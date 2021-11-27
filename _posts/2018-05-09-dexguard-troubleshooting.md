---
layout: post
title: DexGuard 常见问题解决
categories: [Android]
tags: [DexGuard]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

## multidex 错误

Android 对单个工程包含的总方法数有限制，最多是 65535 个。如果工程较大，方法数超过这个数目，就要使用 multidex 技术，否则将会运行错误。multidex 将原来编译出的 dex 文件分割为多个子 dex 文件，APP 运行时再拼接起来。DexGuard 使用自己实现的 multidex 支持，因而使用上需要一些配置。

### **错误日志**

```
Overflow detected while writing dex file 'classes.dex': Too many method references : 92180; max is 65535
You may try using configuration 'dexguard-debug-chrink.pro' for debug builds or enable multilex support.
Please refer to the documentation for more information.`

* What went wrong
Executing failed for task ':chrome:dexguardDebug'.
> Writing of dex file 'classes.dex' failed due to overflow
```

### 解决办法

在 `dexguard-project.txt` 中配置如下字段，打开 DexGuard 的 multidex 支持开关

```
...
-multidex
```

## 压缩了不该压缩的 assets

Android 对于 assets 资源的压缩有限制。因为对于 assets 压缩文件，apk 在运行时需要映射至内存然后解压为直接可用的格式。这就要求被压缩文件不能过大，否则近乎双倍于原文件的内存占用，会造成大量内存消耗。对 mp3、ogg 等天然压缩文件类型，aapt 在打包时会自动忽略压缩，运行时直接映射至内存。而对于非压缩文件类型，如果该文件过大（> 1M），我们就必须手工指定忽略压缩，否则会出现运行错误。

### 错误日志

```
E ApkAssets: Error while loading assets/icudtl.dat:
java.io.FileNotFoundException: This file can not be opened as a file descriptior; it is probably compressed
```

分析：查看 gradle 工程 app 模块的 `build.gradle`，可以看见原 gradle 工程对 dat 后缀文件是指定忽略压缩的，因为 `icudtl.dat` 远大于 1M

```
./build.gradle

...
aaptOptions {
    noCompress "dat", "bin", "pak"
}
```

但是，DexGuard 的打包系统与 aapt 独立，aapt 的配置并不生效，`assets/icudtl.dat` 被误压缩。此外，我们知道 apk 包实际上就是 zip 压缩格式，因此还可以用 unzip -lv 命令看看混淆编译出的 apk 中每个文件的压缩处理情况。其中「method」一栏用 「Stored」 表示该文件为存储型（不压缩）;「Defl」表示该文件是被压缩。博主使用此命令查看存在问题的混淆 apk 版本，发现 `assets/icudtl.dat` 文件标识为「Defl」，可见确实被 DexGuard 错误地压缩了。

### 解决办法

> Dexguard incidentally does perform the final packaging . It has an option -dontcompress to control which files or filetype are left uncompressed . The default configuration leaves . ogg and .Me files uncompressed.

## assets 压缩错误

有些 assets 文件，原 gradle 工程并未指定忽略压缩，但是在 DexGuard build 下却存在压缩异常，混淆版本崩溃时机通常发生在读取 assets 文件时

### 错误日志

```
D ApkAssets: copy from assets/as block to /data/user/0/com.foo.bar/app_hbs/adblock
W System.err: Java.lang.NoClassDefFoundError: Failed resolution of : Lcom/google/devtools/build/android/desugar/runtime/ThrowableExtention;
System.err: at org.chromium.base.ApkAssets.copy
System.err: Caused by: java.lang.ClassNotFoundException: Didn't find class com.google.devtools.build.android.desugar.runtime.ThrowableExtention" on path: DexPathList [[zip file "/data/app/..."], nativeLibraryDirectories=[/data/app/...]]
```

可看到 assets manager 在 copy `adblock` 文件后立即发生崩溃错误，博主猜测是 DexGuard 未正常处理该文件的压缩事宜。

### 解决办法

尝试在 DexGuard 规则文件中使用以下规则排除该文件的压缩

```
-dontcompress **/adblock
```

问题解决
