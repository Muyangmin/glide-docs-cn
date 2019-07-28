---
layout: page
title: "配置"
category: doc
date: 2018/7/9 16:51
order: 9
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/configuration.html){:target="_blank"}

* TOC
{:toc}

### 设置
从 Glide 4.9.0 开始，在某些情形下必须完成必要的设置 (`setup`)。

对于应用程序（application），仅当以下情形时才需要做设置：

* 使用一个或更多集成库
* 修改 Glide 的配置(`configuration`)（磁盘缓存大小/位置，内存缓存大小等）
* 扩展 Glide 的API。

对于库（library），仅当库需要注册一个或多个组件时才需要做设置。

#### 应用程序
应用程序(Applications)如果希望使用集成库和/或 Glide 的 API 扩展，则需要：
1. 恰当地添加一个 [``AppGlideModule``][1] 实现。
2. (可选)添加一个或多个 [``LibraryGlideModule``][2] 实现。
3. 给上述两种实现添加 [``@GlideModule``][5] 注解。
4. 添加对 Glide 的注解解析器的依赖。
5. 在 proguard 中，添加对 [``AppGlideModules``][1] 的 keep 。

在 Glide 的 [Flickr 示例应用][8] 中，有一个 [``AppGlideModule``][1] 的示例实现：
```java
@GlideModule
public class FlickrGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Photo.class, InputStream.class, new FlickrModelLoader.Factory());
  }
}
```

请注意添加对 Glide 的注解和注解解析器的依赖：
```groovy
compile 'com.github.bumptech.glide:annotations:4.9.0'
annotationProcessor 'com.github.bumptech.glide:compiler:4.9.0'
```

最后，你应该在你的 ``proguard.cfg`` 中 keep 住你的 AppGlideModule 实现：
```
-keep public class  extends com.bumptech.glide.module.AppGlideModule
-keep class com.bumptech.glide.GeneratedAppGlideModuleImpl
```

#### 程序库 (Libraries)
程序库若不需要注册定制组件，则不需要做任何配置步骤，可以完全跳过这个章节。

程序库如果需要注册定制组件，例如 `ModelLoader`，可按以下步骤执行：

1. 添加一个或多个 [``LibraryGlideModule``][2] 实现，以注册新的组件。
2. 为每个 [``LibraryGlideModule``][2] 实现，添加 [``@GlideModule``][5] 注解。
3. 添加 Glide 的注解处理器的依赖。

一个 [``LibraryGlideModule``] 的例子，在 Glide 的[OkHttp 集成库][7] 中：

```java
@GlideModule
public final class OkHttpLibraryGlideModule extends LibraryGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```

使用 [``GlideModule``][5] 注解需要使用 Glide 注解的依赖：

```groovy
compile 'com.github.bumptech.glide:annotations:4.9.0'
```

##### 避免在程序库中使用 AppGlideModule
程序库一定 **不要** 包含 ``AppGlideModule`` 实现。这么做将会阻止依赖该库的任何应用程序管理它们的依赖，或配置诸如 Glide 缓存大小和位置之类的选项。
 
此外，如果两个程序库都包含 ``AppGlideModule``，应用程序将无法在同时依赖两个库的情况下通过编译，而不得不在二者之中做出取舍。

这确实意味着程序库将无法使用 Glide 的 generated API，但是使标准的 `RequestBuilder` 和 `RequestOptions` 加载仍然有效（可以在 [选项][42] 页找到例子）。


### 应用程序选项
Glide 允许应用通过 [``AppGlideModule``][1] 实现来完全控制 Glide 的内存和磁盘缓存使用。Glide 试图提供对大部分应用程序合理的默认选项，但对于部分应用，可能就需要定制这些值。在你做任何改变时，请注意测量其结果，避免出现性能的倒退。

#### 内存缓存
默认情况下，Glide使用 [``LruResourceCache``][10] ，这是 [``MemoryCache``][9] 接口的一个缺省实现，使用固定大小的内存和 LRU 算法。[``LruResourceCache``][10] 的大小由 Glide 的 [``MemorySizeCalculator``][11] 类来决定，这个类主要关注设备的内存类型，设备 RAM 大小，以及屏幕分辨率。

