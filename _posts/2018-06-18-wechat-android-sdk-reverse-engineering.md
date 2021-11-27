---
layout: post
title: 微信支付 Android SDK 逆向
categories: [Android]
tags: [微信支付, Reverse Engineering]
seo:
  date_modified: 2020-03-02 22:39:56 +0800
---

## 简介

按照微信官方开发者网站的描述，微信支付按模式不同有刷卡支付、公众号支付、扫码支付、APP 支付、H5 支付、小程序支付六种。其中，「APP 支付」即商户 APP 调用微信提供的 SDK 调用微信支付模块，商户 APP 会跳转到微信中完成支付，支付完后跳回到商户 APP 内，最后展示支付结果。

本文将介绍 Android 平台上的 APP 支付流程，然后通过逆向工程探究微信在手机端是如何对接口进行认证的。

## 支付流程

如图所示是微信「APP 支付」时序图

![微信 APP 支付时序图](/assets/img/post/2018/wechat-pay-diagram.png)

整个过程可归纳为以下几部分：

1. 商户 APP 与后台（商户服务器、微信服务器）通信，生成订单信息。这里关键接口是微信服务器提供的「统一下单 API」，请求参数包含订单请求参数的签名（sign），用于完整性校验
2. 商户 APP 与微信通信，使用前述订单发起支付请求。此时，出现微信的支付 activity，等待用户授权（指纹、密码）
3. 用户确认支付，微信与微信服务器通信，完成最终支付
4. 微信调用回调通知商户 APP 支付结果

## 接口认证过程

### 初探支付 SDK

出于安全考量，微信在 Android 侧对「APP支付」做了接口认证，只有符合要求的 APP 才能使用微信的支付接口，在实际项目开发中就发现如果使用非微信平台注册的 APK 调用微信 APP 支付接口会导致失败。下面通过查看微信官方文档、反编译等手段，还原一下整个过程。

首先，根据[微信开发者文档](https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=8_5)描述，商户 APP 要调用微信「APP支付」接口需集成微信官方提供的 SDK。发起一次支付请求的代码如下

```java
IWXAPI api;
PayReq request = new PayReq();
request.appId = "wxd930ea5d5a258f4f";
request.partnerId = "1900000109";
request.prepayId= "1101000000140415649af9fc314aa427",;
request.packageValue = "Sign=WXPay";
request.nonceStr= "1101000000140429eb40476f8896f4c9";
request.timeStamp= "1398746574";
request.sign= "7FFECB600D7157C5AA49810D2D8F28BC2811827B";
api.sendReq(request);
```

