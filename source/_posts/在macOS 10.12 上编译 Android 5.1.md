
---
title: 在macOS 10.12 上编译 Android 5.1
tag: Android
---

---

官方文档虽然也有介绍，但是macOS平台上的编译环境问题还存在很多坑。本文介绍下如何在在macOS 10.12 上编译 Android 5.1源码，导入源码到Android Studio中，把系统烧录到Nexus6手机中。

---
## 搭建编译环境

### 创建分区
AOSP源码需要一个支持大小写敏感的文件系统，100G是至少要的。[官网](https://source.android.com/source/initializing.html#setting-up-a-mac-os-x-build-environment)有详细的介绍，这里简单列一下。
```bash
$ hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 40g ~/android.dmg
```
然后挂载这个分区：
```bash
$ hdiutil attach ~/android.dmg -mountpoint /Volumes/android;
```

### 切换shell
Android的相关编译只能是使用bash.
```
$ chsh -s /bin/bash
```
重启终端。

### 安装Xcode
1、这里需要两个Xcode，可以用命令切换需要使用的Xcode，会有不同的用处。
 - 去AppStore下载最新的Xcode
 - 到[这里](https://developer.apple.com/download/more/)下载5.1.1的Xcode
 
2、创建一个`/Developer/SDK`文件夹，从Xcode5.1.1中把`MacOSX10.8.sdk`从`Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/`复制到 `/Developer/SDK`中。
3、从`Xcode5.1.1.dmg`复制Xcode.app 到  `/Developer`目录中。
4、从AppStore下载的最新版Xcode会默认放在`/Applications`目录中
5、给两个版本的Xcode都安装command line tools
```bash
$ sudo xcode-select -switch /Developer/Xcode.app
$ xcode-select --install
$ sudo xcode-select -switch /Applications/Xcode.app
$ xcode-select --install
```

### 安装JDK
编译Android5.1需要jdk1.7，去[官网](http://www.oracle.com/technetwork/java/javase/downloads/jre7-downloads-1880261.html)下载.
如果有切换多个版本的jdk需求的话，可以使用`jenv`这个工具，参考[在OS X中使用jEnv管理多个Java版本](http://boxingp.github.io/blog/2015/01/25/manage-multiple-versions-of-java-on-os-x/)


### 安装其他软件
1、安装MacPorts，需要去[官网](https://www.macports.org/install.php)下载对应版本的MacPorts
2、配置port命令环境变量
```bash
$ export PATH=/opt/local/bin:$PATH
```
3、下载依赖包
```bash
$ POSIXLY_CORRECT=1 sudo port install gmake libsdl git gnupg
```
---
## 下载源码
直接去google官方下载会很慢，这里推荐用[中科大镜像](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp)
1、首先下载 repo 工具。
```bash
$ mkdir ~/bin
$ PATH=~/bin:$PATH
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
## 如果上述 URL 不可访问，可以用下面的：
## curl https://storage-googleapis.proxy.ustclug.org/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
```
2、在之前创建的大小写分区上建立一个工作目录，之后源码下载和编译都在这里进行。
```bash
$ mkdir WORKING_DIRECTORY
$ cd WORKING_DIRECTORY
```
3、初始化仓库
```bash
$ repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest
## 如果提示无法连接到 gerrit.googlesource.com，可以编辑 ~/bin/repo，把 REPO_URL 一行替换成下面的：
## REPO_URL = 'https://gerrit-googlesource.proxy.ustclug.org/git-repo'
```
4、选择某个特定的 Android 版本，具体查看[这里](https://source.android.com/source/build-numbers.html#source-code-tags-and-builds),我选择的是android-5.1.1_r14，build号是LMY48M，等会用这个build号下载对应的驱动包，烧录到nexus真机时会用到。
```bash
$ repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-5.1.1_r14
```
5、同步源码树
```bash
$ repo sync
```

源码下载完后，如果没有同步的需求的话，就可以把`.repo`目录删掉了，防止编译时磁盘空间不够用。


## 下载驱动
烧录到真机时需要用到，默认只是用模拟器的话，可以跳过这步。
在https://developers.google.com/android/nexus/drivers#hikey中找到对应设备与源码分支的硬件驱动。刚才选择的源码分支所对应的build码是LMY48M，因此，就下载此代号的驱动程序即可。
下载得到的是三个tgz文件，我们只需依次解压三个文件，得到的是三个shell脚本文件，我们先将其置于源码根目录中。
依次执行这3个脚本将在源码根目录中生成一个vendor文件夹。

---
## 编译
### 设置文件描述符限制
在macOS中，默认限制的同时打开的文件数量很少，不能满足编译过程中的高并发需要，因此需要在shell中运行命令：
```bash
$ ulimit -S -n 1024
```
### 环境设置
在源码根目录下调用下面的命令：
```bash
$ source build/envsetup.sh
```
### 选择设备
因为我编译后需要烧录到Nexus6上，所以选择`aosp_shamu-userdebug`
```bash
$ lunch aosp_shamu-userdebug
```
如果不需要烧录到真机上的话，用默认的`aosp_arm-eng`类型就可以了。

### 开始编译
因为本机CPU的内核是8核的，所以开16个线程加快编译。
```bash
$ make -j16
```
编译成功后，会有类似下面的日志：
```bash
#### make completed successfully (30:28:08 (hh:mm:ss)) ####
```
编译成功的结果都在`out`目录中。
如果lunch的是`aosp_arm-eng`类型，就可以用`$ emulator`命令刷到模拟器了。

---
## 源码导入到Android Studio中
为了方便查看源码，可以把代码导入到AS中。目前看来，只能支持Java的跳转，对c++的支持不太好。
为了让AS理解代码的符号和源码树的结构，需要用如下命令生成一个`android.ipr`工程配置文件。
```bash
$ mmm development/tools/idegen/
$ development/tools/idegen/idegen.sh
```
大约需要十几秒的时间，就能在源码根目录下生成android.ipr和android.iml了。
用AS打开android.ipr就能导入整个源码了。
如果要支持跳转的话，还需要做些配置，可以看这篇教程：[Import AOSP into Android Studio](http://blog.justain.net/index.php/import-aosp-into-android-studio/)

---
## 刷机
Nexus6手机在打开USB调试，连接电脑后允许调试这台手机，并且在设置中打开“允许 OEM 解锁”。然后令手机进入recovery模式，在关机下，输入如下命令即可：
```bash
$ adb reboot bootloader
```
执行如下命令刷机：
```bash
$ fastboot -w flashall
```
刷机成功后，手机会自动重启，新鲜出炉的系统终于跑起来了。
刷机过程中也出现过变砖的情况，可以试一下[这个教程](http://www.shuame.com/faq/restore-tutorial/14679-google-nexus6.html)，亲测有效。

---
## 相关链接
[Build Android 5.0 Lollipop on OSX 10.10 Yosemite](https://medium.com/@raminmahmoodi/build-android-5-0-lollipop-on-osx-10-10-yosemite-441bd00ee77a)
[http://blog.bihe0832.com/macOS-AOSP.html](http://blog.bihe0832.com/macOS-AOSP.html)
[在OS X中使用jEnv管理多个Java版本](http://boxingp.github.io/blog/2015/01/25/manage-multiple-versions-of-java-on-os-x/)
[Import AOSP into Android Studio](http://blog.justain.net/index.php/import-aosp-into-android-studio/)
[Nexus 6 恢复官方兼救砖](http://www.shuame.com/faq/restore-tutorial/14679-google-nexus6.html)
