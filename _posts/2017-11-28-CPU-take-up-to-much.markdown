---
layout:     post
title:      "CPU占用过高分析"
subtitle:   ""
date:       2017-11-28 
author:     "Qiby"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Android
---


#CPU占用过高分析

接手日历两个多月,感慨一下代码很烂.我的工作多数时间都是在接入广告,为了收入嘛,就是略感无聊.前两天测试报出一个日历静置耗电高的问题
拿到问题的时候我是蒙的
Http请求可能耗电,SD卡读写可能耗电,数据库读写,Timer,动画,后台Service 都有可能造成耗电量高.
该从什么地方分析呢? 先查看一下CPU
```
 adb shell top -d 1 | grep com.android.calendar
```
日历在前台时 CPU占用竟然高达15%左右.当APP进程的CPU使用率超过1%的时候，都是耗电比较大的了，正常情况下应该处于0%的值，实际上应该是0点几.
![Alt text](/qibenyu/img/in-post/2017-11-28-3-ScreenShot.png)

好了就是这里.

在这之后当我把日历切换到后台,CPU使用率掉到了0%~1%的正常值,由此猜想此问题可能页面在绘制的时候造成的问题.
进一步验证我的猜想,查看线程占用情况
```
adb shell top -d 1 -t | grep u0_a73
```
![Alt text](/qibenyu/img/in-post/2017-11-28-4-ScreenShot.png)
果然验证了我的猜想,RenderThread占用很高

接下来使用ddms查看了trace log

![Alt text](/qibenyu/img/in-post/2017-11-28-1-ScreenShot.png)

哦哦 应该就是这里 被标记红了
在绘制过程中执行事件过长
![Alt text](/qibenyu/img/in-post/2017-11-28-2-ScreenShot.png)

追到了具体代码发现是推啊sdk广告的问题...那就没办法再追下去了,是时候展现丢锅的技术了.

