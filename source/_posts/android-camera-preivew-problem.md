---
title: android camera 摄像头预览画面变形
date: 2013-09-08 22:59:37
tags: android
permalink: ?p=14
---

问题：最近在处理一下camera的问题，发现在竖屏时预览图像会变形，而横屏时正常。但有的手机则是横竖屏都会变形。

结果：解决了预览变形的问题，同时支持前后摄像头，预览无变形，拍照生成的jpg照片方向正确。
<!--more-->

环境 ： android studio, xiaomi m1s android4.2 miui v5

过程：

## 1. 预览 preview画面变形
以sdk中apidemos里的camera为例，进行修改。先重现下问题，在AndroidManifest.xml中指定了activity 的screenOrientation为landspace，则预览正常。若指定为portrait，则图像会有拉伸变形。

### 找到正确的previewsize
继续看demo代码，mCamera.getParameters().getSupportedPreviewSizes() 可以返回当前设备支持的一组previewSize，例如：

> 1920x1088 1280x720 800x480 768x432 720x480 640x480 576x432 480x320
> 384x288 352x288 320x240 240x160 176x144

而我们根据我们在界面上需要显示的预览大小，来设置camera的预览大小，即在这一组size中选择一个previewsize，找到高宽比和大小最接近的一个size。通过调用 getOptimalPreviewSize(List<Size> sizes, int w, int h) 来进行选择。注意这里的后两个参数，因为**摄像头的预览size是固定的，就那么一组，其高宽比是固定的，且方向也是固定的。**
对于摄像头来说，都是width是长边，即width > height。 所以camera的ratio计算值总是大于1的。所以当手机在横屏的时候，我们的w>h，调用该方法进行选择是没问题的。但是当竖屏后，w < h了，若还是直接调用该方法，则targetRatio 会小于1，按这个targetRatio去找就找不到合适的size了，那么比例不对预览自然就变形了。所以得在调用的地方进行调整，保证参数w > h。

```java
private Size getOptimalPreviewSize(List<Size> sizes, int w, int h) {
    final double ASPECT_TOLERANCE = 0.1;
    double targetRatio = (double) w / h;
    if (sizes == null) return null;

    Size optimalSize = null;
    double minDiff = Double.MAX_VALUE;

    int targetHeight = h;

    // Try to find an size match aspect ratio and size
    for (Size size : sizes) {
        double ratio = (double) size.width / size.height;
        if (Math.abs(ratio - targetRatio) > ASPECT_TOLERANCE) continue;
        if (Math.abs(size.height - targetHeight) < minDiff) {
            optimalSize = size;
            minDiff = Math.abs(size.height - targetHeight);
        }
    }

    // Cannot find the one match the aspect ratio, ignore the requirement
    if (optimalSize == null) {
        minDiff = Double.MAX_VALUE;
        for (Size size : sizes) {
            if (Math.abs(size.height - targetHeight) < minDiff) {
                optimalSize = size;
                minDiff = Math.abs(size.height - targetHeight);
            }
        }
    }
    return optimalSize;
}

 @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
           // We purposely disregard child measurements because act as a
           // wrapper to a SurfaceView that centers the camera preview instead
           // of stretching it.
           final int width = resolveSize(getSuggestedMinimumWidth(), widthMeasureSpec);
           final int height = resolveSize(getSuggestedMinimumHeight(), heightMeasureSpec);
           setMeasuredDimension(width, height);

           if (mSupportedPreviewSizes != null) {
                mPreviewSize = getOptimalPreviewSize(mSupportedPreviewSizes, Math.max(width, height), Math.min(width, height));

           }
    ......
```
### 选择和previewsize一致的比例来布局surfaceview

当我们选择了合适的previewsize后，还有一个因素会影响到预览画面是否正常。即surfaceview的布局。从demo代码可以知道要想在界面上显示camera预览画面，需要添加一个surfaceview，而我们添加了surfaceview后，就需要对其进行布局，设置其大小，位置。