应用程序可以自定义 [``MemoryCache``][9] 的大小，具体是在它们的 [``AppGlideModule``][1] 中使用 [``applyOptions(Context, GlideBuilder)``][12] 方法配置 [``MemorySizeCalculator``][11] ：

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

甚至可以提供自己的 [``MemoryCache``][9] 实现：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setMemoryCache(new YourAppMemoryCacheImpl());
  }
}
```

#### Bitmap 池
Glide 使用 [``LruBitmapPool``][39] 作为默认的 [``BitmapPool``][40] 。[``LruBitmapPool``][39] 是一个内存中的固定大小的 ``BitmapPool``，使用 LRU 算法清理。默认大小基于设备的分辨率和密度，同时也考虑内存类和 [``isLowRamDevice``][41] 的返回值。具体的计算通过 Glide 的 [``MemorySizeCalculator``][11] 来完成，与 Glide 的 [``MemoryCache``][9] 的大小检测方法相似。
 
 应用可以在它们的 [``AppGlideModule``][1] 中定制 [``BitmapPool``] 的尺寸，使用 [``applyOptions(Context, GlideBuilder)``][12] 方法并配置 [``MemorySizeCalculator``][11]:

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    MemorySizeCalculator calculator = new MemorySizeCalculator.Builder(context)
        .setBitmapPoolScreens(3)
        .build();
    builder.setBitmapPool(new LruBitmapPool(calculator.getBitmapPoolSize()));
  }
}
```

应用也可以直接复写这个池的大小：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    int bitmapPoolSizeBytes = 1024 * 1024 * 30; // 30mb
    builder.setBitmapPool(new LruBitmapPool(bitmapPoolSizeBytes));
  }
}
```

甚至可以提供 [``BitmapPool``][40] 的完全自定义实现：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setBitmapPool(new YourAppBitmapPoolImpl());
  }
}
```

#### 磁盘缓存
Glide 使用 [``DiskLruCacheWrapper``][13] 作为默认的 [``磁盘缓存``][14] 。 [``DiskLruCacheWrapper``][13] 是一个使用 LRU 算法的固定大小的磁盘缓存。默认磁盘大小为 [250 MB][15] ，位置是在应用的 [缓存文件夹][17] 中的一个 [特定目录][16] 。

假如应用程序展示的媒体内容是公开的（例如从无授权机制的网站或搜索引擎上加载），那么应用可以将这个缓存位置改到外部存储：
```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDiskCache(new ExternalCacheDiskCacheFactory(context));
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
    builder.setDiskCache(new InternalCacheDiskCacheFactory(context, diskCacheSizeBytes));
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
        new InternalCacheDiskCacheFactory(context, cacheFolderName, diskCacheSizeBytes));
  }
}
```

应用程序还可以自行选择 [``DiskCache``][14] 接口的实现，并提供自己的 [``DiskCache.Factory``][18] 来创建缓存。Glide 使用一个工厂接口来在后台线程中打开 [``磁盘缓存``][14] ，这样方便缓存做诸如检查路径存在性等的IO操作而不用触发 [严格模式][19] 。
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

### 默认请求选项
虽然 [``请求选项``][33] 通常由每个请求单独指定，你也可以通过 [``AppGlideModule``][1] 应用一个 [``请求选项``][33] 的集合以作用于你应用中启动的每个加载：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDefaultRequestOptions(
        new RequestOptions()
          .format(DecodeFormat.RGB_565)
          .disallowHardwareBitmaps());
  }
}
```

一旦你创建了新的请求，这些选项将通过 ``GlideBuilder`` 中的 ``setDefaultRequestOptions`` 被应用上。因此，任何单独请求里应用的选项将覆盖 ``GlideBuilder`` 里设置的冲突选项。

类似地，[``RequestManagers``][34] 允许你为这个特定的 [``RequestManager``][34] 启动的所有加载请求设置默认的 [``请求选项``][33]。 因为每个 ``Activity`` 和 ``Fragment`` 都拥有自己的 [``RequestManager``][34]，你可以使用 [``RequestManager``][34] 的 [``applyDefaultRequestOptions``][35] 方法来设置默认的 [``RequestOption``][33]，并仅作用于一个特定的 ``Activity`` 或 ``Fragment``：

```java
Glide.with(fragment)
  .applyDefaultRequestOptions(
      new RequestOptions()
          .format(DecodeFormat.RGB_565)
          .disallowHardwareBitmaps());
