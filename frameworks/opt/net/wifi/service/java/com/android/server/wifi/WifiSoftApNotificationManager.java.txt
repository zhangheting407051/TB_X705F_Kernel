/*
 * Copyright (C) 2010 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.android.server.wifi;

import android.app.Notification;
import android.app.NotificationManager;
import android.content.Context;
import android.content.Intent;
import android.net.ConnectivityManager;
import android.os.RemoteException;
import android.os.UserHandle;
import android.util.Log;
import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.File;
import java.io.FileDescriptor;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.HashMap;
import android.net.ConnectivityManager.NetworkCallback;

import android.app.NotificationChannel;
import android.content.res.Resources;
import android.app.PendingIntent;
import android.net.wifi.WifiDevice;
import com.android.internal.R;
import android.os.Message;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

public class WifiSoftApNotificationManager  {
    private static Context mContext;
    private static final String TAG = "WifiSoftApNotificationManager";
    private final static boolean DBG = false;
    private static WifiSoftApNotificationManager mWifiSoftApNotificationManager;

    private Notification.Builder softApNotificationBuilder;
    private int mLastSoftApNotificationId = 0;
    // Once STA established connection to hostapd, it will be added
    // to mL2ConnectedDeviceMap. Then after deviceinfo update from dnsmasq,
    // it will be added to mConnectedDeviceMap
    private HashMap<String, WifiDevice> mL2ConnectedDeviceMap = new HashMap<String, WifiDevice>();
    private HashMap<String, WifiDevice> mConnectedDeviceMap = new HashMap<String, WifiDevice>();
    private static final String dhcpLocation = "/data/misc/dhcp/dnsmasq.leases";

    //Notification channel,
    private static String HOTSPOT_NOTIFICATION = "HOTSPOT_NOTIFICATION";
    private String mChannelName;

    // Device name polling interval(ms) and max times
    private static final int DNSMASQ_POLLING_INTERVAL = 1000;
    private static final int DNSMASQ_POLLING_MAX_TIMES = 10;

    private WifiSoftApNotificationManager (Context context) {
        mContext = context;
    }

    public static WifiSoftApNotificationManager getInstance(Context mContext)
    {
        if(mWifiSoftApNotificationManager == null )
          return mWifiSoftApNotificationManager = new WifiSoftApNotificationManager(mContext);
        else
          return mWifiSoftApNotificationManager;
    }

    private ConnectivityManager getConnectivityManager() {
        return (ConnectivityManager) mContext.getSystemService(Context.CONNECTIVITY_SERVICE);
    }

    private void sendConnectDevicesStateChangedBroadcast() {
        if (!getConnectivityManager().isTetheringSupported()) return;

        Intent broadcast = new Intent(ConnectivityManager.TETHER_CONNECT_STATE_CHANGED);
        broadcast.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING |
                Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);

        mContext.sendStickyBroadcastAsUser(broadcast, UserHandle.ALL);
        showSoftApClientsNotification(com.android.internal.R.drawable.stat_sys_tether_wifi);
    }

    private boolean readDeviceInfoFromDnsmasq(WifiDevice device) {
        boolean result = false;
        FileInputStream fstream = null;
        String line;

        try {
            fstream = new FileInputStream(dhcpLocation);
            if (DBG) Log.e(TAG, "dhcpLocation path location" + dhcpLocation );
            DataInputStream in = new DataInputStream(fstream);
            BufferedReader br = new BufferedReader(new InputStreamReader(in));
            while ((null != (line = br.readLine())) && (line.length() != 0)) {
                String[] fields = line.split(" ");

                if (DBG) Log.e(TAG, "lease file data" + line );
                // 1499268032 64:bc:0c:7d:bb:52 192.168.43.169 android-1d166408c556f704 01:64:bc:0c:7d:bb:52
                if (fields.length > 3) {
                    String addr = fields[1];
                    String name = fields[3];
                    if (addr.equals(device.deviceAddress)) {
                        if (DBG) Log.d(TAG, "Successfully poll device info for " +device.deviceAddress);
                        device.deviceName = name;
                        result = true;
                        break;
                    }
                }
            }
        } catch (IOException ex) {
            Log.e(TAG, "readDeviceNameFromDnsmasq: " + ex);
        } finally {
            if (fstream != null) {
                try {
                    fstream.close();
                } catch (IOException ex) {}
            }
        }

        return result;
    }

    private static class DnsmasqThread extends Thread {
        private final WifiSoftApNotificationManager  mWifiSoftApNotificationmgr;
        private int mInterval;
        private int mMaxTimes;
        private WifiDevice mDevice;

        public DnsmasqThread(WifiSoftApNotificationManager  softap,  WifiDevice device,
                             int interval, int maxTimes) {
            super("SoftAp");
            mWifiSoftApNotificationmgr = softap;
            mInterval = interval;
            mMaxTimes = maxTimes;
            mDevice = device;
        }

        public void run() {
            boolean result = false;
            try {
                while (mMaxTimes > 0) {
                    result = mWifiSoftApNotificationmgr.readDeviceInfoFromDnsmasq(mDevice);
                    if (DBG) Log.d(TAG, "Thread Running");
                    if (result) {
                        if (DBG) Log.d(TAG, "Successfully poll device info for " + mDevice.deviceAddress);
                        break;
                    }

                    mMaxTimes --;
                    Thread.sleep(mInterval);
                }
            } catch (Exception ex) {
                result = false;
                Log.e(TAG, "Polling " + mDevice.deviceAddress +  "error" + ex);
            }

            if (!result) {
                if (DBG) Log.d(TAG, "Polling timeout, suppose STA uses static ip " + mDevice.deviceAddress);
            }

            // When STA uses static ip, device info will be unavaiable from dnsmasq,
            // thus no matter the result is success or failure, we will broadcast the event.
            // But if the device is not in L2 connected state, it means the hostapd connection is
            // disconnected before dnsmasq get device info, so in this case, don't broadcast
            // connection event.
               WifiDevice other = mWifiSoftApNotificationmgr.mL2ConnectedDeviceMap.get(mDevice.deviceAddress);
               if (other != null && other.deviceState == WifiDevice.CONNECTED) {
                mWifiSoftApNotificationmgr.mConnectedDeviceMap.put(mDevice.deviceAddress, mDevice);
                mWifiSoftApNotificationmgr.sendConnectDevicesStateChangedBroadcast();
            } else {
                if (DBG) Log.d(TAG, "Device " + mDevice.deviceAddress + "already disconnected, ignoring");
            }
        }

    }

    private void interfaceMessageRecevied(String message , boolean isConnected) {
        // if softap extension feature not enabled, do nothing
        if (!mContext.getResources().getBoolean(com.android.internal.R.bool.config_softap_extension)) {
            return;
        }

        if (DBG) Log.d(TAG, "interfaceMessageRecevied: message=" + message);
            WifiDevice device = new WifiDevice(message, isConnected);

        if (device.deviceState == WifiDevice.CONNECTED) {
            mL2ConnectedDeviceMap.put(device.deviceAddress, device);
            if (DBG) Log.d(TAG, "device: connected");

            // When hostapd reported STA-connection event, it is possible that device
            // info can't fetched from dnsmasq, then we start a thread to poll the
            // device info, the thread will exit after device info avaiable.
            // For static ip case, dnsmasq don't hold the device info, thus thread
            // will exit after a timeout.
            if (readDeviceInfoFromDnsmasq(device)) {
                if (DBG) Log.d(TAG, "readDeviceInfoFromDnsmasq");
                mConnectedDeviceMap.put(device.deviceAddress, device);
                sendConnectDevicesStateChangedBroadcast();
            } else {
                if (DBG) Log.d(TAG, "Starting poll device info for " + device.deviceAddress);
                new DnsmasqThread(this, device,
                    DNSMASQ_POLLING_INTERVAL, DNSMASQ_POLLING_MAX_TIMES).start();
            }
        } else if (device.deviceState == WifiDevice.DISCONNECTED) {
            if (DBG) Log.d(TAG, "device: disconnected");
            mConnectedDeviceMap.remove(device.deviceAddress);
            if (mL2ConnectedDeviceMap.remove(device.deviceAddress) != null) {
                sendConnectDevicesStateChangedBroadcast();
            }
        }

    }


    private void showSoftApClientsNotification(int icon) {
        NotificationManager notificationManager =
                (NotificationManager)mContext.getSystemService(Context.NOTIFICATION_SERVICE);

        if (notificationManager == null) {
            return;
        }

        Intent intent = new Intent();
        intent.setClassName("com.android.settings", "com.android.settings.TetherSettings");
        intent.setFlags(Intent.FLAG_ACTIVITY_NO_HISTORY);

        PendingIntent pi = PendingIntent.getActivityAsUser(mContext, 0, intent, 0,
                null, UserHandle.CURRENT);

        CharSequence message;
        Resources r = Resources.getSystem();
        CharSequence title = r.getText(com.android.internal.R.string.tethered_notification_title);
        mChannelName = r.getText(com.android.internal.R.string.notification_channel_hotspot).toString();
        int size = mConnectedDeviceMap.size();
        if (size == 0) {
            message = r.getText(com.android.internal.R.string.tethered_notification_no_device_message);
        } else if (size == 1) {
            message = String.format((r.getText(com.android.internal.R.string.tethered_notification_one_device_message)).toString(),
                    size);
        } else {
            message = String.format((r.getText(com.android.internal.R.string.tethered_notification_multi_device_message)).toString(),
                    size);
        }
        if (softApNotificationBuilder == null) {

            NotificationChannel channel = new NotificationChannel(HOTSPOT_NOTIFICATION, mChannelName, NotificationManager.IMPORTANCE_MIN);
            notificationManager.createNotificationChannel(channel);
            softApNotificationBuilder = new Notification.Builder(mContext,HOTSPOT_NOTIFICATION);
            softApNotificationBuilder.setWhen(0)
                    .setOngoing(true)
                    .setColor(mContext.getColor(
                            com.android.internal.R.color.system_notification_accent_color))
                    .setVisibility(Notification.VISIBILITY_PUBLIC)
                    .setCategory(Notification.CATEGORY_STATUS);
        }
        softApNotificationBuilder.setSmallIcon(icon)
                .setContentTitle(title)
                .setContentText(message)
                .setContentIntent(pi)
                .setPriority(Notification.PRIORITY_MIN);
        softApNotificationBuilder.setContentText(message);

        mLastSoftApNotificationId = icon + 10;
        notificationManager.notify(mChannelName, mLastSoftApNotificationId, softApNotificationBuilder.build());
    }

    public void clearSoftApClientsNotification() {
        NotificationManager notificationManager =
                (NotificationManager)mContext.getSystemService(Context.NOTIFICATION_SERVICE);
        if (notificationManager != null && mLastSoftApNotificationId != 0) {
            notificationManager.cancel(mChannelName, mLastSoftApNotificationId);
            mLastSoftApNotificationId = 0;
        }
    }

    public void connectionStatusChange(Message message){
        boolean isConnected = message.arg1 == 1;
        Log.d(TAG, "devices status="+isConnected);
        interfaceMessageRecevied((String) message.obj,isConnected);
    }
    public void clearConnectedDevice() {
        mConnectedDeviceMap.clear();
        mL2ConnectedDeviceMap.clear();
    }

    public List<WifiDevice> getConnectedStations() {
        Iterator it;
        List<WifiDevice> getConnectedStationsList = new ArrayList<WifiDevice>();

        if (mContext.getResources().getBoolean(com.android.internal.R.bool.config_softap_extension)) {
            it = mConnectedDeviceMap.keySet().iterator();
            while(it.hasNext()) {
                String key = (String)it.next();
                WifiDevice device = (WifiDevice)mConnectedDeviceMap.get(key);
                if (DBG) {
                    Log.d(TAG, "getTetherConnectedSta: addr=" + key + " name=" + device.deviceName+" address=" + device.deviceAddress);
                }
                getConnectedStationsList.add(device);
            }
        }
        return getConnectedStationsList;
    }

}
