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

import static android.net.wifi.WifiManager.WIFI_AP_STATE_DISABLED;
import static android.net.wifi.WifiManager.WIFI_AP_STATE_DISABLING;
import static android.net.wifi.WifiManager.WIFI_AP_STATE_ENABLED;
import static android.net.wifi.WifiManager.WIFI_AP_STATE_ENABLING;
import static android.net.wifi.WifiManager.WIFI_AP_STATE_FAILED;
import static android.net.wifi.WifiManager.WIFI_STATE_DISABLED;
import static android.net.wifi.WifiManager.WIFI_STATE_DISABLING;
import static android.net.wifi.WifiManager.WIFI_STATE_ENABLED;
import static android.net.wifi.WifiManager.WIFI_STATE_ENABLING;
import static android.net.wifi.WifiManager.WIFI_STATE_UNKNOWN;
import static android.system.OsConstants.ARPHRD_ETHER;

import android.net.wifi.IApInterface;

import android.content.Context;
import android.content.Intent;
import android.net.ConnectivityManager;
import android.net.wifi.WifiConfiguration;
import android.net.wifi.WifiManager;
import android.net.wifi.WifiInfo;
import android.os.BatteryStats;
import android.os.INetworkManagementService;
import android.os.Message;
import android.os.RemoteException;
import android.os.UserHandle;
import android.util.Log;
import android.util.Pair;

import com.android.internal.R;
import com.android.internal.app.IBatteryStats;
import com.android.internal.util.Protocol;
import com.android.internal.util.State;
import com.android.internal.util.StateMachine;
import com.android.server.wifi.util.ApConfigUtil;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Track the state of Wifi SoftAp connectivity. All event handling is done here,
 * and all changes in connectivity state are initiated here.
 *
 * Without STA + SAP Concurrency:
 * Wi-Fi now supports three modes of operation: Client, SoftAp and p2p
 * In the current implementation, we support concurrent wifi p2p and wifi operation.
 * The WifiStateMachine handles SoftAp and Client operations while WifiP2pService
 * handles p2p operation.
 *
 * With STA + SAP Concurrency:
 * Wi-fi can now additionaly support STA and SAP concurrently. This SoftApStateMachine
 * is leveraged from WifiStateMachine to handle concurrent operations.
 *
 * @hide
 */
public class SoftApStateMachine extends StateMachine {

    private static boolean DBG = false;
    private static final String TAG = "SoftApStateMachine";

    private WifiNative mWifiNative;
    private WifiInjector mWifiInjector;
    private WifiApConfigStore mWifiApConfigStore;
    private WifiConfigManager mWifiConfigManager;
    private INetworkManagementService mNwService;
    private ConnectivityManager mCm;

    private String mInterfaceName = null;
    private int mSoftApChannel = 0;
    private WifiStateTracker mWifiStateTracker;

    private Context mContext;
    /* The base for wifi message types */
    static final int BASE = Protocol.BASE_WIFI;
    /* Start the soft ap  */
    static final int CMD_START_AP                                 = BASE + 21;
    /* Indicates soft ap start failed */
    static final int CMD_START_AP_FAILURE                         = BASE + 22;
    /* Stop the soft ap */
    static final int CMD_STOP_AP                                  = BASE + 23;
    /* Soft access point teardown is completed. */
    static final int CMD_AP_STOPPED                               = BASE + 24;

    public static final int CMD_BOOT_COMPLETED                    = BASE + 134;


    /**
     * One of  {@link WifiManager#WIFI_AP_STATE_DISABLED},
     * {@link WifiManager#WIFI_AP_STATE_DISABLING},
     * {@link WifiManager#WIFI_AP_STATE_ENABLED},
     * {@link WifiManager#WIFI_AP_STATE_ENABLING},
     * {@link WifiManager#WIFI_AP_STATE_FAILED}
     */
    private final AtomicInteger mWifiApState
            = new AtomicInteger(WIFI_AP_STATE_DISABLED);
    private final IBatteryStats mBatteryStats;

    /* Temporary initial state */
    private State mInitialState = new InitialState();
    /* Soft ap state */
    private State mSoftApState = new SoftApState();

    public SoftApStateMachine(Context context, WifiInjector wifiInjector,
                            WifiNative wifiNative,
                            INetworkManagementService NwService,
                            IBatteryStats BatteryStats) {
        super("SoftApStateMachine");

        mContext = context;
        mWifiInjector = wifiInjector;
        mWifiStateTracker = wifiInjector.getWifiStateTracker();
        mWifiConfigManager = wifiInjector.getWifiConfigManager();
        mWifiApConfigStore = wifiInjector.getWifiApConfigStore();
        mWifiNative = wifiNative;
        mNwService = NwService;
        mBatteryStats = BatteryStats;

        addState(mInitialState);
            addState(mSoftApState, mInitialState);
        setInitialState(mInitialState);
        start();

    }

    void enableVerboseLogging(int verbose) {
        if (verbose > 0) {
            DBG = true;
        } else {
            DBG = false;
        }
    }

