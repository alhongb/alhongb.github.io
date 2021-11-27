---
title: App Links 及其安全性
layout: post
categories: [Android]
tags: [Android]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

在 Android 上 Google 提供了一套名为 [App Links]( https://developer.android.com/training/app-links/deep-linking ) 和 [Deep Links]( https://developer.android.com/training/app-links/verify-site-associations )  的由 URI/URL 跳转到原生 Android 应用组件的协议及实现，使得用户从能够从原生应用（包括浏览器页面）直接通过链接跳转到指定的应用内容。本文详细介绍了它们如何运作，并剖析了其中的安全性。

## App Links 的实现细节及与 Deep Links 间区别

App Links 是一种特殊类型的 Deep Links，可以看作是 Deep Links 的升级版本（Android 6.0 引入）。二者主要区别在于：当用户点击 App Links 时将直接跳转到应用内容，而 Deep Links 则会弹出应用选择对话框（如果用户设备上有多个应用可以处理相同的 intent）。举例来说：在电子邮件中点击银行发送的 HTTP 网址，如果是 Deep Link 系统可能会显示一个对话框，询问用户是使用浏览器还是银行自己的应用打开此链接，而 App Link 则直接跳转到应用。

此外二者的形式上也存在区别：App Links 和传统 HTTP URL 链接格式保持一致，并且一定关联于某一个 Web 站点 ，这意味着当用户没有安装实体应用时，App Links 就直接呈现为 Web 网站内容，而不会出现 404 页面或异常；Deep Links 则更加宽泛，它的 URI 在 http/https 协议之外还支持[自定义 scheme](https://developer.chrome.com/multidevice/android/intents) ，意味着如果是自定义 schema，Deep Links 虽可在浏览器中使用，但却不指向 Web 网站内容，且当应用不存在时，跳转也将失效。

一言以概之，由于只支持 http/https 协议并关联到网站，App Links 可以简单理解为传统网站内容的应用版本。

在开发层面，创建一个完整的 App Links 包括：

1. 遵循完整的 Deep Links 创建过程 ：在 Android 应用清单中为 URI 创建对应的 intent 过滤器，即在目标 Activity 的 intent-filter 中，data 标签下配置一个或多个 URI（URI 支持细粒度路径，从而让不同 Activity 匹配不同 URI 路径），并且将应用设置为能够正确处理包含来自网站内容的 intent。创建 App Links 的要点仅仅是前述 URI 只能为 http/https 协议。

2. 要求应用和网站之间建立认证关联，这是网站上发布 Digital Asset Links JSON 文件实现的，在安全性中将展开说明。

```xml
<activity
    android:name="com.example.android.GizmosActivity"
    android:label="@string/title_gizmos" >
    <intent-filter android:label="@string/filter_view_http_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "http://www.example.com/gizmos” -->
        <data android:scheme="http"
              android:host="www.example.com"
              android:pathPrefix="/gizmos" />
        <!-- note that the leading "/" is required for pathPrefix-->
    </intent-filter>
    <intent-filter android:label="@string/filter_view_example_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "example://gizmos” -->
        <data android:scheme="example"
              android:host="gizmos" />
    </intent-filter>
</activity>
```

引用[官网的表格](https://developer.android.com/training/app-links/verify-site-associations#the-difference)，完整展示二者的区别：

| | Deep Link | App Link |
| :-------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| intent 网址协议 | `http`、`https` 或自定义协议 | 需要 `http` 或 `https` |
| intent 操作 | 任何操作 | 需要 `android.intent.action.VIEW` |
| intent 类别 | 任何类别 | 需要 `android.intent.category.BROWSABLE` 和 `android.intent.category.DEFAULT` |
| 链接验证 | 无 | 需要通过 HTTPS 协议在您的网站上发布 [Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started?hl=zh-cn) 文件 |
| 用户体验 | 可能会显示一个消除歧义对话框，以供用户选择用于打开链接的应用 | 无对话框；您的应用会打开以处理您的网站链接 |
| 兼容性 | 所有 Android 版本 | Android 6.0 及更高版本 |

## 安全性

因为在逻辑上，App Links 是「网站内容的应用版本」或「网站内容的扩展部分」，并且 Google 的实现要做到无干预跳转，当用户从网站（App Links）跳转到应用时就必须避免用户跳转到仿冒的应用上。同样是上面银行邮件的例子，如果任何应用都能轻易地被作为该银行应用跳转，将造成严重安全问题。也就是说，App Links 最重要的安全特性是对应用进行身份认证。

Google 具体是这样实现网站对应用的认证的：

当应用关联的 Deep Link 想要升级为 App Link（即成为指定 URL 的默认处理程序）时，要在 AndroidManifest Activity 的 intent 过滤器中设置 `android:autoVerify="true"`

```xml
    <application>
      <activity android:name=”MainActivity”>
        <intent-filter android:autoVerify="true">
          <action android:name="android.intent.action.VIEW" />
          <category android:name="android.intent.category.DEFAULT" />
          <category android:name="android.intent.category.BROWSABLE" />
          <data android:scheme="https" android:host="www.example.com" />
          <data android:scheme="https" android:host="mobile.example.com" />
        </intent-filter>
      </activity>
    </application>
```

随后 Android 系统（6.0 及以上版本，安装应用时）就会验证该 intent 过滤器中每一个域名所对应的服务器，即查询并校验相应服务器上位于 `https://hostname/.well-known/assetlinks.json` 的 Digital Asset Links 文件。

`assetlinks.json` 的内容

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```

_注：一个网站可与多个应用相关联，一个应用也可以关联到多个网站：如果关联多个子域名，则需要在每个子域名上放置一个有效的 `assetlinks.json`；如果采用通配符声明域名（如 `*.example.com`）则只需在根域名下放置 `assetlinks.json` 即可。_

其中， `package_name` 为在应用的 build.gradle 文件中声明的应用 ID，`sha256_cert_fingerprints` 为应用签名证书的 SHA256 指纹。

这样，通过指定应用的证书指纹，网站就与经过 Android 系统校验的合法的应用关联在一起了，这种关联（认证）基于应用的数字签名，只要开发者签名私钥安全保管，同时Android系统取到正确的 `assetlinks.json` ，安全性就可以保证。

当然，为了保证系统能取到正确的 `assetlinks.json` ，Google 只允许 `assetlinks.json` 通过 https 访问，无论开发者设置的 App Link 协议是 http 还是 https

这样，经过严格的认证措施，任何未经验证的应用都无法通过仿冒合法应用的方式来被 App Link 跳转。

