---
title: android-DecorView
date: 2016-08-17 11:03:26
categories:
tags:
---
![](http://i.imgur.com/0Snb7zd.png)

>可见，DecorView由一个LinearLayout构成，这个LinearLayout又包括两个FramLayout，一个是ActionBar，一个是content。我们经常使用的setContentView(R.layout.activity_main)设置的就是content，它的id是android.R.id.content。

：DecorView与Window是一一对应的。常见的Window有：Activity，Dialog，Toast。
