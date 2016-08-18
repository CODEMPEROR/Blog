---
title: androdid-适配
date: 2016-08-17 09:53:44
categories: "android"
tags: ["android", "屏幕适配", "限定符"]
---
# androdid-适配
由于android屏幕的多样性，关于尺寸、屏幕大小等就有一系列的属性和单位来保证应用程序在各个机型上的适配:
## 1. 单位:
1. px
2. dp：设备无关的尺寸单位
3. dip：同dp
4. sp:文字的大小，sp = px * scaledDensity, scaledDensity是用户在系统设置里设置的文字缩放比例。
5. dpi:dot per inch, 称为屏幕像素密度，指每英寸上的像素点数。
6. density:px/dp
7. ppi:同dpi, pixel per inch.

## 2. 属性
1. 屏幕宽度：屏幕横向的像素点数，单位px
2. 屏幕高度：屏幕纵向的像素点数，单位px
3. 屏幕尺寸：屏幕对角线的长度，单位为英寸，1 inch = 2.54cm
4. 屏幕分辨率：屏幕横纵方向的像素点数，单位为px， 1px = 1像素点。例如1920 * 1080
5. 屏幕像素密度：指每英寸上的像素点数，即dot per inch $$屏幕像素密度 = 屏幕分辨率 / 屏幕尺寸$$屏幕像素密度一般分为ldpi, mdpi, hdpi, xhdpi, xxhdpi, xxxhdpi这6种类型，他们与屏幕像素密度大小的对象关系如下表：
6. 缩放比例：scaledDensity,用户设置的文字缩放比例

|名称|像素密度大小(px/inch)|
| -------- | --------- |
|ldpi (DENSITY_LOW)|<120|
|mdpi (DENSITY_MEDIUM)|120-160|
|hdpi (DENSITY_HIGH)|160-240|
|xhdpi|240-320|
|xxhdpi|320-480|
|xxxhdpi|480-640|

从上表我们可以看出： mdpi:hdpi:xhdpi:xxhdpi:xxxhdpi = 3:4:6:8:12

以上所有的信息均可在DisplayMetrics类中查询到，具体用法如下:
```
    /**
     * dip/dp 转 px
     *
     * @param context  Context
     * @param dipValue dip/dp value
     * @return the px value
     */
    public static int dip2px(Context context, float dipValue) {
        final float density = context.getResources().getDisplayMetrics().density;
        Log.i(TAG, "dip2px_density = " + density);
        return (int) (dipValue * density + 0.5f);
    }

    /**
     * px转dip/dp
     *
     * @param context context
     * @param pxValue px
     * @return
     */
    public static int px2dip(Context context, float pxValue) {
        final float scale = context.getResources().getDisplayMetrics().density;
        return (int) (pxValue / scale + 0.5f);
    }
    
    /**
     * 将px值转换为sp值
     *
     * @param pxValue px
     * @return sp
     */
    public static int px2sp(Context context, float pxValue) {
        final float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (pxValue / fontScale + 0.5f);
    }

    /**
     * 将sp值转换为px值
     *
     * @param context context
     * @param spValue （DisplayMetrics类中属性scaledDensity）
     * @return
     */
    public static int sp2px(Context context, float spValue) {
        final float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (spValue * fontScale + 0.5f);
    }
    
    /**
     * 获取屏幕宽度
     *
     * @param context context
     * @return 屏幕宽度，单位px
     */
    public static int getDisplayWidthPixels(Context context) {
        if (context == null) {
            return -1;
        }
        DisplayMetrics dm = context.getResources().getDisplayMetrics();
        return dm.widthPixels;
    }
    
    /**
     * 获取屏幕高度
     *
     * @param context context
     * @return 屏幕高度，单位px
     */
    public static int getDisplayWidthPixels(Context context) {
        if (context == null) {
            return -1;
        }
        DisplayMetrics dm = context.getResources().getDisplayMetrics();
        return dm.heightPixels;
    }
    
    /**
     *  获取手机状态栏高度
     * @param context context
     * @return  手机状态栏高度
     */
    public static int getStatusBarHeight(Context context) {
        Class<?> c = null;
        Object obj = null;
        Field field = null;
        int x = 0, statusBarHeight = 0;
        try {
            c = Class.forName("com.android.internal.R$dimen");
            obj = c.newInstance();
            field = c.getField("status_bar_height");
            x = Integer.parseInt(field.get(obj).toString());
            statusBarHeight = context.getResources().getDimensionPixelSize(x);
        } catch (Exception e1) {
            e1.printStackTrace();
        }
        return statusBarHeight;
    }

    /**
     * 获取ActionBar的高度
     * @param context context
     * @return ActionBar的高度
     */
    public static int getActionBarHeight(Context context) {
        TypedValue tv = new TypedValue();
        int actionBarHeight = 0;
        if (context.getTheme().resolveAttribute(android.R.attr.actionBarSize, tv, true))// 如果资源是存在的、有效的
        {
            actionBarHeight = TypedValue.complexToDimensionPixelSize(tv.data, context.getResources().getDisplayMetrics());
        }
        return actionBarHeight;
    }

    /**
     * 获取NavigationBar的高度
     * @param context  context
     * @return NavigationBar的高度
     */
    public static int getNavigationBarHeight(Context context) {
        Resources resources = context.getResources();
        int resourceId = resources.getIdentifier("navigation_bar_height", "dimen", "android");
        //获取NavigationBar的高度
        return resources.getDimensionPixelSize(resourceId);
    }
```

