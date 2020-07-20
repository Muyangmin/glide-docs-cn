---
layout: page
title: "调试"
category: doc
date: 2017-01-09 07:14:59
order: 12 
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/debugging.html){:target="_blank"}

* TOC
{:toc}


### 本地日志(Local Logs)
如果你拥有设备的访问权限，你可以使用 ``adb logcat`` 或你的 IDE 查看一些日志。你可以使用 ``adb shell setprop log.tag.<tag_name> <VERBOSE|DEBUG>`` 操作为任何下面提到的标签(`tag`))开启日志。VERBOSE 级别的日志会显得更加冗余但包含更多有用的信息。根据你要查看的标签的不同，你可以把 VERBOSE 和 DEBUG 级别的信息都尝试一下，以决定哪个级别的信息是你最需要的。

#### 请求错误
最高级别和最容易理解的日志都通过 ``Glide`` 标签打印。Glide 标签将记录成功和失败的请求以及不同级别的详细信息，具体取决于日志级别。VERBOSE 会被用于记录成功的请求，DEBUG则会打印出详细的错误信息。

你也可以通过手动调用 [``setLogLevel(int)``][1] 方法控制Glide标签的冗余度。``setLogLevel`` 允许你--举个栗子--在开发构建(developer builds)时启用更加冗余的日志，而在发布(release builds)构建时则关闭它们。

#### (非预期的)缓存未命中 (`Miss`)
关于 Glide 缓存如何工作，请查阅 [缓存页][13]。

``Engine`` 标签会详细记录请求被填充的全过程，并包括用于存储相应资源的完整内存缓存键。如果你正在尝试调试“内存中明明有这个图片，为什么没在另一个地方用到”的问题，那么 ``Engine`` 标签可以让你直观地比较两者的缓存键的区别。

对于每一个开始了的请求，``Engine`` 标签将会记录这个请求将会从哪个地方加载完成：缓存，活动资源，已存在的加载过程，或者一个新的加载过程。缓存：意味着这个资源暂时没有被用到，但是在内存缓存中可用。活动资源：表示这个资源正在被另一个 ``Target`` 使用，一般是在一个 ``View`` 中。已存在的加载过程：表示这个资源虽然现在在内存中不可用，但是另一个``Target``已经在早先发起了对同一个资源的请求，并且这个请求还在处理中。最后，新的加载过程表示这个资源既不在内存中，也没有被其他地方请求过，那么这将触发一次新的加载。