```
[``RequestManager``][34] 还有一个 [``setDefaultRequestOptions``][36] 方法，可以完全替换掉之前设置的任意的默认 [``请求选项``][33]，无论它是通过 [``AppGlideModule``][1] 的 [``GlideBuilder``] 还是 [``RequestManager``][34]。使用 [``setDefaultRequestOptions``]要小心，因为很容易意外覆盖掉你其他地方设置的重要默认选项。 通常 [``applyDefaultRequestOptions``][35]更安全，使用起来更直观。

### 未捕获异常策略 (UncaughtThrowableStrategy)
在加载图片时假如发生了一个异常 (例如, OOM), Glide 将会使用一个 `GlideExecutor.UncaughtThrowableStrategy` 。

默认策略是将异常打印到设备的 LogCat 中。 这个策略从 Glide 4.2.0 起将可被定制。 你可以传入一个磁盘执行器和/或一个 resize 执行器：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    final UncaughtThrowableStrategy myUncaughtThrowableStrategy = new ...
    builder.setDiskCacheExecutor(newDiskCacheExecutor(myUncaughtThrowableStrategy));
    builder.setResizeExecutor(newSourceExecutor(myUncaughtThrowableStrategy));
  }
}
```

#### 日志级别

你可以使用 [``setLogLevel``][37] (结合 Android 的 [``Log``][38] 定义的值) 来获取格式化日志的子集，包括请求失败时的日志行。通常来说 ``Log.VERBOSE`` 将使日志变得更冗杂，``Log.ERROR`` 会让日志更趋向静默，详细可见 [javadoc][37] 。

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setLogLevel(Log.DEBUG);
  }
}
```

### 注册组件

应用程序和库都可以注册很多组件来扩展 Glide 的功能。可用的组件包括：

1. [``ModelLoader``][23], 用于加载自定义的 Model(Url, Uri,任意的 POJO )和 Data(InputStreams, FileDescriptors)。
2. [``ResourceDecoder``][24], 用于对新的 Resources(Drawables, Bitmaps)或新的 Data 类型(InputStreams, FileDescriptors)进行解码。
3. [``Encoder``][25], 用于向 Glide 的磁盘缓存写 Data (InputStreams, FileDesciptors)。
4. [``ResourceTranscoder``][26]，用于在不同的资源类型之间做转换，例如，从 BitmapResource 转换为 DrawableResource 。
5. [``ResourceEncoder``][27]，用于向 Glide 的磁盘缓存写 Resources(BitmapResource, DrawableResource)。

组件通过 [``Registry``][28] 类来注册。例如，添加一个 ``ModelLoader`` ，使其能从自定义的Model对象中创建一个 InputStream ：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
  }
}
```

在一个 ``GlideModule`` 里可以注册很多组件。[``ModelLoader``][23] 和 [``ResourceDecoder``][24] 对于同样的参数类型还可以有多种实现。

### 剖析(Anatomy)一个请求
被注册组件的集合（包括默认被 Glide 注册的和在 Module 中被注册的），会被用于定义一个加载路径集合。每个加载路径都是从提供给 [``load``][29] 方法的数据模型到 [``as``][30] 方法指定的资源类型的一个逐步演进的过程。一个加载路径（粗略地）由下列步骤组成:

1. 模型(Model) -> 数据(Data) (由``模型加载器(ModelLoader)``处理)
2. 数据(Data)  -> 资源(Resource) (由``资源解析器(ResourceDecoder)``处理)
3. 资源(Resource) -> 转码后的资源(Transcoded Resource) (可选；由``资源转码器(ResourceTranscoder)``处理)

``编码器(Encoder)``可以在步骤2之前往Glide的磁盘缓存中写入数据。
 ``资源编码器(ResourceEncoder)``可以在步骤3之前网Glide的磁盘缓存写入资源。
  
当一个请求开始后，Glide将尝试所有从数据模型到请求的资源类型的可用路径。如果任何一个加载路径成功，这个请求就将成功。只有所有可用加载路径都失败时，这个请求才会失败。

### 排序组件

