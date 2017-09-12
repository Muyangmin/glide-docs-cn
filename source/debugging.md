---
title: "调试"
---
原文链接：[点击查看](http://bumptech.github.io/glide/doc/debugging.html)

### 本地日志(Local Logs)
如果你拥有设备的访问权限，你可以使用``adb logcat``或你的IDE查看一些日志。你可以使用``adb shell setprop log.tag.<tag_name> <VERBOSE|DEBUG>``操作为任何下面提到的标签(`tag`))开启日志。VERBOSE级别的日志会显得更加冗余但包含更多有用的信息。根据你要查看的标签的不同，你可以把VERBOSE和DEBUG级别的信息都尝试一下，以决定哪个级别的信息是你最需要的。

#### Request errors
最高级别和最容易理解的日志都通过``Glide``标签打印。Glide标签将记录成功和失败的请求以及不同级别的详细信息，具体取决于日志级别。VERBOSE会被用于记录成功的请求，DEBUG则会打印出详细的错误信息。

你也可以通过手动调用[``setLogLevel(int)``][1]方法控制Glide标签的冗余度。``setLogLevel``允许你--举个栗子--在开发构建(developer builds)时启用更加冗余的日志，而在发布(release builds)构建时则关闭它们。

#### Unexpected cache misses
``Engine``标签会详细记录请求被填充的全过程，并包括用于存储相应资源的完整内存缓存键。如果你正在尝试调试“内存中明明有这个图片，为什么没在另一个地方用到”的问题，那么``Engine``标签可以让你直观地比较两者的缓存键的区别。

对于每一个开始了的请求，``Engine``标签将会记录这个请求将会从哪个地方加载完成：缓存，活动资源，已存在的加载过程，或者一个新的加载过程。缓存：意味着这个资源暂时没有被用到，但是在内存缓存中可用。活动资源：表示这个资源正在被另一个``Target``使用，一般是在一个``View``中。已存在的加载过程：表示这个资源虽然现在在内存中不可用，但是另一个``Target``已经在早先发起了对同一个资源的请求，并且这个请求还在处理中。最后，新的加载过程表示这个资源既不在内存中，也没有被其他地方请求过，那么这将触发一次新的加载。

#### 图片和本地日志丢失
在某些情况下，你可能会发现某个图片永远不会加载出来，而且这个请求甚至还没有``Glide``标签和``Engine``标签的日志。这可能有以下一些原因。


##### 启动请求失败(Failing to start the request)
请检查你是否为你的请求调用了[``into()``][2] 或者 [``submit()``][3]方法。很显然，如果你忘记了调用这两个方法，Glide不会认为你已经要求开始加载。

##### 未指定尺寸（Missing Size）
如果你确信你调用了[``into()``][2]、[``submit()``][3]之一，并且仍然没有看到日志，那么最可能的解释是，Glide无法决定你即将加载资源的``View``或``Target``的尺寸。

###### 自定义Target(Custom Targets)
如果你正在使用一个自定义的``Target``，请确保你实现了[``getSize``][4]方法并使用了非零的宽高来调用指定的回调方法，或者继承自一个已经为你实现了这个方法的``Target``，例如[``ViewTarget``][5]。

###### Views
如果你只是在往一个``View``中加载资源，那么最大的可能是你的这个view要么还没有被布局(layout)过，要么被指定了零宽或高。View的可见性被设置为``View.GONE``或它并没有被attach，都会导致view不会被layout。如果View和/或它们的父控件被以特定方式组合`wrap_content`和`match_parent`来作为宽高，则view可能会收到一个无效的或为0的宽高值。你可以试验一下，将你的view设定为非0的尺寸，或在请求时使用[``override(int, int)`` API][6]来为Glide传入一个特定的尺寸。

#### RequestListener and custom logs
如果你想使用编程的办法跟踪成功和失败信息、跟踪应用中的整体缓存命中率，或增加对本地日志的控制，你可以使用[``RequestListener``][7] 接口。``RequestListener``可以通过[``RequestBuilder#listener()``][8]方法来添加到单独的加载请求中。下面是一个使用示例：

```java
Glide.with(fragment)
   .load(url)
   .listener(new RequestListener() {
       @Override
       boolean onLoadFailed(@Nullable GlideException e, Object model,
           Target<R> target, boolean isFirstResource) {
         // Log errors here.
       }

       @Override
       boolean onResourceReady(R resource, Object model, Target<R> target,
           DataSource dataSource, boolean isFirstResource) {
         // Log successes here or use DataSource to keep track of cache hits and misses.
       }
    })
    .into(imageView);
```

To save object allocations, you can re-use the same ``RequestListener`` for multiple loads.
为减少对象分配起见，你可以为多个加载重用相同的``RequestListener``。


[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/GlideBuilder.html#setLogLevel-int-
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#into-android.widget.ImageView-
[3]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#submit-int-int-
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/target/Target.html#getSize-com.bumptech.glide.request.target.SizeReadyCallback-
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/target/ViewTarget.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#override-int-int-
[7]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestListener.html
[8]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#listener-com.bumptech.glide.request.RequestListener-
