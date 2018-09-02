# Android App Bundle初体验

Android App Bundle一般简称AAB，是Google最新推出来的动态化交付框架，依托Google Play的支持，基于ProtoBuffer的数据压缩与解析，根据用户的设备信息交付重新组装过的apk，并且可以根据功能需要动态的下发。

![](./imgs/summary)

## 优缺点分析

> 优点

* 安装包更小
* 安装速度更快
* 按需下发非核心功能

> 缺点

* 目前是必须通过Google Play发布的应用
* 必须使用Google Play注册的keystore
* 最低支持android 5.0（5.0以下不支持split和动态下发，但是Google Play也会对包做一定的优化）
* 需要升级AS到3.2（目前还在beta版本~）
* 必须集成Play Core Library
* 对老的工程项目，改动成本比较大

> 如何看待AAB？

AAB可以看作是Google进一步对app开发生态的控制（个人观点~），首先是统一化了安卓开发者的开发模式，再就是相当于对所有开发出的apk，google有了“上帝视角”，可以做任何的事情。在国内，AAB可以说就没啥用武之地了，除非其他的市场能够支持。

但是AAB给我们提供了一种解决问题的思路，我们可以学习它的运行原理，说不定在某些业务场景下可以使用得到。

## 理论基础

> 为什么最低支持Android 5.0?

AAB是基于split Apk理论之上的，而split Apk是Android 5.0开始引入的多apk构建机制。split apk机制可以将一个apk在ABI、屏幕密度和语言维度对apk进行拆分，来减少apk的体积。

我们看下Android API 21的源码：

在`ApplicationInfo.java`中，可以看到每个split apk的路径的定义：

```java
/**
 * Full path to the base APK for this application.
 */
public String sourceDir;

/**
 * Full path to the publicly available parts of {@link #sourceDir},
 * including resources and manifest. This may be different from
 * {@link #sourceDir} if an application is forward locked.
 */
public String publicSourceDir;

/**
 * Full paths to zero or more split APKs that, when combined with the base
 * APK defined in {@link #sourceDir}, form a complete application.
 */
public String[] splitSourceDirs;

/**
 * Full path to the publicly available parts of {@link #splitSourceDirs},
 * including resources and manifest. This may be different from
 * {@link #splitSourceDirs} if an application is forward locked.
 */
public String[] splitPublicSourceDirs;
```

再来看下android系统是如何处理split apk的，我们知道，apk包括2大部分，class的加载和resource的加载，而它们分别对应着`ClassLoader`和`ResourceManager`。

我们使用的`ClassLoader`是怎么来的？看下面的流程：

![](./imgs/classloader.png)

可以看出，核心是在`LoadedApk`这里，这里指定了需要装载的class资源。

```java
public ClassLoader getClassLoader() {
	... ...
   final List<String> zipPaths = new ArrayList<>();
   final List<String> apkPaths = new ArrayList<>();
   final List<String> libPaths = new ArrayList<>();

   ... ...
   
   zipPaths.add(mAppDir);
   if (mSplitAppDirs != null) {
      Collections.addAll(zipPaths, mSplitAppDirs);
   }
   
   ... ...
}
```

从上面的源码可以看到，classloader是包含了split apk的信息的。

那我们使用的`Resource`对象呢？

![](./imgs/resource.png)

这里的核心是在`ResourceManager`里：

```java
Resources getTopLevelResources(String resDir, String[] splitResDirs,
            String[] overlayDirs, String[] libDirs, int displayId,
            Configuration overrideConfiguration, CompatibilityInfo compatInfo) {
	... ...
	
	AssetManager assets = new AssetManager();
    if (resDir != null) {
        if (assets.addAssetPath(resDir) == 0) {
            return null;
        }
    }

    if (splitResDirs != null) {
        for (String splitResDir : splitResDirs) {
            if (assets.addAssetPath(splitResDir) == 0) {
                return null;
            }
        }
    }

    if (overlayDirs != null) {
        for (String idmapPath : overlayDirs) {
            assets.addOverlayPath(idmapPath);
        }
    }

    if (libDirs != null) {
        for (String libDir : libDirs) {
            if (libDir.endsWith(".apk")) {
                // Avoid opening files we know do not have resources,
                // like code-only .jar files.
                if (assets.addAssetPath(libDir) == 0) {
                    Log.w(TAG, "Asset path '" + libDir +
                            "' does not exist or contains no resources.");
                }
            }
        }
    }
        
	... ...
}
```

可以看出通过`AssetManager`的`addAssetPath`方法，将split apk的资源加入到了整个Resource里。

> AAB的改进是啥呢？

AAB在split apk的基础上增加了自己的一些特性，主要是将split的apk的概念更加清晰化了。

它有三大概念：

