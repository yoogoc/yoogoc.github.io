---
title: Ubuntu绿屏怎么办
tags:
- linux
date: 2018-11-23 14:10:01
---
暴力关机损坏了 Ubuntu的图形系统配置，导致图形界面无法正常起来。所以就看到能够登录，却只有一片蓝色。

先进入字符界面：`Ctrl + Alt + F4`

然后安装相应服务，然后重置它！


    sudo apt-get install xserver-xorg-lts-utopic
    sudo dpkg-reconfigure xserver-xorg-lts-utopic
    reboot
