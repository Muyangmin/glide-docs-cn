---
layout: page
title: "配置"
category: doc
date: 2017-03-14 13:37:04
order: 9
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/configuration.html)

* TOC
{:toc}

### 设置
为了让Glide正常工作，库和应用程序需要做一些固定的步骤。不过，假如你的库不希望注册额外的组件，则这些初始化不是必须的。

#### 程序库
程序库(Libraries)需要：
1. 添加一个或多个[``LibraryGlideModule``][2]实现。
2. 为每一个[``LibraryGlideModule``][2]实现添加[``@GlideModule``][5]注解。
3. 添加Glide的注解解析器的依赖。

An example [``LibraryGlideModule``][2] from Glide's [OkHttp integration library][7] looks like this
在Glide的[OkHttp 集成库][7]中有一个[``LibraryGlideModule``][2]的示例实现，它看起来长这样：
```java
@GlideModule
public final class OkHttpLibraryGlideModule extends LibraryGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```

要使用[``@GlideModule``][5]注解，需要有对Glide注解的依赖：
```groovy
compile 'com.github.bumptech.glide:annotations:4.0.0'
```

#### 应用程序
应用程序(Applications)需要：
1. 恰当地添加一个[``AppGlideModule``][1]实现。
2. (可选)添加一个或多个[``LibraryGlideModule``][2]实现。
3. 给上述两种实现添加[``@GlideModule``][5]注解。
4. 添加对Glide的注解解析器的依赖。
5. 在proguard中，添加对[``AppGlideModules``][1]的keep。

在Glide的[Flickr 示例应用][8]中，有一个[``AppGlideModule``][1]的示例实现：
```java
@GlideModule
public class FlickrGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Photo.class, InputStream.class, new FlickrModelLoader.Factory());
  }
}
```

请注意添加对Glide的注解和注解解析器的依赖：
```groovy
compile 'com.github.bumptech.glideannotations4.0.0'
annotationProcessor 'com.github.bumptech.glidecompiler4.0.0'
```

最后，你应该在你的``proguard.cfg``中keep住你的AppGlideModule实现：
```
-keep public class  extends com.bumptech.glide.module.AppGlideModule
-keep class com.bumptech.glide.GeneratedAppGlideModuleImpl
```

### 应用程序选项
Glide允许应用通过[``AppGlideModule``][1]实现来完全控制Glide的内存和磁盘缓存使用。Glide试图提供对大部分应用程序合理的默认选项，但对于部分应用，可能就需要定制这些值。在你做任何改变时，请注意测量其结果，避免出现性能的倒退。

#### 内存缓存
默认情况下，Glide使用[``LruResourceCache``][10]，这是[``MemoryCache``][9]接口的一个缺省实现，使用固定大小的内存和LRU算法。[``LruResourceCache``][10]的大小由Glide的[``MemorySizeCalculator``][11]类来决定，这个类主要关注设备的内存类型，设备RAM大小，以及屏幕分辨率。

应用程序可以自定义[``MemoryCache``][9]的大小，具体是在它们的[``AppGlideModule``][1]中使用[``applyOptions(Context, GlideBuilder)``][12]方法配置[``MemorySizeCalculator``][11]：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    MemorySizeCalculator calculator = new MemorySizeCalculator.Builder(context)
        .setMemoryCacheScreens(2)
        .build();
    builder.setMemoryCache(new LruResourceCache(calculator.getMemoryCacheSize()));
  }
}
```

也可以直接覆写缓存大小：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    int memoryCacheSizeBytes = 1024 * 1024 * 20; // 20mb
    builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));
  }
}
```

甚至可以提供自己的[``MemoryCache``][9]实现：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setMemoryCache(new YourAppMemoryCacheImpl());
  }
}
```

#### 磁盘缓存
Glide使用[``DiskLruCacheWrapper``][13]作为默认的[``磁盘缓存``][14]。[``DiskLruCacheWrapper``][13]是一个使用LRU算法的固定大小的磁盘缓存。默认磁盘大小为[250 MB][15]，位置是在应用的[缓存文件夹][17]中的一个[特定目录][16]。

假如应用程序展示的媒体内容是公开的（从无授权机制的网站上加载，或搜索引擎等），那么应用可以将这个缓存位置改到外部存储：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDiskCache(new ExternalDiskCacheFactory(context));
  }
}
```

