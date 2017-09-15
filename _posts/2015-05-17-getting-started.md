---
layout: page
title: "开始使用"
category: doc
date: 2015-05-17 17:01:02
order: 2
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/getting-started.html){:target="_blank"}

### 基本用法

很多情况下，使用Glide加载图片非常简单，简单到只用一行代码：

```java
Glide.with(fragment)
    .load(myUrl)
    .into(imageView);
```

取消加载同样很简单：

```java
Glide.with(fragment).clear(imageView);
```

尽管及时取消不必要的加载是很好的实践，但是你并不需要做这样的辛苦活。实际上，当你通过 [``Glide.with()``][1] 方法传入Glide的Activity或Fragment被销毁的时候，Glide会自动取消加载并回收资源。

### 应用程序

应用可以添加一个合适的被标记为 [``AppGlideModule``][6] 的实现类来生成一个内联了大部分选项的API，包括那些已经在集成库中定义过的：

```java
package com.example.myapp;

import com.bumptech.glide.annotation.GlideModule;
import com.bumptech.glide.module.AppGlideModule;

@GlideModule
public final class MyAppGlideModule extends AppGlideModule {}
```

API会在[``AppGlideModule``][6]的相同包下被生成出来，默认命名为``GlideApp``。将开始加载的代码从 ``Glide.with()``改成``GlideApp.with()``，应用就可以使用新的API啦。

```java
GlideApp.with(fragment)
   .load(myUrl)
   .placeholder(placeholder)
   .fitCenter()
   .into(imageView);
```

可以访问Glide的 [generated API][7] 页面来获得更多信息。 

### ListView 和 RecyclerView

在ListView或RecyclerView中加载图片的代码和在单独的View中完全一样。Glide已经自动处理了View的重用和请求的取消操作：

```java
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    String url = urls.get(position);
    Glide.with(fragment)
        .load(url)
        .into(holder.imageView);
}
```

你也大可不必对你的url做null检查，如果url为空，Glide会擦除View的内容，或者显示你之前设置的任何 [placeholder Drawable][2] 或 [fallback Drawable][3] 。

Glide唯一的要求就是，对于任何可复用的``View`` [``Target``][5], 如果你在之前的位置可能启动了一个加载请求的话，对于新的位置，要么开始一个新的加载过程，要么使用 [``clear()``][4] API来显式地清空内容。

```java
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    if (isImagePosition(position)) {
        String url = urls.get(position);
        Glide.with(fragment)
            .load(url)
            .into(holder.imageView);
    } else {
        Glide.with(fragment).clear(holder.imageView);
        holder.imageView.setImageDrawable(specialDrawable);
    }
}
```


通过对``View``执行[``clear()``][4] 或 ``into(View)``，之前的加载都会被取消，并且在方法调用完成后，Glide将保证不会改变view的内容。如果你忘记调用[``clear()``][4]，而又没有开启新的加载过程，那么你之前对同一个View执行的加载就可能在你设置了特定的``Drawable``之后完成，并将``View``的内容改成较旧的图片。

虽然这里的示例代码都是针对RecyclerView, 但是同样的原则也适用于ListView。

[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/Glide.html#with-android.app.Fragment-
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#placeholder-int-
[3]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#fallback-int-
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestManager.html#clear-com.bumptech.glide.request.target.Target-
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/target/Target.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[7]: {{ site.url }}/glide/doc/generatedapi.html
