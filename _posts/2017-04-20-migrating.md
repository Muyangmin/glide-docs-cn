---
layout: page
title: "从v3迁移到v4"
category: doc
date: 2017-04-20 07:13:46
order: 14
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/migrating.html){:target="_blank"}

* TOC
{:toc}

## 选项(Options)
Glide v4 中的一个比较大的改动是Glide库处理选项(``centerCrop()``, ``placeholder()`` 等)的方式。在 v3 版本中，选项由一系列复杂的异构建造者(multityped builders)单独处理。在新版本中，由一个单一类型的唯一一个建造者接管一系列选项对象。Glide 的[generated API][11]进一步简化了这个操作：它会合并传入建造者的选项对象和任何已包含的集成库里的选项，以生成一个流畅的 API。

### RequestBuilder
对于这类方法：
```java
listener()
thumbnail()
load()
into()
```

在 Glide v4 版本中，只存在一个 [``RequestBuilder``][5] 对应一个你正在试图加载的类型(``Bitmap``, ``Drawable``, ``GifDrawable`` 等)。 ``RequestBuilder`` 可以直接访问对这个加载过程有影响的选项，包括你想加载的数据模型（url, uri等），可能存在的[``缩略图``][6]请求，以及任何的[``监听器``][7]。``RequestBuilder``也是你使用 [``into()``][8] 或者 [``preload()``][9] 方法开始加载的地方：

```java
RequestBuilder<Drawable> requestBuilder = Glide.with(fragment)
    .load(url);

requestBuilder
    .thumbnail(Glide.with(fragment)
        .load(thumbnailUrl))
    .listener(requestListener)
    .load(url)
    .into(imageView);
```

### 请求选项

对于这类方法：
```java
centerCrop()
placeholder()
error()
priority()
diskCacheStrategy()
```

大部分选项被移动到了一个单独的称为 [``RequestOptions``][10] 的对象中，

```
RequestOptions options = new RequestOptions()
    .centerCrop()
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .priority(Priority.HIGH);
```

``RequestOptions`` 允许你一次指定一系列的选项，然后对多个加载重用它们：

```java
RequestOptions myOptions = new RequestOptions()
    .fitCenter()
    .override(100, 100);

Glide.with(fragment)
    .load(url)
    .apply(myOptions)
    .into(drawableView);

Glide.with(fragment)
    .asBitmap()
    .apply(myOptions)
    .load(url)
    .into(bitmapView);
```

### 变换
Glide v4 里的 [``Transformations``][28] 现在会替换之前设置的任何变换。在 Glide v4 中，如果你想应用超过一个的 [``Transformation``][28]，你需要使用 [``transforms()``][29] 方法：

```java
Glide.with(fragment)
  .load(url)
  .apply(new RequestOptions().transforms(new CenterCrop(), new RoundedCorners(20)))
  .into(target);
```

或使用 [generated API][24]:

```java
GlideApp.with(fragment)
  .load(url)
  .transforms(new CenterCrop(), new RoundedCorners(20))
  .into(target);
```

### 解码格式
在 Glide v3， 默认的 [``DecodeFormat``][30] 是 [``DecodeFormat.PREFER_RGB_565``][31]，它将使用 [``Bitmap.Config.RGB_565``]，除非图片包含或可能包含透明像素。对于给定的图片尺寸，``RGB_565`` 只使用 [``Bitmap.Config.ARGB_8888``] 一半的内存，但对于特定的图片有明显的画质问题，包括条纹(banding)和着色(tinting)。为了避免``RGB_565``的画质问题，Glide 现在默认使用 ``ARGB_8888``。结果是，图片质量变高了，但内存使用也增加了。

要将 Glide v4 默认的 [``DecodeFormat``][30] 改回 [``DecodeFormat.PREFER_RGB_565``][31]，请在 [``AppGlideModule``][2] 中应用一个 ``RequestOption``：
```java
@GlideModule
public final class YourAppGlideModule extends GlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDefaultRequestOptions(new RequestOptions().format(DecodeFormat.PREFER_RGB_565));
  }
}
```

关于使用 [``AppGlideModules``][2] 的更多信息，请查阅 [配置][4] 页面。请注意，为了让 Glide 发现你的 [``AppGlideModule``][2] 实现，你必须确保添加了对 Glide 的注解解析器的依赖。关于如何设置这个库的更多信息，请查看 [下载和设置][34]。

### 过渡选项

对于这类方法：
```java
crossFade()
animate()
```

控制从占位符到图片和/或缩略图到全图的交叉淡入和其他类型变换的选项，被移动到了 [``TransitionOptions``][13] 中。

要应用过渡（之前的动画），请使用下列选项中符合你请求的资源类型的一个：

