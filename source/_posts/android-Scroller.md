---
title: android-Scroller
date: 2016-08-17 11:02:50
categories:
tags:
---
1. View中的mScrollX, mScrollY这两个变量描述的是***现在View的内容的位置（x2， y2）相对于View所在位置(x1, y1)的值***, 即 mScrollX = x1 - x2, mScrollY = y1 - y2。 
2. 通过Scroller, 滑动的**是View的内容**，而不是view本身
3. scrollTo(x, y),x 是相对于View左边而言的，y是相对于view上边而言的。
4. scrollTo没有累加效果，scrollBy有累加效果
5. scroller只是用来做平滑的滑动，也就是说如果你要让某个View随着手机的滑动而滑动则必须使用scrollTo或scrllBy。只要使用了startScroll,就开始自动滑动了。
#####原理解析
> 


----------