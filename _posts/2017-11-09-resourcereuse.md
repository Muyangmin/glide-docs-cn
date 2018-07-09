---
layout: page
title: "资源重用"
category: doc
date: 2018/7/9 17:06
order: 11
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/resourcereuse.html){:target="_blank"}

* TOC
{:toc}

### 资源
Glide 中的资源包含很多东西，例如 ``Bitmap``，``byte[]`` 数组， ``int[]`` 数组，以及大量的 POJO 。无论什么时候，Glide 都会尝试重用这些资源，以限制你应用中的内存抖动数量。

### 好处
任何尺寸的对象的过多分配都会显著增加你应用中的垃圾回收 (GC)。虽然 Android 较新的 ART 运行时的 GC 惩罚比 Dalvik 运行时要低，但无论你使用什么设备，过多内存分配都会降低应用的性能。

### Dalvik 
Dalvik 设备 (Lollipop 之前)在过多分配时将不得不面对特别大的代价，值得在这里讨论一下。

Dalvik 有两种基本的 GC 模式， GC_CONCURRENT 和 GC_FOR_ALLOC ，这两种你都可以在 logcat 中看到。

* GC_CONCURRENT 对于每次收集将阻塞主线程大约 5ms 。因为每个操作都比一帧(16ms)要小，GC_CONCURRENT 通常不会造成你的应用丢帧。
* GC_FOR_ALLOC 是一种 stop-the-world 收集，可能会阻塞主线程达到 125ms 以上。GC_FOR_ALLOC 几乎每次都会造成你的应用丢失多个帧，导致视觉卡顿，特别是在滑动的时候。

不幸的是，Dalvik 似乎甚至连适度的分配（例如一个 16kb 的缓冲区）都处理得不是很好。重复的中等分配，或即使单次大的分配（比如说一个 Bitmap ），将会导致 GC_FOR_ALLOC 。因此，你分配的内存越多，就会招来越多 stop-the-world 的 GC，而你的应用将有更多的丢帧。
 
通过复用中到大尺寸的资源， Glide 可以帮你尽可能地减少这种 GC，以保持应用的流畅。 

## Glide 如何追踪和重用资源
Glide 采用较为宽容的办法来处理资源重用。Glide 会在它相信某个资源可以安全地复用时才这么做，但它并不要求调用者在每次请求之后都回收资源。除非某个调用者显式地表示它已经用完了某个资源（见下文），资源将不会被回收或重用。

### 引用计数 
为决定某个资源是否正在被使用，以及什么时候可以安全地被重用，Glide 为每个资源保持了一个引用计数。

#### 增加引用计数
每次调用 [``into()``][1] 来加载一个资源，这个资源的引用计数会被加一。如果相同的资源被加载到两个不同的 [``Target``][2]，则在两个加载都完成后，它的引用计数将会为二。

#### 减少引用计数
引用计数仅在调用者通过以下方式表示它们用完资源后会减少：
1. 在加载资源的 [``View``][4] 或 [``Target``][2] 上调用 [``clear()``][3] 。  
2. 在这个[``View``][4] 或 [``Target``][2] 上调用对另一个资源请求的 [``into``][1] 方法。 

#### 释放资源
当引用计数到达 0 时，这个资源会被释放并被返回给 Glide 以重用。当资源被返回给 Glide 以重用以后，继续使用它是不安全的，因此以下行为是 **不安全的**：
1. 使用 ``getImageDrawable`` 来取回 ``ImageView`` 中加载的 ``Bitmap`` 或 ``Drawable``，并使用某种方式展示它( ``setImageDrawable``，动画，或 ``TransitionDrawable`` 或其他任何方式 )。
2. 使用 [``SimpleTarget``][5] 来将一个资源加载到 [``View``][4]，但没有实现 [``onLoadCleared()``][6] 方法并在其中将资源从 [``View``][4] 中移除。
3. 对 Glide 加载的任何 ``Bitmap`` 调用 [``recycle()``][7]。

在清理对应的 [``View``][4] 或 [``Target``][2] 之后还保持对资源的引用是不安全的，因为这个资源可能已经被销毁，或被重用于展示一个不同的图片，这可能导致未定义行为，图形损坏，或甚至导致继续使用该资源的应用崩溃。例如，在被释放回 Glide 之后， ``Bitmap`` 可能会被存储在一个 ``BitmapPool`` 中，并在未来的某个时刻被用重用于保存一张新图片的字节数据，或者它们已经被调用了 [``recycle()``]。在这两种情况下继续引用这个 ``Bitmap`` 并期待它们保持原始图像都是不安全的。