经过搜索查资料，[here](http://stackoverflow.com/questions/7580887/android-camera-stretched-in-landscape-modesomewhat-urgent)这里有人回答了原因。

> “The reason is: SurfaceView aspect ratio (width/height) MUST be same as Camera.Size aspect ratio used in preview parameters. And if aspect
> ratio is not the same, you've got stretched image.”

 看到这句话，理解了变形是因为比例错误。surfaceview和cameraSize的比例应该要一致。

在onlayout中，我们对surfaceview进行了layout，根据指定的surfaceview的高宽来布局。demo中会将surfacevidew居中，看代码是根据宽高比和previewsize的宽高比来对比，选择水平居中或垂直居中。前面已经说过，previewsize的width大于height，所以凡是涉及到宽高比计算的地方，两个size我们都需要保持同样的顺序，都是w > h，或w < h。所以当竖屏时，我们在判断前应该交换presize的w和h，才能正确布局居中surfaceview。
```java
 protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int curOrientation =
            getContext().getResources().getConfiguration().orientation;

    if (changed && getChildCount() > 0 || mLastOrientation != curOrientation) {
        mLastOrientation = curOrientation;
        final View child = getChildAt(0);

        final int width = r - l;
        final int height = b - t;

        int previewWidth = width;
        int previewHeight = height;
        if (mPreviewSize != null) {
            previewWidth = mPreviewSize.width;
            previewHeight = mPreviewSize.height;

            if (curOrientation == Configuration.ORIENTATION_PORTRAIT) {
                previewWidth = mPreviewSize.height;
                previewHeight = mPreviewSize.width;
            }
        }

         // Center the child SurfaceView within the parent.
        if (width * previewHeight > height * previewWidth) {
            final int scaledChildWidth = previewWidth * height / previewHeight;
            child.layout((width - scaledChildWidth) / 2, 0,
                    (width + scaledChildWidth) / 2, height);
        } else {
            final int scaledChildHeight = previewHeight * width / previewWidth;
            child.layout(0, (height - scaledChildHeight) / 2,
                    width, (height + scaledChildHeight) / 2);
        }
      }
}
```
至此，经过测试，设备显示正常，横屏竖屏均再无拉伸现象了。

总结下，demo工程在横屏下正常，而在竖屏下出现预览画面变形的原因主要是，onMeasure时选择previewsize和onlayout时布局surfaceview，都是基于横屏考虑的，所以w均大于h。
当activity改为竖屏运行时，就需要调整这两个地方，保证比例一致，才能计算正确，从而显示正常画面。


## 2. 方向问题
刚刚讲了变形的问题，我们应该发现当手机竖屏后，预览画面的方向没有随之改变过来，于是看上去就颠倒了。所以我们还需要处理一下方向的问题。

注意这里的“方向”包括：前置、后置摄像头画面预览方向，前置后置摄像头拍照后的图片方向。


### 预览方向：
通过查询android文档，可以发现如下资料：

> For example, suppose the natural orientation of the device is
> portrait. The device is rotated 270 degrees clockwise, so the device
> orientation is 270. Suppose a back-facing camera sensor is mounted in
> landscape and the top side of the camera sensor is aligned with the
> right edge of the display in natural orientation. So the camera
> orientation is 90. The rotation should be set to 0 (270 + 90).

（后置摄像头）
可以得知，camera是在设置上是固定方向的， camera的顶部是和屏幕自然显示时的右边对齐的。说明camera默认就是横屏方向的。为了在竖屏的时候进行preview预览，我们需要调整camera的方向。

setDisplayOrientation 可以修改camera的预览方向。

> If you want to make the camera image show in the same orientation as
> the display, you can use the following code.

```java
 public static void setCameraDisplayOrientation(Activity activity,
     int cameraId, android.hardware.Camera camera) {
 android.hardware.Camera.CameraInfo info =
         new android.hardware.Camera.CameraInfo();
 android.hardware.Camera.getCameraInfo(cameraId, info);
 int rotation = activity.getWindowManager().getDefaultDisplay()
         .getRotation();
 int degrees = 0;
 switch (rotation) {
     case Surface.ROTATION_0: degrees = 0; break;
     case Surface.ROTATION_90: degrees = 90; break;
     case Surface.ROTATION_180: degrees = 180; break;
     case Surface.ROTATION_270: degrees = 270; break;
 }

 int result;
 if (info.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
     result = (info.orientation + degrees) % 360;
     result = (360 - result) % 360;  // compensate the mirror
 } else {  // back-facing
     result = (info.orientation - degrees + 360) % 360;
 }
 camera.setDisplayOrientation(result);
 }
```

### 拍照方向：

若想修改照片的方向，还需调用 camera.setRotation，google 的文档说的很清楚了，添加一个orientation listener既可以从传感器获取当前设备旋转方向，参考如下代码即可正确设置照片方向。但要注意一点，***这个方向和activity方向（可以在android-manifest设置）无关。*** 无论activity此时是什么方向，只要获取了传感器方向均可以正确调整照片方向，与预览方面一致。

代码方面，由于这个回调调用比较频繁（设备角度一变化就会调用），可以在回调里保存下rotation，然后在拍照的时候再设置camera。由于考虑了前置摄像头，须注意传递正确的cameraId。
```java
    mCamera.setParameters(parameters);
    mCamera.takePicture(shutterCallback, rawCallback,jpegCallback);
```

> CameraInfo.orientation is the angle between camera orientation and
> natural device orientation. The sum of the two is the rotation angle
> for back-facing camera. The difference of the two is the rotation
> angle for front-facing camera. Note that the JPEG pictures of
> front-facing cameras are not mirrored as in preview display. For
> example, suppose the natural orientation of the device is portrait.
> The device is rotated 270 degrees clockwise, so the device orientation
> is 270. Suppose a back-facing camera sensor is mounted in landscape
> and the top side of the camera sensor is aligned with the right edge
> of the display in natural orientation. So the camera orientation is
> 90. The rotation should be set to 0 (270 + 90).

> The reference code is as follows.

```java
public void onOrientationChanged(int orientation) {
    if (orientation == ORIENTATION_UNKNOWN) return;
        android.hardware.Camera.CameraInfo info =
            new android.hardware.Camera.CameraInfo();
        android.hardware.Camera.getCameraInfo(cameraId, info);
        orientation = (orientation + 45) / 90 * 90;
        int rotation = 0;
        if (info.facing == CameraInfo.CAMERA_FACING_FRONT) {
            rotation = (info.orientation - orientation + 360) % 360;
        } else {  // back-facing camera
            rotation = (info.orientation + orientation) % 360;
        }
        mParameters.setRotation(rotation);
 }
```

## 3. 总结
到此为止，我们基本上是实现了一个最简单的拍照应用，能支持前后摄像头，预览正确，照片方向正确。文中一些api在低版本sdk上没有，不能直接用，还须参考资料换用其他方法。
由于手头设备有限，我仅仅在android4.2 小米手机上测试过，pad未测试。

## 补充
setRotation 在一些设备上无效 拍照后得到的图像还是横屏的，所以在save前需要再rotate一下。
```java
private Bitmap adjustPhotoRotationToPortrait(byte[] data) {
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeByteArray(data, 0, data.length, options);
    if (options.outHeight < options.outWidth) {
        int w = options.outWidth;
        int h = options.outHeight;

        Matrix mtx = new Matrix();
        mtx.postRotate(90);
        // Rotating Bitmap
        Bitmap bmp = BitmapFactory.decodeByteArray(data, 0, data.length);
        Bitmap rotatedBMP = Bitmap.createBitmap(bmp, 0, 0, w, h, mtx, true);
        return rotatedBMP;
    } else {
        return null;
    }
}
```








