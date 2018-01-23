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

**Min Sdk Version** - 使用 Glide 需要 min SDK 版本 API **14** (Ice Cream Sandwich) 或更高。

**Compile Sdk Version** - Glide 必须使用 API **26** (Oreo) 或更高版本的 SDK 来编译。

**Support Library Version** - Glide 使用的支持库版本为 **27**。

如果你需要使用不同的支持库版本，你需要在你的 `build.gradle` 文件里去从 Glide 的依赖中去除 `"com.android.support"`。例如，假如你想使用 v26 的支持库： 

```groovy
dependencies {
  implementation ("com.github.bumptech.glide:glide:4.5.0") {
    exclude group: "com.android.support"
  }
  implementation "com.android.support:support-fragment:26.1.0"
}
```
使用与 Glide 依赖的支持库不同的版本可能会导致一些运行时异常 ，例如：

```
java.lang.NoSuchMethodError: No static method getFont(Landroid/content/Context;ILandroid/util/TypedValue;ILandroid/widget/TextView;)Landroid/graphics/Typeface; in class Landroid/support/v4/content/res/ResourcesCompat; or its super classes (declaration of 'android.support.v4.content.res.ResourcesCompat' 
at android.support.v7.widget.TintTypedArray.getFont(TintTypedArray.java:119)
```

也可能造成 Glide 的 API 生成器失败，从而不能正确地生成 `GlideApp` 类.

请参阅 [#2730][8] 获取这方面的更多信息。

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
    compile 'com.github.bumptech.glide:glide:4.5.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.5.0'
}
```

注意：如果可能，请尽量在你的依赖中避免使用 `@aar` 。如果你必须这么做，请添加 `transitive=true` 以确保所有必要的类都被包含到你的 API 中：

```groovy
dependencies {
    implementation ("com.github.bumptech.glide:glide:4.5.0@aar") {
        transitive = true
    }
}
```
在 Gradle 中，`@aar` 意味着 ["Artifact Only"][9]，默认情况下将排除所有依赖。

使用 `@aar` 而不使用 `transitive=true` ,将会排除 Glide 的依赖，并导致运行时异常，例如：

```
java.lang.NoClassDefFoundError: com.bumptech.glide.load.resource.gif.GifBitmapProvider
    at com.bumptech.glide.load.resource.gif.ByteBufferGifDecoder.<init>(ByteBufferGifDecoder.java:68)
    at com.bumptech.glide.load.resource.gif.ByteBufferGifDecoder.<init>(ByteBufferGifDecoder.java:54)
    at com.bumptech.glide.Glide.<init>(Glide.java:327)
    at com.bumptech.glide.GlideBuilder.build(GlideBuilder.java:445)
    at com.bumptech.glide.Glide.initializeGlide(Glide.java:257)
    at com.bumptech.glide.Glide.initializeGlide(Glide.java:212)
    at com.bumptech.glide.Glide.checkAndInitializeGlide(Glide.java:176)
    at com.bumptech.glide.Glide.get(Glide.java:160)
    at com.bumptech.glide.Glide.getRetriever(Glide.java:612)
    at com.bumptech.glide.Glide.with(Glide.java:684)
```


#### Maven

如果使用 Maven，同样可以添加对 Glide 的依赖。再次强调，你依旧需要添加 Android 支持库的依赖。

```xml
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>glide</artifactId>
  <version>4.5.0</version>
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
  <version>4.5.0</version>
  <optional>true</optional>
</dependency>
```

### 设置

针对相应的构建配置，你可能还需要做一些额外的设置。

#### 权限
Glide 假定你要访问的数据都存储在你的应用中，不要求任何权限。也就是说，大部分应用从设备上（DCIM，图库，或SD卡的其他地方）或 Internet 上加载图片。因此，你可能需要包含一条或多条以下列出的权限，这取决于你的应用场景。

##### Internet
如果你计划从 URL 或一个网络连接中加载数据，你需要添加 ``INTERNET`` 和 ``ACCESS_NETWORK_STATE`` 权限到你的 ``AndroidManifest.xml`` 中：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="your.package.name"

    <uses-permission android:name="android.permission.INTERNET"/>
    <!--
    Allows Glide to monitor connectivity status and restart failed requests if users go from a
    a disconnected to a connected network state.
    -->
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>

    <application>
      ...
    </application>
</manifest>
```

从技术上讲，``ACCESS_NETWORK_STATE`` 对于 Glide 加载 URL 并不是必需的，但是它将帮助 Glide 处理 *片状网络(flaky network)* 和飞行模式。请继续阅读下面的连接监视章节以了解详情。