现在我们回头想想，一张640 * 640的图片在不同dpi的屏幕上显示的大小可能是不一样的，例如在mdpi显示为 640/120 = 5.3inch,在hdpi的屏幕上显示为640/24=2.67inch。所以为了保证在各种不同dpi的屏幕上，同一张图片显示的大小尽可能相同，我们需要针对不同dpi的屏幕提供不同大小的图片。例如ic_launcher这张图片：在mdpi~xxxhdpi中大小依次为 36、48、72、96、144px。

## 限定符及其使用
### 限定符的类型
#### 1. large限定符
现在很多应用在平板上都支持双面板模式，左边为选项列表，右侧为选项内容，但在手机上就无法容纳这种双面板模式，需要分别显示。在这种情况下我们就可以使用large限定符来动态选择布局。系统默认情况下，将7英寸及以上的手机及平板当做large来处理。

>res/**layout**/main.xml，单面板（默认）布局：

```

	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    	android:orientation="vertical"
    	android:layout_width="match_parent"
    	android:layout_height="match_parent">

    	<fragment android:id="@+id/headlines"
              android:layout_height="fill_parent"
              android:name="com.example.MenuFragment"
              android:layout_width="match_parent" />
	</LinearLayout>

```

>res/**layout-large**/main.xml,双面板模式布局：

```

	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    	android:layout_width="fill_parent"
    	android:layout_height="fill_parent"
    	android:orientation="horizontal">
    	
    	<fragment android:id="@+id/headlines"
              android:layout_height="fill_parent"
              android:name="com.example.MenuFragment"
              android:layout_width="400dp"
              android:layout_marginRight="10dp"/>
    	<fragment android:id="@+id/article"
              android:layout_height="fill_parent"
              android:name="com.example.MenuDetailFragment"
              android:layout_width="fill_parent" />
	</LinearLayout>

```
但是仅仅使用large限定符，是没有办法来做到精准的适配。例如因为某些特殊的需求需要在5寸，7寸，12寸等大小的手机上以不同的方式显示某些内容，此时large就无法满足需求了。最小宽度限定符就是用来解决解决上述问题的。
#### 2. 最先宽度限定符:swXXdp
最小宽度限定度通过指定某个最小宽度来限定屏幕（以dp为单位）。比如标准7英寸的平板为600dp。

**注意：swXXdp仅适用的android3.2及以上机型。**
>res/layout/main.xml
>
>res/layout-sw600dp/main.xml
#### 3. 布局别名
最小宽度限定符仅适用于 Android 3.2 及更高版本。因此，如果我们仍需使用与较低版本兼容的概括尺寸范围（小、正常、大和特大）。例如，如果要将用户界面设计成在手机上显示单面板，但在 6 英寸平板电脑、电视和其他较大的设备上显示多面板，那么我们就需要提供以下文件：

>* res/**layout**/main.xml: 单面板布局 //Normal
>* res/**layout-large**/main.xml: 多面板布局   //适用于large，低于android3.2的情况
>* res/**layout-sw600dp**/mail.xml: 多面板布局 //适用于android3.2且sw600dp情况

后两个文件是相同的，因为其中sw600dp用于和 Android 3.2 设备匹配，而large则是为使用较低版本 Android 的平板电脑和电视准备的。

要避免平板电脑和电视的文件出现重复，可以使用别名文件。例如，可以定义以下布局：
>* res/layout/main.xml，单面板布局
>* res/layout/main_twopanes.xml，双面板布局

然后添加这两个文件：

>res/**values-large**/layout.xml:


```

	<resources>
    	<item name="main" type="layout">@layout/main_twopanes</item>
	</resources>
```

>res/**values-sw600dp**/layout.xml:

```

	<resources>
    	<item name="main" type="layout">@layout/main_twopanes</item>
	</resources>
```

#### 4. 使用屏幕方向限定符
>* res/layout-land/mail.xml
>* res/layout-port/main.xml

#### 5. 各式各样的限定度使用举例
#####别名
>* res/values-sw600dp-land/layout.xml
>* res/values-large-land/layout.xml
>* res/values-large-port/layout.xml

## 9Patch图片
只需要记得左上限制缩放区域，右下限制显示区域