* **Base APK**：基础apk，包含split apk可以访问的代码和资源，并且包含一些全局控制的逻辑。
* **Configuration APKs**：就是原来基于split配置产生的apks，包括ABI、屏幕密度和语言维度。当用户下载app时，configuration apks会根据用户的手机信息合理的下载。configuration apks是不需要有单独的module的，Google Play会自动我们的配置生成对应的apk。
* **Dynamic feature APKs**：这个就是AAB的创新点了，该apk是不会马上安装到用户手机上的，在后面需要的时候才会动态安装。

<img src="./imgs/aab.png" height=350 width=450 />

## 实际操作

#### 1、下载Android Studio 3.2

[https://developer.android.com/studio/preview/](https://developer.android.com/studio/preview/)

目前3.2是处于beta版本的，同时gradle也需要在3.2及以上。

```gradle
buildscript {
    repositories {
        // Include Google’s Maven repository here
        // and in the repositories block below.
        google()
    }
    dependencies {
        // Use the latest version of the Android plugin listed in
        // Google's Maven repository index.
        classpath 'com.android.tools.build:gradle:3.2.0-beta05'
    }
}
```

#### 2、新建一个project

新建一个project，同时创建一个base的application。这个操作和正常的应用开发是一样的。

#### 3、添加Dynamic feature

* 选择File->New->New Module
* 在New Module弹出框中选择Dynamic Feature Module
* 配置Dynamic feature的基本信息
 * 选择Base application module，就是动态化apk所要依赖的base apk 
 * 确定该module的名称，唯一标识该module，这个值会写到`<manifest split>`里
 * 确定module的包名
 * module支持的最低API，这个值必须跟base apk保持一致
* 配置动态下发的信息
 * 设置Module title。这个跟上面的Module name是不一样的，Module name是为了gradle的管理，Module title则是为了feature的动态下载，这个值会被写到base apk里的（这个AS会主动帮我们弄好）。
 ![](./imgs/module_title.png)
 * 设置按需才下载安装，如果勾选，则该feature将不会跟base apk一起被下载安装，而会根据需要实时下载安装。
 * Fusing设置，设置是否在android 5.0以下是否将此feature一起打包到完整apk中

feature的gradle plugin：
 
```xml
apply plugin: 'com.android.dynamic-feature'
```

base和feature之间的相互建立关系：

base引用feature：

```xml
// In the base module’s build.gradle file.
android {
    ...
    // Specifies dynamic feature modules that have a dependency on
    // this base module.
    dynamicFeatures = [":dynamic-feature", ":dynamic-feature2"]
}
```

feature引用base：

```xml
// In the dynamic feature module’s build.gradle file:
dependencies {
    ...
    // Declares a dependency on the base module, ':app'.
    implementation project(':app')
}
```

我们可以设置feature的混淆（注意，feature里是不能做代码shrinking的）：

```xml
android.buildTypes {
     release {
         // You must use the following property to specify additional ProGuard
         // rules for dynamic feature modules.
         proguardFiles 'proguard-rules-dynamic-features.pro'
     }
}
```

**在feature module的build.gradle里不需要再配置的内容：**

* 签名信息
* `minifyEnabled`
* `versionCode`和`versionName`

## 关于AAB包

构建AAB包的方法：

![](./imgs/build.png)

这样在`project-name/module-name/build/outputs/bundle/`下会生成对应的aab文件。

aab其实是个zip包，包含看了整个项目的所有资源和代码，具体看下文件夹结构：

<img src="./imgs/stucture.png" height=400 />

* **base/、onefeature/**：这个就是每个module的信息，包含每个module的代码和资源。base里一定就是base application，每个dynamic feature module就是它的module name，这里的module name是onefeature，所以在aab文件里就是onefeature文件夹。
* **META-INF**：就是aab文件的元信息，跟正常的apk类似
* ***.pb文件**：个人觉得pb文件跟正常apk的文件没啥差别，这么做的目的纯粹就是为了压缩，因为经过pb压缩后的文件大小可以减少1/3，并且解析的速度也很快。
* **root/**：

Google Play在得到AAB文件后，会生成对应的base APK、dynamic feature APK(s)， configuration APKs，然后就能根据需要进行下载安装了。

解析后的结构如下图所示：

![](./imgs/aab_format.png)

## Play Core Library

动态化的features下载需要使用Play Core Library。

这块内容大家可以参考google sample，具体不分析。

## 参考文献

[https://developer.android.com/guide/app-bundle/](https://developer.android.com/guide/app-bundle/)

[https://developer.android.com/studio/build/configure-apk-splits?authuser=2](https://developer.android.com/studio/build/configure-apk-splits?authuser=2)

[https://github.com/jakkypan/android-dynamic-features](https://github.com/jakkypan/android-dynamic-features)
