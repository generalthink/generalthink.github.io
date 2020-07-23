---
title: 在MacOS系统上编译OpenJDK12并使用CLion调试
date: 2020-07-22 15:44:32
tags: jvm
---

最近在看synchronized 锁优化方面的内容,有些地方看起来不是很方便,干脆就编译个源码来看看。


### 在windows上编译

由于自己常用的电脑操作系统是win10,所以最开始是想要在win10上编译的,但是一来网上文章太少,二来在windows上编译确实麻烦太多了(windows可以参考深入理解JVM虚拟机这本书),故放弃了。


<!--more-->

### MAC环境

![mac信息](/images/macos-jvm/macos.png)

### 准备

#### 获取源码

OpenJDK源码使用Mercurial管理,如果通过版本库下载,则需要安装Mercurial,我们借助homebrew包管理器来安装

```
brew install mercurial
```

如果你的电脑没有安装homebrew,那么可以使用下面的命令来安装

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

安装完成后使用以下命令clone源代码

```
cd ~/jvm
hg clone https://hg.openjdk.java.net/jdk/jdk12
```

> 命令运行之后,openjdk的源码并没有下载下来

由于我并没有使用hg的方式,关于这部分可以参考闪电侠的文章<https://www.jianshu.com/p/ee7e9176632c>