无论使用内部或外部磁盘缓存，应用程序都可以改变磁盘缓存的大小：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    int diskCacheSizeBytes = 1024  1024  100;  100 MB
    builder.setDiskCache(new InternalDiskCacheFactory(context, diskCacheSizeBytes));
  }
}
```

应用程序还可以改变缓存文件夹在外存或内存上的名字：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    int diskCacheSizeBytes = 1024  1024  100;  100 MB
    builder.setDiskCache(
        new InternalDiskCacheFactory(context, cacheFolderName, diskCacheSizeBytes));
  }
}
```

应用程序还可以自行选择[``DiskCache``][14]接口的实现，并提供自己的[``DiskCache.Factory``][18]来创建缓存。Glide使用一个工厂接口来在后台线程中打开[``磁盘缓存``][14]，这样方便缓存做诸如检查路径存在性等的IO操作而不用触发[严格模式][19]。
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDiskCache(new DiskCache.Factory() {
        @Override
        public DiskCache build() {
          return new YourAppCustomDiskCache();
        }
    });
  }
}
```

### 注册组件

应用程序和库都可以注册很多组件来扩展Glide的功能。可用的组件包括：

1. [``ModelLoader``][23], 用于加载自定义的Model(Url, Uri,任意的POJO)和Data(InputStreams, FileDescriptors)。
2. [``ResourceDecoder``][24], 用于对新的Resources(Drawables, Bitmaps)或新的Data类型(InputStreams, FileDescriptors)进行解码。
3. [``Encoder``][25], 用于向Glide的磁盘缓存写Data (InputStreams, FileDesciptors)。
4. [``ResourceTranscoder``][26]，用于在不同的资源类型之间做转换，例如，从BitmapResource转换为DrawableResource。
5. [``ResourceEncoder``][27]，用于向Glide的磁盘缓存写Resources(BitmapResource, DrawableResource)。

组件通过[``Registry``][28]类来注册。例如，添加一个``ModelLoader``，使其能从自定义的Model对象中创建一个InputStream：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
  }
}
```

在一个``GlideModule``里可以注册很多组件。[``ModelLoader``][23]和[``ResourceDecoder``][24]对于同样的参数类型还可以有多种实现。

被注册组件的集合（包括默认被Glide注册的和在Moduke中被注册的），会被用于定义一个加载路径集合。每个加载路径都是从提供给[``load``][29]方法的数据模型到[``as``][30]方法指定的资源类型的一个逐步演进的过程。一个加载路径（粗略地）由下列步骤组成:

1. 模型(Model) -> 数据(Data) (由``模型加载器(ModelLoader)``处理)
2. 数据(Data)  -> 资源(Resource) (由``资源解析器(ResourceDecoder)``处理)
3. 资源(Resource) -> 转码后的资源(Transcoded Resource) (可选；由``资源转码器(ResourceTranscoder)``处理)

``编码器(Encoder)``可以在步骤2之前往Glide的磁盘缓存中写入数据。
 ``资源编码器(ResourceEncoder)``可以在步骤3之前网Glide的磁盘缓存写入资源。
  
当一个请求开始后，Glide将尝试所有从数据模型到请求的资源类型的可用路径。如果任何一个加载路径成功，这个请求就将成功。只有所有可用加载路径都失败时，这个请求才会失败。

在[``Registry``][28]类中定义了``prepend()``, ``append()`` 和``replace()``方法，它们可以用于设置Glide尝试每个``ModelLoader``和``ResourceDecoder``之间的顺序。通过确保优先注册处理最常见类型的``ModelLoader``和``ResourceDecoder``，可以使请求更高效。对组件进行排序还意味着允许你注册一些只处理特定树模型的子集的组件（即只处理特定类型的Uri，或仅仅特定类型的图像格式），但还能在后面追加一个捕获所有类型的组件以处理其他情况。

### 模块类和注解
Glide v4 依赖于两种类，[``AppGlideModule``][1]与[``LibraryGlideModule``][2]，以配置Glide单例。这两种类都允许用于注册额外的组件，例如[``ModelLoaders``][3], [``ResourceDecoders``][4]等。但只有[``AppGlideModule``][1]被允许配置应用特定的设置项，比如缓存实现和缓存大小。

#### AppGlideModule
所有应用都必须添加一个[``AppGlideModule``][1]实现，即使应用并没有改变任何附加设置项，也没有实现[``AppGlideModule``][1]中的任何方法。 [``AppGlideModule``][1]实现是一个信号，它会让Glide的注解解析器生成一个单一的所有已发现的[``LibraryGlideModules``][2]的联合类。
 
对于一个特定的应用，只能存在一个[``AppGlideModule``][1]实现（超过一个会在编译时报错）。因此，程序库不能提供[``AppGlideModule``][1]实现。

