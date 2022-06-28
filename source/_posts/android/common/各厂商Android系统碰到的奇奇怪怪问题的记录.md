---
title: 各厂商Android系统碰到的奇奇怪怪问题的记录
date: 2021-12-23 10:39:00
tags: 常见问题
categories: 
- [Android, 常见问题]
---

# 小米

## MIUI

### Camera2

`CaptureRequest.Builder`的`set`方法，对部分`key`不生效

```java
// MIUI中，CaptureRequest.Builder设置图片方向不生效
captureBuilder.set(CaptureRequest.JPEG_ORIENTATION,getJpegOrientation(deviceRotation));
```

解决方法：获得拍摄好的照片`Bitmap`后，再对其进行旋转

```java
public Bitmap rotateBitmap(Bitmap bitmap, int angle) {
   Matrix matrix = new Matrix();
   matrix.setRotate(angle);
   return Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(), bitmap.getHeight(), matrix, true);
}
```

# 华为

## HarmonyOs

### TextureView

华为ROM（EMUI不确定有没有这种情况）计算`TextureView`边界的代码似乎有bug

现象：
1. 相机预览和拍摄时有概率画面畸形
2. 渲染超过一屏的文本会渲染空白

解决方法：手动管理`TextureView`的销毁和创建

第一步：在对`TextureView`设置`TextureView.SurfaceTextureListener`时，另`onSurfaceTextureDestroyed`返回`false`

```java
mTextureView.setSurfaceTextureListener(new TextureView.SurfaceTextureListener() {
    @Override
    public void onSurfaceTextureAvailable(@NonNull SurfaceTexture surface, int width, int height) {
       ...
    }

    @Override
    public void onSurfaceTextureSizeChanged(@NonNull SurfaceTexture surface, int width, int height) {
    }

    @Override
    public boolean onSurfaceTextureDestroyed(@NonNull SurfaceTexture surface) {
        // 这里默认是返回true，代表系统自动管理，我们把它设为false手动管理
        return false;
    }

    @Override
    public void onSurfaceTextureUpdated(@NonNull SurfaceTexture surface) {
    }
});
```

第二步：在`TextureView`不渲染的时候手动`release`掉其中的`SurfaceTexture`，后面再渲染时，系统调用draw方法后，会自动重新`new`一个`SurfaceTexture`出来

```java
SurfaceTexture surfaceTexture = mTextureView.getSurfaceTexture();
if (surfaceTexture != null) {
    surfaceTexture.release();
}
```

# VIVO

## OriginOS

### 字体

OriginOS中，`TextView`设置了`android:fontFamily`后，不能在设置`android:textStyle`属性，否则会导致使用的字体被系统默认字体覆盖
