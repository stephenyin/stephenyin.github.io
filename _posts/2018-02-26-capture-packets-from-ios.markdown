---
layout: post
title:  "不越狱 iphone 抓包"
date:   2018-02-26 17:34:33 +0800
categories: mobile tcpdump
tags: mobile
---
## 1.获取 iOS 设备 UDID
## 2.安装 RVI，需要使用 rvictl 工具
```bash
rvictl -s 0ea4f781e259bea5737e0aabe956846bbbcbdc5e
```
## 3.用 tcpdump 对虚拟 interface rvi0 抓包即可
## 4.抓包结束后 remove rvi0
```bash
rvictl -x 0ea4f781e259bea5737e0aabe956846bbbcbdc5e
```
## 脚本
```bash
#set -x
if [ $# -ne 1 ]
then
    echo "cmd format : rvi.sh <start|stop>"
    exit
fi

if [ "$1"x = "start"x ]
then
#    ifconfig -l
    rvictl -s 0ea4f781e259bea5737e0aabe956846bbbcbdc5e
#    ifconfig -l
elif [ "$1"x = "stop"x ]
then
#    ifconfig -l
    rvictl -x 0ea4f781e259bea5737e0aabe956846bbbcbdc5e
#    ifconfig -l
else
    echo "wrong parameter"
fi
```