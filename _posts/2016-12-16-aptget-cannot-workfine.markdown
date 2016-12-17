---
layout: post
title:  "VirtualBox 崩溃后 apt-get 不能正常工作的问题"
date:   2016-12-16 10:40:33 +0800
categories: Linux-system
tags: Linux-system
---

### 毒药

在给运行在 VirtualBox 中的 Linux mint 上安装 google-chrome-stable_current_amd64.deb 时 VirtualBox 崩溃了，重启后 apt-get 无法正常工作，得到如下错误：

```
$sudo apt-get install -f
Reading package lists... Done
Building dependency tree       
Reading state information... Done
E: The package google-chrome-stable needs to be reinstalled, but I can't find an archive for it.
```


### 解药

终端运行 sudo dpkg -i 

```
$sudo dpkg -i google-chrome-stable_current_amd64.deb
```