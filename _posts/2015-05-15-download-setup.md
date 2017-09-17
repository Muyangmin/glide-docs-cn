---
layout: page
title: "下载和设置"
category: doc
date: 2017-09-17 09:49:00
order: 1
disqus: 1
translators: [Muyangmin, vincgao]
---

原文链接：[点击查看](http://bumptech.github.io/glide/doc/download-setup.html){:target="_blank"}

* TOC
{:toc}
### Android SDK 要求

**使用最低要求** - 使用 Glide 要求 SDK 版本为 API 14 (Ice Cream Sandwich) 及以上。

**编译最低要求** - 编译 Glide 要在 SDK 版本为 API 26 (Oreo) 及以上。

### 下载

可以使用多种方法获取 Glide 的公开发行版。

#### Jar

你可以直接在 GitHub 下载[最新的 jar 包][1]。并且还需要包含 Android [v4支持库][2] 的 jar 包。

#### Gradle

如果使用 Gradle，可从 Maven Central 或 JCenter 中添加对 Glide 的依赖。同样，你还需要添加 Android 支持库的依赖。

```groovy
repositories {
  mavenCentral()
  maven { url 'https://maven.google.com' }
}

dependencies {
    compile 'com.github.bumptech.glide:glide:4.1.1'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.1.1'
}
```

#### Maven

如果使用 Maven，同样可以添加对 Glide 的依赖。再次强调，你依旧需要添加 Android 支持库的依赖。

```xml
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>glide</artifactId>
  <version>4.1.1</version>
  <type>aar</type>
</dependency>
<dependency>
  <groupId>com.google.android</groupId>
  <artifactId>support-v4</artifactId>
  <version>r7</version>
</dependency>
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>compiler</artifactId>
  <version>4.1.1</version>
  <optional>true</optional>
</dependency>
```

### 设置

针对相应的构建配置，你可能还需要做一些额外的设置。

#### Proguard

如果你有使用到 proguard，那么请把以下代码添加到你的 ``proguard.cfg`` 文件中：
```
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.AppGlideModule
-keep public enum com.bumptech.glide.load.resource.bitmap.ImageHeaderParser$** {
    **[] $VALUES;
    public *;
}
```

#### Jack

Glide 的构建配置需要使用一些 [Jack][3] 目前还不能支持的特性。并且由于 Jack 最近已经被标记为 [deprecated][4]，Glide 需要使用的特性可能在未来也不会被加入了。

#### Java 8

截止目前 (2017年6月) 还没有一个稳定的 Android 工具链能允许你将 Glide 和 Java 8 特性一起使用。如果你希望使用 Java 8，并且允许牺牲一定的稳定性，那么，至少目前已经有一个 alpha 版本的 Android Gradle 插件可以支持 Java 8。但是，Alpha 版本的插件目前还未经 Glide 测试过。如果你对此感兴趣，可以查看 Android 的 [Java 8支持页][5]。

#### Kotlin

如果你在 Kotlin 编写的类里使用 Glide 注解，你需要引入一个 ``kapt`` 依赖，以代替常规的 ``annotationProcessor`` 依赖：

```groovy
dependencies {
  kapt 'com.github.bumptech.glide:compiler:4.1.1'
}
```

关于 Kotlin 的更多 api，可以查看 [Generated API][6]。

[1]: https://github.com/bumptech/glide/releases/download/v3.6.0/glide-3.6.0.jar
[2]: http://developer.android.com/tools/support-library/features.html#v4
[3]: https://source.android.com/source/jack
[4]: https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html
[5]: https://developer.android.com/studio/write/java8-support.html
[6]: {{ site.baseurl }}/doc/generatedapi.html#kotlin

