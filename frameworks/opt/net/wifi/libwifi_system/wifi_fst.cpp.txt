/*
 * Copyright (c) 2015,2017, The Linux Foundation. All rights reserved.
 *
 * Not a Contribution.
 * Copyright 2008, The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/stat.h>

#include <android-base/logging.h>
#define LOG_TAG "WifiFST"
#include <android-base/file.h>
#include <android-base/stringprintf.h>
#include "cutils/properties.h"

#define _REALLY_INCLUDE_SYS__SYSTEM_PROPERTIES_H_
#include <sys/_system_properties.h>
#include <private/android_filesystem_config.h>

using android::base::StringPrintf;
using android::base::WriteStringToFile;
using std::string;

static const char FSTMAN_IFNAME[] = "wlan0";
static const char FSTMAN_WIGIG_IFNAME[] = "wigig0";
static const char FSTMAN_DATA_IFNAME[] = "bond0";
static const char FSTMAN_NAME[] = "fstman";
const char FSTMAN_CONFIG_FILE[] = "/data/vendor/wifi/fstman.ini";
static const char FSTMAN_WIGIG_IF_CHANNEL[] = "2";
static const char FST_RATE_UPGRADE_ENABLED_PROP_NAME[] = "persist.vendor.fst.rate.upgrade.en";
static const char FST_SOFTAP_ENABLED_PROP_NAME[] = "persist.vendor.fst.softap.en";
static const char FST_STA_INTERFACE_PROP_NAME[] = "persist.vendor.fst.wifi.sta.interface";
static const char FST_SAP_INTERFACE_PROP_NAME[] = "persist.vendor.fst.wifi.sap.interface";
static const char FST_DATA_INTERFACE_PROP_NAME[] = "persist.vendor.fst.data.interface";
static const char FST_WIGIG_INTERFACE_PROP_NAME[] = "persist.vendor.fst.wigig.interface";
static const char FST_WIGIG_INTERFACE_CHANNEL_PROP_NAME[] =
    "persist.vendor.fst.wigig.interface.channel";

namespace android {
namespace wifi_system {

static string create_fstman_ini_file(int softap_mode) {
    char fst_iface1[PROPERTY_VALUE_MAX] = { '\0' };
    char fst_iface2[PROPERTY_VALUE_MAX] = { '\0' };
    char fst_data_iface[PROPERTY_VALUE_MAX] = { '\0' };
    char fst_wigig_channel[PROPERTY_VALUE_MAX] = { '\0' };
    string result;

    if (softap_mode)
        property_get(FST_SAP_INTERFACE_PROP_NAME, fst_iface1, FSTMAN_IFNAME);
    else
        property_get(FST_STA_INTERFACE_PROP_NAME, fst_iface1, FSTMAN_IFNAME);
    property_get(FST_WIGIG_INTERFACE_PROP_NAME, fst_iface2, FSTMAN_WIGIG_IFNAME);
    property_get(FST_DATA_INTERFACE_PROP_NAME, fst_data_iface, FSTMAN_DATA_IFNAME);
    property_get(FST_WIGIG_INTERFACE_CHANNEL_PROP_NAME, fst_wigig_channel,
                 FSTMAN_WIGIG_IF_CHANNEL);

    result = StringPrintf(
        "[fst_manager]\n"
        "ctrl_iface=/data/vendor/wifi/hostapd/global\n"
        "groups=%s\n" /* bond0 */
        "\n"
        "[%s]\n" /* bond0 */
        "interfaces=%s,%s\n" /* wlan0,wigig0 */
        "mux_type=bonding\n"
        "mux_ifname=%s\n" /* bond0 */
        "mux_managed=1\n"
        "mac_address_by=%s\n" /* wlan0 */
        "rate_upgrade_master=%s\n" /* wlan0 */
        "txqueuelen=100\n"
        "rate_upgrade_acl_file=/data/vendor/wifi/fst_rate_upgrade.accept\n"
        "\n"
        "[%s]\n" /* wlan0 */
        "priority=100\n"
        "default_llt=3600\n"
        "\n"
        "[%s]\n" /* wigig0 */
        "priority=110\n"
        "wpa_group=GCMP\n"
        "wpa_pairwise=GCMP\n"
        "hw_mode=ad\n"
        "channel=%s\n",
        fst_data_iface,
        fst_data_iface,
        fst_iface1, fst_iface2,
        fst_data_iface,
        fst_iface1,
        fst_iface1,
        fst_iface1,
        fst_iface2,
        fst_wigig_channel
        );

    return result;
}

