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

在Glide中，[Transformations][1] 可以获取资源并修改它，然后返回被修改后的资源。通常变换操作是用来完成剪裁或对位图应用过滤器，但它也可以用于转换GIF动画，甚至自定义的资源类型。

### 内置类型

Glide 提供了很多内置的变换，包括：

* [CenterCrop][4]
* [FitCenter][2]
* [CircleCrop][6]

### 应用

通过 [RequestOptions][9] 类可以应用变换：

#### 默认变换

```java
Glide.with(fragment)
  .load(url)
  .fitCenter()
  .into(imageView);
```

或使用 `RequestOptions` ：

```java
RequestOptions options = new RequestOptions();
options.centerCrop();

Glide.with(fragment)
    .load(url)
    .apply(options)
    .into(imageView);
```

可以查阅 [Options][3] 页来获得更多 `RequestOption` 的相关信息。

#### 多重变换

默认情况下，每个 [``transform()``][17] 调用，或任何特定转换方法(``fitCenter()``, ``centerCrop()``, ``bitmapTransform()`` etc)的调用都会替换掉之前的变换。

如果你想在单次加载中应用多个变换，请使用 [``MultiTransformation``][18] 类，或其快捷方法  [``.transforms()``][19] 。

```java
Glide.with(fragment)
  .load(url)
  .transform(new MultiTransformation(new FitCenter(), new YourCustomTransformation())
  .into(imageView);
```
或使用快捷方法：

```java
Glide.with(fragment)
  .load(url)
  .transform(new FitCenter(), new YourCustomTransformation())
  .into(imageView);
```

请注意，你向 [``MultiTransformation``][18] 的构造器传入变换参数的顺序，决定了这些变换的应用顺序。

### 定制变换

尽管 Glide 提供了各种各样的内置 [``Transformation``][1] 实现，如果你需要额外的功能，你也可以实现你自己的 [``Transformation``][2]。

#### BitmapTransformation

如果你只需要变换 ``Bitmap``，最好是从继承 [``BitmapTransformation``][20] 开始。``BitmapTransformation`` 为我们处理了一些基础的东西，例如，如果你的变换返回了一个新修改的 Bitmap ，``BitmapTransformation``将负责提取和回收原始的 Bitmap。

一个简单的实现看起来可能像这样：

```java
public class FillSpace extends BitmapTransformation {
    private static final String ID = "com.bumptech.glide.transformations.FillSpace";
    private static final String ID_BYTES = ID.getBytes(STRING_CHARSET_NAME);

    @Override
    public Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        if (toTransform.getWidth() == outWidth && toTransform.getHeight() == outHeight) {
            return toTransform;
        }

        return Bitmap.createScaledBitmap(toTransform, outWidth, outHeight, /*filter=*/ true);
    }

    @Override
    public void equals(Object o) {
      return o instanceof FillSpace;
    }

    @Override
    public int hashCode() {
      return ID.hashCode();
    }

    @Override
    public void updateDiskCacheKey(MessageDigest messageDigest)
        throws UnsupportedEncodingException {
      messageDigest.update(ID_BYTES);
    }
}
```

尽管你的 ``Transformation`` 将几乎确定比这个示例更复杂，但它应该包含了相同的基本元素和复写方法。

#### 必需的方法

请特别注意，对于任何 ``Transformation`` 子类，包括 ``BitmapTransformation``，你都有三个方法你 **必须** 实现它们，以使得磁盘和内存缓存正确地工作：

1. ``equals()``
2. ``hashCode()``
3. ``updateDiskCacheKey``

如果你的 [``Transformation``][1] 没有参数，通常使用一个包含完整包限定名的 ``static`` ``final`` ``String`` 来作为一个 ID，它可以构成 ``hashCode()`` 的基础，并可用于更新 ``updateDiskCacheKey()`` 传入的 ``MessageDigest``。如果你的 [``Transformation``][1] 需要参数而且它会影响到 ``Bitmap`` 被变换的方式，它们也必须被包含到这三个方法中。

例如，Glide 的 [``RoundedCorners``][21] 变换接受一个 ``int``，它决定了圆角的弧度。它的``equals()``, ``hashCode()`` 和 ``updateDiskCacheKey`` 实现看起来像这样：

```java
  @Override
  public boolean equals(Object o) {
    if (o instanceof RoundedCorners) {
      RoundedCorners other = (RoundedCorners) o;
      return roundingRadius == other.roundingRadius;
    }
    return false;
  }

  @Override
  public int hashCode() {
    return Util.hashCode(ID.hashCode(),
        Util.hashCode(roundingRadius));
  }

  @Override
  public void updateDiskCacheKey(MessageDigest messageDigest) {
    messageDigest.update(ID_BYTES);

    byte[] radiusData = ByteBuffer.allocate(4).putInt(roundingRadius).array();
    messageDigest.update(radiusData);
  }
```

