---
layout: page
title: "编写定制的 ModelLoader"
category: tut
date: 2017-12-05 19:54:12
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/resourcereuse.html){:target="_blank"}

* TOC
{:toc}

虽然 Glide 内置了大部分常用模型（URL, Uri, 文件路径等）的支持，你还是可能偶尔会遇到一种 Glide 不支持的类型。你也可能会遇到需要定制或调整 Glide 默认行为的情况。你甚至可能会想要集成一种新的拉取图片的方法，或更换 Glide 目前支持的 [集成库][3] 之外的网络库。

好在 Glide 是可扩展的。要添加对一种新的模型（Model）类型的支持，你需要按照以下步骤来执行：

1. 实现一个 [``ModelLoader``][1]；
2. 实现一个 [``DataFetcher``][2]，它可被你的 [``ModelLoader``][1] 返回；
3. 将你的新 [``ModelLoader``][3] 注册到 Glide ，使用 [``AppGlideModule``][4] (或者，如果你正在改造的是一个程序库而不是应用，你需要使用 [``LibraryGlideModule``][5])。

所以让我们跟着这个步骤来，尝试实现一个定制的 ``ModelLoader``，它接受 Base64 编码的图片字符串并使用 Glide 来解码。请注意如果你真要在你的应用中这么做的话，有一个更好的办法是在你的 ``ModelLoader`` 中取回 Base64 编码字符串，这样如果 Glide 之前缓存过你的图片，则可以避免加载图片到内存所需要耗费的 CPU 和内存。

但就我们这片文章的目的而言，我们希望提供一个简单的例子了来加载 Base64 图片，虽然它在真实世界里可能有些低效。

## 编写一个 ModelLoader

第一步是实现一个 [``ModelLoader``][1] 接口。在开始之前，我们需要决定两件事情：

1. 我们要处理什么类型的模型 (Model)？
2. 对于那种模型我们应该产出什么类型的数据 (Data)？

在这个例子中，我们希望处理 Base64 编码的字符串，所以 ``String`` 可能是一个合适的 ``Model`` 类型。稍后我们将需要比单纯的随机字符串更具体的东西，不过现在 String 已经足够开始我们的实现了。

接下来我们需要决定我们应该从 String 中尝试产出什么类型的数据。默认情况下，Glide 提供了两种数据类型的图片解码器 (decoders)：

1. [``InputStream``][7]
2. [``ByteBuffer``][8]

Glide 也为解码视频提供了一个默认的 [``ParcelFileDescriptor``][9]支持。

因为我们是要解码图片，所以我们大概要使用 [``InputStream``][7] 或 [``ByteBuffer``][8]。因为我们已经在内存中拥有了所有的数据，而且我们将用于做实际解码的 [``Base64``][10] 的方法返回 ``byte[]`` ，所以 [``ByteBuffer``][8] 可能是最好的选择。

### 一个空实现

现在我们已经知道了 ``Model`` 和 ``Data`` 的类型，我们可以创建一个类接受正确的参数类型并返回默认值：

```java
package judds.github.com.base64modelloaderexample;

import android.support.annotation.Nullable;
import com.bumptech.glide.load.Options;
import com.bumptech.glide.load.model.ModelLoader;
import java.io.InputStream;
import java.nio.ByteBuffer;

/**
 * Loads an {@link InputStream} from a Base 64 encoded String.
 */
public final class Base64ModelLoader implements ModelLoader<String, ByteBuffer> {

  @Nullable
  @Override
  public LoadData<ByteBuffer> buildLoadData(String model, int width, int height, Options options) {
    return null;
  }

  @Override
  public boolean handles(String model) {
    return false;
  }
}
```
当然这个 ``ModelLoader`` 不会有什么作用，但这是一个开始。

### 实现 ``handles()``

下一步是实现 [``handles()``][11] 方法。正如我们之前提到过的，String 类型可能代表着多种模型类型，包括：

1. URL
2. Uri
3. File

``handles()`` 方法允许 ``ModelLoader`` 高效地检查每个模型以避免加载不支持的类型。

为了简化我们的工作，这里让我们假定我们的 Base64 编码后的字符串将被实际使用 [data URI][12] 的方式传入。 handles 方法的目标是识别任何 Glide 传入的 String 参数，并为那些匹配 data uri 格式的串返回 ``true``。

还好，data URI 格式很直接，因此我们可以只是检查 ``data:`` 前缀：

