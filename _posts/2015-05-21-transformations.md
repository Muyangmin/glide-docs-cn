---
layout: page
title: "变换"
category: doc
date: 2015-05-21 20:04:53
order: 6
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/transformations.html){:target="_blank"}

* TOC
{:toc}

### 关于变换

在Glide中，[Transformations][1]可以获取资源并修改它，然后返回被修改后的资源。通常变换操作是用来完成剪裁或对位图应用过滤器，但它也可以用于转换GIF动画，甚至自定义的资源类型。

### 内置类型

Glide提供了很多内置的变换，包括：

* [CenterCrop][4]
* [FitCenter][2]
* [CircleCrop][6]

### 应用

通过[RequestOptions][9]类可以应用变换：

#### 默认变换

```java
RequestOptions options = new RequestOptions();
options.centerCrop();

Glide.with(fragment)
    .load(url)
    .apply(options)
    .into(imageView);
```

大多数内置的变换都有静态的import，这是为API的流畅性考虑的。例如，你可以通过静态方法应用一个[FitCenter][2]变换：

```java
import static com.bumptech.glide.request.RequestOptions.fitCenterTransform;

Glide.with(fragment)
    .load(url)
    .apply(fitCenterTransform())
    .into(imageView);
```

如果你正在使用[生成的API][16]，那么这些变换方法已经被内联了，所以使用起来甚至更为轻松：

```java
GlideApp.with(fragment)
  .load(url)
  .fitCenter()
  .into(imageView);
```

可以查阅[Options][3]页来获得更多`RequestOption`的相关信息。

#### 多重变换

默认情况下，每个[``transform()``][17]调用，或任何特定转换方法(``fitCenter()``, ``centerCrop()``, ``bitmapTransform()`` etc)的调用都会替换掉之前的变换。

如果你想在单次加载中应用多个变换，请使用[``MultiTransformation``][18]类。

使用[generated API][16]:

```java
Glide.with(fragment)
  .load(url)
  .transform(new MultiTransformation(new FitCenter(), new YourCustomTransformation())
  .into(imageView);
```

请注意，你向 [``MultiTransformation``][18]的构造器传入变换参数的顺序，决定了这些变换的应用顺序。

### Glide中的特殊行为

#### 重用变换
``Transformation``的设计初衷是无状态的。因此，在多个加载中复用``Transformation``应当总是安全的。创建一次``Transformation``并在多个加载中使用它，通常是很好的实践。

#### ImageView的自动变换
在Glide中，当你为一个[ImageView][7]开始加载时，Glide可能会自动应用[FitCenter][2]或[CenterCrop][4]，这取决于view的[ScaleType][8]。如果`scaleType`是``CENTER_CROP``, Glide将会自动应用``CenterCrop``变换。如果`scaleType`为``FIT_CENTER`` 或 ``CENTER_INSIDE``，Glide会自动使用 ``FitCenter``变换。

当然，你总有权利覆写默认的变换，只需要一个带有``Transformation``集合的[RequestOptions][9] 即可。另外，你也可以通过使用[``dontTransform()``][10]确保不会自动应用任何变换。

#### 自定义资源
因为Glide 4.0 允许你指定你将解码的资源的父类型，你可能无法确切地知道将会应用何种变换。例如，当你使用[``asDrawable()``][11](或就是普通的``with()``，因为``asDrawable()``是默认情形)来加载Drawable资源时，你可能会得到[``BitmapDrawable``][12]子类，也有可能得到[``GifDrawable``][13] 子类。

为了确保你添加到``RequestOptions``中的任何变换都会被使用，Glide将``Transformation``添加到一个Map中保存，其Key为你提供变换的资源类型。当资源被成功解码时，Glide使用这个Map来取回对应的``Transformation``。

Glide可以将``Bitmap`` ``Transformation``应用到``BitmapDrawable``, ``GifDrawable``, 以及``Bitmap``资源上，因此通常你只需要编写和应用``Bitmap`` ``Transformation``。然而，如果你添加了额外的资源类型，你可能需要考虑派生[``RequestOptions``][15]类，并让你的资源类型能应用Glide内置的``Bitmap`` ``Transformation``。

[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/Transformation.html
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/bitmap/FitCenter.html
[3]: options.html
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/bitmap/CenterCrop.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/bitmap/CircleCrop.html
[7]: http://developer.android.com/reference/android/widget/ImageView.html
[8]: http://developer.android.com/reference/android/widget/ImageView.ScaleType.html
[9]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[10]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#dontTransform--
[11]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestManager.html#asDrawable--
[12]: http://developer.android.com/reference/android/graphics/drawable/BitmapDrawable.html
[13]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/gif/GifDrawable.html
[14]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#transform-java.lang.Class-com.bumptech.glide.load.Transformation-
[15]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[16]: {{ site.url }}/glide/doc/generatedapi.html
[17]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#transform-java.lang.Class-com.bumptech.glide.load.Transformation-
[18]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/MultiTransformation.html

