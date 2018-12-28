/*
 * Copyright (c) 2016, The Linux Foundation. All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions are
 * met:
 *  * Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 *  * Redistributions in binary form must reproduce the above
 *    copyright notice, this list of conditions and the following
 *    disclaimer in the documentation and/or other materials provided
 *    with the distribution.
 *  * Neither the name of The Linux Foundation nor the names of its
 *    contributors may be used to endorse or promote products derived
 *    from this software without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED "AS IS" AND ANY EXPRESS OR IMPLIED
 * WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NON-INFRINGEMENT
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS
 * BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR
 * BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE
 * OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN
 * IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 */
package com.android.camera.util;

import android.os.SystemProperties;

public class PersistUtil {

    public static final int CAMERA2_DEBUG_DUMP_IMAGE = 1;
    public static final int CAMERA2_DEBUG_DUMP_LOG = 2;
    public static final int CAMERA2_DEBUG_DUMP_ALL = 100;

    private static final int CAMERA_SENSOR_HORIZONTAL_ALIGNED = 0;
    private static final int CAMERA_SENSOR_VERTICAL_ALIGNED = 1;

    private static final int PERSIST_MEMORY_LIMIT =
            SystemProperties.getInt("persist.vendor.camera.perf.memlimit", 60);
    private static final boolean PERSIST_SKIP_MEMORY_CHECK =
            SystemProperties.getBoolean("persist.vendor.camera.perf.skip_memck", false);
    private static final int PERSIST_LONGSHOT_SHOT_LIMIT =
            SystemProperties.getInt("persist.vendor.camera.longshot.shotnum", 50);
    private static final String PERSIST_CAMERA_PREVIEW_SIZE =
            SystemProperties.get("persist.vendor.camera.preview.size", "");
    private static final boolean PERSIST_CAMERA_CAMERA2 =
            SystemProperties.getBoolean("persist.vendor.camera.camera2", false);
    private static final boolean PERSIST_CAMERA_ZSL =
            SystemProperties.getBoolean("persist.vendor.camera.zsl.disabled", false);
    private static final int PERSIST_CAMERA2_DEBUG =
            SystemProperties.getInt("persist.vendor.camera2.debug", 0);
    private static final int PERSIST_CAMERA_CANCEL_TOUCHFOCUS_DELAY =
            SystemProperties.getInt("persist.vendor.camera.focus_delay", 5000);
    private static final int PERSIST_CAMERA_DEBUG =
            SystemProperties.getInt("persist.vendor.camera.debug", 0);
    private static final String PERSIST_CAMERA_STILLMORE_BRCOLR =
            SystemProperties.get("persist.vendor.camera.stm_brcolor", "0.5");
    private static final String PERSIST_CAMERA_STILLMORE_BRINTENSITY =
            SystemProperties.get("persist.vendor.camera.stm_brintensity", "0.6");
    private static final String PERSIST_CAMERA_STILLMORE_SMOOTHINGINTENSITY =
            SystemProperties.get("persist.vendor.camera.stm_smooth", "0");
    private static final int PERSIST_CAMERA_STILLMORE_NUM_REQUIRED_IMAGE =
            SystemProperties.getInt("persist.vendor.camera.stm_img_nums", 5);
    private static final String PERSIST_CAMERA_CS_BRINTENSITY_KEY =
            SystemProperties.get("persist.vendor.camera.sensor.brinten", "0.0");
    private static final String PERSIST_CAMERA_CS_SMOOTH_KEY =
            SystemProperties.get("persist.vendor.camera.sensor.smooth", "0.5");
    private static final int PERSIST_CAMERA_SENSOR_ALIGN_KEY =
            SystemProperties.getInt("persist.vendor.camera.sensor.align",
                    CAMERA_SENSOR_HORIZONTAL_ALIGNED);
    private static final int CIRCULAR_BUFFER_SIZE_PERSIST =
            SystemProperties.getInt("persist.vendor.camera.zsl.buffer.size", 5);
    private static final int SAVE_TASK_MEMORY_LIMIT_IN_MB =
            SystemProperties.getInt("persist.vendor.camera.perf.memlimit", 60);
    private static final boolean PERSIST_CAMERA_UI_AUTO_TEST_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.ui.auto_test", false);
    private static final boolean PERSIST_CAMERA_SAVE_IN_SD_ENABLED =
            SystemProperties.getBoolean("persist.vendor.env.camera.saveinsd", false);
    private static final boolean PERSIST_LONG_SAVE_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.longshot.save", false);
    private static final boolean PERSIST_CAMERA_PREVIEW_RESTART_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.feature.restart", false);
    private static final boolean PERSIST_CAPTURE_ANIMATION_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.capture.animate", true);
    private static final boolean PERSIST_SKIP_MEM_CHECK_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.perf.skip_memck", false);
    private static final boolean PERSIST_ZZHDR_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.zzhdr.enable", false);
    private static final int PERSIST_PREVIEW_SIZE =
            SystemProperties.getInt("persist.vendor.camera.preview.size", 0);
    private static final long PERSIST_TIMESTAMP_LIMIT =
            SystemProperties.getLong("persist.vendor.camera.cs.threshold", 10);
    private static final int PERSIST_BURST_COUNT =
            SystemProperties.getInt("persist.vendor.camera.cs.burstcount", 4);
    private static final boolean PERSIST_DUMP_FRAMES_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.cs.dumpframes", false);
    private static final boolean PERSIST_DUMP_YUV_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.cs.dumpyuv", false);
    private static final int PERSIST_CS_TIMEOUT =
            SystemProperties.getInt("persist.vendor.camera.cs.timeout", 300);
    private static final boolean PERSIST_DUMP_DEPTH_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.cs.dumpdepth", false);
    private static final boolean PERSIST_DISABLE_QCOM_MISC_SETTING =
            SystemProperties.getBoolean("persist.vendor.camera.qcom.misc.disable", false);
    private static final int PREVIEW_FLIP_VALUE =
            SystemProperties.getInt("persist.vendor.debug.camera.preview.flip", 0);
    private static final int PERSIST_VIDEO_FLIP_VALUE =
            SystemProperties.getInt("persist.vendor.debug.camera.video.flip", 0);
    private static final int PERSIST_PICTURE_FLIP_VALUE =
            SystemProperties.getInt("persist.vendor.debug.camera.picture.flip", 0);
    private static final boolean PERSIST_YV_12_FORMAT_ENABLED =
            SystemProperties.getBoolean("persist.vendor.camera.debug.camera.yv12", false);
    private static final String PERSIST_DISPLAY_UMAX =
            SystemProperties.get("persist.vendor.camera.display.umax", "");
    private static final String PERSIST_DISPLAY_LMAX =
            SystemProperties.get("persist.vendor.camera.display.lmax", "");

    public static int getMemoryLimit() {
        return PERSIST_MEMORY_LIMIT;
    }

    public static boolean getSkipMemoryCheck() {
        return PERSIST_SKIP_MEMORY_CHECK;
    }

    public static int getLongshotShotLimit() {
        return PERSIST_LONGSHOT_SHOT_LIMIT;
    }

    public static int getCameraPreviewSize() {
        return SystemProperties.getInt(PERSIST_CAMERA_PREVIEW_SIZE, 0);
    }

    public static boolean isDisableQcomMiscSetting(){
        return PERSIST_DISABLE_QCOM_MISC_SETTING;
    }

    public static int getPreviewFlip() {
        return PREVIEW_FLIP_VALUE;
    }

    public static int getVideoFlip() {
        return PERSIST_VIDEO_FLIP_VALUE;
    }

    public static int getPictureFlip() {
        return PERSIST_PICTURE_FLIP_VALUE;
    }

    public static boolean isYv12FormatEnable() {
        return PERSIST_YV_12_FORMAT_ENABLED;
    }

    public static String getDisplayUMax() {
        return PERSIST_DISPLAY_UMAX;
    }

    public static String getDisplayLMax() {
        return PERSIST_DISPLAY_LMAX;
    }

}
