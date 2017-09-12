---
title: "RecyclerView"
---
原文链接：[点击查看](http://bumptech.github.io/glide/int/recyclerview.html)

### RecyclerView
The RecyclerView library adds a class that will automatically load images just ahead of where a user is scrolling in a RecyclerView. Combined with the right image size and an effective disk cache strategy, this library can dramatically decrease the number of loading tiles/indicators users see when scrolling through lists of images.
RecyclerView库添加了一个类，这个类会在用户滑动RecycelrView时自动加载滑动位置之前的图片。配合使用正确的图片尺寸和合适的磁盘缓存策略，这个库可以在用户滑动图片列表时显著减少用户看到的加载指示器的数量。

**代码在这里:**
[https://github.com/bumptech/glide/tree/master/integration/recyclerview][1]

**Gradle 依赖:**
```groovy
compile ("com.github.bumptech.glide:recyclerview-integration:4.0.0") {
  // Excludes the support library because it's already included by Glide.
  transitive = false
}
```

[1]: https://github.com/bumptech/glide/tree/master/integration/recyclerview