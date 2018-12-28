/*
 * Copyright (c) 2017, The Linux Foundation. All rights reserved.
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
package com.android.camera;

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.ImageFormat;
import android.graphics.Matrix;
import android.graphics.Paint;
import android.graphics.Point;
import android.graphics.Rect;
import android.graphics.RectF;
import android.graphics.YuvImage;
import android.location.Location;
import android.os.Handler;
import android.os.HandlerThread;
import android.os.Looper;
import android.os.Message;
import android.renderscript.Allocation;
import android.renderscript.Element;
import android.renderscript.RenderScript;
import android.renderscript.ScriptIntrinsicYuvToRGB;
import android.renderscript.Type;
import android.util.Log;

import com.android.camera.exif.ExifInterface;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.FileWriter;
import java.io.IOException;
import java.io.Writer;
import java.nio.ByteBuffer;
import java.util.HashMap;
import java.util.Iterator;
import java.util.TimeZone;

import com.android.camera.PhotoModule.NamedImages.NamedEntity;
import com.android.camera.imageprocessor.filter.ImageFilter;
import com.android.camera.util.CameraUtil;

import org.codeaurora.snapcam.filter.ClearSightNativeEngine;
import org.codeaurora.snapcam.filter.DDMNativeEngine;
import org.codeaurora.snapcam.filter.DualCameraEffect;
import org.codeaurora.snapcam.filter.GDepth;
import org.codeaurora.snapcam.filter.GImage;
import android.os.SystemProperties;

public class SnapshotBokehProcessor {
    private static final String TAG = "SnapshotBokehProcessor";

    private static final String PROPERTY_QUEUE = "persist.vendor.snapcam.bokeh.queue";
    private static final String PROPERTY_BOKEH_DEBUG = "persist.vendor.snapcam.bokeh.debug";

    private static final int MAX_PROCESS_QUEUE = SystemProperties.getInt(PROPERTY_QUEUE,4);
    private static final Boolean DEBUG = SystemProperties.getBoolean(PROPERTY_BOKEH_DEBUG,false);
    public static final int LENGTHRATIO_INDEX = 20;
    public static final int STRIDE_INDEX = 12;
    public static final int SCANLINE_INDEX = 16;

    private Context mContext;
    private HandlerThread mBokehProcessThread;
    private HandlerThread mGdepthProcessThread;
    private BokehProcessHandler mBokehHandler;
    private GdepthProcessHandler mGdepthHandler;
    private PhotoModule mModule;
    private BokehCallback mCallback;
    private HashMap<NamedEntity, ProcessTask> mTask = new HashMap<>(MAX_PROCESS_QUEUE);
    private ClearSightNativeEngine.CamSystemCalibrationData mOtpMetaData;


    public SnapshotBokehProcessor(PhotoModule module,
                                  BokehCallback callback) {
        mModule = module;
        mContext = module.getMainActivity().getApplicationContext();
        mCallback = callback;
    }

    public void setJpegForTask(final long captureTime,final byte[] jpeg) {
        mBokehHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG,"start to set primary jpeg capture time = "+captureTime);
                Iterator<NamedEntity> iterator = mTask.keySet().iterator();
                while (iterator.hasNext()) {
                    NamedEntity tmp = iterator.next();
                    if (tmp.date == captureTime) {
                        BokehProcess process = mTask.get(tmp).getBokehProcess();
                        if (process != null) {
                            process.setPrimaryJpeg(jpeg);
                            Log.d(TAG,"set primary jpeg capture time =" + captureTime);
                        }
                    }
                }
            }
        },800);

    }

    public void stopBackgroundThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    if (mGdepthProcessThread != null) {
                        mGdepthProcessThread.join();
                        mGdepthProcessThread = null;
                    }
                    mGdepthHandler = null;
                    if (mBokehProcessThread != null) {
                        mBokehProcessThread.join();
                        mBokehProcessThread = null;
                    }
                    mBokehHandler = null;
                    mTask.clear();
                } catch (InterruptedException e){
                    e.printStackTrace();
                }
            }
        }).start();
    }

    public void startBackgroundThread() {
        mGdepthProcessThread = new HandlerThread("GdepthThread");
        mGdepthProcessThread.start();
        mGdepthHandler = new GdepthProcessHandler(mGdepthProcessThread.getLooper());

        mBokehProcessThread = new HandlerThread("BokehThread");
        mBokehProcessThread.start();
        mBokehHandler = new BokehProcessHandler(mBokehProcessThread.getLooper());
    }

    public boolean createTask(final ByteBuffer primaryYUV, final ByteBuffer auxiliaryYUV,
                              final byte[] primaryParam,
                              final byte[] auxiliaryParam,
                              final NamedEntity namedEntity,
                              final YuvImageSize primary,
                              final YuvImageSize auxiliary,
                              final Location loc,
                              final int orientation) {
        if (mTask.size() >= (MAX_PROCESS_QUEUE - 1)) {
            mCallback.enableShutterLock(false);
        } else {
            mCallback.enableShutterLock(true);
        }
        ProcessTask task = new ProcessTask(namedEntity);
        GdepthProcess gdepthProcess = new GdepthProcess(namedEntity);
        BokehProcess bokehProcess = new BokehProcess(namedEntity,loc,orientation,primary.getFocus());
        task.setGdepthProcess(gdepthProcess);
        task.setBokehProcess(bokehProcess);
        mTask.put(namedEntity, task);
        mGdepthHandler.post(new Runnable() {
            @Override
            public void run() {
                initGdepthProcess(primaryYUV,auxiliaryYUV,primaryParam,auxiliaryParam,
                        primary,auxiliary,namedEntity);
                mGdepthHandler.obtainMessage(
                        GdepthProcessHandler.GENERATE_GDEPTH,namedEntity).sendToTarget();
            }
        });
        mBokehHandler.post(new Runnable() {
            @Override
            public void run() {
                initeBokehProcess(namedEntity,primaryYUV.array(),primary);
            }
        });
        if (DEBUG) {
            mGdepthHandler.post(new Runnable() {
                @Override
                public void run() {
                    Bitmap pri = getBitmapFromYuv(mContext,primaryYUV.array(), primary);
                    Bitmap aux = getBitmapFromYuv(mContext,auxiliaryYUV.array(), auxiliary);

                    byte[] jpeg1 = compressBitmapToJpeg(pri);
                    byte[] jpeg2 = compressBitmapToJpeg(aux);
                    mModule.getMainActivity().getMediaSaveService().addImage(
                            jpeg1,"Debug_bayer_"+namedEntity.date,
                            System.currentTimeMillis(),null,
                            primary.getWidth(),primary.getHeight(),orientation,null,null,
                            mContext.getContentResolver(),PhotoModule.PIXEL_FORMAT_JPEG);
                    mModule.getMainActivity().getMediaSaveService().addImage(
                            jpeg2,"Debug_mono_"+namedEntity.date,
                            System.currentTimeMillis(),null,
                            auxiliary.getWidth(),auxiliary.getHeight(),orientation,null,null,
                            mContext.getContentResolver(),PhotoModule.PIXEL_FORMAT_JPEG);
                }
            });
        }
        return true;
    }

    private void initGdepthProcess(ByteBuffer primaryYUV, ByteBuffer auxiliaryYUV,
                                              byte[] primaryParam, byte[] auxiliaryParam,
                                              YuvImageSize primary, YuvImageSize auxiliary,
                                              NamedEntity nameEntity) {
        GdepthProcess process = mTask.get(nameEntity).getGdepthProcess();
        if (process != null){
            process.setPrimaryImage(primaryYUV, primary, primaryParam);
            process.setAuxiliaryImage(auxiliaryYUV, auxiliary,auxiliaryParam);
        }
    }

    private void initeBokehProcess(NamedEntity namedEntity,byte[] primaryYuv,YuvImageSize size) {
        BokehProcess process = mTask.get(namedEntity).getBokehProcess();
        if (process != null) {
            process.setPrimary(primaryYuv,size);
        }
    }

    class GdepthProcessHandler extends Handler {
        public static final int GENERATE_GDEPTH = 0;

        GdepthProcessHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            Log.d(TAG,"GdepthProcessHandler handle message =" +msg.what);
            switch (msg.what) {
                case GENERATE_GDEPTH:
                    NamedEntity namedEntity = (NamedEntity)msg.obj;
                    GdepthProcess gdepthProcess = mTask.get(namedEntity).mGdepthProcess;
                    if (gdepthProcess != null && gdepthProcess.isReadyToGenerateGdept()) {
                        if (DEBUG) {
                            try {
                                String path = Storage.DIRECTORY + File.separator +
                                        "Debug_parameters_" + namedEntity.date + ".log";
                                File logFile = new File(path);
                                FileWriter fileWriter = new FileWriter(logFile);
                                fileWriter.write(gdepthProcess.dumpParameter());
                                fileWriter.flush();
                                fileWriter.close();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                        gdepthProcess.generateGDepth();
                    }
                    GDepth depth = gdepthProcess.getGDepth();
                    BokehProcess bokehProcess = mTask.get(namedEntity).getBokehProcess();
                    if (depth != null && bokehProcess != null) {
                        Log.d(TAG,"gdepth is generated");
                        bokehProcess.setDepth(depth);
                        mBokehHandler.obtainMessage(BokehProcessHandler.GENERATE_BOKEH,
                                namedEntity).sendToTarget();
                        depth.encoding();
                        if (DEBUG) {
                            Bitmap depthMap = depth.getGdepthBitmap();
                            byte[] depthJpeg = compressBitmapToJpeg(depthMap);
                            mModule.getMainActivity().getMediaSaveService().addImage(
                                    depthJpeg,"Debug_gdepth_"+namedEntity.date,
                                    System.currentTimeMillis(),null,
                                    depthMap.getWidth(),depthMap.getHeight(),0,null,null,
                                    mContext.getContentResolver(),PhotoModule.PIXEL_FORMAT_JPEG);
                        }
                        mTask.get(namedEntity).getGdepthProcess().release();
                        mTask.get(namedEntity).setGdepthProcess(null);
                    } else {
                        mModule.getMainActivity().runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                mCallback.onBokenFailure(BokehCallback.GDEPTH_FAIL);
                            }
                        });
                        mTask.get(namedEntity).release();
                    }
                    break;
                default:
                    Log.d(TAG,"ImageEncodeHandler unknown message = " + msg.what);
            }
        }
    }

    class BokehProcessHandler extends Handler {
        public static final int GENERATE_BOKEH = 0;
        public static final int SAVE = 1;

        BokehProcessHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            Log.d(TAG,"BokehProcessHandler handle message =" +msg.what);
            switch(msg.what) {
                case GENERATE_BOKEH:
                    NamedEntity namedEntity = (NamedEntity)msg.obj;
                    BokehProcess process = mTask.get(namedEntity).getBokehProcess();
                    boolean success = process.generateBokehBitmap();
                    if (DEBUG) {
                        byte[] depthJpeg = compressBitmapToJpeg(process.getBokeh());
                        mModule.getMainActivity().getMediaSaveService().addImage(
                                depthJpeg,"Debug_bokeh_"+namedEntity.date,
                                System.currentTimeMillis(),null,
                                process.getBokeh().getWidth(),process.getBokeh().getHeight(),
                                process.getOrientation(),null,null,
                                mContext.getContentResolver(),PhotoModule.PIXEL_FORMAT_JPEG);
                    }
                    if (success) {
                        Log.d(TAG,"bokeh image is generated");
                        obtainMessage(SAVE,0,0,process).sendToTarget();
                    } else {
                         mModule.getMainActivity().runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                mCallback.onBokenFailure(BokehCallback.BOKEH_FAIL);
                            }
                         });
                         mTask.get(namedEntity).release();
                    }
                    break;
                case SAVE:
                    BokehProcess bokehProcess = (BokehProcess)msg.obj;
                    if (!bokehProcess.isReadyToSave()) {
                        if (msg.arg1 <= 1) {
                            Message message = obtainMessage(SAVE,msg.arg1+1,0,bokehProcess);
                            mBokehHandler.sendMessageDelayed(message,500);
                            Log.d(TAG,"primary jpeg is not existed try time ="+msg.arg1);
                        } else {
                            if (bokehProcess.getPrimary() != null && msg.arg1 == 2) {
                                Log.d(TAG,"compress primary bitmap to jpeg");
                                bokehProcess.compressPrimaryBitmap();
                                obtainMessage(SAVE,3,0,bokehProcess).sendToTarget();
                            } else {
                                mModule.getMainActivity().runOnUiThread(new Runnable() {
                                    @Override
                                    public void run() {
                                        mCallback.onBokenFailure(BokehCallback.PACK_FAIL);
                                    }
                                });
                                mTask.get(bokehProcess.getNameEntity()).release();
                            }
                        }
                    } else {
                        byte[] bokeh = compressBitmapToJpeg(bokehProcess.getBokeh());
                        bokeh = addExifTags(bokeh,bokehProcess.getOrientation());
                        byte[] primary = bokehProcess.getPrimaryJpeg();
                        GImage gImage = new GImage(primary,"image/jpeg");
                        GDepth depth = bokehProcess.getDepth();
                        Log.d(TAG,"images are compressed, start to save");
                        mModule.getMainActivity().getMediaSaveService().addXmpImage(
                                bokeh, gImage, depth,
                                "bokeh_"+bokehProcess.getNameEntity().title,
                                bokehProcess.getNameEntity().date,
                                bokehProcess.getLocation(),
                                bokehProcess.getBokeh().getWidth(),
                                bokehProcess.getBokeh().getHeight(),
                                bokehProcess.getOrientation(),
                                null,null,mModule.getMainActivity().getContentResolver(),
                                PhotoModule.PIXEL_FORMAT_JPEG);
                        mTask.get(bokehProcess.getNameEntity()).release();
                        mModule.getMainActivity().runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                mCallback.onBokehSuccess();
                            }
                        });
                    }
                    break;
            }
        }

    }

    public class GdepthProcess {
        private DDMNativeEngine mNativeEngine;
        private GDepth mGDepth;
        private NamedEntity mNameEntity;

        public GdepthProcess(NamedEntity namedEntity){
            mNameEntity = namedEntity;
            mNativeEngine = new DDMNativeEngine();
            mNativeEngine.setCamSystemCalibrationData(mOtpMetaData);
        }

        public void setPrimaryImage(ByteBuffer yuv, YuvImageSize size, byte[] parameters) {
            float len = byte2float(parameters, LENGTHRATIO_INDEX);
            mNativeEngine.setBayerLensFocusDistance(len);
            mNativeEngine.setBayerReprocessResult(parameters);
            mNativeEngine.setPrimaryImage(yuv,size);
        }

        public void setAuxiliaryImage(ByteBuffer yuv, YuvImageSize size, byte[] parameters) {
            mNativeEngine.setMonoReprocessResult(parameters);
            mNativeEngine.setAuxiliaryImage(yuv,size);
        }

        public boolean isReadyToGenerateGdept() {
            return  mNativeEngine.isReadyForGenerateDepth();
        }

        public void generateGDepth() {
            GDepth.DepthMap depthMap = null;
            int[] size = new int[2];
            if ( mNativeEngine.getDepthMapSize(size) ) {
                int width = size[0];
                int height = size[1];
                Bitmap bmp = Bitmap.createBitmap(width, height,
                        Bitmap.Config.ALPHA_8);
                int stride = bmp.getRowBytes();
                bmp.recycle();
                byte[] depthBuffer = new byte[stride*height];
                Rect roiRect = new Rect();
                if (mNativeEngine.dualCameraGenerateDDM(depthBuffer, stride, roiRect) ) {
                    depthMap = new GDepth.DepthMap(width, height);
                    depthMap.roi = roiRect;
                    depthMap.buffer = depthBuffer;
                    mGDepth = GDepth.createGDepth(depthMap);
                }else{
                    Log.e(TAG, "dualCameraGenerateDDM failure");
                }
            }else{
                Log.e(TAG, "getDepthMapSize failure");
            }
        }

        public Bitmap getGDepthBitmap() {
            return mGDepth.getGdepthBitmap();
        }

        public NamedEntity getNameEntity() {
            return mNameEntity;
        }

        public String dumpParameter() {
            StringBuilder log = new StringBuilder();
            log.append(mNativeEngine.getBayerScaleCrop());
            log.append(mNativeEngine.getMonoScaleCrop());
            log.append(mNativeEngine.getOTPCalibration());
            return log.toString();
        }

        public GDepth getGDepth() {return mGDepth;}

        public void release() {
            mNativeEngine.release();
            mNativeEngine = null;
            mGDepth = null;
        }
    }

    public class BokehProcess {
        private Bitmap mPrimary;
        private Bitmap mBokeh;
        private byte[] mPriJpeg;
        private GDepth mGDepth;
        private NamedEntity mNameEntity;
        private int mOrientation;
        private Location mLoc;
        private Rect mFocus;

        public BokehProcess(NamedEntity namedEntity, Location loc,int orientation,Rect focus) {
            mNameEntity = namedEntity;
            mLoc = loc;
            mOrientation = orientation;
            mFocus = focus;
        }

        public void setDepth(GDepth depth) {
            mGDepth = depth;
        }

        public GDepth getDepth() {
            return mGDepth;
        }

        public Bitmap getBokeh(){
            return mBokeh;
        }

        public void setPrimary(byte[] primaryYuv,YuvImageSize size) {
            mPrimary = getBitmapFromYuv(mContext,primaryYuv,size);
        }

        public Bitmap getPrimary() {
            return mPrimary;
        }

        public Location getLocation() {return mLoc;}

        public int getOrientation() {return  mOrientation;}

        public NamedEntity getNameEntity() {
            return mNameEntity;
        }

        public boolean isReadyToSave() {
            return mPriJpeg != null && mGDepth != null && mBokeh != null;
        }

        public void compressPrimaryBitmap() {
            mPriJpeg = compressBitmapToJpeg(mPrimary);
        }

        public void setPrimaryJpeg(byte[] jpeg) {
            mPriJpeg = jpeg;
        }

        public byte[] getPrimaryJpeg() {
            return mPriJpeg;
        }

        public boolean generateBokehBitmap() {
            boolean ret = false;
            int width = mPrimary.getWidth();
            int height = mPrimary.getHeight();
            Bitmap Gdepth = mGDepth.getBitGdepthBitmap();
            DualCameraEffect dualCameraEffect = DualCameraEffect.getInstance();
            if (!dualCameraEffect.isSupported() || mPrimary == null || Gdepth == null) {
                Log.d(TAG,"can't generate bokeh image");
                return false;
            }
            Rect roiRect = mGDepth.getRoi();
            int roiWidth = roiRect.right - roiRect.left;
            int roiHeight = roiRect.bottom - roiRect.top;
            int renderwidth = roiWidth > width ?
                    width : roiWidth;
            int renderHeight = (roiRect.bottom - roiRect.top) > height ?
                    height : (roiRect.bottom - roiRect.top);
            int[] roi = new int[] {
                    roiRect.left,roiRect.top,roiWidth,roiHeight};
            if (dualCameraEffect.initialize(mPrimary, Gdepth, roi,renderwidth, renderHeight,0.0f)) {
                mBokeh = Bitmap.createBitmap(renderwidth,renderHeight, Bitmap.Config.ARGB_8888);
                Point center;
                if (mFocus != null) {
                    Matrix matrix = new Matrix();
                    CameraUtil.prepareMatrix(matrix,false,0,roiRect);
                    RectF focus = CameraUtil.rectToRectF(mFocus);
                    matrix.mapRect(focus);
                    center = new Point((int)focus.centerX(),(int)focus.centerY());
                }else {
                    center = new Point(roiRect.centerX(),roiRect.centerY());

                }
                dualCameraEffect.map(center);
                ret = dualCameraEffect.render(
                        DualCameraEffect.REFOCUS_CIRCLE, center.x, center.y, mBokeh);
                if (DEBUG) {
                    Canvas canvas = new Canvas(mBokeh);
                    Paint paint = new Paint();
                    paint.setColor(Color.RED);
                    paint.setStrokeWidth(5.0f);
                    canvas.drawCircle(center.x, center.y, 30,paint);
                }
            }
            dualCameraEffect.release();
            return ret;
        }

        public void release() {
            if (mPrimary != null) {
                mPrimary.recycle();
            }
            if (mBokeh != null) {
                mBokeh.recycle();
            }
            mPriJpeg = null;
            mGDepth = null;
        }
    }

    public class ProcessTask {
        private GdepthProcess mGdepthProcess;
        private BokehProcess mBokehProcess;
        private NamedEntity mKey;

        public ProcessTask(NamedEntity namedEntity) {
            mKey = namedEntity;
        }

        public void setGdepthProcess(GdepthProcess process) {
            mGdepthProcess = process;
        }

        public void setBokehProcess(BokehProcess process) {
            mBokehProcess = process;
        }

        public GdepthProcess getGdepthProcess() {
            return  mGdepthProcess;
        }

        public BokehProcess getBokehProcess() {
            return mBokehProcess;
        }

        public void release() {
            if (mGdepthProcess != null) {
                mGdepthProcess.release();
            }
            if (mBokehProcess != null) {
                mBokehProcess.release();
            }
            mTask.remove(mKey);
            if (mTask.size() < MAX_PROCESS_QUEUE) {
                mModule.getMainActivity().runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        mCallback.enableShutterLock(true);
                    }
                });
            }
        }
    }

    public static class YuvImageSize {
        private int mWidth;
        private int mHeight;
        private int mScanline;
        private int[] mStrides;
        private Rect mFocus;

        public YuvImageSize(int width, int height, int[] strides, int scanline){
            mWidth = width;
            mHeight = height;
            mStrides = strides;
            mScanline = scanline;
        }

        public int getWidth() {
            return mWidth;
        }

        public int getHeight() {
            return mHeight;
        }

        public int[] getStrides() {
            return mStrides;
        }

        public int getScanline() {
            return mScanline;
        }

        public Rect getFocus() {return mFocus;}

        public void setFocus(Rect focus) {
            mFocus = focus;
        }
    }

    public void setOtpMetaData(byte[] data){
        byte[] otpData = new byte[data.length -8];
        for(int i = 0; i < otpData.length; i++ ) {
            otpData[i] = data[i + 8];
        }
        mOtpMetaData = ClearSightNativeEngine.CamSystemCalibrationData.createFromBytes(otpData);
    }


    public static Bitmap getBitmapFromYuv(Context context, byte[] yuv, YuvImageSize size) {
        int yLength = size.getStrides()[0] * size.getHeight();
        byte[] data = new byte[yLength * 3 /2];
        System.arraycopy(yuv,0,data,0,yLength);
        System.arraycopy(yuv,size.getStrides()[0] * size.getScanline(),data,yLength,yLength/2);

        RenderScript rs = RenderScript.create(context);
        ScriptIntrinsicYuvToRGB yuvToRgbIntrinsic =
                ScriptIntrinsicYuvToRGB.create(rs, Element.RGBA_8888(rs));
        Type.Builder yuvTypeBuilder = new Type.Builder(rs, Element.U8(rs)).setX(data.length);
        Type.Builder rgbTypeBuilder = new Type.Builder(rs, Element.RGBA_8888(rs));
        rgbTypeBuilder.setX(size.getWidth());
        rgbTypeBuilder.setY(size.getHeight());

        Allocation input = Allocation.createTyped(rs, yuvTypeBuilder.create(),
                Allocation.USAGE_SCRIPT);
        Allocation output = Allocation.createTyped(rs, rgbTypeBuilder.create(),
                Allocation.USAGE_SCRIPT);

        input.copyFrom(data);
        yuvToRgbIntrinsic.setInput(input);
        yuvToRgbIntrinsic.forEach(output);

        Bitmap bitmap = Bitmap.createBitmap(size.getWidth(), size.getHeight(),
                Bitmap.Config.ARGB_8888);
        output.copyTo(bitmap);

        return bitmap;
    }

    public static byte[] compressBitmapToJpeg(Bitmap bitmap) {
        if ( bitmap == null ) {
            Log.d(TAG, " buffer can't be decoded ");
            return null;
        }
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        bitmap.compress(Bitmap.CompressFormat.JPEG, 75, outputStream);
        return outputStream.toByteArray();
    }

    private class BitmapOutputStream extends ByteArrayOutputStream {
        public BitmapOutputStream(int size) {
            super(size);
        }

        public byte[] getArray() {
            return buf;
        }
    }

    private byte[] nv21ToJpeg(ImageFilter.ResultImage resultImage, int orientation) {
        BitmapOutputStream bos = new BitmapOutputStream(1024);
        YuvImage im = new YuvImage(resultImage.outBuffer.array(), ImageFormat.NV21,
                resultImage.width, resultImage.height, new int[]{resultImage.stride, resultImage.stride});
        im.compressToJpeg(resultImage.outRoi, 75, bos);
        byte[] bytes = bos.getArray();
        bytes = addExifTags(bytes, orientation);
        return bytes;
    }

    public byte[] addExifTags(byte[] jpeg, int orientationInDegree) {
        ExifInterface exif = new ExifInterface();
        exif.addOrientationTag(orientationInDegree);
        ByteArrayOutputStream jpegOut = new ByteArrayOutputStream();
        try {
            exif.writeExif(jpeg, jpegOut);
        } catch (IOException e) {
            Log.e(TAG, "Could not write EXIF", e);
        }
        return jpegOut.toByteArray();
    }

    public static float byte2float(byte[] b, int index) {
        int l;
        l = b[index + 0];
        l &= 0xff;
        l |= ((long) b[index + 1] << 8);
        l &= 0xffff;
        l |= ((long) b[index + 2] << 16);
        l &= 0xffffff;
        l |= ((long) b[index + 3] << 24);
        return Float.intBitsToFloat(l);
    }

    public interface BokehCallback {
        public static final int CAPTURE_FAIL = 0;
        public static final int PARAMETER_FAIL = 1;
        public static final int GDEPTH_FAIL = 2;
        public static final int BOKEH_FAIL = 3;
        public static final int QUEUE_FULL = 4;
        public static final int PACK_FAIL = 5;
        public void enableShutterLock(boolean enable);
        public void onBokehSuccess();
        public void onBokenFailure(int reason);
    }
}
