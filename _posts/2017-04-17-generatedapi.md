---
layout: page
title: "Generated API"
category: doc
date: 2017-09-17 16:19:51
order: 3
disqus: 1
translators: [Muyangmin, vincgao]
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/generatedapi.html){:target="_blank"}

* TOC
{:toc}

### 简介

Glide v4 使用 [注解处理器 (Annotation Processor)][1] 来生成出一个 API，它允许应用扩展 Glide 的 API并包含各种集成库提供的组件。

Generated API 模式的设计出于以下两个目的：
1. 集成库可以为 Generated API 扩展自定义选项。
2. 在 Application 模块中可将常用的选项组打包成一个选项在 Generated API 中使用

虽然以上所说的工作均可以通过手动创建 [``RequestOptions``][3] 子类的方式来完成，但想将它用好更具有挑战，并且降低了 API 使用的流畅性。

### 开始使用

#### 有效使用范围

Generated API 目前仅可以在 Application 模块内使用。这一限制可以让我们仅持有一份 Generated API，而不是各个 Library 和 Application 中均有自己定义出来的 Generated API。这一做法会让 Generated API 的调用更简单，并确保 Application 模块中 Generated API 调用的选项在各处行为一致。这一限制在接下来的版本中也许会被取消（以实验性或其他的方式给出）。

#### Java

要在 Application 模块中使用 Generated API，你需要执行以下两步：

1. 添加 Glide 注解处理器的依赖：

   ```groovy
   repositories {
     mavenCentral()
   }

   dependencies {
     annotationProcessor 'com.github.bumptech.glide:compiler:4.11.0'
   }
   ```

   参阅 [下载和设置][12] 页面了解更多。

2. 在 Application 模块中包含一个 [``AppGlideModule``][4] 的实现：

   ```java
   package com.example.myapp;

   import com.bumptech.glide.annotation.GlideModule;
   import com.bumptech.glide.module.AppGlideModule;

   @GlideModule
   public final class MyAppGlideModule extends AppGlideModule {}
   ```
   
你不必去重写 `AppGlideModule` 中的任何一个方法。子类中完全可以不用写任何东西，它只需要继承 `AppGlideModule` 并且添加 `@GlideModule` 注解。

[``AppGlideModule``][4] 的实现必须使用 [``@GlideModule``][5] 注解标记。如果注解不存在，该 module 将不会被 Glide 发现，并且在日志中收到一条带有 ``Glide`` tag 的警告，表示 module 未找到。

**注意：** 程序库 (Library) **不** 应该包含 [`AppGlideModule`][4] 实现，详见 [配置][15]。

#### Kotlin

如果你正在使用Kotlin，你可以选择：

1. 使用 Java 按前面所述实现所有的 Glide 注解类([``AppGlideModule``][4]， [``LibraryGlideModule``][13]，以及 [``GlideExtension``][6] )。

2. 使用 Kotlin 实现注解类，但需要添加一个 ``kapt`` 依赖以替换 Glide 的``annotationProcessor`` 依赖：

   ```groovy
   dependencies {
     kapt 'com.github.bumptech.glide:compiler:4.11.0'
   }
   ```
   注意，你还需要在你的 ``build.gradle`` 文件中包含 ``kotlin-kapt`` 插件：
   
   ```groovy
   apply plugin: 'kotlin-kapt'
   ```
    
    此外，如果你有其他的注解处理器，它们都必须全部被从 ``annotationProcessor`` 转换为 ``kapt``：

   ```groovy
   dependencies {
     kapt "android.arch.lifecycle:compiler:1.0.0"
     kapt 'com.github.bumptech.glide:compiler:4.11.0'
   }
   ```

   关于``kapt``的使用，请查看[官方文档][14]。

#### Android Studio

Android Studio 在大多数时候都可以正确地处理注解处理器 (annotation processor) 和 generated API。然而，当你第一次添加你的 ``AppGlideModule`` 或做了某些类型的修改后，你可能需要重新构建 (rebuild) 你的项目。 无论何时，如果你发现 API 没有被 import ，或看起来已经过期，你可以通过以下方法重新构建：
1. 打开 Build 菜单；
2. 点击 Rebuild Project。

### 使用 Generated API

Generated API 默认名为 `GlideApp` ，与 Application 模块中 [`AppGlideModule`][4]的子类包名相同。在 Application 模块中将 `Glide.with()` 替换为 `GlideApp.with()`，即可使用该 API 去完成加载工作：

```java
GlideApp.with(fragment)
   .load(myUrl)
   .placeholder(R.drawable.placeholder)
   .fitCenter()
   .into(imageView);
```

与 ``Glide.with()`` 不同，诸如  ``fitCenter()`` 和 ``placeholder()`` 等选项在 Builder 中直接可用，并不需要再传入单独的 [``RequestOptions``][3] 对象。
​    
### GlideExtension

Glide Generated API 可在 Application 和 Library 中被扩展。扩展使用被注解的静态方法来添加新的选项、修改现有选项、甚至添加额外的类型支持。

[``@GlideExtension``][6] 注解用于标识一个扩展 Glide API 的类。任何扩展 Glide API 的类都必须使用这个注解来标记，否则其中被注解的方法就会被忽略。

