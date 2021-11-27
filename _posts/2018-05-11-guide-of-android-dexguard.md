---
title: Android DexGuard 混淆指南
layout: post
categories: [Android]
tags: [DexGuard]
seo:
  date_modified: 2020-03-02 22:39:56 +0800
---

## 简介

DexGuard 是一款付费代码混淆软件，主要功能是对 Java 代码进行混淆，使得反编译后得到的源代码可读性差，从而加大破解的难度。DexGuard 与 Android 上主流的混淆工具 ProGuard 同属一家公司开发，但相比免费的 ProGuard 功能更多，混淆力度也更大。详细异同参考：[DexGuard vs. ProGuard](https://www.guardsquare.com/en/blog/dexguard-vs-proguard)。

本篇教程将使用 DexGuard 8.1.14 版本，在 Ubuntu 14.04 server 编译环境下对 Android Gradle 工程进行混淆。

## 入门

要使用 DexGuard，首先要在原 Android 工程中集成 DexGuard 插件。

编辑工程的 `build.gradle` 文件，导入 DexGuard 依赖

```sh
vim project_root_build.gradle

buildscript {
    ...
    repositories {
        ...
        flatDir { dirs 'path_to_DexGuard-8.1.14/lib'}
    }
    dependencies {
        ...
        classpath ':dexguard:'
    }
}
```

编辑 app 模块的 `build.gradle` 使能 Dexguard 插件


```sh
vim project_root_app_module_dir_build.gradle

apply plugin: 'dexguard'
    android {
        ...
        defaultConfig {
        ...
 //DedGuard will handle multidex in stead of Gradle, so if your project needs multdex support, disable Gradle's multidex support here and add -multidex to your dexguard-projext.txt.
 //multiDexEnabled true
        ndk {
            abiFilters "armeabi-v7a"
        }
    }
    buildTypes {
        release {
 proguardFile getDefaultDexGuardFile('dexguard-release.pro')
 proguardFile 'dexguard-project.txt'
            proguardFile 'proguard-project.txt'
        }
    debug {
 proguardFile getDefaultDexGuardFile('dexguard-debug.pro')
 proguardFile 'dexguard-project.txt'
            proguardFile 'proguard-project.txt'
        }
    }
 }
```

其中，`dexguard-project.txt` 为自定义规则文件，如果不存在，创建并将其放置在 app 模块目录。后续章节介绍如何制定规则。

DexGuard 是付费软件，因此还需要指定 license。将你的 license 文件放置在用户 home 根目录下即可。你也可以参考 `docs/` 下的指导文档，配置环境变量来指定 license 文件。

配置好以后，就可以使用 Gradle 命令启动混淆版本的编译了


```sh
./gradlew assembRelease
```

DexGuard 会参与 Gradle 的 release 编译和 debug 编译，但为了方便开发调试，默认只有 release 版本才会应用 DexGuard 的各种功能，包括优化、混淆、压缩等。要注意尽管 debug 版本不会应用 DexGuard 功能，其打包过程还是由 DexGuard 接管（ 代替 aapt） 。最后，你可通过反编译 apk 观察最终的混淆效果。

另外，随编译输出的还有 `mapping.txt` 文件，该文件记录了本次混淆的映射表，后续可以用来恢复 stack trace。

## 混淆基本原理与语法

DexGuard 对 Java 的混淆原理是将类名或方法、成员名自动替换为无意义且难以阅读的字符，这存在一个问题：DexGuard 并非完全精确，一些依赖于类名、方法名的调用，类名、方法名可能被错误地混淆，从而导致无法链接而调用失败。举例来说：如果 JNI 中的 Java Native 方法名被混淆，将导致 C++ 库无法根据方法名链接 Java 代码。对于这种情况，我们就需要额外制定混淆规则保留 native 方法名及其所在的类名。

```
-keepclasseswithmembernames,includedescriptorclasses class * {
    native <method>;
}
```

实际上，混淆规则的本质是帮助 DexGuard 识别程序的入口（Entry Point）。所谓入口，最容易理解的是程序的 main 方法，如果 main 方法都被混淆了，那么 Java 程序肯定就启动不了。此外， Entry Point 通常还包括 applets，midlets，activities。与此相反，类内部的 private 方法则一般不是 Entry Point，因为只在类内部调用，并不会暴露给外部。

DexGuard 的混淆规则语法与 Proguard 兼容。下面列举比较常用的语法：

`keep` - 将指定的类及成员保留 `*` - 匹配任意类名，但只匹配当前包目录，不跨界 `**` - 匹配任意类名，且可跨目录匹配

例如：

```
-keep class android.support.v7.widget.SearchView {
    public <method>;
}
```
上面的规则表示将 `class android.support.v7.widget.SearchView` 类的类名名和该类的 public 方法保留。

更多语法内容请参考[官网](https://www.guardsquare.com/en/proguard/manual/usage)。

## 规则制定过程总结

DexGuard 实际上已经能够自动识别一些 Entry Point，例如部分反射调用，此外还会根据你的工程所属平台启用子带的通用型规则。但是特定工程还是需要人工适配混淆规则。这里博主结合此前使用 ProGuard 的经验，提供 DexGuard 制定规则的一般步骤与思路。

**Step 1** - 对于使用第三方库，或基于第三方开源软件开发的工程，首先去对应的第三方项目中寻找混淆规则，第三方库通常已经为我们做了这项工作

例如，代码量极其庞大的 Chromium 工程，比较明智的做法是站在巨人肩膀上。

**Step 2** - 编译输出中包含 `Warning` 和 `Note` 告警信息，导致编译不通过，根据告警提供的信息使用相应的混淆规则或确认误报，然后使用如下规则清除告警：


```
-dontwarn class_filter
-dontnote class_filter
```

**Step 3** - 识别项目特定规则

跨 runtime 的方法调用，例如 V8 JavaScript 使用方法名调用 Java native 方法（JS/Native Bridge），显然不能让 Java 方法被混淆。

JS/Native Java 代码

```java
@JSMethod
public void pay(String payReq, JSCallback success, JSCallback fail, JSCallback complete) {...}
```

混淆规则

```
-keepclassmembers class * {
    @com.taobao.weex.annotation.JSMethod <method>;
}
```

这里我们利用  `@JSMethod` annotation 来匹配工程中所有的 JS/Native 方法。

**Step 4** - 观察日志，根据关键崩溃信息制定规则。典型场景如类名误混淆导致的崩溃 stack trace 如下

```
W System.err: Caused by: java.lang.ClassNotFoundException: Didn't find class "org.chromium.chrome.browser.preferences.ExpandablePreferenceGroup"
```

解决方法也很简单，在混淆规则中使用 `-keep` 语句保留住该类即可

```
-keep class org.chromium.chrome.browser.preferences.ExpandablePreferenceGroup
```

混淆规则制定过程中 crash 在所难免，有一个小技巧是通过大范围的 `-keep` 先让工程跑起来，然后逐步缩小 `keep` 的范围，从而定位具体误混淆的类。

## ReTrace 恢复日志

混淆加大了黑客反编译的难度，但也给开发者造成了障碍：经过混淆的 apk 异常抛出错误时，trace 信息也是混淆的，不利于开发调试。

DexGuard 提供的 ReTrace 工具能够读取混淆的 stack trace 然后恢复成未混淆时的模样。这个过程依赖上文提到的 mapping 文件，该文件记录了混淆前和混淆后的类名、成员名的映射关系

```
<project_root>/<app_module>/build/outputs/mapping/release/mapping.txt
```

mapping 文件每个编译版本不同（除非特别指定），开发者需要记录发布版本和 mapping 文件的对应关系。

要恢复 stack trace， 准备好你的 `mapping.txt` 和 `stacktrace_obfuscated.log`，使用下面的命令完成恢复

```sh
cd DexGuard-8.1.14/bin
./retrace.sh mapping.txt stacktrace_obfuscated.log > stacktrace_restored.log
```

## 常见问题

参见此前的文章 - [DexGuard Android Project Troubleshooting](https://linhongbo.com/2018/05/09/dexguard-troubleshooting.html)
