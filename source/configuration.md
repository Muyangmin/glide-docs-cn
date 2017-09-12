---
title: "配置"
---
原文链接：[点击查看](http://bumptech.github.io/glide/doc/configuration.html)

### Setup
为了让Glide正常工作，库和应用程序需要做一些固定的步骤。不过，假如你的库不希望注册额外的组件，则这些初始化不是必须的。

#### Libraries
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

#### Applications
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

### Application Options
Glide允许应用通过[``AppGlideModule``][1]实现来完全控制Glide的内存和磁盘缓存使用。Glide试图提供对大部分应用程序合理的默认选项，但对于部分应用，可能就需要定制这些值。在你做任何改变时，请注意测量其结果，避免出现性能的倒退。

#### Memory cache
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

#### Disk Cache
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

### Registering Components

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

The set of registered components, including both those registered by default in Glide and those registered in Modules are used to define a set of load paths. Each load path is a step by step progression from the the Model provided to [``load()``][29] to the Resource type specified by [``as()``][30]. A load path consists (roughly) of the following steps


1. Model - Data (handled by ``ModelLoader``s)
2. Data - Resource (handled by ``ResourceDecoder``s)
3. Resource - Transcoded Resource (optional, handled by ``ResourceTranscoder``s).

``Encoder``s can write Data to Glide's disk cache cache before step 2.
``ResourceEncoder``s can write Resource's to Glide's disk cache before step 3. 

When a request is started, Glide will attempt all available paths from the Model to the requested Resource type. A request will succeed if any load path succeeds. A request will fail only if all available load paths fail.  

The ``prepend()``, ``append()``, and ``replace()`` methods in [``Registry``][28] can be used to set the order in which Glide will attempt each ``ModelLoader`` and ``ResourceDecoder``. Requests can be made somewhat more efficient by making sure the ``ModelLoader``s and ``ResourceDecoder``s that handle the most common types are registered first. Ordering components can also allow you to register components that handle specific subsets of models or data (ie only certain types of Uris, or only certain image formats) while also having an appended catch-all component to handle the rest.

### Module classes and annotations.
Glide v4 relies on two classes, [``AppGlideModule``][1] and [``LibraryGlideModule``][2], to configure the Glide singleton. Both classes are allowed to register additional components, like [``ModelLoaders``][3], [``ResourceDecoders``][4] etc. Only the [``AppGlideModules``][1] are allowed to configure application specific settings, like cache implementations and sizes. 

#### AppGlideModule
All applications must add a [``AppGlideModule``][1] implementation, even if the Application is not changing any additional settings or implementing any methods in [``AppGlideModule``][1]. The [``AppGlideModule``][1] implementation acts as a signal that allows Glide's annotation processor to generate a single combined class with with all discovered [``LibraryGlideModules``][2].

There can be only one [``AppGlideModule``][1] implementation in a given application (having more than one produce errors at compile time). As a result, libraries must never provide a [``AppGlideModule``][1] implementation. 

#### @GlideModule
In order for Glide to properly discover [``AppGlideModule``][1] and [``LibraryGlideModule``][2] implementations, all implementations of both classes must be annotated with the [``@GlideModule``][5] annotation. The annotation will allow Glide's [annotation processor][6] to discover all implementations at compile time. 

#### Annotation Processor
In addition, to enable discovery of the [``AppGlideModule``][1] and [``LibraryGlideModules``][2] all libraries and applications must also include a dependency on Glide's annotation processor. 

### Conflicts
Applications may depend on multiple libraries, each of which may contain one or more [``LibraryGlideModules``][2]. In rare cases, these [``LibraryGlideModules``][2] may define conflicting options or otherwise include behavior the application would like to avoid. Applications can resolve these conflicts or avoid unwanted dependencies by adding an [``@Excludes``][20] annotation to their [``AppGlideModule``][1].

For example if you depend on a library that has a [``LibraryGlideModule``][2] that you'd like to avoid, say ``com.example.unwanted.GlideModule``

```java
@Excludes(com.example.unwanted.GlideModule)
@GlideModule
public final class MyAppGlideModule extends AppGlideModule { }
```

You can also excludes multiple modules

```java
@Excludes({com.example.unwanted.GlideModule, com.example.conflicing.GlideModule})
@GlideModule
public final class MyAppGlideModule extends AppGlideModule { }
```

[``@Excludes``][20] can be used to exclude both [``LibraryGlideModules``][2] and legacy, deprecated [``GlideModule``][21] implementations if you're still in the process of migrating from Glide v3.


### Manifest Parsing
To maintain backward compatibility with Glide v3's [``GlideModules``][21], Glide still parses ``AndroidManifest.xml`` files from both the application and any included libraries and will include any legacy [``GlideModules``][21] listed in the manifest. Although this functionality will be removed in a future version, we've retained the behavior for now to ease the transition.

If you've already migrated to the Glide v4 [``AppGlideModule``][1] and [``LibraryGlideModule``][2], you can disable manifest parsing entirely. Doing so can improve the initial startup time of Glide and avoid some potential problems with trying to parse metadata. To disable manifest parsing, override the [``isManifestParsingEnabled()``][22] method in your [``AppGlideModule``][1] implemenation

```java
@GlideModule
public final class MyAppGlideModule extends AppGlideModule {
  @Override
  public boolean isManifestParsingEnabled() {
    return false;
  }
}
```

[1] {{ site.url }}glidejavadocs400combumptechglidemoduleAppGlideModule.html
[2] {{ site.url }}glidejavadocs400combumptechglidemoduleLibraryGlideModule.html
[3] {{ site.url }}glidejavadocs400combumptechglideloadmodelModelLoader.html
[4] {{ site.url }}glidejavadocs400combumptechglideloadResourceDecoder.html
[5] {{ site.url }}glidejavadocs400combumptechglideannotationGlideModule.html
[6] {{ site.url }}glidejavadocs400combumptechglideannotationcompilerGlideAnnotationProcessor.html
[7] httpsgithub.combumptechglideblobmasterintegrationokhttp3srcmainjavacombumptechglideintegrationokhttp3OkHttpLibraryGlideModule.java
[8] httpsgithub.combumptechglideblobmastersamplesflickrsrcmainjavacombumptechglidesamplesflickrFlickrGlideModule.java
[9] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheMemoryCache.html
[10] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheLruResourceCache.html
[11] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheMemorySizeCalculator.html
[12] {{ site.url }}glidejavadocs400combumptechglidemoduleAppGlideModule.html#applyOptions-android.content.Context-com.bumptech.glide.GlideBuilder-
[13] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheDiskLruCacheWrapper.html
[14] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheDiskCache.html
[15] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheDiskCache.Factory.html#DEFAULT_DISK_CACHE_SIZE
[16] {{ site.url }}glidejavadocs400combumptechglideloadenginecacheDiskCache.Factory.html#DEFAULT_DISK_CACHE_DIR
[17] httpsdeveloper.android.comreferenceandroidcontentContext.html#getCacheDir()
[18] {{ site.url}}glidejavadocs400combumptechglideloadenginecacheDiskCache.Factory.html
[19] httpsdeveloper.android.comreferenceandroidosStrictMode.html
[20] {{ site.url }}glidejavadocs400combumptechglideannotationExcludes.html
[21] {{ site.url }}glidejavadocs400combumptechglidemoduleGlideModule.html
[22] {{ site.url }}glidejavadocs400combumptechglidemoduleAppGlideModule.html#isManifestParsingEnabled--
[23] {{ site.url }}glidejavadocs400combumptechglideloadmodelModelLoaderFactory.html
[24] {{ site.url }}glidejavadocs400combumptechglideloadResourceDecoder.html
[25] {{ site.url }}glidejavadocs400combumptechglideloadEncoder.html
[26] {{ site.url }}glidejavadocs400combumptechglideloadresourcetranscodeResourceTranscoder.html
[27] {{ site.url }}glidejavadocs400combumptechglideloadResourceEncoder.html
[28] {{ site.url }}glidejavadocs400combumptechglideRegistry.html
[29] {{ site.url }}glidejavadocs400combumptechglideRequestBuilder.html#load-java.lang.Object-
[30] {{ site.url }}glidejavadocs400combumptechglideRequestManager.html#as-java.lang.Class-