被 [``@GlideExtension``][6] 注解的类应以工具类的思维编写。这种类应该有一个私有的、空的构造方法，应为 final 类型，并且仅包含静态方法。被注解的类可以含有静态变量，可以引用其他的类或对象。

在 Application 模块中可以根据需求实现任意多个被 [``@GlideExtension``][6] 注解的类，在 Library 模块中同样如此。当 [``AppGlideModule``][4] 被发现时，所有有效的 [Glide 扩展类][6] 会被合并，所有的选项在 API 中均可以被调用。合并冲突会导致 Glide 的 Annotation Processor 抛出编译错误。

被 `@GlideExtention` 注解的类有两种扩展方式：

1. [``GlideOption``][7]  - 为 [``RequestOptions``][3] 添加一个自定义的选项。
2. [``GlideType``][8] - 添加对新的资源类型的支持(GIF，SVG 等等)。


#### GlideOption

用 [``@GlideOption``][7] 注解的静态方法用于扩展 [``RequestOptions``][3] 。``GlideOption`` 可以：

1. 定义一个在 Application 模块中频繁使用的选项集合。
2. 创建新的选项，通常与 Glide 的 [``Option``][10] 类一起使用。

要定义一个选项集合，你可以这么写：

```java
@GlideExtension
public class MyAppExtension {
  // Size of mini thumb in pixels.
  private static final int MINI_THUMB_SIZE = 100;

  private MyAppExtension() { } // utility class

  @NonNull
  @GlideOption
  public static BaseRequestOptions<?> miniThumb(BaseRequestOptions<?> options) {
    return options
      .fitCenter()
      .override(MINI_THUMB_SIZE);
  }
```

这将会在 [``RequestOptions``][3] 的子类中生成一个方法，类似这样：

```java
public class GlideOptions extends RequestOptions {
  
  public GlideOptions miniThumb() {
    return (GlideOptions) MyAppExtension.miniThumb(this);
  }

  ...
}
```

你可以为方法任意添加参数，但要保证第一个参数为 [``RequestOptions``][9]。

```java
@GlideOption
public static BaseRequestOptions<?> miniThumb(BaseRequestOptions<?> options, int size) {
  return options
    .fitCenter()
    .override(size);
}
```

在自动生成的方法中新添的参数同样被加了进来：

```java
public GlideOptions miniThumb(int size) {
  return (GlideOptions) MyAppExtension.miniThumb(this);
}
```

之后你就可以使用生成的 ``GlideApp`` 类调用你的自定义方法：

```java
GlideApp.with(fragment)
   .load(url)
   .miniThumb(thumbnailSize)
   .into(imageView);
```

使用 ``@GlideOption`` 标记的方法应该为静态方法，并且返回值为 `BaseRequestOptions<?>`。请注意，这些生成的方法在标准的 ``Glide`` 和 ``RequestOptions`` 类里不可用，只存在于生成的等效类中。

#### GlideType

被 [``@GlideType``][8] 注解的静态方法用于扩展 [``RequestManager``][11] 。被 ``@GlideType`` 注解的方法允许你添加对新的资源类型的支持，包括指定默认选项。

例如，为添加对 GIF 的支持，你可以添加一个被 ``@GlideType`` 注解的方法：

```java
@GlideExtension
public class MyAppExtension {
  private static final RequestOptions DECODE_TYPE_GIF = decodeTypeOf(GifDrawable.class).lock();

  @NonNull
  @GlideType(GifDrawable.class)
  public static RequestBuilder<GifDrwable> asGif(RequestBuilder<GifDrawable> requestBuilder) {
    return requestBuilder
      .transition(new DrawableTransitionOptions())
      .apply(DECODE_TYPE_GIF);
  }
}
```

这样会生成一个包含对应方法的 [``RequestManager``][11] ：

```java
public class GlideRequests extends RequesetManager {

  public GlideRequest<GifDrawable> asGif() {
    return (GlideRequest<GifDrawable> MyAppExtension.asGif(this.as(GifDrawable.class));
  }
  
  ...
}
```

之后你可以使用生成的 ``GlideApp`` 类调用你的自定义类型：

```java
GlideApp.with(fragment)
  .asGif()
  .load(url)
  .into(imageView);
```

被 ``@GlideType`` 标记的方法必须使用 [``RequestBuilder<T>``][2] 作为其第一个参数，这里的泛型 ``<T>`` 对应 [``@GlideType``][8] 注解中传入的类。该方法应为静态方法，且返回值为 `RequestBuilder<T>` 。方法必须定义在一个被 [``@GlideExtension``][6] 注解标记的类中。


[1]: https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html
[2]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestBuilder.html
[3]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[4]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[5]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideModule.html
[6]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideExtension.html
[7]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideOption.html
[8]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/annotation/GlideType.html
[9]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[10]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/load/Option.html
[11]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/RequestManager.html
[12]: {{ site.baseurl }}/doc/download-setup.html
[13]: {{ site.baseurl }}/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html
[14]: https://kotlinlang.org/docs/reference/kapt.html
[15]: {{ site.baseurl }}/doc/configuration.html#avoid-appglidemodule-in-libraries


