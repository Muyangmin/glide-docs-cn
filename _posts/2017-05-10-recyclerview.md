---
layout: page
title: "RecyclerView"
category: int
date: 2017-05-10 07:47:20
order: 4
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/int/recyclerview.html){:target="_blank"}

### 关于
RecyclerView 集成库使你在你的应用中能够使用 [``RecyclerViewPreloader``][2] ，它可以在用户滑动 RecyclerView 时自动加载稍微超前一些的图片。

配合使用正确的图片尺寸和高效率的磁盘缓存策略，这个库可以显著减少用户滑动图片列表时看到的加载指示器的数量。

### Gradle 依赖
要使用 RecyclerView 集成库，在你的 ``build.gradle`` 文件中添加一个依赖：
```groovy
compile ("com.github.bumptech.glide:recyclerview-integration:4.5.0") {
  // Excludes the support library because it's already included by Glide.
  transitive = false
}
```
当然你还需要确保你的应用中有 ``RecyclerView`` 的依赖并正在你的应用中使用 ``RecyclerView`` :) 。

### 设置
为了使用 ``RecyclerView`` 集成库，你需要做以下步骤：

1. 创建一个 [``PreloadSizeProvider``][3]
2. 创建一个 [``PreloadModelProvider``][6]
3. 创建一个 [``RecyclerViewPreloader``][2] 并将你前两步创建的 [``PreloadSizeProvider``][3] 和 [``PreloadModelProvider``][6] 赋值进去
4. 将你的 [``RecyclerViewPreloader``][2] 添加到你的 ``RecyclerView`` 做为一个 scroll listener。

上面的每一个步骤在下面会详细讲解。

#### PreloadSizeProvider

在添加完 gradle 依赖后，你接下来需要创建一个 [``PreloadSizeProvider``][3]。[``PreloadSizeProvider``][3] 负责保证你的 ``RecyclerViewPreloader`` 使用与你的适配器中 ``onBindViewHolder`` 方法一样的尺寸来加载图片。

如果你的 ``RecyclerView`` 里有统一的 ``View`` 尺寸、你使用 ``into(ImageView)``来加载图片并且你没有使用 ``override()`` 方法来设置一个不同的尺寸，那么你可以使用 [``ViewPreloadSizeProvider``][4]。

如果你使用 ``override()`` 方法或其他情况导致加载的图片尺寸并不完全匹配你的 ``View`` 尺寸，你可以使用 [``FixedPreloadSizeProvider``][5]。

如果在你的 ``RecyclerView`` 中决定给定位置下图片尺寸的逻辑并不适合上述两种场景，你还可以编写你自己的 [``PreloadSizeProvider``]的实现。

如果你使用固定尺寸加载你的图片，通常 [``FixedPreloadSizeProvider``][5] 是最简单的：
```java
private final imageWidthPixels = 1024;
private final imageHeightPixels = 768;

...

PreloadSizeProvider sizeProvider = 
    new FixedPreloadSizeProvider(imageWidthPixels, imageHeightPixels);
```

#### PreloadModelProvider

下一步是实现你的 [``PreloadModelProvider``][6]。[``PreloadModelProvider``][6]主要做两个事情，第一个是收集并返回一个给定位置的 ``Model``（即你传给 Glide 的 ``load(Object)`` 方法的对象，例如 URL 或 文件路径）列表。第二是取出一个 ``Model`` 并生产一个 ``RequestBuilder``，用于预加载给定的 ``Model`` 到内存中。

例如，假设我们有一个 ``RecyclerView``，它包含一个图片的 url 列表，``RecyclerView``的每个位置展示一个 URL 。然后，假设你在你的 ``RecyclerView.Adapter`` 的 ``onBindViewHolder`` 方法中这样加载图片：

```java
private List<String> myUrls = ...;

...

@Override
public void onBindViewHolder(ViewHolder viewHolder, int position) {
  ImageView imageView = ((MyViewHolder) viewHolder).imageView;
  String currentUrl = myUrls.get(position);

  GlideApp.with(fragment)
    .load(currentUrl)
    .override(imageWidthPixels, imageHeightPixels)
    .into(imageView);
}
```

这样的话，你的 [``PreloadModelProvider``][6] 实现大概看起来像这样：

```java
private List<String> myUrls = ...;

...

private class MyPreloadModelProvider implements PreloadModelProvider {
  @Override
  @NonNull
  List<U> getPreloadItems(int position) {
    String url = myUrls.get(position);
    if (TextUtils.isEmpty(url)) {
      return Collections.emptyList();
    }
    return Collections.singletonList(url);
  }

  @Override
  @Nullable
  RequestBuilder getPreloadRequestBuilder(String url) {
    return 
      GlideApp.with(fragment)
        .load(url) 
        .override(imageWidthPixels, imageHeightPixels);
  }
}
```