    /* API to check whether WifiConfig shares its network or not */
    public boolean isExtendingNetworkCoverage() {
        WifiStateMachine mWifiStateMachine = mWifiInjector.getWifiStateMachine();
        WifiConfiguration currentStaConfig = mWifiStateMachine.getCurrentWifiConfiguration();
        return (currentStaConfig != null && currentStaConfig.shareThisAp);
    }

    public void setSoftApInterfaceName(String iface) {
        mInterfaceName = iface;
    }

    public void setSoftApChannel(int channel) {
        mSoftApChannel = channel;
    }


   /*  Leverage from WiFiStateMachine */
    public void setHostApRunning(SoftApModeConfiguration softApConfig, boolean enable) {
        if (enable) {
            sendMessage(CMD_START_AP, softApConfig);
        } else {
            sendMessage(CMD_STOP_AP);
        }
    }

   /*  Leverage from WiFiStateMachine */
    public int syncGetWifiApState() {
        return mWifiApState.get();
    }

   /*  Leverage from WiFiStateMachine */
    public String syncGetWifiApStateByName() {
        switch (mWifiApState.get()) {
            case WIFI_AP_STATE_DISABLING:
                return "disabling";
            case WIFI_AP_STATE_DISABLED:
                return "disabled";
            case WIFI_AP_STATE_ENABLING:
                return "enabling";
            case WIFI_AP_STATE_ENABLED:
                return "enabled";
            case WIFI_AP_STATE_FAILED:
                return "failed";
            default:
                return "[invalid state]";
        }
    }

   /*  Leverage from WiFiStateMachine */
    private void setWifiApState(int wifiApState, int reason, String ifaceName, int mode) {
        final int previousWifiApState = mWifiApState.get();

        try {
            if (wifiApState == WIFI_AP_STATE_ENABLED) {
                mBatteryStats.noteWifiOn();
            } else if (wifiApState == WIFI_AP_STATE_DISABLED) {
                mBatteryStats.noteWifiOff();
            }
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to note battery stats in wifi");
        }

        if ((wifiApState == WIFI_AP_STATE_DISABLED)
               || (wifiApState == WIFI_AP_STATE_FAILED)) {
            boolean skipUnload = false;
            WifiStateMachine mWifiStateMachine = mWifiInjector.getWifiStateMachine();
            int wifiState = mWifiStateMachine.syncGetWifiState();
            int operMode = mWifiStateMachine.getOperationalMode();
            if ((wifiState ==  WifiManager.WIFI_STATE_ENABLING) ||
                    (wifiState == WifiManager.WIFI_STATE_ENABLED) ||
                     (operMode == WifiStateMachine.SCAN_ONLY_WITH_WIFI_OFF_MODE)) {
                Log.d(TAG, "Avoid unload driver, WIFI_STATE is enabled/enabling");
                skipUnload = true;
            }
            if (!skipUnload) {
                mWifiStateMachine.cleanup();
            } else {
                mWifiNative.tearDownAp();
            }
        }
        // Update state
        mWifiApState.set(wifiApState);

        if (DBG) Log.d(TAG,"setWifiApState: " + syncGetWifiApStateByName());

        final Intent intent = new Intent(WifiManager.WIFI_AP_STATE_CHANGED_ACTION);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
        intent.putExtra(WifiManager.EXTRA_WIFI_AP_STATE, wifiApState);
        intent.putExtra(WifiManager.EXTRA_PREVIOUS_WIFI_AP_STATE, previousWifiApState);
        if (wifiApState == WifiManager.WIFI_AP_STATE_FAILED) {
            //only set reason number when softAP start failed
            intent.putExtra(WifiManager.EXTRA_WIFI_AP_FAILURE_REASON, reason);
        }

        if (ifaceName == null) {
            Log.e(TAG, "Updating wifiApState with a null iface name");
        }
        intent.putExtra(WifiManager.EXTRA_WIFI_AP_INTERFACE_NAME, ifaceName);
        intent.putExtra(WifiManager.EXTRA_WIFI_AP_MODE, mode);

        mContext.sendStickyBroadcastAsUser(intent, UserHandle.ALL);
    }

   /*  Leverage from WiFiStateMachine */
    private void checkAndSetConnectivityInstance() {
        if (mCm == null) {
            mCm = (ConnectivityManager) mContext.getSystemService(Context.CONNECTIVITY_SERVICE);
        }
    }

     /*
      *  SoftApStateMAchine states
      *      (InitialState)
      *           |
      *           |
      *           V
      *      (SoftApState)
      *
      * InitialState : By default control sits in this state after
      *                SoftApStateMachine is getting initialized.
      *                It unload wlan driver if it is loaded.
      *                On request turn on softap, it movies to
      *                SoftApState.
      * SoftApState  : Once driver and firmware successfully loaded
      *                it control sits in this state.
      *                On request to stop softap, it movies back to
      *                InitialState.
      *
      */

   /*  Leverage from WiFiStateMachine */
    class InitialState extends State {
        @Override
        public boolean processMessage(Message message) {
            switch (message.what) {
                case CMD_START_AP:
                    transitionTo(mSoftApState);
                    break;
                default:
                    return NOT_HANDLED;
            }
            return HANDLED;
        }
    }

