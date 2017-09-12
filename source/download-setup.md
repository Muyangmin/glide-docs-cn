---
title: "下载和初始化"
---
原文链接：[点击查看](http://bumptech.github.io/glide/doc/download-setup.html)

### 下载

可以使用多种方法访问Glide的公开发行版。

#### Jar

你可以直接从GitHub下载[最新的jar包][1]。请注意，你还需要包含Android的[v4支持库][2]。

#### Gradle

如果你使用Gradle，那么你可以添加对Glide的依赖，从Maven Central或者JCenter都可以。同样，你还需要包含对支持库的依赖。

```groovy
repositories {
  mavenCentral()
}

dependencies {
    compile 'com.github.bumptech.glide:glide:4.0.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.0.0'
    compile 'com.android.support:support-v4:25.3.1'
}
```

#### Maven

如果你使用Maven，你也可以添加对Glide的依赖。再说一遍，你需要包含对支持库的依赖。

```xml
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>glide</artifactId>
  <version>4.0.0</version>
  <type>aar</type>
</dependency>
<dependency>
  <groupId>com.google.android</groupId>
  <artifactId>support-v4</artifactId>
  <version>r7</version>
</dependency>
```

### 设置

你可能需要做一些额外的设置步骤，这取决于你的构建配置。

#### Proguard

如果你使用proguard，那么请把以下代码添加到你的``proguard.cfg``：
```
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.AppGlideModule
-keep public enum com.bumptech.glide.load.resource.bitmap.ImageHeaderParser$** {
    **[] $VALUES;
    public *;
}
```

#### Jack

Glide的构建配置需要使用一些[Jack][3]目前还不能支持的特性。并且由于Jack最近已经被标记为[deprecated][4]，Glide需要使用的特性可能在未来也不会被加入了。

#### Java 8

目前(2017年6月)还没有一个稳定的Android工具链能允许你将Glide和Java 8特性一起使用。如果你希望使用Java 8，并且允许牺牲一定的稳定性，那么，至少目前已经有一个alpha版本的Android Gradle插件可以支持Java 8. 但是，Alpha版本的插件目前还未经Glide测试过。如果你对此感兴趣，可以查看Android的[Java 8支持页][5]。

#### Kotlin

如果你在使用Kotlin实现的类上使用Glide的注解，你需要引入一个``kapt``依赖，以代替常规的``annotationProcessor`` 依赖：

```groovy
dependencies {
  kapt 'com.github.bumptech.glide:compiler:4.0.0'
}
```

关于Kotlin的更多api，可以查看[Generated API][6]。

[1]: https://github.com/bumptech/glide/releases/download/v3.6.0/glide-3.6.0.jar
[2]: http://developer.android.com/tools/support-library/features.html#v4
[3]: https://source.android.com/source/jack
[4]: https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html
[5]: https://developer.android.com/studio/write/java8-support.html
[6]: {{ site.url }}/glide/doc/generatedapi.html#kotlin