```java
  @Override
  public boolean handles(String model) {
    return model.startsWith("data:");
  }
```

取决于我们对该实现所要求的鲁棒性程度，我们可能还希望检查嵌入的图片类型或数据格式，以避免我们加载错误的数据，例如一个 HTML 页面。但在这里为了简化起见我们将省略这些问题。

### 实现 ``buildLoadData``

现在我们可以识别 data URI 了，下一步是提供一个对象，它将在给定模型、尺寸和选项的 ``ByteBuffer`` 未被缓存的时候做实际的解码工作。为此我们需要实现 [``buildLoadData``][13] 方法。

作为一个开始，我们的方法仅仅返回 ``null``，这是完全合法的，虽然几乎没什么用：

```java
  @Nullable
  @Override
  public LoadData<ByteBuffer> buildLoadData(String model, int width, int height, Options options) {
    return null;
  }
```

为了让这个方法更有用，让我们从返回一个新的 [``LoadData<ByteBuffer>``][14] 开始。为此你需要做两件事情：

1. 一个 [``Key``][15] ，它将被用于磁盘缓存键的一部分 (模型的 ``equals`` 和 ``hashCode()`` 方法被用于内存缓存键)。
2. 一个 [``DataFetcher``][2]，它可以从我们的特定模型中建立一个 ``ByteBuffer``。

#### 选择 ``Key``

在这个例子，用于磁盘缓存键的 [``Key``][15] 是很直接的，因为我们的模型类型是 ``String`` 。如果你有一个模型类型可以被使用 [``toString()``][16] 序列化，则你可以直接将这个模型传入一个新的 [``ObjectKey``][17]：

```java
Key diskCacheKey = new ObjectKey(model);
```

否则，你就要在这里实现 [``Key``][15] 接口，并确保 ``equals``, ``hashCode()``，以及 [``updateDiskCacheKey``][18] 方法都被合理地填写并可以唯一地 (unique) 、一致 (consistent) 地标识你的特定模型。

因为我们现在只是与字符串字面值打交道，因此 [``ObjectKey``][17] 就可以满足需求。

#### 选择 DataFetcher

因为我们正在添加一个新模型的支持，我们实际上将需要定制一个 DataFetcher 。在某些情况下，[``ModelLoader``][1] 可能只是对 [``buildLoadData``][13] 传入的参数做一些简单的解析然后委托 (delegate) 给另一个 [``ModelLoader``][1]，但这里我们可没这么幸运。

现在我们先传入一个 ``null``（虽然它并不是一个合法的值），然后转到我们实际的 ``DataFetcher`` 实现中去：

```java
  @Nullable
  @Override
  public LoadData<ByteBuffer> buildLoadData(String model, int width, int height, Options options) {
    return new LoadData<>(new ObjectKey(model), /*fetcher=*/ null);
  }
```

## 编写 DataFetcher

和 [``ModelLoader``][1] 类似， [``DataFetcher``][2] 接口是泛型的，它要求我们指定所期待的返回类型。好在我们已经决定了我们是想要加载 ``ByteBuffer``，所以这里做决定并不困难。

因此，我们可以很快创建一个存根 (stub) 实现出来：

```java
public class Base64DataFetcher implements DataFetcher<ByteBuffer> {

  @Override
  public void loadData(Priority priority, DataCallback<? super ByteBuffer> callback) {}

  @Override
  public void cleanup() {}

  @Override
  public void cancel() {}

  @NonNull
  @Override
  public Class<ByteBuffer> getDataClass() {
    return null;
  }

  @NonNull
  @Override
  public DataSource getDataSource() {
    return null;
  }
}
```

虽然这里有很多个方法，但它们中的大部分都很容易实现。

#### getDataClass

``getDataClass`` 没什么好讨论的，我们要加载 ``ByteBuffer``：

```java
  @NonNull
  @Override
  public Class<ByteBuffer> getDataClass() {
    return ByteBuffer.class;
  }
```

#### getDataSource

``getDataSource`` 也基本不重要，但它有一些影响。Glide 对本地图片和远程图片的默认缓存策略是不同的。Glide 假定获取本地图片是简单廉价的，因此我们默认在它们被下采样和变换之后才缓存它们。相反，Glide 假定获取远程图片是困难而且昂贵的，因此我们将默认缓存获取到的原始数据。

