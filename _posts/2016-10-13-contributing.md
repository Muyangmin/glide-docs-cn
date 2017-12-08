---
layout: page
title: "贡献代码或文档"
category: dev
date: 2016-10-13 08:29:39
disqus: 1
order: 2
---

原文链接：[点击查看](http://bumptech.github.io/glide/dev/contributing.html){:target="_blank"}

* TOC
{:toc}

### 贡献代码

非常欢迎你为 Glide 源码做贡献！

#### 工作流

如果你想向 Glide 贡献代码，你需要：

1. 从 GitHub 上 [Fork][Github Fork] [Glide 仓库][1] 。
2. [Clone][Github Clone] 你的仓库到你的电脑上：

   ```sh
   git clone https://github.com/<your_username>/glide.git
   cd glide
   ```
3. 在 Android Studio 中打开 (目录为 Android Studio 3.0+ 的结构)
    1. 打开 Android Studio 
    2. 点击 "Import Project"
    3. 浏览并转到你先前克隆下来的路径
    4. 点击 'settings.gradle'
    5. 点击 'Open'
4. 贡献你的代码。
5. 提交你的修改:

   ```sh
   git add . 
   git commit -m "Describe your change here."
   ```
   
6. 往你自己的 fork 推送修改：  

   ```sh
   git push origin master
   ```
   
7. 在 Github 上打开你的 fork (`https://github.com/<your_username>/glide`)
8. 在 GitHub 上发送一个 [pull request][2] 到主仓库。

#### 构建项目

要构建项目，你通常需要在项目的根目录下运行一个 gradle 命令：

``./gradlew build``


#### 测试你的修改

##### 测试
Glide 拥有两种类型的测试，在你的本地机器上运行的单元测试（unit test），和在模拟器或设备上执行的仪器测试 (instrumentation test)。

##### 单元测试
Glide 的单元测试是作为 Glide 的构建过程的一部分来允许的，所以你可以直接使用：

 ``./gradlew build``
 
为了加快开发周期，你也可以只运行主库的单元测试：

``./gradlew :library:testDebugUnitTest``

##### 仪器测试
为了运行 Glide 的仪器测试，你需要插入一个真机，或使用 Android Studio 添加一个模拟器。现在在 Android Studio 中添加模拟器已经很容易，并且 x86 的模拟器启动和运行也相当快。因此，我通常推荐你在一个模拟器上执行 Glide 的仪器测试。

要执行 Glide 的仪器测试：
1. [在 Android Studio 中配置一个模拟器][Android Studio emulator] (我通常使用 x86 和 API 26)
2. 执行:

    ``./gradlew :instrumentation:connectedDebugAndroidTest``

##### Sample 项目
Glide 的测试并不特别全面。为了验证你的修改能用且不会给性能带来消极影响，尝试运行 Glide 的示例项目是一个好主意。

Glide 的示例项目位于 samples/ 中。示例项目可以被 gradle 构建和安装到设备或模拟器中：

``./gradlew :samples:<sample_name>:run``

例如，运行 Flickr demo：

``./gradlew :samples:flickr:run``

#### 代码风格

Glide使用 [Google Java 风格指南][3]。

为了让 Android Studio 自动使用 Google 风格，你需要做以下步骤：

1. 打开 [https://raw.githubusercontent.com/google/styleguide/gh-pages/intellij-java-google-style.xml][4]；
2. 保存 intellij-java-google-style.xml 到你电脑上；
3. 打开 Android Studio
4. 打开 Preferences...
5. 打开 Editor > Code Style
6. 在 'Schema' 旁边，点击 'Manage'
7. 点击 Import...
8. 高亮 'Intellij IDEA code style XML' 然后点击 'Ok'
9. 查看你第2步下载的 intellij-java-google-style.xml 的路径，选中该文件，点击 'Ok'
10. 点击'Ok'（你可以选择性地修改 GoogleStyle 这个名字）
11. 在 Code Style Schemes 对话框中，高亮你刚刚创建的风格，然后点击 'Copy to project'
12. 点击 ok ，关闭偏好设置对话框。

在添加这个风格指南到 Android Studio 之后，要重新格式化代码，你只需要打开 Code 菜单，然后点击 'Reformat Code' 。

所有的新代码都应该遵循这个指定的风格指南，并且在 Glide 的测试用例里有一些自动的约束。修复 Glide 既有代码的风格问题的 Pull request 也很欢迎。然而，通常最好把修复既有代码风格问题的修改集与添加新代码的修改集分开。开两个 pull request 是完全没有问题的，如果你想修复一些风格问题并同时贡献一些新功能或 bug 修复的话。

如有疑问，请给我们一个单一的 pull request ，我们可以为你提供帮助。

### 贡献文档

#### 上传一个修改

如果你想贡献文档的话：

1. 从 GitHub 上 [Fork][Github Fork] [Glide 仓库][1] 。
2. [Clone][Github Clone] 你的仓库到你的电脑上：

   ```sh
   git clone https://github.com/<your_username>/glide.git
   cd glide
   ```
3. 检出 gh-pages 分支：
   
   ```sh
   git checkout -t origin/gh-pages
   ```
4. 完成你的修改。
5. 提交你的修改:

   ```sh
   git add . 
   git commit -m "Describe your change here."
   ```
6. 向你的 Glide fork 推送修改：

   ```sh
   git push origin gh-pages 
   ```
  
7. 在 Github 上打开你的 fork (`https://github.com/<your_username>/glide`)
8. 在 GitHub 上发送一个 [pull request][2] 到主仓库，注意使用 ``gh-pages`` 分支。

#### 修改已有的页面

你在文档中看到的所有页面都位于 ``_pages`` 文件夹下，并可以在那里修改它们。

#### Adding a new page
新页面可以使用 ``./bin/jekyll-page <page_name> <category>`` 来添加。 其中，``<page_name>`` 是页面标题， ``category`` 对应左边导航菜单的章节。通常 ``<category>`` 应为 ``doc``，以使页面在 ``Documentation`` 章节下展示。

当你添加一个新页面时，请确保在头部添加 ``disqus: 1`` 和 ``order: <n>``。其中你赋予 order 的值用于在子章节中排序页面，0 表示第一页。如果只是简单地将页面添加在尾部（很适合默认情况），请找出上个页面的 order 值，并为你的新页面使用这个值+1。

最终的头部看起来像这样：
```
---

title: "Targets"
category: doc
date: 2015-05-26 07:03:23
order: 6
disqus: 1
---
```

#### 查看你的本地修改

要查看你的本地修改，你需要安装 jekyll 和 gems ：

``sudo gem install jekyll redcarpet pygments.rb``

然后你可以本地运行 jekyll :

``jekyll serve --watch``

最后，你可以查看一个包含你本地修改的网站版本：``http://127.0.0.1:4000/glide/``。 jekyll 会告诉你确切的地址。

[1]: https://github.com/bumptech/glide
[2]: https://help.github.com/articles/creating-a-pull-request/
[3]: https://google.github.io/styleguide/javaguide.html
[4]: https://raw.githubusercontent.com/google/styleguide/gh-pages/intellij-java-google-style.xml
[Github Clone]: https://help.github.com/articles/cloning-a-repository/
[Github Fork]: https://help.github.com/articles/fork-a-repo/
[Android Studio Emulator]: https://developer.android.com/studio/run/managing-avds.html#createavd