#### 请求监听器与定制日志
如果你想使用编程的办法跟踪成功和失败信息、跟踪应用中的整体缓存命中率，或增加对本地日志的控制，你可以使用 [``RequestListener``][7] 接口。 ``RequestListener`` 可以通过 [``RequestBuilder#listener()``][8] 方法来添加到单独的加载请求中。下面是一个使用示例：

```java
Glide.with(fragment)
   .load(url)
   .listener(new RequestListener() {
       @Override
       boolean onLoadFailed(@Nullable GlideException e, Object model,
           Target<R> target, boolean isFirstResource) {
         // Log the GlideException here (locally or with a remote logging framework):
         Log.e(TAG, "Load failed", e);

         // You can also log the individual causes:
         for (Throwable t : e.getRootCauses()) {
           Log.e(TAG, "Caused by", t);
         }
         // Or, to log all root causes locally, you can use the built in helper method:
         e.logRootCauses(TAG);

         return false; // Allow calling onLoadFailed on the Target.
       }

       @Override
       boolean onResourceReady(R resource, Object model, Target<R> target,
           DataSource dataSource, boolean isFirstResource) {
         // Log successes here or use DataSource to keep track of cache hits and misses.

         return false; // Allow calling onResourceReady on the Target.
       }
    })
    .into(imageView);
```

请注意，每个 GlideException 都有多个 ``Throwable`` root cause。在 Glide 中可能有有任意多的方法使得 注册组件(``ModelLoader``, ``ResourceDecoder``, ``Encoder`` 等)作用于从给定的模型（URL, File 等）加载给定的资源 (``Bitmap``, ``GifDrawable`` 等)。每个 ``Throwable`` root cause 描述了一个特定的 Glide 组件组合为什么失败。理解某个特定请求为何失败可能需要检查所有的 root cause。

然而，你也可能会发现某个单一的 root cause 比其他的要重要一些。例如你正在加载 URL 并试图找出特定的 HttpException (它意味着你的加载是由于一个网络错误而失败)，你可以遍历所有的 root cause 并使用 ``instanceof`` 来检查其类型：

```java
for (Throwable t : e.getRootCauses()) {
  if (t instanceof HttpException) {
    Log.e(TAG, "Request failed due to HttpException!", t);
    break;
  }
}
```

当然你也可以使用类似的迭代过程和 ``instanceof`` 操作符来检查 Http 错误之外其他你关心的异常类型。

为减少对象分配起见，你可以为多个加载重用相同的``RequestListener``。

#### 图片和本地日志丢失
在某些情况下，你可能会发现某个图片永远不会加载出来，而且这个请求甚至还没有``Glide``标签和``Engine``标签的日志。这可能有以下一些原因。


##### 启动请求失败(Failing to start the request)
请检查你是否为你的请求调用了 [``into()``][2] 或者 [``submit()``][3] 方法。很显然，如果你忘记了调用这两个方法，Glide 不会认为你已经要求开始加载。

##### 未指定尺寸（Missing Size）
如果你确信你调用了 [``into()``][2] 、 [``submit()``][3] 之一，并且仍然没有看到日志，那么最可能的解释是，Glide 无法决定你即将加载资源的 ``View`` 或 ``Target`` 的尺寸。

###### 自定义Target(Custom Targets)
如果你正在使用一个自定义的 ``Target`` ，请确保你实现了 [``getSize``][4] 方法并使用了非零的宽高来调用指定的回调方法，或者继承自一个已经为你实现了这个方法的 ``Target`` ，例如 [``ViewTarget``][5]。

###### Views
如果你只是在往一个 ``View`` 中加载资源，那么最大的可能是你的这个view要么还没有被布局(layout)过，要么被指定了零宽或高。View 的可见性被设置为 ``View.GONE`` 或它并没有被 attach ，都会导致 view 不会被 layout 。如果 View 和/或它们的父控件被以特定方式组合 `wrap_content` 和 `match_parent` 来作为宽高，则 view 可能会收到一个无效的或为 0 的宽高值。你可以试验一下，将你的 view 设定为非0的尺寸，或在请求时使用 [``override(int, int)`` API][6] 来为 Glide 传入一个特定的尺寸。

### Out of memory 错误
几乎所有的 OOM 错误都是因为宿主应用出了问题，而不是 Glide 本身。
应用里两种常见的 OOM 错误分别是：
1. 过大的内存分配 (Excessively large allocations)
2. 内存泄露(Memory leaks, 被分配的内存没有被释放)

#### 过大的内存分配
如果在打开一个单独页面或加载一个单独图片导致了 OOM , 那么你的应用可能在加载一个不必要的大图。

使用 Bitmap 显示一张图片所需的内存数量为 宽(width) * 高(height) * 每像素字节数(bytes per pixel)。 每像素字节数取决于显示图片所使用的 ``Bitmap.Config``，但通常对于 ``ARGB_8888`` 的位图来说，每个像素即为四个字节。因此，即使是一张普通的 1080P 图片也需要 8MB 内存。图片越大，所需要的内存就越多，因此一个 12M 像素的图片会要求相当庞大的 48MB 内存。

Glide 会将图片自动下采样 (downsample)，这是基于 ``Target``，``ImageView`` 或 ``override()`` 提供的尺寸。 如果你在 Glide 中看到了特别大的内存分配，通常意味着你的 ``Target`` 或 ``override()`` 提供的尺寸太大，或你使用了 ``Target.SIZE_ORIGINAL`` 而又恰好碰上了一个大图。

要解决这种过大的内存分配，请避免使用 ``Target.SIZE_ORIGINAL`` 并确保你的 ``ImageView`` 尺寸或你通过 ``override()`` 方法提供给 Glide 的尺寸是合理的。

#### 内存泄露
如果在你的应用中持续重复特定步骤会逐步增加你应用的内存使用并最终导致 OOM ，你可能有内存泄露。

[Android 官方文档][10] 中有很多关于追踪和调试内存使用的有用信息。为了调查内存泄露，你几乎肯定需要 [捕捉一个 heap dump][11] 并查看 Fragments, Activities 和以及其他不再被使用但却仍被持有的对象。

要修复内存泄露，你需要对已销毁的 ``Fragment`` 或 ``Activity`` 在生命周期的合适时机移除对它们的引用，以避免持有过多的对象。使用 heap dump 来帮助查找你应用中持有其他内存的方式并在找到后移除不必要的引用。通常你可以从列出对 Bitmap 对象（使用 [MAT][12] 或其他内存分析器)的最短路径（不含弱引用）开始，然后寻找可疑的引用链。你还可以在你的内存分析器中搜索 ``Activity`` 和 ``Fragment``，以确保每个 ``Activity`` 不超过一个实例，并且 ``Fragment`` 的实例数目也在期望范围内。