**当然我非常不建议你使用Mercurial的方式下载,这种方式不久要等很久,而且还需要科学上网才稳妥。我墙裂推荐你直接通过页面下载[OpenJdk12源码压缩包](https://hg.openjdk.java.net/jdk/jdk12))**

![下载OpenJDK12源码](/images/macos-jvm/openjdk12-source-code.png)

比如我下载的是gz格式的文件(选择什么格式并不重要)。

> 如果你使用的Chrome,那么可以开启多线程下载,速度会有一个很明显的提升。

#### Bootstrap JDK

因为OpenJDK的各个组成部分有的是使用C++编写的,有的是使用Java编写的,因此编译这些Java代码需要使用到一个可用的JDK,官方称这个JDK为“Bootstrap JDK",一般来说只需要比编译的JDK第一个版本,这里采用OpenJDK11,可以通过这个网址 <https://jdk.java.net/archive/> 下载 
记住一定要下载一个适合Mac平台的OpenJDK11。

我使用的Bootstrap JDK版本如下

```
openjdk version "11.0.2" 2019-01-15
OpenJDK Runtime Environment 18.9 (build 11.0.2+9)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.2+9, mixed mode)
```

### 安装依赖

```
# 用于生成shell脚本的工具,可以使软件包在不同的系统下都可以编译
brew install autoconf

# 字体引擎
brew install freetype
```

#### XCode 和 Command Line Tools for Xcode

这两个SDK提供了OpenJDK所需的编译器以及Makefile中用到的外部命令。一般电脑上都自带安装了。

网上很多编译出错都是因为XCode版本问题,导致报错,我也遇到了这个坑。

```
configure: error: No xcodebuild tool and no system framework headers found, use --with-sysroot or --with-sdk-name to provide a path to a valid SDK

/Users/.../.../xxx.sh: line 82: 5: Bad file descriptor

configure exiting with result code 1

```
最终我通过官网下载<https://developer.apple.com/download/more/>安装了9.4.1版本的XCode才搞定了。
> 下载xcode需要登录,如果没有账号可以申请一个


我选择的是手动下载xcode安装的方式,你可以先只下载Command Line Tools (macOS 10.13) for Xcode 9.4.1 安装看能不能解决你的问题,如果不能再下载Xcode(这个很大,大概4个多G)

![xcode下载](/images/macos-jvm/download-xcode.png)


### 编译jdk

源码下载好之后,我解压放到了这个`/Users/naver/jvm/jdk12-06222165c35f`目录下,下面的命令均是在这个目录下执行的。

```
bash configure  --with-boot-jdk='/usr/local/jdk-11.0.2.jdk/Contents/Home' --with-debug-level=slowdebug --with-target-bits=64 --disable-warnings-as-errors --enable-dtrace --with-jvm-variants=server
```

--with-boot-jdk：指定Bootstrap JDK路径
--with-debug-level：编译级别,可选值为release、fastdebug、slowdebug和optimized,默认值为release,如果我们要调试的话,需要设定为fastdebug或者slowdebug,建议设置为slowdebug
--with-target-bits：指定编译32位还是64位的虚拟机
--disable-warnings-as-errors：避免因为警告而导致编译过程中断
--enable-dtrace：开启一个性能工具
--with-jvm-variants：编译特定模式下的虚拟机,一般这里编译server模式
--with-conf-name：指定编译配置的名称,如果没有指定,则会生成默认的配置名称macosx-x86_64-server-slowdebug,我这里采用默认生成配置

在很多场景下编译OpenJDK都会使用--enable-ccache参数,来通过ccache加快编译速度,但我没有采用,因为目前编译速度其实不慢,再有就是如果增加了这个参数,后续导入到CLion的时候,会出现很多红字提示,看着好像不影响使用,但总归看着不太舒服


#### 生成Compilation Database

在配置CLion的时候,直接import编译好之后的jdk源码,你会发现头文件都是红色的,无法找到提示,是因为CLion生产的CMakeLists.txt有问题,如果想要解决这个问题就需要修改这个文件,很明显我不会修。

最后通过JetBrains说的利用Compilation Database (<https://blog.jetbrains.com/clion/2020/03/openjdk-with-clion/>) 在CLion中构建OpenJDK解决了这个问题。


```
make CONF=macosx-x86_64-server-slowdebug compile-commands
```

执行完该命令,就会在${source_root}/build/macosx-x86_64-server-slowdebug下生成compile_commands.json文件。

![compile_commands.json](/images/macos-jvm/compile_commands_json.png)

#### 编译

在导入CLion之前,要编译一下,因为某些模块使用了预编译头,如果不编译,CLion会在索引过程中提示找不到各种各样的文件。

```
make CONF=macosx-x86_64-server-slowdebug
```

#### 测试

![open-jdk-version](/images/macos-jvm/open-jdk-version.png)

至此,证明我们已经编译完成了JDK12

### CLion调试


#### 导入project

在导入project之前先配置好Toolchains(Preferences进入)

![toolchains](/images/macos-jvm/toolchains.png)


配置好Toolchains后,通过`File -> Open`功能,选中`${source_root}/build/macosx-x86_64-server-slowdebug/compile_commands.json`,`As a project`打开,这样就导入了Compilation Database文件,接下来CLion开始进行索引。

这时候,你会发现你是看不到源码的,所以下面需要修改项目的根目录,通过`Tools -> Compilation Database -> Change Project Root`功能,选中你的源码目录,也就是`${source_root}`,这样设置就可以在CLion中看到源代码啦。

> ${source_root}指的是 /Users/naver/jvm/jdk12-06222165c35f

#### debug之前配置

需要在`Preferences --> Build, Exceution, Deployment --> Custom Build Targets`配置构建目标

![build](/images/macos-jvm/build.png)

![clean](/images/macos-jvm/clean.png)

由于我这里已经配置好了,所以这里显示的是编辑页面,第一次配置要点击+,进行新增即可。

**通过这两个配置每次构建之前都会重新编译我们的jdk,修改jvm代码之后可以直接进行重新调试。**


#### debug 配置

![debug配置](/images/macos-jvm/debug_config.png)

Executable：选择`${source_root}/build/macosx-x86_64-server-slowdebug/jdk/bin/java`,或者其它你想调试的文件,比如javac；
Before luanch：这个下面新增的时候有一个bug,去掉就不会每次执行都去Build,节省时间,但其实OpenJDK增量编译的方式,每次Build都很快,所以就看个人选择了。

#### debug

在`${source_root}/src/java.base/share/native/libjli/java.c`的401行打断点,点击Debug,然后F9放掉,不出意外你会遇到下面这个问题

![sigsegv信息](/images/macos-jvm/debug-sigsegv.png)


由于我们使用的LLDB进行debug的,所以在进入第一个断点的时候在LLDB下执行以下命令可以避免此类问题

```
pro hand -p true -s false SIGSEGV SIGBUS
```

![解决sigsegv问题](/images/macos-jvm/lldb-pro-hand.png)

最终就可以看到java -version的输出效果如下

![java-version](/images/macos-jvm/java-version.png)



不过每次debug的时候都要输入这么一句就很麻烦,所以我们可以在**~/.lldbinit**文件中,使用如下命令,实现每次Debug时自动打个断点,然后输入`pro hand -p true -s false SIGSEGV SIGBUS`,最后继续执行后续流程,文件内容如下(其中main.c文件的路径自行替换)

```
breakpoint set --file /Users/naver/jvm/jdk12-06222165c35f/src/java.base/share/native/launcher/main.c --line 98 -C "pro hand -p true -s false SIGSEGV SIGBUS" --auto-continue true
```

#### 与Java程序联合debug

上面演示的实际是java -version如何debug,那么如何做到通过自己编写的java代码作为程序入口来调试呢？

首先java代码如下(我用idea编写的):

![java-code](/images/macos-jvm/java-code.png)

CLion中配置如下

![debug-java-code](/images/macos-jvm/debug-java-code.png)

运行结果如下:

![result](/images/macos-jvm/debug-java-code-result.png)


### 参考文档

<<深入理解Java虚拟机：JVM高级特性与最佳实践>>
[mac下编译openjdk1.9及集成clion动态调试](https://www.jianshu.com/p/ee7e9176632c)
[在MacOS系统上使用CLion编译并调试OpenJDK](https://www.howieli.cn/posts/macos-clion-build-debug-openjdk12.html)