##### 连接监听 (Connectivity Monitoring)
如果你正在从 URL 加载图片，Glide 可以自动帮助你处理片状网络连接：它可以监听用户的连接状态并在用户重新连接到网络时重启之前失败的请求。如果 Glide 检测到你的应用拥有 ``ACCESS_NETWORK_STATUS`` 权限，Glide 将自动监听连接状态而不需要额外的改动。

你可以通过检查 ``ConnectivityMonitor`` 日志标签来验证 Glide 是否正在监听网络状态: 

```
adb shell setprop log.tag.ConnectivityMonitor DEBUG
```
如果你成功添加了 ``ACCESS_NETWORK_STATUS`` 权限，你将在 logcat 中看到类似这样的日志：

```
11-18 18:51:23.673 D/ConnectivityMonitor(16236): ACCESS_NETWORK_STATE permission granted, registering connectivity monitor
11-18 18:48:55.135 V/ConnectivityMonitor(15773): connectivity changed: false
11-18 18:49:00.701 V/ConnectivityMonitor(15773): connectivity changed: true
```

而如果权限缺失，你将看到一条错误：

```
11-18 18:51:23.673 D/ConnectivityMonitor(16236): ACCESS_NETWORK_STATE permission missing, cannot register connectivity monitor
```

##### 本地存储 (Local Storage)
要从本地文件夹或 DCIM 或图库中加载图片，你将需要添加 ``READ_EXTERNAL_STORAGE`` 权限：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="your.package.name"

    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />

    <application>
      ...
    </application>
</manifest>
```

而如果要使用 [``ExternalPreferredCacheDiskCacheFactory``][7] 来将 Glide 的缓存存储到公有 SD 卡上，你还需要添加 ``WRITE_EXTERNAL_STORAGE`` 权限：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="your.package.name"

    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

    <application>
      ...
    </application>
</manifest>
```

#### Proguard

如果你有使用到 proguard，那么请把以下代码添加到你的 ``proguard.cfg`` 文件中：
```
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.module.AppGlideModule
-keep public enum com.bumptech.glide.load.resource.bitmap.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}

# for DexGuard only
-keepresourcexmlelements manifest/application/meta-data@value=GlideModule
```

#### Jack

Glide 的构建配置需要使用一些 [Jack][3] 目前还不能支持的特性。并且由于 Jack 最近已经被标记为 [deprecated][4]，Glide 需要使用的特性可能在未来也不会被加入了。如果你希望使用 Java 8 编译，请看下文。

#### Java 8
从 Android Studio 3.0 和 Android Gradle plugin 3.0 版本开始，你可以使用 Java 8 来编译你的项目和 Glide 。关于更多详细信息，请访问 Android 开发者网站的 [Use Java 8 LaAnguage Features][5] 。

Glide 本身没有使用，也不要求你使用 Java 8 来编译或在你项目中使用 Glide。Glide 最终肯定也将需要使用 Java 8 来编译，但是我们将尽量为开发者们留出时间来先更新他们自己的应用，因此看起来 Java 8 在未来的几个月或数年内都不会成为一个需求（截止 11/2017）。

#### Kotlin

如果你在 Kotlin 编写的类里使用 Glide 注解，你需要引入一个 ``kapt`` 依赖，以代替常规的 ``annotationProcessor`` 依赖：

```groovy
dependencies {
  kapt 'com.github.bumptech.glide:compiler:4.5.0'
}
```

请注意，你还需要在你的 `build.gradle` 文件中包含 `kotlin-kapt`插件：

```groovy
apply plugin: 'kotlin-kapt'
```

关于 Kotlin 的更多 api，可以查看 [Generated API][6]。

[1]: https://github.com/bumptech/glide/releases/download/v3.6.0/glide-3.6.0.jar
[2]: http://developer.android.com/tools/support-library/features.html#v4
[3]: https://source.android.com/source/jack
[4]: https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html
[5]: https://developer.android.com/studio/write/java8-support.html
[6]: {{ site.baseurl }}/doc/generatedapi.html#kotlin
[7]: {{ site.baseurl }}/javadocs/431/com/bumptech/glide/load/engine/cache/ExternalPreferredCacheDiskCacheFactory.html
[8]: https://github.com/bumptech/glide/issues/2730
[9]: https://docs.gradle.org/current/userguide/dependency_management.html#ssub:artifact_dependencies