有一点十分重要，从 ``getPreloadRequestBuilder`` 中返回的 ``RequestBuilder`` ，必须与你从 ``onBindViewHolder`` 里启动的请求使用完全相同的一组选项 (占位符， 变换等) 和完全相同的尺寸。如果对于一个给定位置，两种方法提供的任何选项不完全相同，你的预加载请求将被浪费，因为它加载的图片会被缓存，但是缓存键却与你在 ``onBindViewHolder`` 里图片的缓存键不同。如果您无法使这些缓存键匹配，请参阅[调试页面] [7]。

如果对于一个给定的位置你不需要加载任何东西，你可以从 ``getPreloadItems`` 返回一个空列表。如果你稍晚时候发现你无法从一个给定的 ``Model`` 创建一个 ``RequestBuilder``，你可以从 ``getPreloadRequestBuilder`` 方法返回 ``null`` 。

#### RecyclerViewPreloader

当你创建好你的 [``PreloadSizeProvider``][3] 和 [``PreloadModelProvider``][6]，你就已经准备好创建 [``RecyclerViewPreloader``][2] 了：

```java
private final imageWidthPixels = 1024;
private final imageHeightPixels = 768;
private List<String> myUrls = ...;
 
...

PreloadSizeProvider sizeProvider = 
    new FixedPreloadSizeProvider(imageWidthPixels, imageHeightPixels);
PreloadModelProvider modelProvider = new MyPreloadModelProvider();
RecyclerViewPreloader<Photo> preloader = 
    new RecyclerViewPreloader<>(
        Glide.with(this), modelProvider, sizeProvider, 10 /*maxPreload*/);
```

这里使用 10 作为 maxPreload 仅仅十个占位符，关于如何选择这个数字更详细的讨论，请直接看下一节。

##### maxPreload
``maxPreload`` 是一个整数，指示你想预加载多少条数据。最优解因你的图片尺寸，质量，``RecyclerView`` 的布局而异，有些时候甚至与你的应用所运行的设备有关。

一个好的起点是，选择一个足够大的数，大到能包含两到三行的所有图片。一旦你选择了你的初始数字，你可以尝试在几个不同的设备上运行你的应用，并做必要的微调来最大化缓存命中率。

一个过大的数字将意味着你预加载得过远，不实用。过小的数字则会阻止你提前加载足够的图片。

#### RecyclerView
最后一步，当你拥有你的 [``RecyclerViewPreloader``][2] 之后，就可以将它添加为你的 ``RecyclerView`` 的一个滑动监听器:

```java
RecyclerView myRecyclerView = (RecyclerView) findViewById(R.id.recycler_view);
myRecyclerView.addOnScrollListener(preloader);
```

将 [``RecyclerViewPreloader``][2] 添加为一个滑动监听器，将允许 [``RecyclerViewPreloader``][2] 自动提前加载用户滑动方向上的图片，并监听滑动方向和加速度的改变。

**警告** - Glide 的默认滑动监听器 [``RecyclerToListViewScrollListener``][8] 假定你在使用一个 [``LinearLayoutManager``][9] 或其子类，如果实际情况并非如此将会发生 crash 。如果你在使用不同的 ``LayoutManager`` 类型，你需要实现你自己的 [``OnScrollListener``][10]，并翻译一下 ``RecyclerView`` 提供位置的调用，以及  [``RecyclerViewPreloader``][2] 关于这些位置的调用。

#### 总体代码
当你完成了上述步骤之后，你的代码看起来应该类似这样：

```java
public final class ImagesFragment extends Fragment {
  // These are totally arbitrary, pick sizes that are right for your UI.
  private final imageWidthPixels = 1024;
  private final imageHeightPixels = 768;
  // You will need to populate these urls somewhere...
  private List<String> myUrls = ...;

  @Override
  public View onCreateView(LayoutInflater inflater, ViewGroup container,
      Bundle savedInstanceState) {
    
    View result = inflater.inflate(R.layout.images_fragment, container, false);

    PreloadSizeProvider sizeProvider = 
        new FixedPreloadSizeProvider(imageWidthPixels, imageHeightPixels);
    PreloadModelProvider modelProvider = new MyPreloadModelProvider();
    RecyclerViewPreloader<Photo> preloader = 
        new RecyclerViewPreloader<>(
            Glide.with(this), modelProvider, sizeProvider, 10 /*maxPreload*/);

    RecyclerView myRecyclerView = (RecyclerView) result.findViewById(R.id.recycler_view);
    myRecyclerView.addOnScrollListener(preloader);
   
    // Finish setting up your RecyclerView etc.
    myRecylerView.setLayoutManager(...);
    myRecyclerView.setAdapter(...);

    ... 

    return result;
  }

  private class MyPreloadModelProvider implements PreloadModelProvider {
    @Override
    @NonNull
    public List<U> getPreloadItems(int position) {
      String url = myUrls.get(position);
      if (TextUtils.isEmpty(url)) {
        return Collections.emptyList();
      }
      return Collections.singletonList(url);
    }

    @Override
    @Nullable
    public RequestBuilder getPreloadRequestBuilder(String url) {
      return 
        GlideApp.with(fragment)
          .load(url) 
          .override(imageWidthPixels, imageHeightPixels);
    }
  }
}
```