### 池化 (Pooling)
尽管 Glide 的大部分回收逻辑主要针对 Bitmap，但所有的 [``Resource``][8] 实现均可实现 [``recycle()``][9] 方法并将它们包含的任意可重用的数据池化。 [``ResourceDecoder``][10] 可以返回开发者希望的任意 [``Resource``] API，因此开发者可以定制或提供额外的池化规则，只需要实现它们自己的 [``Resource``][8] 和 [``ResourceDecoder``][10]。

特别地，对于 ``Bitmap``，Glide 提供了一个 [``BitmapPool``][11] 接口，以允许 [``Resource``][8] 获取和重用 [``Bitmap``] 对象。 Glide 的 [``BitmapPool``] 可以从任意的 ``Context`` 中使用 Glide 的单例获取到：

```java
Glide.get(context).getBitmapPool();
```

类似地，希望为 ``Bitmap`` 池化施加更多控制的用户可以直接实现他们自己的 [``BitmapPool``][11]，然后可以通过 ``GlideModule`` 的方式提供给 Glide。参见[配置页][12].

## 常见错误
然而，允许池化让保证用户不会误用资源或``Bitmap``变得很困难。 Glide 会在可能的地方尝试添加一些断言，但是因为我们并不持有底层的 ``Bitmap``，我们无法保证调用者在告诉我们 [``clear()``][3] 或一个新请求之后，会立即停用这些资源。

### 资源重用错误的征兆
有多种迹象可能暗示 ``Bitmap`` 或其他在 Glide 中被池化的资源出了问题。我们列出了一些最常见的现象，但这不是一个完备的列表。

#### Cannot draw a recycled Bitmap
Glide 的 ``BitmapPool`` 是固定大小的。当 ``Bitmap`` 从中被踢出而没有被重用时，Glide 将会调用 [``recycle()``][7]。如果应用在向 Glide 指出可以安全地回收之后 "不经意间" 继续持有 ``Bitmap``，则应用可能尝试绘制这个 ``Bitmap``，进而在 ``onDraw`` 方法中造成崩溃。

一种可能的情况是，一个目标被用于两个``ImageView``，而其中一个在 ``Bitmap`` 被放到 ``BitmapPool`` 中后仍然试图访问被回收后的 ``Bitmap``。基于以下因素，要复现这种复用错误可能很困难：1）Bitmap 何时被放入池中，2）Bitmap 何时被回收，3）何种尺寸的 ``BitmapPool`` 和内存缓存会导致 ``Bitmap`` 的回收。可以在你的 ``GlideModule`` 中加入下面的代码片段，以使这个问题更容易复现：

```java
@Override
public void applyOptions(Context context, GlideBuilder builder) {
    int bitmapPoolSizeBytes = 1024 * 1024 * 0; // 0mb
    int memoryCacheSizeBytes = 1024 * 1024 * 0; // 0mb
    builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));
    builder.setBitmapPool(new LruBitmapPool(bitmapPoolSizeBytes));
}
```

上面的代码确保没有内存缓存，且 ``BitmapPool`` 的尺寸为0；因此 ``Bitmap`` 如果恰好没有被使用，它将立刻被回收。这是为了调试目的让它更快出现。

#### Can't call reconfigure() on a recycled bitmap
资源将在它们不再被使用时被返回到 Glide 的 ``BitmapPool`` 中。这里的内部实现基于 ``Request``(它控制着[``Resource``][8]) 的生命周期管理。如果在这些 Bitmap 上调用了 [``recycle()``][7]，但它们仍然在池中，就会使 Glide 无法重用它们而导致你的应用崩溃并抛出这个信息。这里的一个关键点是，这个崩溃很可能发生在未来的某个点，而不在这个违例代码的执行处！

#### View 在图片之间闪烁或相同的图像在多个 View 中展示
如果一个 ``Bitmap`` 被多次返回到 ``BitmapPool`` 中，或它已被返回到池中单仍然被一个 [``View``][4] 持有，另一个图片可能会被解码到这个 ``Bitmap`` 对象中。如果这种情况发生，就会使得 ``Bitmap`` 的内容会被替换为新的图片。 在这个过程中，``View`` 可能仍然试图绘制这个 ``Bitmap``，而这将导致原始的 ``View`` 展示一张新的图片。

### 重用错误的原因
一些常见的重用错误原因已被列在下面。就像上面的征兆一样，要列出全面的列表是很困难的，但是在尝试调试应用程序中的重用错误时，这些是您应该考虑的一些事情。