   /*  Leverage from WiFiStateMachine */
    class SoftApState extends State {
        private SoftApManager mSoftApManager;
        private String mIfaceName;
        private int mMode;

        private class SoftApListener implements SoftApManager.Listener {
            @Override
            public void onStateChanged(int state, int reason) {
                if (state == WIFI_AP_STATE_DISABLED) {
                    mWifiNative.addOrRemoveInterface(mInterfaceName, false, false);
                    sendMessage(CMD_AP_STOPPED);
                } else if (state == WIFI_AP_STATE_FAILED) {
                    mWifiNative.addOrRemoveInterface(mInterfaceName, false, false);
                    sendMessage(CMD_START_AP_FAILURE);
                }

                setWifiApState(state, reason, mIfaceName, mMode);
            }
        }

        @Override
        public void enter() {
            final Message message = getCurrentMessage();
            if (message.what != CMD_START_AP) {
                throw new RuntimeException("Illegal transition to SoftApState: " + message);
            }
            SoftApModeConfiguration config = (SoftApModeConfiguration) message.obj;
            mMode = config.getTargetMode();

            // If required, new interface will added by setupForSoftApMode
            IApInterface apInterface = null;
            Pair<Integer, IApInterface> statusAndInterface = mWifiNative.setupForSoftApMode(mInterfaceName, false);
            if (statusAndInterface.first == WifiNative.SETUP_SUCCESS) {
                apInterface = statusAndInterface.second;
            } else {
                Log.e(TAG, "Error in wifi onFailure due to HAL");
            }

            if (apInterface == null) {
                mWifiNative.addOrRemoveInterface(mInterfaceName, false, false);
                setWifiApState(WIFI_AP_STATE_FAILED,
                        WifiManager.SAP_START_FAILURE_GENERAL, null, mMode);
                /**
                 * Transition to InitialState to reset thE
                 * driver/HAL back to the initial state.
                 */
                transitionTo(mInitialState);
                return;
            }

            try {
                mIfaceName = apInterface.getInterfaceName();
            } catch (RemoteException e) {
                // Failed to get the interface name. The name will not be available for
                // the enabled broadcast, but since we had an error getting the name, we most likely
                // won't be able to fully start softap mode.
            }


            WifiStateMachine mWifiStateMachine = mWifiInjector.getWifiStateMachine();
            WifiConfiguration currentStaConfig = mWifiStateMachine.getCurrentWifiConfiguration();
            if (currentStaConfig != null && currentStaConfig.shareThisAp) {
                config = new SoftApModeConfiguration(mMode, currentStaConfig);
                currentStaConfig = mWifiConfigManager.getConfiguredNetworkWithPassword(currentStaConfig.networkId);
                config.mConfig.SSID = ApConfigUtil.removeDoubleQuotes(currentStaConfig.SSID);
                config.mConfig.apBand = currentStaConfig.apBand;
                config.mConfig.apChannel = currentStaConfig.apChannel;
                config.mConfig.hiddenSSID = currentStaConfig.hiddenSSID;
                config.mConfig.preSharedKey = ApConfigUtil.removeDoubleQuotes(currentStaConfig.preSharedKey);
                config.mConfig.allowedKeyManagement = currentStaConfig.allowedKeyManagement;
                mSoftApChannel = currentStaConfig.apChannel;
                WifiInfo mWifiInfo = mWifiStateMachine.getWifiInfo();
                if (mWifiInfo != null && config.mConfig.apChannel == 0) {
                    config.mConfig.apChannel = ApConfigUtil.convertFrequencyToChannel(mWifiInfo.getFrequency());
                    mSoftApChannel = config.mConfig.apChannel;
                }
           } else if (currentStaConfig != null && currentStaConfig.shareThisAp == false) {
                mSoftApChannel = 0;
           }

            checkAndSetConnectivityInstance();
            mSoftApManager = mWifiInjector.makeSoftApManager(mNwService,
                                                             new SoftApListener(),
                                                             apInterface,
                                                             config.getWifiConfiguration());
            if (mSoftApChannel != 0) {
                mSoftApManager.setSapChannel(mSoftApChannel);
            }
            mSoftApManager.start();
            mWifiStateTracker.updateState(WifiStateTracker.SOFT_AP);
        }

        @Override
        public void exit() {
            mSoftApManager = null;
        }

        @Override
        public boolean processMessage(Message message) {

            switch(message.what) {
                case CMD_START_AP:
                    /* Ignore start command when it is starting/started. */
                    break;
                case CMD_STOP_AP:
                    mSoftApManager.stop();
                    break;
                case CMD_START_AP_FAILURE:
                    transitionTo(mInitialState);
                    break;
                case CMD_AP_STOPPED:
                    transitionTo(mInitialState);
                    break;
                default:
                    return NOT_HANDLED;
            }
            return HANDLED;
        }
    }

    /**
     * arg2 on the source message has a unique id that needs to be retained in replies
     * to match the request
     * <p>see WifiManager for details
     */
    private Message obtainMessageWithWhatAndArg2(Message srcMsg, int what) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.arg2 = srcMsg.arg2;
        return msg;
    }
}