在 [``Registry``][28] 类中定义了 ``prepend()`` , ``append()`` 和 ``replace()`` 方法，它们可以用于设置 Glide 尝试每个 ``ModelLoader`` 和 ``ResourceDecoder`` 之间的顺序。对组件进行排序允许你注册一些只处理特定树模型的子集的组件（即只处理特定类型的Uri，或仅仅特定类型的图像格式），并可以在后面追加一个捕获所有类型的组件以处理其他情况。

##### prepend()
假如你的 ``ModelLoader`` 或者 ``ResourceDecoder`` 在某个地方失败了，这时候你想将已有的数据交由 Glide 的默认行为来处理，可以使用 ``prepend()``。 ``prepend()`` 将确保你的 ``ModelLoader`` 或 ``ResourceDecoder`` 先于之前注册的其他组件并被首先执行。如果你的 ``ModelLoader`` 或者 ``ResourceDecoder`` 从其 ``handles()`` 方法中返回了一个 ``false`` 或失败，所有其他的 ``ModelLoader`` 或 ``ResourceDecoder`` 将以它们被注册的顺序执行，一次一个，作为一种回退方案。

##### append()
要处理新的数据类型或提供一个到 Glide 默认行为的回退，使用 ``append()``。``append()`` 将确保你的 ``ModelLoader`` 或 ``ResourceDecoder`` 仅在 Glide 的默认组件被尝试后才会被调用。 如果你正在尝试处理 Glide 的默认组件能处理的某些子类型 (例如一些特定的 Uri 授权或子类型)，你可能需要使用 ``prepend()`` 来确保 Glide 的默认组件不会在你的定制组件之前加载。

##### replace()
要完全替换 Glide 的默认行为并确保它绝不运行，请使用 ``replace()``。 ``replace()`` 将移除所有处理给定模型和数据类的 ``ModelLoaders``，并添加你的 ``ModelLoader`` 来代替。 ``replace()`` 在使用库(例如 OkHttp 或 Volley)替换掉 Glide 的网络逻辑时尤其有用，这种时候你会希望确保仅 OkHttp 或 Volley 被调用。

