---
title: "过渡"
---
原文链接：[点击查看](http://bumptech.github.io/glide/doc/transitions.html)

### About
在Glide中，[``Transitions``(直译为"过渡")][1]允许你定义Glide如何从占位符到新加载的图片，或从缩略图到全尺寸图像过渡。Transition在单一请求的上下文中工作，而不会跨多个请求。因此，[``Transitions``][1]并不能让你定义从一个请求到另一个请求的动画（比如，交叉淡入效果）。

### Default behavior
在Glide中，图像可能从四个地方中的任何一个位置加载出来：

1. Glide的内存缓存
2. Glide的磁盘缓存
3. 设备本地可用的一个源文件或Uri
4. 仅远程可用的一个源Url或Uri

如果图像从Glide的内存缓存中加载出来，Glide的默认过渡将不会执行。然而，在另外三种场景下，Glide的默认过渡都会被执行。

要改变这种默认的过渡行为，请查看接下来的[custom transitions][20]章节。

### Specifying Transitions

你可以查看[Options documentation][18]以获取概览和示例代码。

[``TransitionOptions``][12]用于给一个特定的请求指定transition。 每个请求可以使用[``RequestBuilder``][14]中的[``transition()``][13]方法来设定[``TransitionOptions``][12]。还可以通过使用[``BitmapTransitionOptions``][15] 或 [``DrawableTransitionOptions``][16]来指定类型特定的过渡动画。对于Bitmap和Drawable之外的资源类型，可以使用[``GenericTransitionOptions``][17]。

### Performance Tips
Android中的动画代价是比较大的，尤其是同时开始大量动画的时候。 交叉淡入和其他涉及alpha变化的动画显得尤其昂贵。 此外，动画通常比图片解码本身还要耗时。在列表和网格中滥用动画可能会让图像的加载显得缓慢而卡顿。为了提升性能，请在使用Glide向ListView, GridView, 或RecyclerView加载图片时考虑避免使用动画，尤其是大多数情况下，你希望图片被尽快缓存和加载的时候。作为替代方案，请考虑预加载，这样当用户滑动到具体的item的时候，图片已经在内存中了。

### Common Errors

#### Cross fading with placeholders and transparent images
Glide 的默认交叉淡入效果使用了[``TransitionDrawable``][8]。它提供两种动画模式，由[``setCrossFadeEnabled()``][9]控制。当交叉淡入被禁用时，正在过渡的图片会在原先显示的图像上面淡入。当交叉淡入被启用时，原先显示的图片会从不透明过渡到透明，而正在过渡的图片则会从透明变为不透明。

在Glide中，我们默认禁用了交叉淡入，这样通常看起来要好看一些。实际的交叉淡入，如上所述对两个图片同时改变alpha值，通常会在过渡的中间造成一个短暂的白色闪屏，这个时候两个图片都是部分不透明的。

不幸的是，虽然禁用交叉淡入通常是一个比较好的默认行为，当待加载的图片包含透明像素时仍然可能造成问题。当占位符比实际加载的图片要大，或者图片部分为透明时，禁用交叉淡入会导致动画完成后占位符在图片后面仍然可见。 如果你在加载透明图片时使用了占位符，你可以启用交叉淡入，具体办法是调整[``DrawableCrossFadeFactory``][10]里的参数并将结果传到[``transition()``][11]中。

#### Cross fading across requests.
[``Transitions``][1]并不能让你在不同请求中加载的两个图像之间做过渡。当新的加载被应用到View或Target(查看[Target的文档][19])上时，Glide默认会取消任何已经存在的请求。因此，如果你想加载连个个不同的图片并在它们之间做动画，你无法直接通过Glide来完成。等待第一个加载完成并在View外持有这个Bitmap或Drawable，然后开始新的加载并手动在这两者之间做动画，诸如此类的策略看起来有效，但是实际上不安全，并可能导致程序崩溃或图像错误。

相反，最简单的办法是使用包含两个[``ImageView``][3]的[``ViewSwitcher``][2]来完成。将第一张图片加载到[``getNextView()``][4]的返回值里面，然后将第二张图片加载到[``getNextView()``][4]的下一个返回值中，并使用一个[``RequestListener``][5]在第二张图片加载完成时调用[``showNext()``][6]。为了更好地控制，你也可以使用[developer documentation][7]指出的策略。但要记住与[``ViewSwitcher``] [2]一样，仅在第二次图像加载完成后才开始交叉淡入淡出。

### Custom Transitions
如果要定义一个自定义的过渡动画，你需要完成以下两个步骤：

1. 实现[``TransitionFactory``][21].
2. 使用[``DrawableTransitionOptions#with``][22]来将你自定义的``TransitionFactory``应用到加载中。

如果要改变你的transition的默认行为，以更好地控制它在不同的加载源（内存缓存，磁盘缓存，或uri）下是否被应用，你可以检查一下你的[``TransitionFactory``][21]中传递给[``build()``][24]方法的那个[``DataSource``][23]。

如需示例代码，请查看[``DrawableCrossFadeFactory``][25].

[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/transition/Transition.html
[2]: https://developer.android.com/reference/android/widget/ViewSwitcher.html
[3]: https://developer.android.com/reference/android/widget/ImageView.html
[4]: https://developer.android.com/reference/android/widget/ViewSwitcher.html#getNextView()
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestListener.html
[6]: https://developer.android.com/reference/android/widget/ViewAnimator.html#showNext()
[7]: https://developer.android.com/training/animation/crossfade.html
[8]: https://developer.android.com/reference/android/graphics/drawable/TransitionDrawable.html
[9]: https://developer.android.com/reference/android/graphics/drawable/TransitionDrawable.html#setCrossFadeEnabled(boolean)
[10]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/transition/DrawableCrossFadeFactory.html
[11]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/TransitionOptions.html#transition-com.bumptech.glide.request.transition.TransitionFactory-
[12]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/TransitionOptions.html
[13]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#transition-com.bumptech.glide.TransitionOptions-
[14]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html
[15]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/bitmap/BitmapTransitionOptions.html
[16]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/drawable/DrawableTransitionOptions.html
[17]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/GenericTransitionOptions.html
[18]: /glide/doc/options.html#transitionoptions
[19]: /glide/doc/targets.html#targets-and-automatic-cancellation
[20]: {{ site.url }}/glide/transitions#custom-transitions
[21]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/transition/TransitionFactory.html
[22]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/drawable/DrawableTransitionOptions.html#with-com.bumptech.glide.request.transition.TransitionFactory-
[23]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/DataSource.html
[24]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/transition/TransitionFactory.html#build-com.bumptech.glide.load.DataSource-boolean-
[25]: https://github.com/bumptech/glide/blob/8f22bd9b82349bf748e335b4a31e70c9383fb15a/library/src/main/java/com/bumptech/glide/request/transition/DrawableCrossFadeFactory.java#L35 