* [``GenericTransitionOptions``][14]
* [``DrawableTransitionOptions``][15]
* [``BitmapTransitionOptions``][16]

如果你想移除任何默认的过渡，可以使用 ``TransitionOptions.dontTransition()``][17] 。

过渡动画通过 [``RequestBuilder``][5] 应用到请求上：

```java
Glide.with(fragment)
    .load(url)
    .transition(withCrossFade(R.anim.fade_in, 300));
```

#### 交叉淡入  (Cross fade)
不同于 Glide v3，Glide v4 将**不会**默认应用交叉淡入或任何其他的过渡效果。每个请求必须手动应用过渡。

要为一个特定的加载应用一个交叉淡入变换效果，你可以使用：

```java
import static com.bumptech.glide.load.resource.drawable.DrawableTransitionOptions.withCrossFade;

Glide.with(fragment)
  .load(url)
  .transition(withCrossFade())
  .into(imageView);
```

或:

```java
Glide.with(fragment)
  .load(url)
  .transition(
      new DrawableTransitionOptions
        .crossFade())
  .into(imageView);
```

### Generated API

为了让使用 Glide v4 更简单轻松，Glide 现在也提供了一套可以为应用定制化生成的 API。应用可以通过包含一个标记了 [``AppGlideModule``][[2] 的实现来访问生成的 API。如果你不了解这是怎么工作的，可以查看 [Generated API][11] 。

Generated API添加了一个 ``GlideApp`` 类，该类提供了对 ``RequestBuilder`` 和 ``RequestOptions`` 子类的访问。``RequestOptions`` 的子类包含了所有 ``RequestOptions`` 中的方法，以及 [``GlideExtensions``][12] 中定义的方法。``RequestBuilder`` 的子类则提供了生成的 ``RequestOptions`` 中所有方法的访问，而不需要你再手动调用 ``apply`` 。举个例子：

在没有使用 Generated API 时，请求大概长这样：

```java
Glide.with(fragment)
    .load(url)
    .apply(centerCropTransform()
        .placeholder(R.drawable.placeholder)
        .error(R.drawable.error)
        .priority(Priority.HIGH))
    .into(imageView);
```

使用 Generated API，``RequestOptions`` 的调用可以被内联：

```java
GlideApp.with(fragment)
    .load(url)
    .centerCrop()
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .priority(Priority.HIGH)
    .into(imageView);
```

你仍然可以使用生成的 ``RequestOptions`` 子类来应用相同的选项到多次加载中；但生成的 ``RequestBuilder`` 子类可能在多数情况下更为方便。

## 类型(Type)与目标(Target)

### 选择资源类型

Glide 允许你指定你想加载的资源类型。如果你指定了一个超类型，Glide 会尝试加载任何可用的子类型。比如，如果你请求的是 Drawable ，Glide 可能会加载一个 BitmapDrawable 或一个 GifDrawable 。而如果你请求的是一个 GifDrawable ，要么会加载出一个 GifDrawable，要么报错--只要图片不是 GIF 的话（即使它凑巧是一个完全有效的图片也是如此）。

默认请求的类型是 Drawable：

```java
Glide.with(fragment).load(url)
```
  
如果要明确指定请求 Bitmap：

```java
Glide.with(fragment).asBitmap()
```

如果要创建一个文件路径（本地图片的最佳选项）：

```java
Glide.with(fragment).asFile()
```

如果要下载一个远程文件到缓存然后创建文件路径：

```java
Glide.with(fragment).downloadOnly()
// or if you have the url already:
Glide.with(fragment).download(url);
```

### Drawables

Glide v3 版本中的 ``GlideDrawable`` 类已经被移除，支持标准的Android [``Drawable``][18]。 ``GlideBitmapDrawable`` 也已经被删除，由 [``BitmapDrawable``][19] 代替之。

如果你想知道某个 Drawable 是否是动画(animated)，可以检查它是否为 [``Animatable``][20] 的实例。

```java
boolean isAnimated = drawable instanceof Animatable;
```

### Targets

``onResourceReady`` 方法的签名做了一些修改。例如，对于 ``Drawables``:

```java
onResourceReady(GlideDrawable drawable, GlideAnimation<? super GlideDrawable> anim) 
```

现在改为:

```java
onResourceReady(Drawable drawable, Transition<? super Drawable> transition);
```

类似地, ``onLoadFailed`` 的签名也有一些变动：
```java
onLoadFailed(Exception e, Drawable errorDrawable)
```

改为：

```java
onLoadFailed(Drawable errorDrawable)
```

如果你想要获得更多导致加载失败的错误信息，你可以使用 [``RequestListener``][21] 。


#### 取消请求

``Glide.clear(Target)`` 方法被移动到了 [``RequestManager``][22] 中:

```java
Glide.with(fragment).clear(target)
```

使用 ``RequestManager`` 清除之前由它启动的加载过程，通常能提高性能，虽然这并不是强制要求的。Glide v4 会为每一个 Activity 和 Fragment 跟踪请求，所以你需要在合适的层级去清除请求。


## 配置
在 Glide v3 中，配置使用一个或多个 [``GlideModule``][1] 来完成。而在 Glide v4 中，配置改为使用一个类似但稍微复杂的系统来完成。

关于这个新系统的细节，可以查看[配置][4]页面。

### 应用程序

在早期版本中使用了一个 [``GlideModule``][1] 的应用，可以将它转换为一个 [``AppGlideModule``][2] 。

在 Glide v3 中，你可能会有一个像这样的 ``GlideModule`` ：

```java
public class GiphyGlideModule implements GlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setMemoryCache(new LruResourceCache(10 * 1024 * 1024));
  }

  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Api.GifResult.class, InputStream.class, new GiphyModelLoader.Factory());
  }
}
```

在 Glide v4 中，你需要将其转换成一个 ``AppGlideModule`` ，它看起来像这样：

```java
@GlideModule
public class GiphyGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setMemoryCache(new LruResourceCache(10 * 1024 * 1024));
  }

  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Api.GifResult.class, InputStream.class, new GiphyModelLoader.Factory());
  }
}
```

请注意，``@GlideModule`` 注解不能省略。
 
如果你的应用拥有多个 ``GlideModule``，你需要把其中一个转换成 ``AppGlideModule``，剩下的转换成 [``LibraryGlideModule``][3] 。除非存在``AppGlideModule`` ，否则程序不会发现 ``LibraryGlideModule`` ，因此您不能仅使用 ``LibraryGlideModule`` 。

### 程序库

拥有一个或多个 ``GlideModule`` 的程序库应该使用 [``LibraryGlideModule``][3] 。程序库不应该使用 [``AppGlideModule``][2] ，因为它在一个应用里只能有一个。因此，如果你试图在程序库里使用它，将不仅会妨碍这个库的用户设置自己的选项，还会在多个程序库都这么做时造成冲突。

例如，v3 版本中 Volley 集成库的 ``GlideModule`` ：

```java
public class VolleyGlideModule implements GlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    // Do nothing.
  }

  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new VolleyUrlLoader.Factory(context));
  }
}
```

在 v4 版本中可以转换成为一个 ``LibraryGlideModule`` ：

```java
@GlideModule
public class VolleyLibraryGlideModule extends LibraryGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new VolleyUrlLoader.Factory(context));
  }
}
```

### 清单解析

为了简化迁移过程，尽管清单解析和旧的 [``GlideModule``][1] 接口已被废弃，但它们在 v4 版本中仍被支持。``AppGlideModule``，``LibraryGlideModule``，与已废弃的 ``GlideModule`` 可以在一个应用中共存。

然而，为了避免检查元数据的性能天花板（以及相关的 bugs ），你可以在迁移完成后禁用掉清单解析，在你的 ``AppGlideModule`` 中复写一个方法：

```java
@GlideModule
public class GiphyGlideModule extends AppGlideModule {
  @Override
  public boolean isManifestParsingEnabled() {
    return false;
  }

  ...
}
```

### ``using()``, ModelLoader, StreamModelLoader.

#### ModelLoader

[``ModelLoader``][26] API 在 v4 版本中仍然存在，并且它的设计目标仍然和它在 v3 中一样，但有一些细节变化。

第一个细节，``ModelLoader`` 的子类型如 ``StreamModelLoader`` ，现在已没有存在的必要，用户可以直接实现 ``ModelLoader`` 。例如，一个``StreamModelLoader<File>`` 类现在可以通过 ``ModelLoader<File, InputStream>`` 的方式来实现和引用。


第二， ``ModelLoader`` 现在并不直接返回 ``DataFetcher``，而是返回 [``LoadData``][27] 。[``LoadData``] 是一个非常简单的封装，包含一个磁盘缓存键和一个 ``DataFetcher`` 。

第三， ``ModelLoaders`` 有一个 ``handles()`` 方法，这使你可以为同一个类型参数注册超过一个的 ModelLoader 。

将一个 ``ModelLoader`` 从 v3 API转换到 v4 API ，通常是很简单直接的。如果你在你的 v3  ``ModelLoader`` 中只是简单滴返回一个 ``DataFetcher`` ：

```java
public final class MyModelLoader implements StreamModelLoader<File> {

  @Override
  public DataFetcher<InputStream> getResourceFetcher(File model, int width, int height) {
    return new MyDataFetcher(model);
  }
}
```

那么你在 v4 替代类上需要做的仅仅只是封装一下这个 data fetcher ：

```java
public final class MyModelLoader implements ModelLoader<File, InputStream> {

  @Override
  public LoadData<InputStream> buildLoadData(File model, int width, int height,
      Options options) {
    return new LoadData<>(model, new MyDataFetcher(model));
  }

  @Override
  public void handles(File model) {
    return true;
  }
}
```

请注意，除了 ``DataFetcher`` 之外，模型也被传递给 ``LoadData`` 作为缓存键的一部分。这个规则为某些特殊场景提供了更多对磁盘缓存键的控制。大部分实现可以直接将 model 传入 ``LoadData`` ，就像上面这样。

如果你仅仅是想为某些 model（而不是所有）使用你的 ModelLoader，你可以在你尝试加载 model 之前使用 ``handles()`` 方法来检查它。如果你从 ``handles`` 方法中返回了 ``false`` ，那么你的 ``ModelLoader`` 将不能加载指定的 model ，即使你的 ``ModelLoader`` 类型 (在这个例子里是 ``File`` 和 ``InputStream``) 与之匹配。

举个例子，如果你在某个指定文件夹下写入了加密的图片，你可以使用 ``handles`` 方法来实现一个 ``ModelLoader`` 以从那个特定的文件夹下解密图片，但是并不用于加载其他文件夹下的 ``File`` ：

```java
public final class MyModelLoader implements ModelLoader<File, InputStream> {
  private static final String ENCRYPTED_PATH = "/my/encrypted/folder";

  @Override
  public LoadData<InputStream> buildLoadData(File model, int width, int height,
      Options options) {
    return new LoadData<>(model, new MyDataFetcher(model));
  }

  @Override
  public void handles(File model) {
    return model.getAbsolutePath().startsWith(ENCRYPTED_PATH);
  }
}
```


#### ``using()``

``using`` API在 Glide v4 中被删除了，这是为了鼓励用户使用 [``AppGlideModule``][2] 一次性地 [注册][35] 所有组件，避免对象重用(re-use, 原文如此 --译者注)。你无需每次加载图片时都创建一个新的 ``ModelLoader`` ；你应该在 [``AppGlideModule``][2] 中注册一次，然后交给 Glide 在每次加载时检查 model (即你传入 [``load()``][25] 方法的对象)来决定什么时候使用你注册的 ``ModelLoader` 。

为了确保你仅为特定的 model 使用你的 ``ModelLoader`` ，请像上面展示的那样实现 ``handles`` 方法：检查每个 model ，但仅在应当使用你的 ``ModelLoader`` 时才返回 true 。

[1]: {{ site.baseurl }}/javadocs/360/com/bumptech/glide/module/GlideModule.html
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[3]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html
[4]: configuration.html
[5]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#thumbnail-com.bumptech.glide.RequestBuilder-
[7]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#listener-com.bumptech.glide.request.RequestListener-
[8]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#into-Y-
[9]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#preload-int-int-
[10]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[11]: generatedapi.html
[12]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideExtension.html
[13]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/bitmap/BitmapTransitionOptions.html
[14]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/GenericTransitionOptions.html
[15]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/drawable/DrawableTransitionOptions.html
[16]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/bitmap/BitmapTransitionOptions.html
[17]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/TransitionOptions.html#dontTransition--
[18]: https://developer.android.com/reference/android/graphics/drawable/Drawable.html
[19]: https://developer.android.com/reference/android/graphics/drawable/BitmapDrawable.html
[20]: https://developer.android.com/reference/android/graphics/drawable/Animatable.html
[21]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestListener.html
[22]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestManager.html
[23]: {{ site.baseurl }}/javadocs/380/com/bumptech/glide/RequestManager.html#using(com.bumptech.glide.load.model.stream.StreamByteArrayLoader)
[24]: {{ site.baseurl }}/doc/generatedapi.html
[25]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#load-java.lang.Object-
[26]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/model/ModelLoader.html
[27]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/model/ModelLoader.LoadData.html
[28]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/load/Transformation.html
[29]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/request/RequestOptions.html#transforms-com.bumptech.glide.load.Transformation...-
[30]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/load/DecodeFormat.html
[31]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/load/DecodeFormat.html#PREFER_RGB_565
[32]: https://developer.android.com/reference/android/graphics/Bitmap.Config.html#RGB_565
[33]: https://developer.android.com/reference/android/graphics/Bitmap.Config.html#ARGB_8888
[34]: download-setup.html
[35]: {{ site.baseurl }}/doc/configuration.html#registering-components
