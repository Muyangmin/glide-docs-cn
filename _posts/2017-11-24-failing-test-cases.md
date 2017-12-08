---
layout: page
title: "问题测试用例"
category: tut
date: 2017-11-24 14:48:43
disqus: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/tut/failing-test-cases.html){:target="_blank"}

* TOC
{:toc}

在为 Glide 报告 bug 的时候，如果您能同时提供一个 Pull Request 包含失败的测试用例 (failing test case) 以演示你正在报告的问题，会对我们很有帮助。失败测试用例可以协助避免交流问题，使维护者容易复现问题，并可在一定程度上提供在将来不再复现该问题的一些保障。

这个指南将手把手地带您撰写一个失败测试用例。

## 初始化设置

在编写任何代码之前，你需要有少许的一些前置条件，但如果您正在定期做与 Android 应用相关的工作，这其中大部分您应该都已经满足：

1. 安装并设置 [Android Studio](https://developer.android.com/studio/index.html)
2. 在 Android Studio 中创建一个 [Android 模拟器](https://developer.android.com/studio/run/managing-avds.html#createavd)，可以使用 x86 和 API 26。
3. Fork 并 Clone Glide 仓库，然后在 Android Studio 中打开（如有问题，请参阅 [贡献代码或文档](http://bumptech.github.io/glide/dev/contributing.html#contribution-workflow)）

## 添加一个仪器测试 (Instrumentation test)

现在你已经在 Android Studio 中打开了 Glide 了，下一步是编写一个仪器测试，它将会因为你将要报告的 bug 而失败。

Glide 的仪器测试存在于项目根目录下一个叫做 ``instrumentation`` 的 module 中。完整的仪器测试路径为 `glide/instrumentation/src/androidTest/java`。

### 添加一个测试文件

让我们来添加一个新的仪器测试文件：

1. 在 Android Studio 的 Project 窗口中，展开 `instrumentation/src/androidTest/java`
2. 右击 `com.bumptech.glide` （或任何合适的 package ）
3. 高亮 `New` 然后选择 `Java Class`
4. 输入一个合适的名字(如果你有 Issue 编号则可以使用 `Issue###Test`，否则可以使用其他描述问题的名称)
5. 点击 `OK`

你现在应该看到一个新的 Java 类，看起来像这样：

```java
package com.bumptech.glide;

public class IssueXyzTest {

}
```

到这里，你已经准备好继续编写你的测试了。

### 编写你的仪器测试

添加了你的测试文件之后，在编写之前，你需要做一些小的设置来让你的测试可以可靠地执行。

#### 设置
首先，你需要为你的测试类添加 `@RunWith(AndroidJUnit4.class)` 来指定 JUnit 4 测试执行器：
 
```java
package com.bumptech.glide;

import android.support.test.runner.AndroidJUnit4;
import org.junit.runner.RunWith;

@RunWith(AndroidJUnit4.class)
public class IssueXyzTest {

}
```

接下来你需要添加 ``TearDownGlide`` 规则，它可以确保其他测试的线程或配置不会与你的测试重合。只需要在你的文件顶部添加一行代码：

```java
package com.bumptech.glide;

import android.support.test.runner.AndroidJUnit4;
import com.bumptech.glide.test.TearDownGlide;
import org.junit.Rule;
import org.junit.runner.RunWith;

@RunWith(AndroidJUnit4.class)
public class IssueXyzTest {
  @Rule public final TearDownGlide tearDownGlide = new TearDownGlide();

}
```

然后我们将创建一个 Glide 的 ``ConcurrencyHelper`` 实例来帮助我们确保我们的步骤有序执行：

```java
package com.bumptech.glide;

import android.support.test.runner.AndroidJUnit4;
import com.bumptech.glide.test.ConcurrencyHelper;
import com.bumptech.glide.test.TearDownGlide;
import org.junit.Rule;
import org.junit.runner.RunWith;

@RunWith(AndroidJUnit4.class)
public class IssueXyzTest {
  @Rule public final TearDownGlide tearDownGlide = new TearDownGlide();
  private final ConcurrencyHelper concurrency = new ConcurrencyHelper();

}
```

最后，我们将添加一个 ``@Before`` 步骤来创建一个 ``Context`` 对象，我们将在大部分测试和帮助方法中用到它：

```java
package com.bumptech.glide;

import android.support.test.runner.AndroidJUnit4;
import com.bumptech.glide.test.ConcurrencyHelper;
import com.bumptech.glide.test.TearDownGlide;
import org.junit.Rule;
import org.junit.runner.RunWith;

@RunWith(AndroidJUnit4.class)
public class IssueXyzTest {
  @Rule public final TearDownGlide tearDownGlide = new TearDownGlide();
  private final ConcurrencyHelper concurrency = new ConcurrencyHelper();
  private Context context;

  @Before
  public void setUp() {
    context = InstrumentationRegistry.getTargetContext();
  }
}
```

就是这些！你已经准备好编写你的实际测试了。

#### 添加一个测试方法
接下来是添加你的特定测试方法。在类文件中添加一个方法，它需要被 ``@Test`` 注解以使 JUnit 知道要执行它：

```java
@Test
public void method_withSomeSetup_producesExpectedResult() {
}
```

理想情况下，测试方法命名应该如上例一样填入特定于你的问题的信息，但没有除了 ``@Test`` 注解之外的强制要求。

#### 编写一个失败测试

因为我们需要编写一些有用的测试用例，我们将使用 [Issue #2638](https://github.com/bumptech/glide/issues/2638) 来作为例子，并编写一个测试以覆盖这里报告的问题。

这个问题似乎是报告者先执行：

```java
byte[] data = ...
Glide.with(context)
  .load(data)
  .into(imageView);
```

然后执行:

```java
byte[] otherData = ...
Glide.with(context)
  .load(data)
  .into(imageView);
```

即使传给 Glide 的两个 ``byte[]`` 数组包含的数据并不相同，``ImageView`` 中显示的图片也没有改变。

我们可以相当简单地复制这个问题，通过创建两个 ``byte[]`` 并包含不同的图片，然后将他们依次加载到一个 ImageView 中，然后断言这个 ImageView 上设置的 ``Drawable`` 是不同的。

##### 创建测试方法

首先让我们创建一个方法，并合理命名：

```java
@Test
public void intoImageView_withDifferentByteArrays_loadsDifferentImages() {
  // TODO: fill this in.
}
```

因为我们将需要一个 ``ImageView`` 来加载图片，所以我们也需要创建它：

```java
@Test
public void intoImageView_withDifferentByteArrays_loadsDifferentImages() {
  final ImageView imageView = new ImageView(context);
  imageView.setLayoutParams(new LayoutParams(/*w=*/ 100, /*h=*/ 100));
}
```

##### 获取测试数据

接下来我们将需要我们将要加载的实际数据。 Glide 的仪器测试包含了一个标准的测试图片，我们可以使用它作为第一个图片。我们需要编写一个函数以加载这个图片的字节：

```java
private byte[] loadCanonicalBytes() throws IOException {
  int resourceId = ResourceIds.raw.canonical;
  Resources resources = context.getResources();
  InputStream is = resources.openRawResource(resourceId);
  return ByteStreams.toByteArray(is);
}
```

接下来我们需要编写一个函数提供不同的图片的字节。我们可以添加另一个资源到 `instrumentation/src/main/res/raw` 或 `instrumentation/src/main/res/drawable` 并复用我们的上一个方法，但我们也可以通过另一个方法，仅仅修改我们的标准图片的一个像素：

```java
private byte[] getModifiedBytes() throws IOException {
  byte[] canonicalBytes = getCanonicalBytes();
  BitmapFactory.Options options = new BitmapFactory.Options();
  options.inMutable = true;
  Bitmap bitmap = 
      BitmapFactory.decodeByteArray(canonicalBytes, 0 ,canonicalBytes.length, options);
  bitmap.setPixel(0, 0, Color.TRANSPARENT);
  ByteArrayOutputStream os = new ByteArrayOutputStream();
  bitmap.compress(CompressFormat.PNG, /*quality=*/ 100, os);
  return os.toByteArray();
}
```

##### 运行 Glide

现在只剩下编写上面的两行加载代码：

```java
@Test
public void intoImageView_withDifferentByteArrays_loadsDifferentImages() throws IOException {
  final ImageView imageView = new ImageView(context);
  imageView.setLayoutParams(new LayoutParams(/*w=*/ 100, /*h=*/ 100));

  final byte[] canonicalBytes = getCanonicalBytes();
  final byte[] modifiedBytes = getModifiedBytes();

  concurrency.loadOnMainThread(Glide.with(context).load(canonicalBytes), imageView);
  Bitmap firstBitmap = ((BitmapDrawable) imageView.getDrawable()).getBitmap();

  concurrency.loadOnMainThread(Glide.with(context).load(modifiedBytes), imageView);
  Bitmap secondBitmap = ((BitmapDrawable) imageView.getDrawable()).getBitmap();
}
```

这里我们使用了 `ConcurrencyHelper`，以使 Glide 的加载在主线程执行，并等待它完成。如果我们只是直接使用 `into()`，加载将会异步发生，而在下一行执行之前可能并没有完成，而我们在下一行将试图取回 ``ImageView`` 里的 ``Bitmap``。然后它将抛出一个异常，因为我们最后在一个 ``null`` ``Drawable`` 上调用了 ``getBitmap``。

最后，我们需要添加我们的断言：两个 Bitmap 实际上包含不同的数据：

##### 断言输出

```java
BitmapSubject.assertThat(firstBitmap).isNotSameAs(secondBitmap);
```

``BitmapSubject`` 是一个 Glide 里的辅助类，可以帮助你在一起测试中对 ``Bitmap`` 做比较时做一些基本的断言操作。

##### 总结

我们现在已经编写了一个测试，它会生成一些测试数据，在 Glide 中执行一些方法，获取这些 Glide 方法的输出，并比较输出结果以确保它符合我们的预期。

我们的完整测试来看起来像这样：

```java
package com.bumptech.glide;

import android.content.Context;
import android.content.res.Resources;
import android.graphics.Bitmap;
import android.graphics.Bitmap.CompressFormat;
import android.graphics.BitmapFactory;
import android.graphics.Color;
import android.graphics.drawable.BitmapDrawable;
import android.support.test.InstrumentationRegistry;
import android.support.test.runner.AndroidJUnit4;
import android.widget.AbsListView.LayoutParams;
import android.widget.ImageView;
import com.bumptech.glide.test.BitmapSubject;
import com.bumptech.glide.test.ConcurrencyHelper;
import com.bumptech.glide.test.ResourceIds;
import com.bumptech.glide.test.TearDownGlide;
import com.google.common.io.ByteStreams;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.concurrent.ExecutionException;
import org.junit.Before;
import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;

@RunWith(AndroidJUnit4.class)
public class Issue2638Test {
  @Rule public final TearDownGlide tearDownGlide = new TearDownGlide();
  private final ConcurrencyHelper concurrency = new ConcurrencyHelper();
  private Context context;

  @Before
  public void setUp() {
    context = InstrumentationRegistry.getTargetContext();
  }

  @Test
  public void intoImageView_withDifferentByteArrays_loadsDifferentImages()
      throws IOException, ExecutionException, InterruptedException {
    final ImageView imageView = new ImageView(context);
    imageView.setLayoutParams(new LayoutParams(/*w=*/ 100, /*h=*/ 100));

    final byte[] canonicalBytes = getCanonicalBytes();
    final byte[] modifiedBytes = getModifiedBytes();

    Glide.with(context)
        .load(canonicalBytes)
        .submit()
        .get();

    concurrency.loadOnMainThread(Glide.with(context).load(canonicalBytes), imageView);
    Bitmap firstBitmap = ((BitmapDrawable) imageView.getDrawable()).getBitmap();

    concurrency.loadOnMainThread(Glide.with(context).load(modifiedBytes), imageView);
    Bitmap secondBitmap = ((BitmapDrawable) imageView.getDrawable()).getBitmap();

    BitmapSubject.assertThat(firstBitmap).isNotSameAs(secondBitmap);
  }

  private byte[] getModifiedBytes() throws IOException {
    byte[] canonicalBytes = getCanonicalBytes();
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inMutable = true;
    Bitmap bitmap =
        BitmapFactory.decodeByteArray(canonicalBytes, 0, canonicalBytes.length, options);
    bitmap.setPixel(0, 0, Color.TRANSPARENT);
    ByteArrayOutputStream os = new ByteArrayOutputStream();
    bitmap.compress(CompressFormat.PNG, /*quality=*/ 100, os);
    return os.toByteArray();
  }

  private byte[] getCanonicalBytes() throws IOException {
    int resourceId = ResourceIds.raw.canonical;
    Resources resources = context.getResources();
    InputStream is = resources.openRawResource(resourceId);
    return ByteStreams.toByteArray(is);
  }
}
```

现在只需要执行这个测试并看看它是否工作。

## 执行仪器测试

现在你已经有了一个测试用例，你可以通过下面的方法来执行它：

1. 右击测试文件名，可以在 Project 窗口中，也可以在你的编辑器顶部 Tab 上
2. 点击 `Run 'IssueXyzTest'`
3. 如果打开了一个标题为 `Edit Configuration` 的窗口，则：
    1. 在 `General` Tab 中
    2. 点击 `Target` 然后选择 `Emulator`
    3. 点击 `Run`
4. 如果打开了一个设备列表：
    1. 在 `Available Virtual Devices` 中：
    2. 点击任意模拟器 (推荐 X86 和 API 26)
    3. 点击 `OK`

你将会看到一个模拟器启动，大概等待 30 秒或一分钟左右直到启动完成。

在模拟器启动之后，你将在 Android Studio 编辑器下方的一个窗口看到测试结果，结果可能为 `All Test Passed` 或 `N tests failed` 并伴随一个异常信息。

在你完成仪器测试的遍历之后，你还应该检查代码风格问题或常见 bug ，请执行：

```sh
./gradlew build
```

**如果你的测试成功了也OK!**

请将成功和失败的测试一并发送 Pull Request 给我们。如果没有其他，则成功的测试可以帮助我们排除一些无法复现你的 bug 的场景，因此我们的精力可以更集中在其他一些可以复现的场景上。我们也可能会建议做出调整或其他可能导致测试失败的变种，并最终找出问题所在。

## 创建一个 Pull Request

现在你已经编写好了你的测试用例，你需要上传到你的 Glide fork 中并发送一个 Pull Request。

首先你需要提交你的新测试文件：

```sh
git add intrumentation/src/androidTest/java/com/bumptech/glide/IssueXyzTest.java
git commit -m "Adding test case for issue XYZ"
```

如果你有多个文件需要添加，你可以使用 ``git add .``，但请特别小心，因为这么做可能会导致你意外添加一些不需要的文件到提交中。

接下来，推送你的修改到你 GitHub 上的 Glide fork 仓库中：

```sh
git push origin master
```

然后创建一个 Pull Request：
1. 在你的 GitHub 上打开你的 fork 仓库 (``https://github.com/<your_username>/glide``)
2. 点击 `New pull request` 按钮
3. 继续点击绿色的大大的 `Create pull request` 按钮
4. 添加一个标题 (`Tests for IssueXyz`)
5. 尽可能地填充模板信息
6. 点击 `Create Pull Request`

大功告成！你的 Pull Request 将会被发送，而我们将尽快查看。