对于 Base64 ``String``，你的应用最好的选择可能取决于你如何获取到这些 ``String``。如果它们是从一个本地数据库取得的，则 ``DataSource.LOCAL`` 最有意义。如果你每次通过 HTTP 取回它们，则 ``DataSource.REMOTE`` 比较合适。

现在让我们假设 ``String`` 是从本地获取的：

```java
  @NonNull
  @Override
  public DataSource getDataSource() {
    return DataSource.LOCAL;
  }
```

#### cancel

对于可以取消的网络连接库或长时间加载，实现 ``cancel()`` 方法是一个好主意。这将帮助加速其他队列里的加载，并节约一些 CPU ，内存或其他资源。

在我们这个场景中， [``Base64``][10] 没有提供取消的 API ，所以我们可以把这里留空：

```java
  @Override
  public void cancel() {
    // Intentionally empty.
  }

```

#### cleanup

``cleanup()`` 方法很有意思。如果你正在加载一个 [``InputStream``][7] 或打开任何 I/O 类的资源，你肯定要在 ``cleanup()`` 方法中关闭并清理这些 ``InputStream`` 或资源。

然而，在我们这个场景下我们仅仅只是要解码一个内存中的模型到内存中的数据。因此，这里没有东西需要清理，所以我们的方法也可以留空：

```java
  @Override
  public void cleanup() {
    // Intentionally empty only because we're not opening an InputStream or another I/O resource!
  }
```

警告！请务必确认，如果你是打开 I/O 资源或一个 ``InputStream``，你必须在这里关闭它们！我们仅仅是因为没有这么做所以这里才留了空 :)

#### loadData

现在最有趣的地方到了！``loadData()`` 是 Glide 所期待你做那些繁重工作的地方。你可以安排一个异步任务，开启一个网络请求，从磁盘加载数据，或做任何你想做的事情。``loadData()`` 只会被 Glide 的某个后台线程所调用。一个给定的 ``DataFetcher`` 在同一时间只会被一个后台线程使用，因此它不需要做到线程安全。然而，多个 ``DataFetcher`` 可能会被并行执行，因此 ``DataFetcher`` 所访问的任何共享资源应当是线程安全的。

``loadData()`` 提供了两个参数：

1. [``Priority``][19]，如果你正在使用网络库或其他队列系统，它可以用于含有优先级的请求。
2. [``DataCallback``][20]，你需要使用你解码出来的数据来调用它，如果因为任何原因解码失败，你也可以使用错误消息来调用。

我们可以排队一个异步任务然后异步调用 [``DataCallback``][20]，也可以在 ``loadData()`` 方法中做一些工作然后直接调用 [``DataCallback``][20]。

在我们的例子中我们没有网络任务或队列需要调用，因此我们可以直接使用一行代码完成。

请注意，这里遗漏了一个重要的事情，我们居然没有对模型的引用！这是因为，每个 [``DataFetcher``][2] 都是一个简单的闭包，可以用于从特定的模型中获取数据。因此，Glide 希望你在 [``DataFetcher``][2] 的构造器中传入一个模型：

```java
  private final String model;

  Base64DataFetcher(String model) {
    this.model = model;
  }
```

事实证明，我们的 ``loadData()`` 方法实际上很简单。我们只需要将 Base64 部分从 data Uri 中解析出来即可：

```java
  private String getBase64SectionOfModel() {
    // See https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs.
    int startOfBase64Section = model.indexOf(',');
    return model.substring(startOfBase64Section + 1);
  }
```

然后我们需要对着个 Base64 部分解码成 ``byte[]``： 

```java
  byte[] data = Base64.decode(base64Section, Base64.DEFAULT);
```

并转换为一个 ``ByteBuffer``：

```java
  ByteBuffer byteBuffer = ByteBuffer.wrap(data);
```

然后我们只需要使用这个解码出来的 ``ByteBuffer`` 调用回调即可：

```java
  callback.onDataReady(byteBuffer);
```

完成这一切之后，我们就有了一个 ``loadData()`` 的完整实现：

```java
  @Override
  public void loadData(Priority priority, DataCallback<? super ByteBuffer> callback) {
    String base64Section = getBase64SectionOfModel();
    byte[] data = Base64.decode(base64Section, Base64.DEFAULT);
    ByteBuffer byteBuffer = ByteBuffer.wrap(data);
    callback.onDataReady(byteBuffer);
  }

  private String getBase64SectionOfModel() {
    // See https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs.
    int startOfBase64Section = model.indexOf(',');
    return model.substring(startOfBase64Section + 1);
  }
```