#### @GlideModule
为了让Glide正确地发现[``AppGlideModule``][1]和[``LibraryGlideModule``][2]的实现类，它们的所有实现都必须使用[``@GlideModule``][5]注解来标记。这个注解将允许Glide的[注解解析器][6]在编译时去发现所有的实现类。

#### 注解处理器 
另外，为了发现[``AppGlideModule``][1] 和 [``LibraryGlideModules``][2]，所有的库和应用还必须包含一个Glide的注解解析器的依赖。

### 冲突
应用程序可能依赖多个程序库，而它们每一个都可能包含一个或更多的[``LibraryGlideModules``][2]。在极端情况下，这些[``LibraryGlideModules``][2]可能定义了相互冲突的选项，或者包含了应用程序希望避免的行为。应用程序可以通过给他们的[``AppGlideModule``][1]添加一个[``@Excludes``][20]注解来解决这种冲突，或避免不需要的依赖。

例如，如果你依赖了一个库，它有一个[``LibraryGlideModule``][2]叫做``com.example.unwanted.GlideModule``，而你不想要它：

```java
@Excludes(com.example.unwanted.GlideModule)
@GlideModule
public final class MyAppGlideModule extends AppGlideModule { }
```

你也可以排除多个模块：

```java
@Excludes({com.example.unwanted.GlideModule, com.example.conflicing.GlideModule})
@GlideModule
public final class MyAppGlideModule extends AppGlideModule { }
```

``@Excludes``][20]注解不仅可用于排除[``LibraryGlideModules``][2]。如果你还在从Glide v3到新版本的迁移过程中，你还可以用它来排除旧的，废弃的[``GlideModule``][21]实现。


### 清单解析
为了维持对Glide v3的[``GlideModules``][21]的向后兼容性，Glide仍然会解析应用程序和所有被包含的库中的``AndroidManifest.xml``文件，并包含在这些清单中列出的旧 [``GlideModules``][21]模块类。

如果你已经迁移到Glide v4的[``AppGlideModule``][1] 和 [``LibraryGlideModule``][2]，你可以完全禁用清单解析。这样可以改善Glide的初始启动时间，并避免尝试解析元数据时的一些潜在问题。要禁用清单解析，请在你的[``AppGlideModule``][1]实现中复写[``isManifestParsingEnabled()``][22]方法：

```java
@GlideModule
public final class MyAppGlideModule extends AppGlideModule {
  @Override
  public boolean isManifestParsingEnabled() {
    return false;
  }
}
```

[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html
[3]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/model/ModelLoader.html
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/GlideModule.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/compiler/GlideAnnotationProcessor.html
[7]: https://github.com/bumptech/glide/blob/master/integration/okhttp3/src/main/java/com/bumptech/glide/integration/okhttp3/OkHttpLibraryGlideModule.java
[8]: https://github.com/bumptech/glide/blob/master/samples/flickr/src/main/java/com/bumptech/glide/samples/flickr/FlickrGlideModule.java
[9]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/MemoryCache.html
[10]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/LruResourceCache.html
[11]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/MemorySizeCalculator.html
[12]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/AppGlideModule.html#applyOptions-android.content.Context-com.bumptech.glide.GlideBuilder-
[13]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskLruCacheWrapper.html
[14]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.html
[15]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html#DEFAULT_DISK_CACHE_SIZE
[16]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html#DEFAULT_DISK_CACHE_DIR
[17]: https://developer.android.com/reference/android/content/Context.html#getCacheDir()
[18]: {{ site.url}}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html
[19]: https://developer.android.com/reference/android/os/StrictMode.html
[20]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/Excludes.html
[21]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/GlideModule.html
[22]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/AppGlideModule.html#isManifestParsingEnabled--
[23]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/model/ModelLoaderFactory.html
[24]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html
[25]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/Encoder.html
[26]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/resource/transcode/ResourceTranscoder.html
[27]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/ResourceEncoder.html
[28]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/Registry.html
[29]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html#load-java.lang.Object-
[30]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestManager.html#as-java.lang.Class-
[31]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html#registerComponents-android.content.Context-com.bumptech.glide.Glide-com.bumptech.glide.Registry-
[32]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html
[33]: {{ site.url }}/glide/javadocs/410/com/bumptech/glide/request/RequestOptions.html
[34]: {{ site.url }}/glide/javadocs/410/com/bumptech/glide/RequestManager.html
[35]: {{ site.url }}/glide/javadocs/410/com/bumptech/glide/RequestManager.html#applyDefaultRequestOptions-com.bumptech.glide.request.RequestOptions-
[36]: {{ site.url }}/glide/javadocs/410/com/bumptech/glide/RequestManager.html#setDefaultRequestOptions-com.bumptech.glide.request.RequestOptions-