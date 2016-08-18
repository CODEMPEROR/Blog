---
title: androdid-view-自定义view
date: 2016-08-17 09:53:16
categories: "android"
tags: ["android", "View绘制原理分析", "自定义View"]
---

# 一、 View绘制原理
任何一个view从开始到绘制在屏幕上都要经历三个过程：测量、布局、绘制，分别对应为方法为：onMeasure(),onLayout, onDraw。measure用来测量view的宽和高，layout用来确定view在父容器中的**放置位置和最终大小**，而draw则负责将view绘制在屏幕上。其中onMeasure和onLayout可能不仅调用一次。和触摸事件的分发一样，view的绘制过程也是从DecorView开始的，接下来我们详细分析一下具体的过程。

> view的测绘都是从ViewRoot开始的，viewRoot并不是一个view，而是一个handler。

## 1. onMeasure(MeasureSpec, MeasureSpec)过程分析
MeasureSpec是4字节整数，用来计量view的大小，高2位表示该view的绘制模式specMode，低30位来表示view的大小specSize。之所以要把SpecMode和SpecSize打包成一个int值，是为了避免过多的对象内存分配，方便操作。view的绘制模式一共有3种：
>* Exactly : 父容器明确知道view的大小，即specSize记录了view精确的大小。对应LayoutParams中的match_parent 和 100dp。
>* at_most : 父容器制定了一个最大值，view的大小不能大于它。对应LP中的wrap_content。
>* unspecified : 父容器对view没有任何限制，要过大给多大，这种情况一般用于系统内部，表示一种测量状态。

所谓绘制模式，其实是ViewGroup对view（或者子viewGroup）的绘制限制，请记住**不管是哪种绘制模式，parent对child的限制都是基于parent剩余的空间大小的**

>MeasureSpec的形成。我们知道DecorView是viewTree最外层的view，那么它的MeasureSpec的生成必定是根据屏幕的宽高来生成的。而其他View的MeasureSpec都是在父容器的Spec及自己的LayoutParams的共同作用下生成的。

> **ChildMeasureSpec = ChildLayoutParams + ParentMeasureSpec;**
**DecorMeasureSpec = WindowSize + DecorLayoutParams**
具体生成规则如表3.3.1-1所示：

 表3.3.1-1

![View#MeasureSpec生成规则](http://img.blog.csdn.net/20160724194418541)

从表3.3.1-1中的结果可知：在自定义View时，如果要使用wrap_content属性，则必须重写onMeasure()方法，并为该View设置一个默认大小，否则默认为parent的大小。具体实现如下：

```
    protected void onMeasure(int widthMeasureSpec, int heightMeaureSpec){
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        
        if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(DEFAULT_WIDTH, DEFAULT_HEIGHT);
        } else if (widthMode == MeasureSpec.AT_MOST) {
            int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
            setMeasuredDimension(DEFAULT_WIDTH, heightSize);
        } else if (heightMode == MeasureSpec.AT_MOST) {
             int widhtSpecSize = MeasureSpec.getSize(widthMeasureSpec);
             setMeasuredDimension(widthSpecSize, DEFAULT_HEIGHT);
        }
        
    }

```

## 2. layout过程
layout过程的作用是由ViewGroup用来确定child位置。当ViewGroup的位置确定之后，他会在onLayout方法中调用所有child的layout方法来确定child的位置。因为每一种不同的ViewGroup对child都有不同的规则限制，所以在ViewGroup#onLayout是abstract的，在每一具体的ViewGroup中（比如LinearLayout）中实现该方法来确定child的位置，然而view#onLayout是一个空方法。

## 3. draw过程
在确定了view的位置之后，就会开始绘制。当然draw也是从ViewGroup开始执行的。对于每一个view，ViewGroup的绘制过程都是：

1. draw the backround
（**drawBackground(canvas);**）
2. draw the content
（**onDraw(canvas);**）
3. draw the children
（**dispatchDraw(canvas);**）
4. draw the decorations：*eg:foregroung, scrollbars*
(**onDrawForeground(canvas);**)

## 正确获取View高度的几种方案：
1. Activity/View#onWindowFocusChanged
2. view.post(new Runnable(){...});
3. view.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener(){...});

----------


## 二、 自定义View
>**自定义view须知**

1. 支持padding属性
2. 如何自定义属性
3. 让view支持wrap_content
4. View#post vs Handler
5. view中如有动画或线程，请及时停止。View#onDeatchedFromWindow
6. 滑动冲突的处理

接下来我们会逐个解释上面6个**须知**，不过在这之前请先了解一下下面四个问题：坐标与坐标系，构造函数，手势处理、自定义view与自定义ViewGroup的区别。

## 1、 坐标与坐标系

在android系统中，坐标原点是屏幕左上角，x向右为正，y向下为证。

view在绘制的时候，都是将其当做一个矩形来处理的，所以只需要左上、右下两个顶点即可确定view的位置，对应view的四个属性：mLeft, mTop, mRight, mBottom。除此之外，view还有其他几个参数：x, y, translationX, translationY。***请记住：view的这8个参数都是相对于父容器而言的。***他们之间的关系可以用下面的图和公式来说明：


$$mWidth = mRight - mLeft$$
$$mHeight = mBottom - mRight$$
$$x = mLeft + translationX$$
$$y = mTop + translationY$$

>需要注意的是:view在平移的过程中，top和left的值并不会发生变化，从上面的公式中也可以看出，变化的仅是translationX, translationY, x, y。

## 2、 构造函数
view有三种构造函数，每一种均有不同的用途，接下来我们以自定义CustomView为例逐个说明解析。

1. public CustomView(Context context){}
一般用于直接new一个CustomView：CustomView pie = new CustomView(context);
2. public CustomView(Context context, AttributeSet attrs){}
在xml中使用时，会调用该构造函数，attrs中存储了在xml中定义的各种属性，包含系统属性及自定义属性。
3. public CustomView(Context context, AttributeSet attrs, int defStyleAttr)
该构造函数系统不会主动调用，一般都是在第二个构造函数中调用来为CustomView来指定style。系统默认实现的button中，其第二个构造函数如下：
```
public Button(Context context, AttributeSet attrs) {
    this(context, attrs, com.android.internal.R.attr.buttonStyle);
}
```

所以如果我们要给CustomView指定一个默认的style，只需要像系统Button的实现一样。如果不调用第三个构造函数，即不指定defStyleAttr属性，则默认使用当前Activity或Application的style。

## 3、 手势处理
参见3.7节及第4章

## 4、 自定义view和自定义ViewGroup的区别

####问题汇总及解析：

> 

----------


# 三、 OverDraw
## 常用的消除OverDraw的方法
### 1. 设置window的背景为null;


```
getWindow().setBackgroundDrawable(null);
```

### 2. 尽量少的设置布局的背景