#### 完整的 DataFetcher

现在让我们把在 ``DataFetcher`` 中实现的方法放到一起，看看完整的代码：

```java
package judds.github.com.base64modelloaderexample;

import android.support.annotation.NonNull;
import android.util.Base64;
import com.bumptech.glide.Priority;
import com.bumptech.glide.load.DataSource;
import com.bumptech.glide.load.data.DataFetcher;
import java.nio.ByteBuffer;

public class Base64DataFetcher implements DataFetcher<ByteBuffer> {

  private final String model;

  Base64DataFetcher(String model) {
    this.model = model;
  }

  @Override
  public void loadData(Priority priority, DataCallback<? super ByteBuffer> callback) {
    String base64Section = getBase64SectionOfModel();
    byte[] data = Base64.decode(base64Section, Base64.DEFAULT);
    ByteBuffer byteBuffer = ByteBuffer.wrap(data);
    callback.onDataReady(byteBuffer);
  }

  private String getBase64SectionOfModel() {
    // See https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs.
    int startOfBase64Section = model.indexOf(',');
    return model.substring(startOfBase64Section + 1);
  }

  @Override
  public void cleanup() {
    // Intentionally empty only because we're not opening an InputStream or another I/O resource!
  }

  @Override
  public void cancel() {
    // Intentionally empty.
  }

  @NonNull
  @Override
  public Class<ByteBuffer> getDataClass() {
    return ByteBuffer.class;
  }

  @NonNull
  @Override
  public DataSource getDataSource() {
    return DataSource.LOCAL;
  }
}
```

## 完成 ModelLoader

回想我们刚刚在写 [``ModelLoader``][1] 时，我们的 [``buildLoadData``][13] 方法有一些不完整，我们返回了一个 ``null``，而不是一个合适的 ``DataFetcher``：

```java
  @Nullable
  @Override
  public LoadData<ByteBuffer> buildLoadData(String model, int width, int height, Options options) {
    return new LoadData<>(new ObjectKey(model), /*fetcher=*/ null);
  }
```

现在我们已经有了一个 ``DataFetcher`` 实现，我们可以将它填进去：

```java
  @Override
  public LoadData<ByteBuffer> buildLoadData(String model, int width, int height, Options options) {
    return new LoadData<>(new ObjectKey(model), new Base64DataFetcher(model));
  }
```

我们可以删除 ``@Nullable`` 因为实际上在我们的实现中永远不会返回 ``null``。 如果我们委托给另一个封装的 ``ModelLoader``，我们需要检查那个 ``ModelLoader`` 的返回值并确保它返回 ``null`` 时我们也返回 ``null``。实际上，在某些情形下我们可能会在尝试解析数据时发现我们并不能加载它，这时候我们也可以返回 ``null``。

## 使用 Glide 注册我们的 ModelLoader

我们已经接近完工，不过还差最后一步。我们的 ``ModelLoader`` 实现已经完成了，但完全没有使用它。要完成我们的项目，我们需要告诉 Glide 以使得 Glide 知道需要使用它。

### 添加 AppGlideModule

为此，我们将按照 [配置][21] 里的步骤为我们的应用添加一个 [``AppGlideModule``][22]，如果你还没有做的话：

```java
package judds.github.com.base64modelloaderexample;

import com.bumptech.glide.annotation.GlideModule;
import com.bumptech.glide.module.AppGlideModule;

@GlideModule
public class MyAppGlideModule extends AppGlideModule { }
```

不要忘了也给你的 build.gradle 文件里添加对 Glide 的注解解析器的依赖：

```groovy
annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0'
```

接下来我们要获取 Glide 的 [``Registry``][23]，因此我们将在我们的 ``AppGlideModule`` 中实现 [``registerComponents``][24] 方法：

```java
package judds.github.com.base64modelloaderexample;

import com.bumptech.glide.annotation.GlideModule;
import com.bumptech.glide.module.AppGlideModule;

@GlideModule
public class MyAppGlideModule extends AppGlideModule { 
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    // TODO: implement this.
  }
}
```

### 选择注册方法

