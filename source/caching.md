---
title: "Caching"
---

### Glide里的缓存

默认情况下，Glide会在开始一个新的图片请求之前检查以下多级的缓存：

1. 活动资源 (Active Resources) - 现在是否有另一个View正在展示这张图片？
2. 内存缓存 (Memory cache) - 该图片是否最近被加载过并仍存在于内存中？
3. 资源类型（Resource） - 该图片是否之前曾被解码、转换并写入过磁盘缓存？
4. 数据来源 (Data) - 构建这个图片的资源是否之前曾被写入过文件缓存？

前两步检查图片是否在内存中，如果是则直接返回图片。后两步则检查图片是否在磁盘上，以便快速但异步地返回图片。

如果四个步骤都未能找到图片，则Glide会返回到原始资源以取回数据（原始文件，Uri, Url等）。

### 缓存键(Cache Keys)

在 Glide 4里，所有缓存键都包含至少两个元素：

1. 请求加载的model（File, Url, Url）
2. 一个可选的[``签名``(Signature)][1]

另外，步骤1-3(活动资源，内存缓存，资源磁盘缓存)的缓存键还包含一些其他数据，包括：

1. 宽度和高度
2. 可选的``变换（Transformation）``
3. 额外添加的任何[``选项(Options)``][2]
4. 请求的数据类型 (Bitmap, GIF, 或其他)

活动资源和内存缓存使用的键还和磁盘资源缓存略有不同，以适应内存[``选项(Options)``][2]，比如影响Bitmap配置的选项或其他解码时才会用到的参数。

为了生成磁盘缓存上的缓存键名称，以上的每个元素会被哈希化以创建一个单独的字符串键名，并在随后作为磁盘缓存上的文件名使用。

### 配置缓存

Glide提供一系列的选项，以允许你选择加载请求与Glide缓存如何交互。

#### 磁盘缓存策略（Disk Cache Strategy）

[``DiskCacheStrategy``][10] 可被[``diskCacheStrategy``][11]方法应用到每一个单独的请求。 目前支持的策略允许你阻止加载过程使用或写入磁盘缓存，选择性地仅缓存无修改的原生数据，或仅缓存变换过的缩略图，或是兼而有之。

默认的策略叫做[``AUTOMATIC``][12]，它会尝试对本地和远程图片使用最佳的策略。当你加载远程数据（比如，从URL下载）时，``AUTOMATIC``策略仅会存储未被你的加载过程修改过(比如，变换，裁剪--译者注)的原始数据，因为下载远程数据相比调整磁盘上已经存在的数据要昂贵得多。对于本地数据，``AUTOMATIC``策略则会仅存储变换过的缩略图，因为即使你需要再次生成另一个尺寸或类型的图片，取回原始数据也很容易。

指定[``DiskCacheStrategy``][10]非常容易:

```java
GlideApp.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.ALL)
  .into(imageView);
```

#### 仅从缓存加载图片

某些情形下，你可能希望只要图片不在缓存中则加载直接失败（*比如省流量模式？--译者注*）。如果要完成这个目标，你可以在单个请求的基础上使用[``onlyRetrieveFromCache``][8]方法：

```java
GlideApp.with(fragment)
  .load(url)
  .onlyRetrieveFromCache(true)
  .into(imageView);
```

如果图片在内存缓存或在磁盘缓存中，它会被展示出来。否则只要这个选项被设置为true，这次加载会视同失败。

#### 跳过缓存

如果你想确保一个特定的请求跳过磁盘和/或内存缓存（*比如，图片验证码 --译者注*），Glide也提供了一些替代方案。

仅跳过内存缓存，请使用[``skipMemoryCache()``][15]:

```java
GlideApp.with(fragment)
  .load(url)
  .skipMemoryCache(true)
  .into(view);
```

仅跳过磁盘缓存，请使用[``DiskCacheStrategy.NONE``][16]:

```java
GlideApp.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.NONE)
  .into(view);
```

这两个选项可以同时使用:

```java
GlideApp.with(fragment)
  .load(url)
  .diskCacheStrategy(DiskCacheStrategy.NONE)
  .skipMemoryCache(true)
  .into(view);
```

虽然提供了这些办法让你跳过缓存，但你通常应该不会想这么做。从缓存中加载一个图片，要比拉取-解码-转换成一张新图片的完整流程快得多。

