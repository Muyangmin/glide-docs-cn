---
layout: page
title: "快照版本"
category: dev
date: 2017-05-20 12:37:05
disqus: 1
order: 1
---

原文链接：[点击查看](http://bumptech.github.io/glide/dev/snapshots.html){:target="_blank"}

## 关于快照(Snapshots)
对于那些等不了 Glide 的下一个稳定版的，喜欢在刀尖上跳舞的用户【注】，我们在[Sonatype's snapshot repo][2]部署了 Glide 库的快照版本。  
> 原文"willing to live on the bleeding edge"，请自行感受…… --译者注


在每次 push 到 GitHub 的 master 分支上后，Glide 会通过[travis-ci][1]构建。如果构建成功，我们将自动部署最新版本的库到 Sonatype 上。

每个集成库都有它自己的快照，与主 Glide 库一样。如果你使用了 Glide 库的快照版本，你使用的任何集成库也要使用快照版本，反之亦然。

## 获取快照
Sonatype 的快照仓库的工作原理与其他maven仓库一样，所以快照可以多种方式访问：jar, maven，或者 gradle。

### Jar
你可以直接从 [Sonatype][3] 下载。请务必检查日期以确保你正在获取的是最新的版本。

### Gradle

首先你需要把快照仓库添加到你的仓库列表：

```gradle
repositories {
  jcenter()
  maven {
    name 'glide-snapshot'
    url 'http://oss.sonatype.org/content/repositories/snapshots'
  }
}
```

然后修改你的依赖为快照版本：

```gradle
dependencies {
  compile 'com.github.bumptech.glide:glide:4.0.0-SNAPSHOT'
  compile 'com.github.bumptech.glide:okhttp-integration:4.0.0-SNAPSHOT'
}
```

### Maven
*请注意，这种方法未经测试，是从 [Stack Overflow][4] 的这个问题而来的。关于本节有任何建议，欢迎提出！*

请将下列代码添加到你的`~/.m2/settings.xml`:

```xml
<profiles>
  <profile>
     <id>allow-snapshots</id>
     <activation><activeByDefault>true</activeByDefault></activation>
     <repositories>
       <repository>
         <id>snapshots-repo</id>
         <url>https://oss.sonatype.org/content/repositories/snapshots</url>
         <releases><enabled>false</enabled></releases>
         <snapshots><enabled>true</enabled></snapshots>
       </repository>
     </repositories>
   </profile>
</profiles>
```

然后修改你的依赖为快照版本：

```xml
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>glide</artifactId>
  <version>4.0.0-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>okhttp-integration</artifactId>
  <version>4.0.0-SNAPSHOT</version>
</dependency>
```

### 本地构建快照
如果你想获取将发布的相同文件以本地编译，请执行下面这个命令：
```shell
gradlew clean buildArchives uploadArchives --stacktrace --info -PSNAPSHOT_REPOSITORY_URL=file://p:\path\to\repo -PRELEASE_REPOSITORY_URL=file://p:\path\to\repo
```
这将创建一个 m2 仓库文件夹，你可以用 Gradle 在一个工程里测试你的修改：
```gradle
repositories {
  //确保这行在 glide-snapshot之前，使它成为首先被查询的仓库
  maven { name 'glide-local'; url 'p:\\path\\to\\repo' }
}
dependencies {
  //开启这个选项，确保所有变更生效
  //configurations.compile.resolutionStrategy.cacheChangingModulesFor 0, 'seconds'
  compile 'com.github.bumptech.glide:glide:x.y.z-SNAPSHOT'
}
```

[1]: https://travis-ci.org/bumptech/glide
[2]: https://oss.sonatype.org/content/repositories/snapshots/
[3]: https://oss.sonatype.org/content/repositories/snapshots/com/github/bumptech/glide/
[4]: http://stackoverflow.com/questions/7715321/how-to-download-snapshot-version-from-maven-snapshot-repository
