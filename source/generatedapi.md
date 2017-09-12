---
title: "生成的API"
---
原文链接：[点击查看](http://bumptech.github.io/glide/doc/generatedapi.html)

### 关于生成的API

Glide v4 使用了[annotation processor][1] 技术来生成API。这个API允许应用统一而流畅地访问 [``RequestBuilder``][2], [``RequestOptions``][3]里的所有选项和引入的任何集成库。

生成的API有两个目的：
1. 集成库可以使用定制选项扩展Glide的API。
2. 应用可以通过添加方法来扩展Glide的API。

虽然这些任务可以通过编写定制的[``RequestOptions``][3]子类，但这么做是非常具有挑战性的，并且降低了API的流畅性。

### 开始使用

#### Java

要使用生成的API，你需要执行两步操作：

1. 添加Glide的注解处理器(`annotation processor`)的依赖：

   ```groovy
   repositories {
     mavenCentral()
   }
   
   dependencies {
     annotationProcessor 'com.github.bumptech.glide:compiler:4.0.0'
   }
   ```
   
   请参阅[下载和设置][12]页面。

2. 在你的应用中包含一个[``AppGlideModule``][4]实现：
 
   ```java
   package com.example.myapp;
   
   import com.bumptech.glide.annotation.GlideModule;
   import com.bumptech.glide.module.AppGlideModule;
   
   @GlideModule
   public final class MyAppGlideModule extends AppGlideModule {}
   ```

    [``AppGlideModule``][4] 实现必须使用[``@GlideModule``][5]注解标记。如果注解不存在，这个module将不会被发现，并且你会在你的日志中收到一个使用``Glide``标签的警告，表示module未找到。

#### Kotlin

如果你正在使用Kotlin，你可以：

1. 使用Java实现所有的Glide注解类([``AppGlideModule``][4], [``LibraryGlideModule``][13], 以及 [``GlideExtension``][6])。
2. 使用Kotlin实现注解类，但是添加一个``kapt``依赖以替换Glide的``annotationProcessor``依赖：

   ```groovy
   dependencies {
     kapt 'com.github.bumptech.glide:compiler:4.0.0'
   }
   ```
   
   关于``kapt``的使用，请查看[官方文档][14]。

### 使用生成的API
 
API会在应用提供的[``AppGlideModule``][4]实现类的相同包下，默认命名为``GlideApp``。将开始加载的代码从 ``Glide.with()``改成``GlideApp.with()``，应用就可以使用新的API啦：
 
```java
GlideApp.with(fragment)
   .load(myUrl)
   .placeholder(R.drawable.placeholder)
   .fitCenter()
   .into(imageView);
```
 
与``Glide.with()``不同，其他诸如 ``fitCenter()``和``placeholder()``的选项在Builder中直接可用，并不需要额外传入单独的[``RequestOptions``][3]对象。
    
### Glide扩展

Glide生成的API可以被应用和库扩展。扩展使用被注解的静态方法来添加新的选项，或修改现有选项，甚至添加额外的类型支持。

[``GlideExtension``][6]注解用于标识一个扩展Glide API的类。任何扩展Glide API的类都必须使用这个注解来标记，否则方法上的注解就会被忽略。

使用[``GlideExtension``][6]标记的类被认为是工具类。这种类应该有一个私有的空构造器，应为final并且仅包含静态方法。被注解的类可以含有静态变量，可以引用其他的类或对象。

一个应用可以按自己的喜好而实现任意多的使用[``GlideExtension``][6]注解的类，Library也一样。当[``AppGlideModule``][4]被发现时，所有可用的[``GlideExtensions``][6]会被合并而构造出一个包含所有可用扩展的API。合并冲突会导致Glide的annotation processor抛出编译错误。

使用`GlideExtention`标记的类可以定义两种不同类型的扩展方法：

1. [``GlideOption``][7]  - 为[``RequestOptions``][3]添加一个自定义的选项。
2. [``GlideType``][8] - 添加对新的资源类型的支持(GIF，SVG, 等等)。


#### Glide选项

[``GlideOption``][7] 注解用于标记扩展[``RequestOptions``][3]的静态方法。``GlideOption``可以：

1. 定义一组应用中频繁使用的选项。
2. 添加新的选项，通常与Glide的[``Option``][10]类一起使用。

要定义一组选项，你可以这么写：

```java
@GlideExtension
public class MyAppExtension {
  // Size of mini thumb in pixels.
  private static final int MINI_THUMB_SIZE = 100;

  private MyAppExtension() { } // utility class

  @GlideOption
  public static void miniThumb(RequestOptions options) {
    options
      .fitCenter()
      .override(MINI_THUMB_SIZE);
  }
```

这将会在[``RequestOptions``][3]的子类中生成一个方法，看起来像这样：

```java
public class GlideOptions extends RequestOptions {
  
  public GlideOptions miniThumb() {
    MyAppExtension.miniThumb(this);
  }

  ...
}
```

你可以按你自己的想法包含尽可能多的额外选项，只要第一个参数保证为[``RequestOptions``][9]就行。

```java
@GlideOption
public static void miniThumb(RequestOptions options, int size) {
  options
    .fitCenter()
    .override(size);
}
```

额外的参数将会被作为参数添加到生成的方法中：

```java
public GlideOptions miniThumb(int size) {
  MyAppExtension.miniThumb(this);
}
```

之后你可以使用生成的``GlideApp``类调用你的自定义方法：

```java
GlideApp.with(fragment)
   .load(url)
   .miniThumb(thumbnailSize)
   .into(imageView);
```

使用``GlideOption``标记的方法应该为静态的(static)并且返回值为空(void)。注意，生成的方法在标准的``Glide``和``RequestOptions``类里不可用。

#### Glide类型

[``GlideType``][8]注解用于标记扩展[``RequestManager``][11]的静态方法。``GlideType``标记的方法允许你添加对新的资源类型的支持，包括指定默认选项。

例如，为添加对GIF的支持，你可以添加一个``GlideType``方法；

```java
@GlideExtension
public class MyAppExtension {
  private static final RequestOptions DECODE_TYPE_GIF = decodeTypeOf(GifDrawable.class).lock();

  @GlideType(GifDrawable.class)
  public static void asGif(RequestBuilder<GifDrawable> requestBuilder) {
    requestBuilder
      .transition(new DrawableTransitionOptions())
      .apply(DECODE_TYPE_GIF);
  }
}
```

这么做会生成一个[``RequestManager``][11]，包含一个方法，看起来像这样：

```java
public class GlideRequests extends RequesetManager {

  public RequestBuilder<GifDrawable> asGif() {
    RequestBuilder<GifDrawable> builder = as(GifDrawable.class);
    MyAppExtension.asGif(builder);
    return builder;
  }
  
  ...
}
```

之后你可以使用生成的``GlideApp``类调用你的自定义类型：

```java
GlideApp.with(fragment)
  .asGif()
  .load(url)
  .into(imageView);
```

``GlideType``标记的方法必须使用[``RequestBuilder<T>``][2]作为它们的第一个参数，这里``<T>``对应提供[``GlideType``][8]的类。方法应该为静态并且返回值为空。方法必须定义在一个被[``GlideExtension``][6]注解标记的类中。


[1]: https://docs.oracle.com/javase/8/docs/api/javax/annotation/processing/Processor.html
[2]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestBuilder.html
[3]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[4]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/AppGlideModule.html
[5]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/GlideModule.html
[6]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/GlideExtension.html
[7]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/GlideOption.html
[8]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/annotation/GlideType.html
[9]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/request/RequestOptions.html
[10]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/load/Option.html
[11]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/RequestManager.html
[12]: {{ site.url }}/glide/doc/download-setup.html
[13]: {{ site.url }}/glide/javadocs/400/com/bumptech/glide/module/LibraryGlideModule.html
[14]: https://kotlinlang.org/docs/reference/kapt.html
