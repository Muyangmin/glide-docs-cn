---
layout: page
title: "硬件位图"
category: doc
date: 2018-02-11 09:27:59
order: 13
disqus: 1
---
* TOC
{:toc}

### 什么是硬件位图（Hardware Bitmaps）？
[`Bitmap.Config.HARDWARE`][3] 是一种 Android O 添加的新的位图格式。硬件位图仅在显存 (graphic memory) 里存储像素数据，并对图片仅在屏幕上绘制的场景做了优化。

### 我们为什么应该使用硬件位图?
因为硬件位图仅储存像素数据的一份副本。一般情况下，应用内存中有一份像素数据（即像素字节数组），而在显存中还有一份副本（在像素被上传到 GPU之后）。而硬件位图仅持有 GPU 中的副本，因此：

 * 硬件位图仅需要**一半**于其他位图配置的内存；
 * 硬件位图可避免绘制时上传纹理导致的内存抖动。

### 如何启用硬件位图?
目前，你可以在 Glide 请求中将默认的 [`DecodeFormat`][1] 设置为 [`DecodeFormat.PREFER_ARGB_8888`][2]。要为应用中的所有请求都应用该操作，你需要在你的 `GlideModule` 中修改默认选项的 `DecodeFormat`，详见 [配置页][4]。

未来 Glide 将默认加载硬件位图而不需要额外的启用配置，只保留禁用的选项。

### 如何禁用硬件位图?
如果你需要禁用硬件位图，你应当仅在以下的一些缓慢的或根本不可用 (broken) 的情况下才尝试去做。你可以使用 [`disallowHardwareConfig()`][5] 来为一个特定的请求禁用硬件位图。

如果你在使用 generated API：

```java
GlideApp.with(fragment)
  .load(url)
  .disallowHardwareConfig()
  .into(imageView);
```

或直接使用 `RequestOptions`:

```java
RequestOptions options = new RequestOptions().disallowHardwareConfig();
Glide.with(fragment)
  .load(url)
  .apply(options)
  .into(imageView);
```

### 哪些情况不能使用硬件位图?
在显存中存储像素数据意味着这些数据不容易访问到，在某些情况下可能会发生异常。已知的情形列举如下：
* 在 Java 中读写像素数据，包括：
  * [Bitmap#getPixel](https://developer.android.com/reference/android/graphics/Bitmap.html#getPixel(int, int))
  * [Bitmap#getPixels](https://developer.android.com/reference/android/graphics/Bitmap.html#getPixels(int[], int, int, int, int, int, int))
  * [Bitmap#copyPixelsToBuffer](https://developer.android.com/reference/android/graphics/Bitmap.html#copyPixelsToBuffer(java.nio.Buffer))
  * [Bitmap#copyPixelsFromBuffer](https://developer.android.com/reference/android/graphics/Bitmap.html#copyPixelsFromBuffer(java.nio.Buffer))
* 在本地 (native) 代码中读写像素数据
* 使用软件画布 (software Canvas) 渲染硬件位图:
```java
Canvas canvas = new Canvas(normalBitmap)
canvas.drawBitmap(hardwareBitmap, 0, 0, new Paint());
```
* 在绘制位图的 View 上使用软件层 (software layer type) （例如，绘制阴影）
```java
ImageView imageView = …
imageView.setImageBitmap(hardwareBitmap);
imageView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

* 打开过多的文件描述符
. 
    每个硬件位图会消耗一个文件描述符。这里存在一个每个进程的文件描述符限制 ( Android O 及更早版本一般为 1024，在某些 O-MR1 和更高的构建上是 32K)。Glide 将尝试限制分配的硬件位图以保持在这个限制以内，但如果你已经分配了大量的文件描述符，这可能是一个问题。

* 需要`ARGB_8888 Bitmaps` 作为前置条件
* 在代码中触发截屏操作，它会尝试使用 ``Canvas`` 来绘制视图层级。

    作为一个替代方案，在 Android O 以上版本你可以使用 [`PixelCopy`][6].  

* 共享元素过渡 (shared element transition)(OMR1已修复)

以下是一个示例 trace:
```
java.lang.IllegalStateException: Software rendering doesn't support hardware bitmaps
  at android.graphics.BaseCanvas.throwIfHwBitmapInSwMode(BaseCanvas.java:532)
  at android.graphics.BaseCanvas.throwIfCannotDraw(BaseCanvas.java:62)
  at android.graphics.BaseCanvas.drawBitmap(BaseCanvas.java:120)
  at android.graphics.Canvas.drawBitmap(Canvas.java:1434)
  at android.graphics.drawable.BitmapDrawable.draw(BitmapDrawable.java:529)
  at android.widget.ImageView.onDraw(ImageView.java:1367)
[snip]
  at android.view.View.draw(View.java:19089)
  at android.transition.TransitionUtils.createViewBitmap(TransitionUtils.java:168)
  at android.transition.TransitionUtils.copyViewImage(TransitionUtils.java:102)
  at android.transition.Visibility.onDisappear(Visibility.java:380)
  at android.transition.Visibility.createAnimator(Visibility.java:249)
  at android.transition.Transition.createAnimators(Transition.java:732)
  at android.transition.TransitionSet.createAnimators(TransitionSet.java:396)
[snip]
```

### 使用硬件位图有什么缺点?
在某些情况下为了避免打断用户，`Bitmap` 类将执行一次昂贵的显存复制。在某些使用这些方法的情况下，你应该根据使用这些缓慢方法的使用频率来考虑避免使用硬件位图配置。如果你确实要使用这些方法，系统将会打印一条信息： `“Warning attempt to read pixels from hardware bitmap, which is very slow operation”`，并触发一次 [`StrictMode#noteSlowCall`][7]。
* [Bitmap#copy](https://developer.android.com/reference/android/graphics/Bitmap.html#copy(android.graphics.Bitmap.Config, boolean))
* [Bitmap#createBitmap*](https://developer.android.com/reference/android/graphics/Bitmap.html#createBitmap(android.graphics.Bitmap, int, int, int, int))
* [Bitmap#writeToParcel](https://developer.android.com/reference/android/graphics/Bitmap.html#writeToParcel(android.os.Parcel, int))
* [Bitmap#extractAlpha](https://developer.android.com/reference/android/graphics/Bitmap.html#extractAlpha())
* [Bitmap#sameAs](https://developer.android.com/reference/android/graphics/Bitmap.html#sameAs(android.graphics.Bitmap))

[1]: https://bumptech.github.io/glide/javadocs/460/com/bumptech/glide/load/DecodeFormat.html
[2]: https://bumptech.github.io/glide/javadocs/460/com/bumptech/glide/load/DecodeFormat.html#PREFER_ARGB_8888
[3]: https://developer.android.com/reference/android/graphics/Bitmap.Config.html#HARDWARE
[4]: https://bumptech.github.io/glide/doc/configuration.html#default-request-options
[5]: https://bumptech.github.io/glide/javadocs/460/com/bumptech/glide/request/RequestOptions.html#disallowHardwareConfig--
[6]: https://developer.android.com/reference/android/view/PixelCopy.html
[7]: https://developer.android.com/reference/android/os/StrictMode.html#noteSlowCall(java.lang.String)
