---
title: android-MultiDex
date: 2016-08-17 10:00:30
categories: "android"
tags: ["android", "multidex"]
---

#  MultiDex相关问题
dex file :dalvik executable file
## 为什么需要MultiDex
用Dalvik虚拟机的Android手机，在安装app的时候，会有一个优化dex的过程，使用dexopt将dex优化的更加高效于运行存储为odex，但是dexopt把每个类的方法id检索的链表长度使用了short，4字节，所以无论如何，就导致了如果**一个dex**中的方法数（包含Android framework methods, library methods, and methods in your own code）不能超过65535。因此当我们项目中的方法书超过65535时就会报错：
```
早期版本报错如下：
Conversion to Dalvik format failed:
Unable to execute dex: method ID not in [0, 0xffff]: 65536

最近的版本报错如下：
trouble writing output:
Too many field references: 131000; max is 65536.
You may try using --multi-dex option.
```
>这个限制通常被叫做：64k preference limit。在dexopt对dex优化时，会将dex中的方法存储在一个叫做LinearAlloc的缓冲区中，LinearAlloc缓冲区在不同版本的android手机上大小是不一样的，2.2、2.3仅有5M，android4.x提升到了8M或16M，所以当方法数过多时，dexopt就会奔溃导致程序异常终止。

在android5.0之前，dalvik规定每一个apk只能包含一个dex文件，所以为了摆脱这个限制，你必须使用Google官方提供的MutiDex support library。当project编译完毕之后，MultiDex lib就会加载到primary dex中的文件，这样就可以管理其他dex的加载。
>注意：如果你的工程要支持api20及以下，那么必须禁用android studio的instant run。

在android50及以后，开始使用了art环境取代Dalvik，而art架构本身支持多dex文件的加载。在安装apk时，art会在扫描class(1...n).dex时进行预编译，将其打包成一个单独的.oat文件，在apk运行的时候就从这个单独的oat文件读取资源。
>注意：如果你的minSdkVersion大于等于21，且你开启了instant run，那么android studio则会自动配置multidex。因为instant run仅在debug时起作用，那么在发布debug版时，你必须自己去配置multidex来突破64k限制。

## 避免64k限制
尽管已经有一些办法去解决64k限制了，但我们应当尽量去避免方法数超过64k，因为multidex的支持有很多限制：
>* 如果第二个dex文件过大可能会导致应用ANR或crash。
>* 在android4.0及以下，因为linearAlloc的bug，可能导致multidex不启用。
>* 因为Dalvik LinearAlloc的限制，multidex可能会导致很大的内存分配，从而crash。在android4.0及以前，有很大的几率会发生crash，在android5.0之前的版本也有一定的几率会发生crash
>* 

谷歌提供了两种方法来解决这个问题：

1. Review your app's direct and transitive dependencies（然并卵）
2. Remove unused code with ProGuard
```
...
android {

    ...
    defaultConfig {
        ...
        // Enabling multidex support.
        multiDexEnabled true
        //移除无用的resource文件
        shrinkResources true
    }
    ...
}
```

## 配置Multidex
有时我们无论怎么努力，因为某些第三方库的原因无可避免的达到了64k，那么我们就要学会怎么去配置multidex。在Android SDK Build Tools 21.1 及以上，开始支持在build.gradle中配置multidex，所以首先我们必须将sdk更新到21.1以上,然后按照以下步骤来操作。

1 修改主module的build.gradle文件
```
android {
    compileSdkVersion 21
    buildToolsVersion "21.1.0"

    defaultConfig {
        ...
        minSdkVersion 14
        targetSdkVersion 21
        ...

        // Enabling multidex support.
        multiDexEnabled true
    }
    ...
}

dependencies {
  //在sdk大于等于23时，则不需要了
  compile 'com.android.support:multidex:1.0.0'
}
```
2 继承或绑定MultiDexApplicaiton
```
    //方法1
   public class  MyApplication extends MultiDexApplication
   
```
如果MyApplication已经集成了其他Application，那么则只需要重写Application的attachBaseContext() 方法即可。
```
    public class  MyApplication extends OtherApplication{
    
        ...
        protected void attachBaseContext(Context base) {
            super.attachBaseContext(base);
            MultiDex.install(this);
        }
        ...
    
    }
```

## 优化Multidex
multidex会带来很大的性能问题，因为它需要在运行时抉择把哪些文件加载到primary dex中,哪些文件加载到second dex中，这当然要比直接执行更加耗时。为了缓解该性能问题，谷歌提供了一种方案：

```
android {
    productFlavors {
        // Define separate dev and prod product flavors.
        dev {
            // dev utilizes minSDKVersion = 21 to allow the Android gradle plugin
            // to pre-dex each module and produce an APK that can be tested on
            // Android Lollipop without time consuming dex merging processes.
            minSdkVersion 21
        }
        prod {
            // The actual minSdkVersion for the application.
            minSdkVersion 14
        }
    }
          ...
    buildTypes {
        release {
            runProguard true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                                                 'proguard-rules.pro'
        }
    }
}
dependencies {
  compile 'com.android.support:multidex:1.0.0'
}
```
After you have completed this configuration change, you can use the devDebug variant of your app, which combines the attributes of the dev productFlavor and the debug buildType. Using this target creates a debug app with proguard disabled, multidex enabled, and minSdkVersion set to Android API level 21. These settings cause the Android gradle plugin to do the following:

Build each module of the application (including dependencies) as separate dex files. This is commonly referred to as pre-dexing.
Include each dex file in the APK without modification.
Most importantly, the module dex files will not be combined, and so the long-running calculation to determine the contents of the primary dex file is avoided.