static bool write_fstman_ini_file(const char *fname, const string& contents) {
    if (!WriteStringToFile(contents, fname, 0660, AID_WIFI, AID_WIFI)) {
        int error = errno;
        LOG(ERROR) << "Cannot write fstman.ini to \""
                   << fname << "\": " << strerror(error);
        struct stat st;
        int result = stat(fname, &st);
        if (result == 0) {
            LOG(ERROR) << "fstman.ini uid: "<< st.st_uid << ", gid: " << st.st_gid
                       << ", mode: " << st.st_mode;
        } else {
            LOG(ERROR) << "Error calling stat() on fstman.ini: " << strerror(errno);
        }
        return false;
    }
    return true;
}

int is_fst_enabled()
{
    char prop_value[PROPERTY_VALUE_MAX] = { '\0' };

    if (property_get(FST_RATE_UPGRADE_ENABLED_PROP_NAME, prop_value, NULL) &&
        strcmp(prop_value, "1") == 0) {
        return 1;
    }

    return 0;
}

int is_fst_softap_enabled() {
    char prop_value[PROPERTY_VALUE_MAX] = { '\0' };

    if (is_fst_enabled() &&
        property_get(FST_SOFTAP_ENABLED_PROP_NAME, prop_value, NULL) &&
        strcmp(prop_value, "1") == 0) {
        return 1;
    }

    return 0;
}

static void get_fstman_props(int softap_mode,
			     char *fstman_svc_name, int fstman_svc_name_len,
			     char *fstman_init_prop, int fstman_init_prop_len)
{
    if (softap_mode)
        strlcpy(fstman_svc_name, FSTMAN_NAME, fstman_svc_name_len);
    else
        snprintf(fstman_svc_name, fstman_svc_name_len, "%s_%s",
                 FSTMAN_NAME, FSTMAN_IFNAME);
    snprintf(fstman_init_prop, fstman_init_prop_len, "init.svc.%s",
             fstman_svc_name);
}

int wifi_start_fstman(int softap_mode)
{
    char fstman_status[PROPERTY_VALUE_MAX] = { '\0' };
    char fstman_svc_name[PROPERTY_VALUE_MAX] = { '\0' };
    char fstman_init_prop[PROPERTY_VALUE_MAX] = { '\0' };
    int count = 50; /* wait at most 5 seconds for completion */
    string config;

    if (!is_fst_enabled() ||
        (softap_mode && !is_fst_softap_enabled())) {
        return 0;
    }

    config = create_fstman_ini_file(softap_mode);
    if (config.empty())
        return -1;
    if (!write_fstman_ini_file(FSTMAN_CONFIG_FILE, config))
        return -1;

    get_fstman_props(softap_mode, fstman_svc_name, sizeof(fstman_svc_name),
                     fstman_init_prop, sizeof(fstman_init_prop));

    /* Check whether already running */
    if (property_get(fstman_init_prop, fstman_status, NULL) &&
        strcmp(fstman_status, "running") == 0)
        return 0;

    LOG(DEBUG) << "Starting FST Manager";
    property_set("ctl.start", fstman_svc_name);
    sched_yield();

    while (count-- > 0) {
        if (property_get(fstman_init_prop, fstman_status, NULL) &&
            strcmp(fstman_status, "running") == 0)
                return 0;
        usleep(100000);
    }

    LOG(ERROR) << "Failed to start FST Manager";
    return -1;
}

int wifi_stop_fstman(int softap_mode)
{
    char fstman_status[PROPERTY_VALUE_MAX] = { '\0' };
    char fstman_svc_name[PROPERTY_VALUE_MAX] = { '\0' };
    char fstman_init_prop[PROPERTY_VALUE_MAX] = { '\0' };
    int count = 50; /* wait at most 5 seconds for completion */

    if (!is_fst_enabled() ||
        (softap_mode && !is_fst_softap_enabled())) {
        return 0;
    }

    get_fstman_props(softap_mode, fstman_svc_name, sizeof(fstman_svc_name),
                     fstman_init_prop, sizeof(fstman_init_prop));

    /* Check whether already stopped */
    if (property_get(fstman_init_prop, fstman_status, NULL) &&
        strcmp(fstman_status, "stopped") == 0)
        return 0;

    LOG(DEBUG) << "Stopping FST Manager";
    property_set("ctl.stop", fstman_svc_name);
    sched_yield();

    while (count-- > 0) {
        if (property_get(fstman_init_prop, fstman_status, NULL) &&
            strcmp(fstman_status, "stopped") == 0)
                return 0;
        usleep(100000);
    }

    LOG(ERROR) << "Failed to stop fstman";
    return -1;
}

}  // namespace wifi_system
}  // namespace android