从中可以观察到，商户 APP 需要构造一个 `PayReq` 支付请求对象，用于承载商户微信在平台的注册 ID（appId）、服务器端生成的订单号（prepayId）、订单请求的签名（sign）等信息，然后通过 `sendReq()` 方法向微信发起该支付请求。`sendReq()` 在[微信支付 SDK](https://bintray.com/wechat-sdk-team/maven) 里实现，反编译查看 `sendReq()` 方法

```java
WXApiImplV10.class

public final boolean sendReq(BaseReq paramBaseReq)
{
    if (this.detached) {
        throw new IllegalStateException("sendReq fail, WXMsgImpl has been detached");
    }
    if (!WXApiImplComm.validateAppSignatureForPackage(this.context, "com.tencent.mm", this.checkSignature))
    {
    Log.e("MicroMsg.SDK.WXApiImplV10", "sendReq failed for wechat app signature check failed");
    return false;
}
    if (!paramBaseReq.checkArgs())
    {
        Log.e("MicroMsg.SDK.WXApiImplV10", "sendReq checkArgs fail");
        return false;
    }
    Log.i("MicroMsg.SDK.WXApiImplV10", "sendReq, req type = " + paramBaseReq.getType());
    Bundle localBundle = new Bundle();
    paramBaseReq.toBundle(localBundle);
    if (paramBaseReq.getType() == 5) {
        return sendPayReq(this.context, localBundle);
    }
    ...
}
```

这里 `PayReq` 对象的 Type 为 5， 订单请求对象 `paramBaseReq` 转化为 `localBundle` Android Bundle 对象，作为入参进入 `sendPayReq()` 分支

```java
WXApiImplV10.class

private boolean sendPayReq(Context paramContext, Bundle paramBundle)
{
    if (wxappPayEntryClassname == null)
    {
        wxappPayEntryClassname = new MMSharedPreferences(paramContext).getString("_wxapp_pay_entry_classname_", null);
    Log.d("MicroMsg.SDK.WXApiImplV10", "pay, set wxappPayEntryClassname = " + wxappPayEntryClassname);
    if (wxappPayEntryClassname == null) {
        try
        {
            wxappPayEntryClassname = paramContext.getPackageManager().getApplicationInfo("com.tencent.mm", 128).metaData.getString("com.tencent.mm.BuildInfo.OPEN_SDK_PAY_ENTRY_CLASSNAME", null);
        }
        catch (Exception localException)
        {
            Log.e("MicroMsg.SDK.WXApiImplV10", "get from metaData failed : " + localException.getMessage());
        }
    }
    if (wxappPayEntryClassname == null)
    {
        Log.e("MicroMsg.SDK.WXApiImplV10", "pay fail, wxappPayEntryClassname is null");
        return false;
    }
    }
    MMessageActV2.Args localArgs;
    (localArgs = new MMessageActV2.Args()).bundle = paramBundle;
    localArgs.targetPkgName = "com.tencent.mm";
    localArgs.targetClassName = wxappPayEntryClassname;
    return MMessageActV2.send(paramContext, localArgs);
}
```

这里构造了请求微信支付所需的额外信息，和订单信息一起封装为 `paramContext` 对象，作为参数传递。程序走到 `MMessageActV2.send()`

```java
MMessageActV2.class
public static boolean send(Context paramContext, Args paramArgs)
{
    if ((paramContext == null) || (paramArgs == null))
    {
        Log.e("MicroMsg.SDK.MMessageAct", "send fail, invalid argument");
        return false;
    }
    if (d.a(paramArgs.targetPkgName))
    {
        Log.e("MicroMsg.SDK.MMessageAct", "send fail, invalid targetPkgName, targetPkgName = " + paramArgs.targetPkgName);
        return false;
    }
    if (d.a(paramArgs.targetClassName)) {
        paramArgs.targetClassName = (paramArgs.targetPkgName + ".wxapi.WXEntryActivity");
    }
    Log.d("MicroMsg.SDK.MMessageAct", "send, targetPkgName = " + paramArgs.targetPkgName + ", targetClassName = " + paramArgs.targetClassName);
    Intent localIntent;
    (localIntent = new Intent()).setClassName(paramArgs.targetPkgName, paramArgs.targetClassName);
    if (paramArgs.bundle != null) {
        localIntent.putExtras(paramArgs.bundle);
    }
    String str = paramContext.getPackageName();
    localIntent.putExtra("_mmessage_sdkVersion", 620823552);
    localIntent.putExtra("_mmessage_appPackage", str);
    localIntent.putExtra("_mmessage_content", paramArgs.content);
    localIntent.putExtra("_mmessage_checksum", b.a(paramArgs.content, 620823552, str));
    if (paramArgs.flags == -1) {
        localIntent.addFlags(268435456).addFlags(134217728);
    } else {
        localIntent.setFlags(paramArgs.flags);
    }
    try
    {
        paramContext.startActivity(localIntent);
    }
    catch (Exception paramContext)
    {
        Log.e("MicroMsg.SDK.MMessageAct", "send fail, ex = " + paramContext.getMessage());
        return false;
    }
    Log.d("MicroMsg.SDK.MMessageAct", "send mm message, intent=" + localIntent);
    return true;
}
```

看到熟悉的 Intent 和 startActivity 了，如设想的一样，支付 SDK 最终启动了一个入口 activity，即微信暴露的 `WXEntryActivity`。

上面的代码特别注意到两行

```java
String str = paramContext.getPackageName();
...
localIntent.putExtra("_mmessage_appPackage", str);
```

在构造传递给微信的 Intent 时，微信 SDK 使用 Android Framework 提供的  `getPackageName()` API 添加了商户 APP 的包名，猜测包名关系到后续的接口认证。 我们回到微信官方开发者网，其中有一段描述是对商户接入微信支付平台的要求：商户需要在网站后台中预置「应用包名」和「应用签名」（精确的描述应当是「应用签名证书的摘要」），同时微信在网站上提供了一个专门的工具，用于获取「应用签名」

![GenSignature App screenshot](/assets/img/post/2018/gensignature-apk-screenshot.png) 

注意该工具的提示信息「此工具仅用于获取安装到手机的第三方应用签名」，说明预注册于微信平台获取的签名来源于已安装的 APP，而未安装的APP是获取不到的，容易联想到用来做接口认证。凭借以往的开发经验，这个专门的工具一定用到了系统（Android Framework）接口来获取「应用签名」，同时我们有理由推测，微信在做接口认证时一样使用这个接口获取调用方的「应用签名」。

按照这个思路简单搜索一下就可以发现 Android 其实提供了 [PackageManager.getPackageInfo()](https://developer.android.com/reference/android/content/pm/PackageManager.html#getPackageInfo(java.lang.String,%20int)) 接口，该方法可返回指定应用的签名证书信息（signature），即微信所说的 「应用签名」

 [![](/assets/img/post/2018/packageinfo.signature.png)](https://developer.android.com/reference/android/content/pm/PackageInfo.html#signatures)

[⇒ 查看 getPackageInfo 的样例代码](https://stackoverflow.com/questions/9293019/get-certificate-fingerprint-from-android-app)

题外话：由于 APK 可以被多个私钥签名，因此接口返回的是 Signature 数组，在查看 Signature 类的 developer 文档时，发现一个有意思的描述：

> This class name is slightly misleading, since it's not actually a signature.

看起来「应用签名」错误叫法始作俑者是 Google？:)

言归正传，为了验证微信使用的是 getPackageInfo() 的推测，我们反编译 `Gen_Signature_Android.apk`，发现事情确实如此，下面我摘抄了部分反编译出的关键 smali 代码

```
.method private
    getRawSignature(Landroid/content/Context;Ljava/lang/String;)[Landroid/content/pm/Signature;
    ...
    invoke-virtual {v2, p2, v4}, Landroid/content/pm/PackageManager;->getPackageInfo(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;
    ...
    :cond_2
    iget-object v3, v1, Landroid/content/pm/PackageInfo;->signatures:[Landroid/content/pm/Signature;
    goto :goto_0
.end method
.method private getSign(Ljava/lang/String;)V
    ...
    invoke-direct {p0, p0, p1}, Lcom/tencent/mm/openapi/MainAct;->getRawSignature(Landroid/content/Context;Ljava/lang/String;)[Landroid/content/pm/Signature;
    ...
    .local v0, "sign":Landroid/content/pm/Signature;
    invoke-virtual {v0}, Landroid/content/pm/Signature;->toByteArray()[B
move-result-object v5
    invoke-static {v5}, Lcom/tencent/mm/openapi/MD5;->getMessageDigest([B)Ljava/lang/String;
    move-result-object v1
    .line 75
    .local v1, "signMd5":Ljava/lang/String;
invoke-direct {p0, v1}, Lcom/tencent/mm/openapi/MainAct;->stdout(Ljava/lang/String;)V
    .line 73
    add-int/lit8 v3, v3, 0x1
    goto :goto_0
.end method
```

现在我们知道获取应用的 signature 需要输入目标应用的包名，而包名通过 Android PackageManger 提供的 getPackageName() API 获取的。

到目前为止，我们还只是分析了 Client-Server 架构中 client 侧的逻辑，基本可以推测出整体接口认证的实现，但终究还是要到微信 APP 中一探究竟。

### 深入微信 APP

下载微信v6.6.6，反编译后直奔入口 `WXEntryActivity.samli`

微信客户端的 Java 代码做了混淆，但不影响我们分析。

首先 `getIntent()` 获取商户 APP 发送过来的 Intent

```
.class public Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;
    ...
    .method public onCreate(Landroid/os/Bundle;)V
    .locals 3
    .prologue
    .line 394
    invoke-virtual {p0}, Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;->getIntent()Landroid/content/Intent;
    ...
.end method
```

然后，从 Intent 中提取商户 APP 包名，初始化为 `WXEntryActivity` 的成员变量 `uN`

```
.class public Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;

.method private B(Landroid/content/Intent;)Z
    ...
    const-string/jumbo v1, "_mmessage_appPackage"
    invoke-static {p1, v1}, Lcom/tencent/mm/sdk/platformtools/s;->j(Landroid/content/Intent;Ljava/lang/String;)Ljava/lang/String;
    move-result-object v1
    iput-object v1, p0, Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;->uN:Ljava/lang/String;
    ...
.end method
```

跟踪 `uN` 的去向，来到 `WXEntryActivity.h()` 方法

```
.class public Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;

.method private h(Lcom/tencent/mm/ab/l;)Z
    ...
    iget-object v4, p0, Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;->appId:Ljava/lang/String;
    invoke-static {v4, v1}, Lcom/tencent/mm/pluginsdk/model/app/g;->be(Ljava/lang/String;Z)Lcom/tencent/mm/pluginsdk/model/app/f;
    move-result-object v4
    .line 601
    if-nez v4, :cond_2
    .line 602
    const-string/jumbo v1, "MicroMsg.WXEntryActivity"
    const-string/jumbo v2, "app not reg, do nothing"
    invoke-static {v1, v2}, Lcom/tencent/mm/sdk/platformtools/w;->w(Ljava/lang/String;Ljava/lang/String;)V
    goto :goto_0
    .line 612
    :cond_2
    iget-object v5, p0, Lcom/tencent/mm/plugin/base/stub/WXEntryActivity;->uN:Ljava/lang/String;
    invoke-static {p0, v4, v5}, Lcom/tencent/mm/pluginsdk/model/app/p;->b(Landroid/content/Context;Lcom/tencent/mm/pluginsdk/model/app/f;Ljava/lang/String;)Z
    move-result v5
    if-nez v5, :cond_3
    ...
.end method
```

注意其中一行

```
`invoke-static {p0, v4, v5}, Lcom/tencent/mm/pluginsdk/model/app/p;->b(Landroid/content/Context;Lcom/tencent/mm/pluginsdk/model/app/f;Ljava/lang/String;)Z`
```

这里调用了 `com.tencent.mm.pluginsdk.model.app.p` 类的 `b()` 方法，第三个入参 `v5` 寄存器即为 `uN`

那么第二个入参 `v4` 寄存器又是什么呢？

我们注意到，在上述的代码中还构造了一个 `com.mm.pluginsdk.model.app.f` 对象，而这个对象是用 `appId` 初始化的。还记得 appId 吧，是商户在微信平台注册的 APP 唯一ID。进一步观察 `f` 类，发现这实际上代表了「商户APP信息」类，这个类的成员包含了：

`field_appName` `field_packageName` `field_signature` ...

Good，这些不就是商户在微信开发者平台预注册时的信息吗？

有理由推断在构造 `f` 类对象时，微信应当是根据 `appId` 从微信服务器端取回的这些信息， 也就是「已注册的服务端商户 APP 对象」。时间有限，与服务器端的通信这块内容就不做深入分析了。

回到接收了 `v4` （即「f 类对象」），`v5`（即 `uN`，接口调用方包名）的`com.tencent.mm.pluginsdk.model.app.p.b()` 方法

```
.class public final Lcom/tencent/mm/pluginsdk/model/app/p;

    .method public static b(Landroid/content/Context;Lcom/tencent/mm/pluginsdk/model/app/f;Ljava/lang/String;)Z
    ...
    iget-object v2, p1, Lcom/tencent/mm/pluginsdk/model/app/f;->field_packageName:Ljava/lang/String;
    invoke-virtual {v2, p2}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z
    move-result v2
    if-nez v2, :cond_9
    const-string/jumbo v2, "MicroMsg.AppUtil"
    const-string/jumbo v4, "isAppValid, packageName is diff, src:%s,local:%s"
    ...
.end method
```

这里 `p1` 寄存器是「f 类对象」，而 `p2` 是 `uN` 即待验证调用方包名。`b()` 方法通过 `iget-object` 获取 `p1` 的 `field_packageName` 成员，然后与 `p2` 一起进行字符串比较 —— 比较待校验包名与根据 `appId` 获取的包名是否一致，如果不一致则走错误分支。原来，除了「应用签名」，微信其实还对调用方的包名和服务器端预注册的包名做了校验。

接下来是「应用签名」的校验，与包名校验类似，但是在校验之前，使用系统提供的 PackageManager API 获取到了 `uN` （`p2` 寄存器）对应的「应用签名」

```
.class public final Lcom/tencent/mm/pluginsdk/model/app/p;

.method public static b(Landroid/content/Context;Lcom/tencent/mm/pluginsdk/model/app/f;Ljava/lang/String;)Z
    invoke-static {p0, p2}, Lcom/tencent/mm/pluginsdk/model/app/p;->bh(Landroid/content/Context;Ljava/lang/String;)[Landroid/content/pm/Signature;
    ...
    :cond_9
    const-string/jumbo v2, "MicroMsg.AppUtil"
    const-string/jumbo v5, "server signatures:%s"
    new-array v6, v0, [Ljava/lang/Object;
    iget-object v7, p1, Lcom/tencent/mm/pluginsdk/model/app/f;->field_signature:Ljava/lang/String;
    aput-object v7, v6, v1
    invoke-static {v2, v5, v6}, Lcom/tencent/mm/sdk/platformtools/w;->i(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/Object;)V
    .line 276
    array-length v5, v4
    move v2, v1
    :goto_1
    if-ge v2, v5, :cond_b
    aget-object v6, v4, v2
    .line 277
    invoke-virtual {v6}, Landroid/content/pm/Signature;->toByteArray()[B
    move-result-object v6
    invoke-static {v6}, Lcom/tencent/mm/a/g;->u([B)Ljava/lang/String;
    move-result-object v6
    invoke-static {v6}, Lcom/tencent/mm/pluginsdk/model/app/p;->SZ(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v6
    .line 278
    const-string/jumbo v7, "MicroMsg.AppUtil"
    const-string/jumbo v8, "local signatures:%s"
    new-array v9, v0, [Ljava/lang/Object;
    aput-object v6, v9, v1
    invoke-static {v7, v8, v9}, Lcom/tencent/mm/sdk/platformtools/w;->i(Ljava/lang/String;Ljava/lang/String;[Ljava/lang/Object;)V
    .line 279
    iget-object v7, p1, Lcom/tencent/mm/pluginsdk/model/app/f;->field_signature:Ljava/lang/String;
    if-eqz v7, :cond_a
    iget-object v7, p1, Lcom/tencent/mm/pluginsdk/model/app/f;->field_signature:Ljava/lang/String;
    invoke-virtual {v7, v6}, Ljava/lang/String;->equals(Ljavalang/Object;)Z
    move-result v6
    if-eqz v6, :cond_a
    .line 280
    invoke-interface {v3, p1}, Lcom/tencent/mm/plugin/ac/a/a;->d(Lcom/tencent/mm/pluginsdk/model/app/f;)V
    goto/16 :goto_0
    .line 276
    :cond_a
    add-int/lit8 v2, v2, 0x1
    goto :goto_1
    .line 285
    :cond_b
    const-string/jumbo v0, "MicroMsg.AppUtil"
    const-string/jumbo v2, "isAppValid, signature is diff"
    invoke-static {v0, v2}, Lcom/tencent/mm/sdk/platformtools/w;- >w(Ljava/lang/String;Ljava/lang/String;)V
    .line 286
    invoke-interface {v3, p1}, Lcom/tencent/mm/plugin/ac/a/a;->c(Lcom/tencent/mm/pluginsdk/model/app/f;)V
    move v0, v1
    .line 287
    goto/16 :goto_0
.end method
```

至此，完成微信端的接口认证分析。

## 总结

微信支付的端侧接口认证实际上就是将 appId、packageName、调用方应用证书指纹（「应用签名」是不准确的说法）与服务器端预注册的值进行比较，从而防止非法应用接入微信支付系统。然而，这三个待验证参数中，只有调用方应用证书指纹是来自操作系统，而 appId、packageName 都是由调用方传入的（packageName 虽然由微信 SDK 通过系统接口获取，但是由于在客户端执行，实际上很容易绕过），这导致了仿冒的可能：恶意 APP 可以任意仿冒已安装于用户手机上的合法应用来仿冒其发起支付，通过在支付请求中构造被仿冒应用的 appId 和 packageName（这些信息都是公开的）。

从安全角度看，微信这样的设计是不正确的。安全的设计应当是由服务端微信 APP 通过 Android 系统接口获取调用方包名，从而防止客户端仿冒。然而，博主在一番搜索之后并没有发现 Android Framework 提供了这样的接口，例如，通过发送方的 Intent 获取源应用包名，这或许能解释为什么微信竟然用客户端传递的包名来做认证。

我想，造成这个安全问题的根本原因是 Android 系统并未提供从调用方 Intent 中获取包名的能力，这也许是 Android 的缺陷。
