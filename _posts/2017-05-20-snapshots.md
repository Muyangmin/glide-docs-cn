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
对于那些等不了 Glide 的下一个稳定版的用户，我们在 [Sonatype's snapshot repo][2] 部署了 Glide 库的快照版本。  

在每次 push 到 GitHub 的 master 分支上后，Glide 会通过 [travis-ci][1] 构建。如果构建成功，我们将自动部署最新版本的库到 Sonatype 上。

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
  compile 'com.github.bumptech.glide:glide:4.12.0-SNAPSHOT'
  compile 'com.github.bumptech.glide:okhttp-integration:4.12.0-SNAPSHOT'
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
  <version>4.12.0-SNAPSHOT</version>
</dependency>
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>okhttp-integration</artifactId>
  <version>4.12.0-SNAPSHOT</version>
</dependency>
```

### 修复你的快照依赖
使用 Glide 的 快照 (``-SNAPSHOT`) 版本可能为应用带来风险，因为 gradle 将拉取的快照版本代码将依赖于你何时首次构建你的项目。假如你添加了一个快照依赖，并随后在你的本地机器上做好了测试，然后将这个构建配置推送到了构建服务器，构建服务器可能最终会使用另一个版本的 Glide 代码来完成构建。快照版本将在每次成功 push 到 GitHub 后发生改变，并且这种改变可能在任何时候发生。

要解决这种问题，你可以从 sonatype 指定一个特定的版本而不是依赖于 ``-SNAPSHOT``。例如在 Gradle 中：

```gradle
dependencies {
  compile 'com.github.bumptech.glide:glide:4.3.0-20171024.022226-26'
  compile 'com.github.bumptech.glide:okhttp-integration:4.3.0-20171024.022226-26'
}
```

或使用 Maven(未测试):

```xml
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>glide</artifactId>
  <version>4.3.0-20171024.022226-26</version>
</dependency>
<dependency>
  <groupId>com.github.bumptech.glide</groupId>
  <artifactId>okhttp-integration</artifactId>
  <version>4.3.0-20171024.022226-26</version>
</dependency>
```

这里的版本号 ``4.3.0-20171024.022226-26``，是从 Sonatype 仓库中取得的。你可以通过以下方法选择特定的版本：

1. 打开 [Sonatype][3]；
2. 点击你需要使用的 package 。通常你可以只使用 [glide][5]；
3. 点击你想使用的 Glide 快照版本，例如 [4.4.0-SNAPSHOT][6]；
4. 从列出的 artifact 中选择一个并复制粘贴版本号即可。例如你看到 ``glide-4.3.0-20171024.022211-26-javadoc.jar``, 那么它的版本号就是 ``4.3.0-20171024.022211-26``。通常你会想要使用最新可用的 artifact 。你可以检查修改日期列来确认这一点，但一般而言最近的 artifact 都展示在页面的底部。 

尽管选择特定快照版本要稍微麻烦一些，但相比你在应用程序或库的生产版本中使用 Glide 的快照版本依赖的话，这通常是一个更安全的选择。

### 本地构建快照
Maven 允许你在特定的 Maven 仓库中安装 artifact 并在其他项目中依赖这些 artifact。通常这是一个较为简单的方法来在第三方项目中测试对 Glide 的修改。你可以在两个地方：

#### 在默认的本地 Maven 库中安装
如果你只是简单地想在你的项目中测试一些你对 Glide 的修改 (或使用 Glide 的某个特定版本或提交来编译你的项目)，你可以在默认的本地 Maven 仓库中安装 Glide。

为了这样做，你需要将以下代码添加到你的 ``build.gradle`` 文件的 ``repositories`` 部分中：

```groovy
repositories {
  mavenLocal()
}
```

然后使用 ``-PLOCAL`` 来构建 Glide：

```shell
./gradlew uploadArchives --parallel -PLOCAL
```

#### 在特定的本地或远程仓库中安装
如果你需要指定一个特定的本地或远程仓库来安装 Glide ，你可以使用以下命令：

```shell
./gradlew uploadArchives --stacktrace --info -PSNAPSHOT_REPOSITORY_URL=file://p:\path\to\repo -PRELEASE_REPOSITORY_URL=file://p:\path\to\repo
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
[5]: https://oss.sonatype.org/content/repositories/snapshots/com/github/bumptech/glide/glide/
[6]: https://oss.sonatype.org/content/repositories/snapshots/com/github/bumptech/glide/glide/4.4.0-SNAPSHOT/