#### 尝试往相同的 Target 加载两个不同的资源
在 Glide 中没有安全的办法来加载多个资源到单一的 Target 中。用户可以使用 [``thumbnail()``][13] API 来加载一系列资源到一个 [``Target``][2]，但也仅仅在下一个 [``onResourceReady()``][14] 调用之前才可以安全地引用早前的一个资源。

通常一个更好的答案是使用第二个 [``View``][4] 并将第二章图片加载到这第二个 View 上。 [``ViewSwitcher``][19] 可以很好地允许你在两个单独请求的不同图片之间做交叉淡入效果 (cross fade)。你可以仅添加一个 ``ViewSwitcher`` 在你的布局中，使用两个 ``ImageView`` 作为其子控件，然后使用两次 [``into(ImageView)``][20]方法，每次一个子控件，来加载两张图片。

对于绝对要求将多个资源加载到相同 [``View``][4] 的用户，可以使用两个单独的 [``Target``][2]。为确保每个加载都不会取消另一个，用户还需要避免使用 [``ViewTarget``][15] 子类，或使用一个自定义的 [``ViewTarget``] 子类并复写(override)其 [``setRequest()``][16] 和 [``getRequest()``][17] 以使得它们不使用 [``View``][4] 的 tag 来存储 [``Request``][18]。这属于高级用法，一般不推荐。

#### 往Target中加载资源，清除或重用Target，并继续引用该资源
最简单的比较这个错误的办法是确保所有对资源的引用都在 [``onLoadCleared()``][6] 调用时置空。通常，加载一个 ``Bitmap`` 然后对 ``Target`` 解引用，并且不要再次调用 [``into()``][1] 或 [``clear()``][3]，这样是安全的。然而，加载了一个 ``Bitmap``，清除这个 ``Target``，并在之后继续持有 ``Bitmap`` 引用是不安全的。类似地，加载资源到一个 ``View`` 上然后从 View 中获取这个资源 (通过 ``getImageDrawable()`` 或任何其他手段)，并在其他某个地方继续引用它，也是不安全的。


#### 在 ``Transformation<Bitmap>`` 中回收原始Bitmap
正如在 [``变换``][21] 的 JavaDoc 中所说，传入 [``transform()``][22] 的原始 ``Bitmap`` 将会自动被回收，只要这个 ``Transformation`` 返回的 ``Bitmap`` 和原始传入 [``transoform()``][22] 的不是同一个实例。这是和其他加载库很重要的一个不同，例如 Picasso。 [``BitmapTransformation``][23] 提供了 Glide 的资源创建的模板，但它的回收是在内部完成的，所以不管是 ``Transformation`` 还是 ``BitmapTransformation`` 都不要回收传入的 ``Bitmap`` 或 ``Resource``。

另外值得注意的是，任何定制的 ``BitmapTransformation`` 从 ``BitmapPool`` 中创建、但没有从 [``transform()``][22] 返回的中间 ``Bitmap``，都会被返回到 ``BitmapPool`` 或被回收，但不会两种情况同时发生。你永远都不应该 [``recycle()``][7] 从 Glide 中创建的 ``Bitmap``。


[1]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/RequestBuilder.html#into-Y-
[2]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/Target.html
[3]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/RequestManager.html#clear-com.bumptech.glide.request.target.Target-
[4]: http://d.android.com/reference/android/view/View.html?is-external=true
[5]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/SimpleTarget.html
[6]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/Target.html#onLoadCleared-android.graphics.drawable.Drawable-
[7]: https://developer.android.com/reference/android/graphics/Bitmap.html#recycle()
[8]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/Resource.html
[9]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/Resource.html#recycle--
[10]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/ResourceDecoder.html
[11]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/bitmap_recycle/BitmapPool.html
[12]: {{ site.baseurl }}/doc/configuration.html#bitmap-pool
[13]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/RequestBuilder.html#thumbnail-com.bumptech.glide.RequestBuilder-
[14]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/Target.html#onResourceReady-R-com.bumptech.glide.request.transition.Transition-
[15]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/ViewTarget.html
[16]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/Target.html#setRequest-com.bumptech.glide.request.Request-
[17]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/target/Target.html#getRequest--
[18]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/request/Request.html
[19]: https://developer.android.com/reference/android/widget/ViewSwitcher.html
[20]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/RequestBuilder.html#into-android.widget.ImageView-
[21]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/Transformation.html
[22]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/resource/bitmap/BitmapTransformation.html#transform-com.bumptech.glide.load.engine.bitmap_recycle.BitmapPool-android.graphics.Bitmap-int-int-
[23]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/resource/bitmap/BitmapTransformation.html