原来的 ``String`` 仍然保留，但 ``roundingRadius`` 被包含到了三个方法中。这里，``updateDiskCacheKey`` 方法还演示了你可以如何使用 ``ByteBuffer`` 来包含基本参数到你的 ``updateDiskCacheKey`` 实现中。

#### 不要忘记 equals() / hashCode()!

值得重申的一点是，为了让内存缓存正常地工作你是否必须实现 ``equals()`` 和 ``hashCode()`` 方法。很不幸，即使你没有复写这两个方法，``BitmapTransformation`` 和 ``Transformation`` 也能通过编译，但这并不意味着它们能正常工作。我们正在探索一些方案，以使在 Glide 的未来版本中，使用默认的 ``equals()`` 和 ``hashCode`` 方法将抛出一个编译时错误。

### Glide中的特殊行为

#### 重用变换
``Transformation`` 的设计初衷是无状态的。因此，在多个加载中复用 ``Transformation`` 应当总是安全的。创建一次 ``Transformation`` 并在多个加载中使用它，通常是很好的实践。

#### ImageView的自动变换
在Glide中，当你为一个 [ImageView][7] 开始加载时，Glide可能会自动应用 [FitCenter][2] 或 [CenterCrop][4] ，这取决于view的 [ScaleType][8] 。如果 `scaleType` 是 ``CENTER_CROP`` , Glide 将会自动应用 ``CenterCrop`` 变换。如果 `scaleType` 为 ``FIT_CENTER`` 或 ``CENTER_INSIDE`` ，Glide会自动使用 ``FitCenter`` 变换。

当然，你总有权利覆写默认的变换，只需要一个带有 ``Transformation`` 集合的 [RequestOptions][9] 即可。另外，你也可以通过使用 [``dontTransform()``][10] 确保不会自动应用任何变换。

#### 自定义资源
因为 Glide 4.0 允许你指定你将解码的资源的父类型，你可能无法确切地知道将会应用何种变换。例如，当你使用 [``asDrawable()``][11] (或就是普通的 ``with()`` ，因为 ``asDrawable()`` 是默认情形)来加载 Drawable 资源时，你可能会得到 [``BitmapDrawable``][12] 子类，也有可能得到 [``GifDrawable``][13] 子类。

为了确保你添加到 ``RequestOptions`` 中的任何变换都会被使用，Glide将 ``Transformation`` 添加到一个Map中保存，其Key为你提供变换的资源类型。当资源被成功解码时，Glide使用这个Map来取回对应的 ``Transformation`` 。

Glide可以将 ``Bitmap`` ``Transformation``应用到 ``BitmapDrawable`` , ``GifDrawable`` , 以及 ``Bitmap`` 资源上，因此通常你只需要编写和应用 ``Bitmap`` ``Transformation`` 。然而，如果你添加了额外的资源类型，你可能需要考虑派生 [``RequestOptions``][15] 类，并且，在内置的这些 ``Bitmap`` ``Transformations`` 之外，你还需要为你的自定义资源类型提供一个 ``Transformation`` 。

[1]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/Transformation.html
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/bitmap/FitCenter.html
[3]: options.html
[4]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/bitmap/CenterCrop.html
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/bitmap/CircleCrop.html
[7]: http://developer.android.com/reference/android/widget/ImageView.html
[8]: http://developer.android.com/reference/android/widget/ImageView.ScaleType.html
[9]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[10]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#dontTransform--
[11]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestManager.html#asDrawable--
[12]: http://developer.android.com/reference/android/graphics/drawable/BitmapDrawable.html
[13]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/resource/gif/GifDrawable.html
[14]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#transform-java.lang.Class-com.bumptech.glide.load.Transformation-
[15]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[16]: {{ site.baseurl }}/doc/generatedapi.html
[17]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html#transform-java.lang.Class-com.bumptech.glide.load.Transformation-
[18]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/MultiTransformation.html
[19]: {{ site.baseurl }}/javadocs/410/com/bumptech/glide/request/RequestOptions.html#transforms-com.bumptech.glide.load.Transformation...-
[20]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/resource/bitmap/BitmapTransformation.html
[21]: {{ site.baseurl }}/javadocs/440/com/bumptech/glide/load/resource/bitmap/RoundedCorners.html


