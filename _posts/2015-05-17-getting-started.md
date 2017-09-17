---
layout: page
title: "开始使用"
category: doc
date: 2017-09-17 10:54:00
order: 2
disqus: 1
translators: [Muyangmin, vincgao]
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/getting-started.html){:target="_blank"}

* TOC
{:toc}
### 基本用法

多数情况下，使用Glide加载图片非常简单，一行代码足以：

```java
Glide.with(fragment)
    .load(myUrl)
    .into(imageView);
```

取消加载同样很简单：

```java
Glide.with(fragment).clear(imageView);
```

尽管及时取消不必要的加载是很好的实践，但这并不是必须的操作。实际上，当 [``Glide.with()``][1] 中传入的 Activity 或 Fragment 实例销毁时，Glide 会自动取消加载并回收资源。

### 在 Application 模块中的使用

在 Application 模块中，可创建一个添加有 `@GlideModule` 注解，继承自 `AppGlideModule` 的类。此类可生成出一个流式 API，内联了多种选项，和集成库中自定义的选项：

```java
package com.example.myapp;

import com.bumptech.glide.annotation.GlideModule;
import com.bumptech.glide.module.AppGlideModule;

@GlideModule
public final class MyAppGlideModule extends AppGlideModule {}
```

生成的 API 默认名为 `GlideApp` ，与 [``AppGlideModule``][6] 的子类包名相同。在 Application 模块中将 ``Glide.with()`` 替换为 ``GlideApp.with()``，即可使用该 API 去完成加载工作。

```java
GlideApp.with(fragment)
   .load(myUrl)
   .placeholder(placeholder)
   .fitCenter()
   .into(imageView);
```

可以访问 Glide 的 [generated API][7] 页面来获得更多信息。 

### 在 ListView 和 RecyclerView 中的使用

在 ListView 或 RecyclerView 中加载图片的代码和在单独的 View 中加载完全一样。Glide 已经自动处理了 View 的复用和请求的取消：

```java
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    String url = urls.get(position);
    Glide.with(fragment)
        .load(url)
        .into(holder.imageView);
}
```

对 url 进行 null 检验并不是必须的，如果 url 为 null，Glide 会清空 View 的内容，或者显示 [placeholder Drawable][2] 或 [fallback Drawable][3] 的内容。

Glide 唯一的要求是，对于任何可复用的 ``View`` 或 [``Target``][5] ，如果它们在之前的位置上，用 Glide 进行过加载操作，那么在新的位置上要去执行一个新的加载操作，或调用 [``clear()``][4] API 停止 Glide 的工作。

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

对 ``View`` 调用 [``clear()``][4] 或 ``into(View)``，表明在此之前的加载操作会被取消，并且在方法调用完成后，Glide 不会改变 view 的内容。如果你忘记调用 [``clear()``][4]，而又没有开启新的加载操作，那么就会出现这种情况，你已经为一个 View 设置好了一个 ``Drawable``，但之前在该位置上使用 Glide 进行过加载操作，加载完毕后可能会将这个 View 改回成原来的内容。

这里的代码以 RecyclerView 的使用为例，但规则同样适用于 ListView。

[1]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/Glide.html#with-android.app.Fragment-
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#placeholder-int-
[3]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#fallback-int-
[4]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestManager.html#clear-com.bumptech.glide.request.target.Target-
[5]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/target/Target.html
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[7]: {{ site.baseurl }}/doc/generatedapi.html