#### 添加一个 ModelLoader
举个例子，添加一个 ``ModelLoader``，它从一个新的自定义 Model 对象中建立一个 InputStream：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
  }
}
```

在这里，``append()``可以被安全地使用，因为 Photo.class 是一个你的应用定制的模型对象，所以你知道 Glide 的默认行为中并没有你需要替换的东西。

相反，如果要处理一种新的 [``BaseGlideUrlLoader``][32] 中的 String Url类型，你应该使用 ``prepend()`` 以使你的 ``ModelLoader`` 在 Glide 对 ``Strings`` 的默认 ``ModelLoaders`` 之前运行：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.prepend(String.class, InputStream.class, new CustomUrlModelLoader.Factory());
  }
}
```
最后，如果要完全移除和替换 Glide 对某种特定类型的默认处理，例如一个网络库，你应该使用 ``replace()``:

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```


### 模块类和注解
Glide v4 依赖于两种类，[``AppGlideModule``][1] 与 [``LibraryGlideModule``][2] ，以配置 Glide 单例。这两种类都允许用于注册额外的组件，例如 [``ModelLoaders``][3] , [``ResourceDecoders``][4] 等。但只有 [``AppGlideModule``][1] 被允许配置应用特定的设置项，比如缓存实现和缓存大小。

#### AppGlideModule
如果应用希望实现  [``AppGlideModule``][1]  的任意方法或使用集成库，则可以添加一个 [``AppGlideModule``][1] 实现。 [``AppGlideModule``][1] 实现是一个信号，它会让 Glide 的注解解析器生成一个单一的所有已发现的 [``LibraryGlideModules``][2] 的联合类。
 
对于一个特定的应用，只能存在一个 [``AppGlideModule``][1] 实现（超过一个会在编译时报错）。因此，程序库不能提供 [``AppGlideModule``][1] 实现。

#### @GlideModule
为了让 Glide 正确地发现 [``AppGlideModule``][1] 和 [``LibraryGlideModule``][2] 的实现类，它们的所有实现都必须使用 [``@GlideModule``][5] 注解来标记。这个注解将允许 Glide 的 [注解解析器][6] 在编译时去发现所有的实现类。

#### 注解处理器 
另外，为了发现 [``AppGlideModule``][1] 和 [``LibraryGlideModules``][2]，所有的库和应用还必须包含一个Glide的注解解析器的依赖。

### 冲突
应用程序可能依赖多个程序库，而它们每一个都可能包含一个或更多的 [``LibraryGlideModules``][2] 。在极端情况下，这些 [``LibraryGlideModules``][2] 可能定义了相互冲突的选项，或者包含了应用程序希望避免的行为。应用程序可以通过给他们的 [``AppGlideModule``][1] 添加一个 [``@Excludes``][20] 注解来解决这种冲突，或避免不需要的依赖。

例如，如果你依赖了一个库，它有一个 [``LibraryGlideModule``][2] 叫做``com.example.unwanted.GlideModule``，而你不想要它：

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

[``@Excludes``][20] 注解不仅可用于排除 [``LibraryGlideModules``][2] 。如果你还在从 Glide v3 到新版本的迁移过程中，你还可以用它来排除旧的，废弃的 [``GlideModule``][21] 实现。


### 清单解析
为了维持对 Glide v3 的 [``GlideModules``][21] 的向后兼容性，Glide 仍然会解析应用程序和所有被包含的库中的 ``AndroidManifest.xml`` 文件，并包含在这些清单中列出的旧 [``GlideModules``][21] 模块类。

如果你已经迁移到 Glide v4 的 [``AppGlideModule``][1] 和 [``LibraryGlideModule``][2] ，你可以完全禁用清单解析。这样可以改善 Glide 的初始启动时间，并避免尝试解析元数据时的一些潜在问题。要禁用清单解析，请在你的 [``AppGlideModule``][1] 实现中复写 [``isManifestParsingEnabled()``][22] 方法：

```java
@GlideModule
public final class MyAppGlideModule extends AppGlideModule {
  @Override
  public boolean isManifestParsingEnabled() {
    return false;
  }
}
```

[1]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html
[3]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/model/ModelLoader.html
[4]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html
[5]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideModule.html
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/compiler/GlideAnnotationProcessor.html
[7]: https://github.com/bumptech/glide/blob/master/integration/okhttp3/src/main/java/com/bumptech/glide/integration/okhttp3/OkHttpLibraryGlideModule.java
[8]: https://github.com/bumptech/glide/blob/master/samples/flickr/src/main/java/com/bumptech/glide/samples/flickr/FlickrGlideModule.java
[9]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/MemoryCache.html
[10]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/LruResourceCache.html
[11]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/MemorySizeCalculator.html
[12]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html#applyOptions-android.content.Context-com.bumptech.glide.GlideBuilder-
[13]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/DiskLruCacheWrapper.html
[14]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.html
[15]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html#DEFAULT_DISK_CACHE_SIZE
[16]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html#DEFAULT_DISK_CACHE_DIR
[17]: https://developer.android.com/reference/android/content/Context.html#getCacheDir()
[18]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.Factory.html
[19]: https://developer.android.com/reference/android/os/StrictMode.html
[20]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/Excludes.html
[21]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/GlideModule.html
[22]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html#isManifestParsingEnabled--
[23]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/model/ModelLoaderFactory.html
[24]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html
[25]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/Encoder.html
[26]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/transcode/ResourceTranscoder.html
[27]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/ResourceEncoder.html
[28]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/Registry.html
[29]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html#load-java.lang.Object-
[30]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestManager.html#as-java.lang.Class-
[31]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html#registerComponents-android.content.Context-com.bumptech.glide.Glide-com.bumptech.glide.Registry-
[32]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html
[33]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/request/RequestOptions.html
[34]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/RequestManager.html
[35]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/RequestManager.html#applyDefaultRequestOptions-com.bumptech.glide.request.RequestOptions-
[36]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/RequestManager.html#setDefaultRequestOptions-com.bumptech.glide.request.RequestOptions-
[37]: {{ site.baseurl }}/javadocs/420/com/bumptech/glide/GlideBuilder.html#setLogLevel-int-
[38]: https://developer.android.com/reference/android/util/Log.html
[39]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/bitmap_recycle/LruBitmapPool.html
[40]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/bitmap_recycle/BitmapPool.html
[41]: https://developer.android.com/reference/android/app/ActivityManager.html#isLowRamDevice()
[42]: {{ site.baseurl }}/doc/options.html