为了告知 Glide 我们的 ``ModelLoader``，我们需要使用 ``ModelLoader`` 的方法之一将其添加到 [``Registry``][23]。
 
 ``ModelLoader`` 是按照它们注册的顺序存储在一个列表中的。当你开启一个新的加载后，Glide 将查找所有已注册的 ``ModelLoader``，使用你的提供的模型类型按照它们的注册顺序逐个尝试。
 
因此，如果你正在添加一种新的 Model 类型支持，你通常想要 [``prepend``][25] 你的 ``ModelLoader``，这样 Glide 将在默认的 ``ModelLoader`` 之前尝试它。在我们的例子中，我们确实是在做这件事，添加一种新模型类型，因此我们使用 [``prepend``][25]。

不过，这里还有一点小麻烦。[``prepend``][25] 要求一个 [``ModelLoaderFactory``][26]，而不是一个 [``ModelLoader``][1]。这允许你委托给其他的 ``ModelLoader``，甚至它们可以被动态注册，但这也使得你在定义新的加载器时需要再多实现一个接口。

### 实现 ModelLoaderFactory

还好，[``ModelLoaderFactory``][26] 接口非常简单，所以我们很容易添加它：

```java
package judds.github.com.base64modelloaderexample;

import com.bumptech.glide.load.model.ModelLoader;
import com.bumptech.glide.load.model.ModelLoaderFactory;
import com.bumptech.glide.load.model.MultiModelLoaderFactory;
import java.nio.ByteBuffer;

public class Base64ModelLoaderFactory implements ModelLoaderFactory<String, ByteBuffer> {

  @Override
  public ModelLoader<String, ByteBuffer> build(MultiModelLoaderFactory unused) {
    return new Base64ModelLoader();
  }

  @Override
  public void teardown() { 
    // Do nothing.
  }
}
```

[``ModelLoaderFactory``][26] 的类型需要完全匹配我们在 ``ModelLoader`` 中使用的类型。

### 注册 ModelLoader

最后，我们只需要更新我们的 ``AppGlideModule`` 来使用我们的新 Factory :

```java
@GlideModule
public class MyAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.prepend(String.class, ByteBuffer.class, new Base64ModelLoaderFactory());
  }
}
```

这就行了！

现在我们可以使用任何带有 Base64 编码图片的 Data URI 并让 Glide 来加载它：

```java
String dataUri = "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYA..."
Glide.with(fragment)
  .load(dataUri)
  .into(imageView);
```

## 完整示例

