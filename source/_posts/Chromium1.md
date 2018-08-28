---
title: (Chromium源码系列一)Chromium简介及源代码获取和编译
categories:
    - '技术'
    - '源码阅读'
tags:
    - chrome
    - 源码阅读
---

### Chromium简介

​        [`Chromium`](https://www.chromium.org/Home)是一个由`Google`主导开发的网页浏览器，以`BSD`许可证等多重自由版权发行并开放源代码。`Chromium`的开发早自2006年即开始，设计思想基于简单、高速、稳定、安全等理念，在架构上使用了`Apple`发展出来的`WebKit`排版引擎、`Safari`的部分源代码与`Firefox`的成果，并采用`Google`独家开发出的`V8`引擎以提升解析`JavaScript`的效率，而且设计了[沙盒]、[黑名单]、[无痕浏览]等功能来实现稳定与安全的网页浏览环境。

​	在[此](https://www.netmarketshare.com/)可以查看各个浏览器的市场占有率。

<!--more-->

#### Chromium vs Chrome

`Chromium`与`Chrome`的差异主要表现在以下方面：

1. 程序图标：两者图标只在色彩上不同，`Chromium`是天蓝色，而`Chrome`是`Google`公司的代表色（红、黄、蓝、绿）；
   ![chromium logo](/pictures/Chromium1/chromium-logo.jpg)
   ![chrome logo](/pictures/Chromium1/chrome-logo.jpg)

   2. 自动更新：`Chromium`不开放自动更新功能，所以用户需要手动下载更新，而`Chrome`则可自动脸上`Google`的服务器更新，但新版的推出很慢；
   3. 安装模式：`Chromium`可以免安装，下载[`zip压缩包`](https://www.chromium.org/getting-involved/download-chromium)后解压即可使用，而`Chrome`则只有安装板；
   4. 功能差异：新功能会率先在`Chromium`上推出，`Chrome`则会相对落后很多。

### 获取Chromium源代码

可以先看一下[官方文档](https://chromium.googlesource.com/chromium/src/+/master/docs/mac_build_instructions.md)中的说明。简单来讲，获取`Chromium`源代码之前，需要能翻墙，系统需要满足一定的要求，我这里使用的是`Mac`系统，就按照`Mac`的要求来做，另外我使用的shadowsocks来访问google。

#### 安装depot_tools

`depot_tools`是`Google`官方提供的一个用来`checkout`、`compile`、`run`和`submit`的工具集，可以帮助我们更好的学习和调试`Chromium`代码，因此我们先安装`depot_tools`。

1. 克隆 `depot_tools repository`

   ```sh
   git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
   ```

2. 添加`depot_tools`路径到`PATH`变量

   ```sh
   export PATH="$PATH:/path/to/depot_tools"
   ```

   假设你把`depot_tools`放置在目录`path/to/`目录下。最好将上述命令添加到`~/.bashrx`或者`~/.zshrc`中，然后执行`source ~/.bashrx`或者`source ~/.zshrx`。

#### 获取代码

1. 首先确保`Unicode`文件名不会破坏`HFS`。执行

   ```sh
   git config --global core.precomposeUnicode true
   ```

2. 创建`chromium`目录，切记`chromium`所在的目录名中没有空格。我在下载`depot_tools`之前已经创建了`chromium`目录，并且将`depot_tools`和`src`都放在了`chromium`目录下。执行

   ```sh
   mkdir chromium && cd chromium
   git config --global core.precomposeUnicode true
   ```

3. 使用`depot_tools`获取`chromium`代码。执行

   ```sh
   fetch chromium
   # or
   fetch chromium --no-history # 不下载全部的代码提交历史，推荐使用这个
   ```

下载完成后，会有一个`.gclient`文件，以及源代码目录`src`，之后的所有操作都在src中执行。

#### 构建工程

`Chromium`使用[`Ninja`](https://ninja-build.org/)和[`GN`](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/quick_start.md)作为主要的构建工具。执行

   ```sh
   gn gen out/Default
   ```

其中`out/`是在`src`目录下，`Default`可以是别的名字，但是一定要在`out`目录下。

#### 构建Chromium

使用`Ninja`来构建`Chromium`程序。执行

   ```sh
   ninja -C out/Default chrome
   ```

执行这条命令需要挺长时间，我跑了十多个小时才完成`build`，不过好的一点是，即使中途中断了，再重启也可以在之前的基础上使用`gclient sync`命令继续构建。完成之后就可以在`out/Default`目录中看到Chromium浏览器的应用程序了。
![chromium build structure](/pictures/Chromium1/chromium-build-structure.png)

#### 使用`Xcode`来构建Chromium

我们要使用`Xcode`来阅读和调试`Chromium`代码，因此我们需要执行

   ```sh
   gn gen out/gn --ide=xcode
   ```

在用`Ninja`和`GN`构建完成之后，执行这条命令需要的时间就比较少了。然后就可以用`Xcode`来打开这个工程了。执行

   ```sh
   open out/gn/ninja/all.xcworkspace
   ```

下面是用`Xcode`打开工程后的代码结构。![chromium code structure](/pictures/Chromium1/code-structure.png)

至此，我们就在本地构建好了Chromium的源代码，可以开始Chromium源代码的阅读之旅了。