### 其他常见问题

#### "You can't start or clear loads in RequestListener or Target callbacks"

如果你尝试在一个 ``Target`` 或 ``RequestListener`` 里的 ``onResourceReady`` 或 ``onLoadFailed`` 中开始一次新的加载，Glide 将会抛出一个异常。之所以抛出这个异常，是因为要处理和回收这种在通知过程中的 (notifying) 加载对我们来说是一个巨大的挑战。

好在这个问题很好解决。从 Glide 4.3.0 开始，你可以很轻松地使用 [``.error()``][14] 方法。这个方法接受一个任意的 [``RequestBuilder``][15]，它会且只会在主请求失败时开始一个新的请求：

```java
Glide.with(fragment)
  .load(url)
  .error(Glide.with(fragment)
     .load(fallbackUrl))
  .into(imageView);
```

对于 Glide 4.3.0 以前的版本，你也可以使用一个 Android [``Handler``][16] 来 post 一个 Runnable 给你的请求：

```java
private final Handler handler = new Handler();
...

Glide.with(fragment)
  .load(url)
  .listener(new RequestListener<Drawable>() {
      ...

      @Override
      public boolean onLoadFailed(@Nullable GlideException e, Object model, 
          Target<Drawable> target, boolean isFirstResource) {
        handler.post(new Runnable() {
            @Override
            public void run() {
              Glide.with(fragment)
                .load(fallbackUrl)
                .into(imageView);
            }
        });
      }
  )
  .into(imageView);
```

#### "cannot resolve symbol 'GlideApp'"

当使用 Generated API 时，你可能会遇到一些错误从而导致无法生成 Glide API。有时这些错误与你的 [设置][17] 有关，但有时可能完全无关。  
不相关的错误经常会被大量的非 root cause 的错误消息掩盖。有可能因错误过多而使得你无法在构建日志中找出 root cause。如果遇到这种情况而且你正在使用 Gradle，可以尝试添加以下代码以增加 Gradle 打印的错误信息数量：

```groovy
allprojects {
  gradle.projectsEvaluated {
    tasks.withType(JavaCompile) {
        options.compilerArgs << "-Xmaxerrs" << "1000"
    }
  }
}
```

参见: 

  *    [https://github.com/bumptech/glide/issues/1945](https://github.com/bumptech/glide/issues/1945)
  *    [https://stackoverflow.com/questions/3115537/java-compilation-errors-limited-to-100/35707023#35707023](https://stackoverflow.com/questions/3115537/java-compilation-errors-limited-to-100/35707023#35707023)
    
[1]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/GlideBuilder.html#setLogLevel-int-  
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#into-android.widget.ImageView-  
[3]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#submit-int-int-  
[4]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/target/Target.html#getSize-com.bumptech.glide.request.target.SizeReadyCallback-  
[5]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/target/ViewTarget.html  
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#override-int-int-  
[7]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestListener.html  
[8]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#listener-com.bumptech.glide.request.RequestListener-  
[9]: https://github.com/bumptech/glide/blob/6b137c2b1d4b2ab187ea2aa56834dea039daa090/library/src/main/java/com/bumptech/glide/load/engine/Engine.java#L33  
[10]: https://developer.android.com/studio/profile/investigate-ram.html  
[11]: https://developer.android.com/studio/profile/investigate-ram.html#HeapDump  
[12]: http://www.eclipse.org/mat/  
[13]: {{ site.baseurl }}/doc/caching.html  
[14]: http://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/RequestBuilder.html#error-com.bumptech.glide.RequestBuilder-  
[15]: http://bumptech.github.io/glide/javadocs/430/com/bumptech/glide/RequestBuilder.html  
[16]: https://developer.android.com/reference/android/os/Handler.html  
[17]: {{ site.baseurl }}/doc/generatedapi.html