这里有一个完整的示例项目，它使用了我们写的这些代码： [https://github.com/sjudd/Base64ModelLoaderExample](https://github.com/sjudd/Base64ModelLoaderExample/commit/ae004dc4b325ee39814f197cc196d7371fbccdf1)

在示例中的提交信息是按照我们编写的顺序来的：

1. [An empty project with a blank Activity](https://github.com/sjudd/Base64ModelLoaderExample/commit/d9ee7eb9285ed1a7279cc085b3abd0f1369f92dd)
2. [A ModelLoader that just implements handles](https://github.com/sjudd/Base64ModelLoaderExample/commit/83ae04155b79056487299f65f70e172c11ff53ae)
3. [A data fetcher implementation](https://github.com/sjudd/Base64ModelLoaderExample/commit/70a4facda7504c375ca9150ea2a6789077bbd7e1)
4. [A ModelLoader with a complete buildLoadData implementation](https://github.com/sjudd/Base64ModelLoaderExample/commit/8c641cb1be3afbf1ff0d8bcba7b37b1778f06dc4)
5. [An AppGlideModule and registered ModelLoader](https://github.com/sjudd/Base64ModelLoaderExample/commit/d4e1cd9dcc011bb6d2910301c5783290fbe3bb89)
6. [An example data Uri loaded into a View](https://github.com/sjudd/Base64ModelLoaderExample/commit/ae004dc4b325ee39814f197cc196d7371fbccdf1)

## 注意事项

实际上，Glide 已经支持了 Data URI，所以如果你只是想使用 Data URI 加载 Base64 字符串的话并不需要做什么。这份代码只是作为示例目的。

如果你确实想实现 Data uri 的支持，你可能想在 ``handles()`` 方法或 ``DataFetcher`` 中做更多的错误检查，来处理索引超出 URI 边界时被截断的字符串。

## 高级使用场景

我们上面的示例并不适合某些高级使用场景。我们将在这里单独讨论。

### 委托(delegate)给另一个 ModelLoader

我们之前提到过，但没有详细讨论的一个事情是，Glide 允许你在一个定制的 ModelLoader 中委托给一个已存在的 ModelLoader 。拥有一个 Glide 不理解的自定义模型类型并不罕见，但是能够相对容易地从自定义类型中提取一个 Glide 能够理解的模型类型，如 URL ，Uri 或文件路径。

使用委托技术，你可以支持你的定制模型，而将 Glide 能理解和委托的模型提取出来。

举个例子，在 Glide 的[Giphy 示例应用][28] 中，我们从 Giphy 的 API 中获取了一个 JSON 对象，包含一个 URL 集合：

```java
/**
 * A POJO mirroring an individual GIF image returned from Giphy's API.
 */
public static final class GifResult {
  public String id;
  GifUrlSet images;

  @Override
  public String toString() {
    return "GifResult{" + "id='" + id + '\'' + ", images=" + images
        + '}';
  }
}
```

尽管我们可以在我们的 View 逻辑中提取出 URL，就像这样：

```java
Glide.with(fragment)
  .load(gifResult.images.fixed_width)
  .into(imageView);
```

但如果我们可以直接传入 ``GifResult``，这个代码将更简洁：

```java
Glide.with(fragment)
  .load(gifResult)
  .into(imageView);
```

如果我们必须重写所有的 URL 逻辑来才能完成这样的想法，这显然不值得。然而我们可以委托，因此最后我们得到了一个很简单的 ``ModelLoader`` 实现：

```java
public final class GiphyModelLoader extends BaseGlideUrlLoader<Api.GifResult> {
  private final ModelLoader<GlideUrl, InputStream> urlLoader;

  private GiphyModelLoader(ModelLoader<GlideUrl, InputStream> urlLoader) {
    this.urlLoader = urlLoader;
  }

  @Override
  public boolean handles(@NonNull Api.GifResult model) {
    return true;
  }

  @Override
  public LoadData<InputStream> buildLoadData(
      @NonNull Api.GifResult model, int width, int height, @NonNull Options options) {
    return urlLoader.buildLoadData(model.images.fixed_width, width, height, options);
  }
}
```

``ModelLoader`` 的构造器所要求的 ``ModelLoader<GlideUrl, InputStream>`` 通过 ``ModelLoaderFactory`` 提供，对于一个特定的模型和数据类型（在这个场景中是 ``GlideUrl`` 和 ``InputSteam``），它可以查找当前注册的 ``ModelLoader``：

```java
/**
 * The default factory for {@link com.bumptech.glide.samples.giphy.GiphyModelLoader}s.
 */
public static final class Factory implements ModelLoaderFactory<GifResult, InputStream> {
  @Override
  public ModelLoader<Api.GifResult, InputStream> build(MultiModelLoaderFactory multiFactory) {
    return new GiphyModelLoader(multiFactory.build(GlideUrl.class, InputStream.class));
  }

  @Override public void teardown() {}
}
```

Glide 的 ``ModelLoader`` 采用懒惰构建，且会在新注册了 ``ModelLoader`` 之后将旧的剔除 (tear down)，因此当你使用这种委托模式时，你永远不会用到一个旧的 ``ModelLoader``。因此，我们的 ``GiphyModelLoader`` 是与我们实际用于加载 URL 的网络库完全解耦的。

[``MultiModelLoaderFactory``][29] 可用于获取任何已注册的 ``ModelLoader``。如果同一个给定的类型有多个 ``ModelLoader``注册，则 ``MultiModelLoaderFactory`` 将返回一个被包装 (wrap) 过的 ``ModelLoader`` ，它将依次尝试每个 [``handles()``][11] 方法返回 ``true`` 的 ``ModelLoader``，直到其中一个成功为止。

### 在 ModelLoader 中处理自定义尺寸

即使用了委托，上面这个 Giphy 的例子好像还是为了一个更轻便好看的 API 做了不少工作。不过，拥有你自己的 ``ModelLoader`` 还有一些额外的好处，特别是像 Giphy 这样你可以从多个 URL 中选择的 API 。

尽管我们之前在 Base64 ``ModelLoader`` 中实现了 [``buildLoadData()``][13]，但我们从没有讨论过这里提供的除 model 之外的参数。``buildLoadData()`` 还传入了一个宽度和高度，可以用于选择最合适尺寸的图片，如此可通过只拉取、缓存和解码最小的必要图片来节省带宽、内存、CPU，以及磁盘空间等。

``buildLoadData()`` 传入的宽度和高度可能是被 [``Target``][30] 提供的，或如果你指定的话则为请求 [``override()``][31] 的尺寸。如果你正在往 ``ImageView`` 中加载数据，则这个宽度和高度即为 ``ImageView`` 的宽高（同样，除非你使用了 ``override()``）。如果你使用了 ``Target.SIZE_ORIGINAL``，那么宽高将为常量值 ``Target.SIZE_ORIGINAL``。

实际的 [``GiphyModelLoader``][32] 有一个简单的示例，使用 ``buildLoadData()`` 提供的尺寸来选取最佳的 URL ：

```java
@Override
protected String getUrl(Api.GifResult model, int width, int height, Options options) {
  Api.GifImage fixedHeight = model.images.fixed_height;
  int fixedHeightDifference = getDifference(fixedHeight, width, height);
  Api.GifImage fixedWidth = model.images.fixed_width;
  int fixedWidthDifference = getDifference(fixedWidth, width, height);
  if (fixedHeightDifference < fixedWidthDifference && !TextUtils.isEmpty(fixedHeight.url)) {
    return fixedHeight.url;
  } else if (!TextUtils.isEmpty(fixedWidth.url)) {
    return fixedWidth.url;
  } else if (!TextUtils.isEmpty(model.images.original.url)) {
    return model.images.original.url;
  } else {
    return null;
  }
}
```

在 Glide 的 [Flickr 示例应用][33]，我们也看到了一个类似的模式，尽管它使用了多种缩略尺寸在一定程度上增强了鲁棒性。

如果你能访问的 API 允许你指定特定尺寸或提供了大量的缩略图尺寸变种，那么使用一个定制的 ``ModelLoader`` 可以显著提升你的应用性能。

### BaseGlideUrlLoader

为了节省在编写定制 ``ModelLoader`` 时仅仅委托给默认网络库的一些模板代码，Glide 包含了一个 [``BaseGlideUrlLoader``][34] 抽象类。我们之前的一些示例，包括 [``GiphyModelLoader``][32] 和 [``FlickrModelLoader``][35] 都使用了这个类。

``BaseGlideUrlLoader`` 提供了一些基础的缓存，以最小化 ``String`` 分配，以及两个简便方法：

1. [``getUrl()``][37]，它为一个给定的模型返回一个 ``String`` URL；
2. [``getHeaders``][38]，可选实现，用于为一个给定的模型和尺寸返回一个 HTTP [``Headers``][36]，如果你需要添加一个授权或其他类型的 header 的话。 

#### getUrl

如果你阅读了之前关于处理自定义尺寸的章节，你可能已经注意到我们在 [``GiphyModelLoader``][32] 中引用的方法并非 [``buildLoadData()``]。它实际上仅仅是 ``getUrl`` 这个简便方法：

```java
@Override
protected String getUrl(Api.GifResult model, int width, int height, Options options) {
  Api.GifImage fixedHeight = model.images.fixed_height;
  int fixedHeightDifference = getDifference(fixedHeight, width, height);
  Api.GifImage fixedWidth = model.images.fixed_width;
  int fixedWidthDifference = getDifference(fixedWidth, width, height);
  if (fixedHeightDifference < fixedWidthDifference && !TextUtils.isEmpty(fixedHeight.url)) {
    return fixedHeight.url;
  } else if (!TextUtils.isEmpty(fixedWidth.url)) {
    return fixedWidth.url;
  } else if (!TextUtils.isEmpty(model.images.original.url)) {
    return model.images.original.url;
  } else {
    return null;
  }
}
```

使用 ``BaseGlideUrlLoader`` 允许你跳过构造磁盘缓存键和 ``LoadData``，并允许你避免处理委托，只是 ``ModelLoader <GlideUrl，InputStream>`` 你必须传入构造函数。

#### getHeaders

尽管 Glide 的实例应用都不需要使用 ``getHeaders()``，但在拉取非公开图片时需要填入一些授权表单并不罕见。``getHeaders()`` 方法可以被选择实现，来为一个给定的模型返回任意合适的 HTTP 头。

例如，如果你有一个字符串授权令牌 (token)，你可以使用 [``LazyHeaders``][39] 类来编写，就像这样：
 
```java
@Nullable
@Override
protected Headers getHeaders(GifResult gifResult, int width, int height, Options options) {
  return new LazyHeaders.Builder()
      .addHeader("Authorization", getAuthToken())
      .build();
}
```

如果你的 ``getAuthToken()`` 方法特别昂贵，你应该使用 [``LazyHeaderFactory``][40] 来代替：

```java
@Override
protected Headers getHeaders(GifResult gifResult, int width, int height, Options options) {
  return new LazyHeaders.Builder()
      .addHeader("Authorization", new LazyHeaderFactory() {
        @Nullable
        @Override
        public String buildHeader() {
          return getAuthToken();
        }
      })
      .build();
}
```

使用 ``LazyHeaderFactory`` 将避免执行昂贵的调用，直到 ``DataFetcher`` 中发生 HTTP 请求。尽管 ``ModelLoader`` 方法是在后台线程中被调用，
``buildLoadData()`` 即使在对应的图片已经被缓存到 Glide 的磁盘缓存时也会被调用。因此，在 ``buildLoadData()`` 方法或任何 ``BaseGlideUrlLoader`` 方法中去做昂贵的工作是很浪费的，因为其结果可能根本不会使用到。使用 ``LazyHeaderFactory`` 将延迟其工作，这将显著节省昂贵的获取 header 的时间。

## 鸣谢 (Credits)

感谢 @jasonch 为 Glide 的 [Data Uri ModelLoader 实现][27] 所做的工作。

[1]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/ModelLoader.html
[2]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/data/DataFetcher.html
[3]: {{ site.baseurl }}/int/about.html
[4]: {{ site.baseurl }}/doc/configuration.html#applications
[5]: {{ site.baseurl }}/doc/configuration.html#libraries
[6]: https://github.com/bumptech/glide/issues/2677
[7]: https://developer.android.com/reference/java/io/InputStream.html
[8]: https://developer.android.com/reference/java/nio/ByteBuffer.html
[9]: https://developer.android.com/reference/android/os/ParcelFileDescriptor.html
[10]: https://developer.android.com/reference/android/util/Base64.html
[11]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/ModelLoader.html#handles-Model-
[12]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs
[13]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/ModelLoader.html#buildLoadData-Model-int-int-com.bumptech.glide.load.Options-
[14]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/ModelLoader.LoadData.html
[15]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/Key.html
[16]: https://developer.android.com/reference/java/lang/Object.html#toString()
[17]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/signature/ObjectKey.html
[18]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/Key.html#updateDiskCacheKey-java.security.MessageDigest-
[19]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/Priority.html
[20]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/data/DataFetcher.DataCallback.html
[21]: {{ site.baseurl }}/doc/configuration.html#applications
[22]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/module/AppGlideModule.html
[23]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/Registry.html
[24]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/module/LibraryGlideModule.html#registerComponents-android.content.Context-com.bumptech.glide.Glide-com.bumptech.glide.Registry-
[25]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/Registry.html#prepend-java.lang.Class-java.lang.Class-com.bumptech.glide.load.model.ModelLoaderFactory-
[26]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/ModelLoaderFactory.html
[27]: https://github.com/bumptech/glide/blob/c3dafde00a061bafcd43a739336ca3503af13a7d/library/src/main/java/com/bumptech/glide/load/model/DataUrlLoader.java#L19
[28]: {{ site.baseurl }}/ref/samples.html#giphy
[29]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/MultiModelLoaderFactory.html
[30]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/request/target/Target.html
[31]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/request/RequestOptions.html#override-int-int-
[32]: https://github.com/bumptech/glide/blob/b4b45791cca6b72345a540dcaa71a358f5706276/samples/giphy/src/main/java/com/bumptech/glide/samples/giphy/GiphyModelLoader.java#L31
[33]: {{ site.baseurl }}/ref/samples.html#flickr
[34]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html
[35]: https://github.com/bumptech/glide/blob/b4b45791cca6b72345a540dcaa71a358f5706276/samples/flickr/src/main/java/com/bumptech/glide/samples/flickr/FlickrModelLoader.java#L21
[36]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/Headers.html
[37]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html#getUrl-Model-int-int-com.bumptech.glide.load.Options-
[38]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html#getHeaders-Model-int-int-com.bumptech.glide.load.Options-
[39]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/LazyHeaders.html
[40]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/model/LazyHeaderFactory.html
