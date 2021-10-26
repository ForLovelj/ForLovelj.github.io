---
layout: post
title: Android WebRTC源码下载编译-M72版本
author: clow
date: 2019-03-29 16:36:55
categories:
- Android
tags: WebRTC
---
## 1. 概述
WebRTC (Web Real-Time Communications) 是一项实时通讯技术，它允许网络应用或者站点，在不借助中间媒介的情况下，建立浏览器之间点对点（Peer-to-Peer）的连接，实现视频流和（或）音频流或者其他任意数据的传输。WebRTC包含的这些标准使用户在无需安装任何插件或者第三方的软件的情况下，创建点对点（Peer-to-Peer）的数据分享和电话会议成为可能。

编译过程有点曲折，整理一下。

## 2. 环境准备
1.编译需要linux环境，我用的是vm虚拟机 + [ubuntu16.04.6](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/16.04/)（建议磁盘容量分配>=50g）
2.VPN （我用的是SSR）
3.配置代理 以SSR为例 先在ubuntu的“系统设置”的“网络设置"中将代理设置为手动，地址指向代理IP（本示例装在虚拟机中，翻墙是在主机中使用SSR，因此指向物理机地址：192.168.1.47，端口1080）。
**代理非常重要，自行百度，后续很多坑可能都是代理配置不正确导致的！！！**

## 3. 源码下载环境准备
1. 更新软件源
```
sudo apt-get update
```
2. 安装git python>= 2.7
```
sudo apt-get install -y git python
```
3. 安装depot_tools
```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```
配置depot_tools到环境变量
```
//打开.bashrc
gedit ~/.bashrc
//添加环境变量 将下面命令添加到bashrc最后一行 保存退出
export PATH=pwd/depot_tools:$PATH
//执行脚本使其生效 
source ~/.bashrc
//执行命令查看环境变量是否追加到path
echo $PATH
//执行gclient 
gclient
```
等待一会儿如果输出下图所示，证明配置成功，否则检查代理设置和环境变量。如果确保都没问题，禁用depot_tools更新。
>depot_tools updates itself automatically when running gclient tool. To disable auto update, set the environment variable DEPOT_TOOLS_UPDATE=0.
>To update package manually, run update_depot_tools.bat on Windows, or ./update_depot_tools on Linux or Mac.
>On Windows only, running gclient will install git and python.

按照官方提示的来，配置环境变量将export DEPOT_TOOLS_UPDATE=0追加到.bashrc末尾，（类似下面配置Boto），然后重新输入gclient。（如果gclient不能更新，感觉代理可能还是存在问题）

![gclient输出](https://ForLovelj.github.io/img/WebRTC下载和编译)

设置depot_tools代理（gclient sync出现download_from_google_storage错误时的解决方法）
新建一个文件gclient.boto（比如/home/cllow/gclient.boto）,添加以下内容
```
[Boto]
proxy=192.168.1.47
proxy_port=1080
```
将其添加到环境变量
```
//打开.bashrc
gedit ~/.bashrc
//添加环境变量 将下面命令添加到bashrc最后一行 保存退出
export NO_AUTH_BOTO_CONFIG=pwd/gclient.boto
//执行脚本使其生效 
source ~/.bashrc
//可以执行export查看
export
```
4. 源码下载
```
//创建目录
mkdir webrtc
cd webrtc
//拉取代码
fetch --nohooks webrtc_android
gclient sync // 异常断开后，可多执行几次 必须要sync同步完才能开始编译
```
5. 编译
```
//安装jdk 8
sudo apt-get install default-jre
sudo apt-get install default-jdk
//安装编译依赖环境
sudo src/build/install-build-deps-android.sh
//进入到src目录下 执行，配置环境变量
. build/android/envsetup.sh
//此时是master分支，如有需要编译特定版本自行切换需要编译的分支，然后再次执行gclient sync比如：
git checkout <branch>
gclient sync
//编译
//debug
gn gen out/Debug --args='target_os="android" target_cpu="arm"'
ninja -C out/Debug
//release
gn gen out/Release --args='target_os="android" target_cpu="arm" is_debug=false'
ninja -C out/Release
```
编译成功后，输出的关键文件如下：
```
out/Release /lib.java/sdk/android/libwebrtc.jar
out/Release /lib.java/sdk/android/libvpx_vp8_java.jar
out/Release /lib.java/sdk/android/libvpx_vp9_java.jar
out/Release /libjingle_peerconnection_so.so
```
由于生成的gradle工程的源码并不是放在一个位置，而且分散在webrtc各个文件夹中，可以将各个对应文件夹下的源码文件整合到一起。
PS：可以自己通过gradle文件的依赖分析查看源码文件夹的引用路径。
java源码目录如下：
```
//android端demo工程源码
examples/androidapp/src  
//java源码 有个目录是chromium的源码注意区分
modules/audio_device/android/java/src  
base/android/java/src  
rtc_base/java/src  
sdk/android/api 
sdk/android/src/java 
//so库，位于编译目录下
libjingle_peerconnection_so.so
```
其他参阅[WebRTC官方文档](https://webrtc.org/native-code/android/)