### 示例代码
Glide 的 [示例应用][11] 包含了一些 [``RecyclerViewPreloader``] 的使用方式样本，包括：
1. [FlickrPhotoGrid][12]，使用一个 [``FixedPreloadSizeProvider``][5] 来在 flickr sample 的两个小型照片网格视图中做预加载。
2. [FlickrPhotoList][13], 使用一个 [``ViewPreloadSizeProvider``][4] 来在 flickr sample 的较大的列表中做预加载。
3. [MainActivity][14]，在 Giphy sample 中，使用一个 [``ViewPreloadSizeProvider``][4] 在滑动过程中做 GIF 预加载。
4. [HorizontalGalleryFragment][15] 在Gallery sample 中，使用一个自定义的 ``PreloadSizeProvider`` 以在水平滑动时预加载本地图片。

### 提示和陷阱 (Tip and tricks)
1. 使用 ``override()`` 来确保你加载的图片在你的 ``Adapter`` 和你的 [``RecyclerViewPreloader``][2] 中使用统一的尺寸。 你并不需要使你传入 ``override`` 方法的尺寸与 ``View`` 的实际尺寸完全一致，因为 Android 的 ``ImageView`` 类可以很容易地放大或缩小尺寸上的细微差别。
2. 少量较大的图片通常比很多小图片要快。启动每个请求有相当多的开销，所以如果可以，在您的UI中加载更少和更大的图像。
3. 如果滑动效果较差，考虑使用 ``override()`` 来故意减小图片尺寸。在 Android 上更新纹理(位图)开销可能较大，尤其是对于大图来说。你可以使用 ``override()`` 来强制图片变得比你的 ``View``更小，以获得更平滑的滑动。你甚至可以在用户停止滑动时再使用较高画质的图片替换较低画质的图片。
4. 如果你在 ``Adapter`` 使用你的 [``RecyclerViewPreloader``][2] 加载的图片时遇到问题，请查阅调试文档页面的 [预料之外的缓存丢失][7] 章节。

### 代码在这里
[https://github.com/bumptech/glide/tree/master/integration/recyclerview][1]

[1]: https://github.com/bumptech/glide/tree/master/integration/recyclerview
[2]: {{ site.baseurl }}/javadocs/420/com/bumptech/glide/integration/recyclerview/RecyclerViewPreloader.html
[3]: {{ site.baseurl }}/javadocs/420/com/bumptech/glide/ListPreloader.PreloadSizeProvider.html
[4]: {{ site.baseurl }}/javadocs/420/com/bumptech/glide/util/ViewPreloadSizeProvider.html
[5]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/util/FixedPreloadSizeProvider.html
[6]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/ListPreloader.PreloadModelProvider.html
[7]: /glide/doc/debugging.html#unexpected-cache-misses
[8]: {{ site.baseurl }}/javadocs/420/com/bumptech/glide/integration/recyclerview/RecyclerToListViewScrollListener.html
[9]: https://developer.android.com/reference/android/support/v7/widget/LinearLayoutManager.html
[10]: https://developer.android.com/reference/android/support/v7/widget/RecyclerView.OnScrollListener.html
[11]: /glide/ref/samples.html
[12]: https://github.com/bumptech/glide/blob/853c0d94f1ad353048b3d2556b49729ef3534430/samples/flickr/src/main/java/com/bumptech/glide/samples/flickr/FlickrPhotoGrid.java#L107
[13]: https://github.com/bumptech/glide/blob/853c0d94f1ad353048b3d2556b49729ef3534430/samples/flickr/src/main/java/com/bumptech/glide/samples/flickr/FlickrPhotoList.java#L68
[14]: https://github.com/bumptech/glide/blob/853c0d94f1ad353048b3d2556b49729ef3534430/samples/giphy/src/main/java/com/bumptech/glide/samples/giphy/MainActivity.java#L48
[15]: https://github.com/bumptech/glide/blob/853c0d94f1ad353048b3d2556b49729ef3534430/samples/gallery/src/main/java/com/bumptech/glide/samples/gallery/HorizontalGalleryFragment.java#L53