如果你只是想更新缓存中的某个条目，请继续阅读下面关于[``invalidation``][17]一节的介绍。

#### Implementation

如果内置的选项不满足你的需求，你也可以编写你自己的[``DiskCache``][13]实现。请查看[配置][14]页获得更多信息。

### Cache Invalidation
因为磁盘缓存使用的是哈希键，所以**并没有**一个比较好的方式来简单地删除某个特定url或文件路径对应的所有缓存文件。如果你只允许加载或缓存原始图片的话，问题可能会变得更简单，但因为Glide还会缓存缩略图和提供多种变换(`transformation`)，它们中的任何一个都会导致在缓存中创建一个新的文件，而要跟踪和删除一个图片的所有版本无疑是困难的。

在实践中，使缓存文件无效的最佳方式是在内容发生变化时（url，uri，文件路径等）更改你的标识符。

#### Custom Cache Invalidation
因为通常改变标识符比较困难或者根本不可能，所以Glide也提供了[``签名``][1]API来混合（你可以控制的）额外数据到你的缓存键中。签名(`signature`)适用于媒体内容，也适用于你可以自行维护的一些版本元数据。

* MediaStore 内容 - 对于媒体存储内容，你可以使用Glide的[``MediaStoreSignature``][3]类作为你的签名。``MediaStoreSignature`` 允许你混入修改时间、MIME类型，以及item的方向到缓存键中。这三个属性能够可靠地捕获对图片的编辑和更新，这可以允许你缓存媒体存储的缩略图。
* 文件 - 你可以使用[``ObjectKey``][4]来混入文件的修改日期。
* Url - 尽管最好的让url失效的办法是让server保证在内容变更时对URL做出改变，你仍然可以使用[``ObjectKey``][4]来混入任意数据（比如版本号）。

将签名传入加载请求很简单：

```java
GlideApp.with(yourFragment)
    .load(yourFileDataModel)
    .signature(new ObjectKey(yourVersionMetadata))
    .into(yourImageView);
```

媒体存储签名对于MediaStore数据来说也很直接：
```java
GlideApp.with(fragment)
    .load(mediaStoreUri)
    .signature(new MediaStoreSignature(mimeType, dateModified, orientation))
    .into(view);
```

你还可以定义你自己的签名，只要实现[``Key``][5]接口就好。请确保正确地实现``equals()``, ``hashCode()`` 和``updateDiskCacheKey()`` 方法:

```java
public class IntegerVersionSignature implements Key {
    private int currentVersion;

    public IntegerVersionSignature(int currentVersion) {
         this.currentVersion = currentVersion;
    }
   
    @Override
    public boolean equals(Object o) {
        if (o instanceof IntegerVersionSignature) {
            IntegerVersionSignature other = (IntegerVersionSignature) o;
            return currentVersion = other.currentVersion;
        }
        return false;
    }
 
    @Override
    public int hashCode() {
        return currentVersion;
    }

    @Override
    public void updateDiskCacheKey(MessageDigest md) {
        messageDigest.update(ByteBuffer.allocate(Integer.SIZE).putInt(signature).array());
    }
}
```

请记住，为了避免降低性能，您将需要在后台批量加载任何版本元数据，以便在要加载图像时即已处于可用状态。

如果这些努力都无法奏效，您不能更改标识符，也不能跟踪任何合理的版本元数据的情况下，也可以使用[``diskCacheStrategy（）`][6]和[`DiskCacheStrategy.NONE``][7]来完全禁用磁盘缓存。

[1]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#signature-com.bumptech.glide.load.Key-
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/Option.html
[3]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/signature/MediaStoreSignature.html
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/signature/ObjectKey.html
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/Key.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#diskCacheStrategy-com.bumptech.glide.load.engine.DiskCacheStrategy-
[7]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/DiskCacheStrategy.html#NONE
[8]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#onlyRetrieveFromCache-boolean-
[9]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[10]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/DiskCacheStrategy.html
[11]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#diskCacheStrategy-com.bumptech.glide.load.engine.DiskCacheStrategy-
[12]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/DiskCacheStrategy.html#AUTOMATIC
[13]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/cache/DiskCache.html
[14]: {{ site.url }}/glide/doc/configuration.html#disk-cache
[15]: {{ site.rul }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html#skipMemoryCache-boolean-
[16]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/engine/DiskCacheStrategy.html#NONE
[17]: {{ site.url }}/glide/doc/caching.html#cache-invalidation
