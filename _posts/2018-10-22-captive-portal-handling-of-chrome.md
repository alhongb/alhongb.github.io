---
layout: post
title: Chrome 的 Captive Portal 处理机制
categories: [Web]
tags: [Chromium]
seo:
  date_modified: 2020-03-02 22:56:24 +0800
---

Google Chrome 在 M63 版本引入了一项修改以解决 Captive Portal 场景下的用户登录问题 —— 当未登录的用户访问 HTTPS 网站时，由于 Captive Portal 的拦截，浏览器会出现网络超时、SSL/TLS 告警等问题。本文详细介绍了 Chrome M63 的这项机制。

## 背景

Captive Portal 中文通常译作“强制主页”或“强制登录门户”，是用户被网关授权访问 Internet 前的登录页面，在很多公共地点（如咖啡厅、机场、酒店）的网络连接都会使用到这项技术来要求用户认证后上网。

Captive Portal 工作原理无非两种

- 将所有 DNS 请求指向自己的 portal 地址
- 直接劫持 HTTP/HTTPS 流量，响应自己的页面

### Captive Portal 的 HTTPS 之痛

无论是哪种工作原理的 Captive Portal，当用户访问 HTTP 网址时会直接劫持到 portal 登录页，浏览器端不会有异常发生；当用户访问 HTTPS 网站时则会因为 Captive Portal 无法提供该网站的合法证书而抛出 SSL/TLS 安全告警页。

当遇见 SSL/TLS 错误时，用户有时可在浏览器告警页上找到“继续访问”按钮而进入 portal 登录页，有时则有因浏览器未提供“继续访问”按钮而阻塞访问（通常是受限于 [HSTS](https://tools.ietf.org/html/rfc6797#section-12.1)）。不论用户刷新页面还是更换其他 HTTPS 网址都无法解决问题，造成了糟糕的体验。

**提示：**HSTS 规范（[RFC 6797](https://tools.ietf.org/html/rfc6797#section-12.1)）要求发生 SSL/TLS 错误时浏览器应禁止用户继续访问。Why？我们都知道 HTTPS 被中间人攻击时浏览器会有 SSL/TLS 错误，但如果用户养成习惯于“我不明白这是什么，请继续访问”，那么网站部署 HTTPS 就毫无意义了，尤其是高度重视安全的网站：银行、金融网站如果在发生 SSL/TLS 错误时允许用户跳过，将发生非常严重的安全后果。因此，这些网站的安全管理员会开启 HSTS，发生错误时不允许用户绕过。包括 Safari、Firefox 在内的国际浏览器都不允许 HSTS 下的 SSL/TLS 错误被绕过。

可见，实际上只有 HTTPS 场景下才会有 Captive Portal 体验问题。

## Chrome 的 Captive Portal 处理机制

对于用户登录之前发起的 HTTPS 连接，不同类型的 Captive Portal 通常分有两种处理方式：静默地丢弃设备发送的 HTTPS 数据包或劫持为自己的 HTTPS 响应。

当所处的 Captive Portal 属于前者，访问任何 HTTPS 页面，例如 `https://google.com` 时，都将产生超时；当所处的 Captive Portal 属于后者，访问任何 HTTPS 页面，例如 `https://google.com` 时，由于采用 SSL/TLS 安全连接，无论 Captive Portal 所采用的技术是将 `gmali.com` 地址指向自己的 Web 服务、替换 `gmail.com` 的证书为自签名证书，还是直接响应明文 HTTP 报文，都将必然发生安全错误。

前面已经提到，浏览器位于 Captive Portal 下且未登录时，用户访问明文 HTTP 是不存在阻塞问题的，因此针对上述不同的场景，Chrome 的解决方案是引入一个 Captive Portal 检测机制，检测到 Cpative Portal 时引导用户打开一个明文 HTTP 页面。

### 检测 Captive Portal

具体来说，当一次 HTTPS 加载耗用了一段时间或出现 SSL/TLS 错误、`ERR_SSL_PROTOCOL_ERROR` 错误时，Chrome 会在后台发起一次对 `http://www.gstatic.com/generate_204` 的请求，然后根据其响应判断当前环境是否处在 Captive Portal 下。这个探测 URL 对应 Google 的服务器正常情况下会返回 HTTP 204 （“No Content”）响应（按 Google 的说法不会记录任何 cookie 和日志）。

如果 Chrome 收到了 204 响应码，说明当前网络能够访问 Internet，浏览器不在 Captive Portal 下或已经登录，此时 Chrome 不需要特别处理，按照通常的网络问题、SSL/TLS 错误处理即可；如果 Chrome 遇到了一个登录页或重定向的响应（这是 Captive Portal 通用的做法），说明浏览器正处在一个 Captive Portal 下；如果发生了错误（Chrome 认为除了 HTTP 2xx、3xx、511 状态码外，均为错误）或响应了非 HTTP 报文，说明浏览器不在 Captive Portal 下或 Captive Portal 本身有异常，无法进入登录页，此时 Chrome 也无需特别处理。

无论 Captive Portal 是何种类型，如何处理浏览器的探测 URL 请求（篡改其 DNS 请求还是劫持 HTTP 报文），都不影响 Chrome 的这种检测机制。

### 引导用户打开 HTTP 窗口

当 Chrome 检测到用户在提供登录页的 Captive Portal 下访问 HTTPS 网页时，会在原始 HTTPS 页面内通过 UI 引导用户打开一个新的 Tab，在这个新的 Tab 中 Chrome 将加载先前提到的 204 响应网址。如前所述，`http://www.gstatic.com/generate_204` 是明文 HTTP 连接，不存在 HTTPS 场景下的各种问题，因此用户顺利进入 Captive Portal 登录页。

### 使用操作系统的 Captive Portal 检测能力

一些操作系统如 Lion、Windows 8/10、高版本 Android 具有系统原生 Captive Portal 检测机制，Chrome 会在这些平台上使用平台提供的接口检测是否处在 Captive Portal 下，而早期的 Windwos 版本虽然也具有检测能力，但可靠性不佳，Chrome 在这些版本 Windows 上依然会采用内建的检测机制。

进入登录页后，Chrome 会持续检测网络连接。当用户完成登录操作连接到 Internet 时，Chrome 刷新所有先前阻塞在 HTTPS 超时、SSL/TLS 错误的 tab （POST 请求除外）

### 代码实现

在 `SSLErrorHandler::StartHandlingError` 中，Chrome 处理 SSL 错误时，会检测是否是处在 Captive Portal 下，如果是，调用 `ShowCaptivePortalInterstitial` 展示 Chrome 内建的 Captive Portal 连接提示页面。

[代码参见](https://cs.chromium.org/chromium/src/chrome/browser/ssl/ssl_error_handler.cc?l=758&gs=kythe%253A%252F%252Fchromium%253Flang%253Dc%25252B%25252B%253Fpath%253Dsrc%252Fchrome%252Fbrowser%252Fssl%252Fssl_error_handler.cc%2523QwZJlwfNQgMwVE2B2CC4htstj%25252FSxA7%25252BrX1he1%25252FWZnq0%25253D&gsn=StartHandlingError&ct=xref_usages)

## 本文发布时的一些失效场景和 bug

根据 Google 的设计文档，Chrome 仅仅在 SSL/TLS 超时和某些特定的错误（SSL 证书错误）场景才会触发 Captive Portal 检测，如果所有的网络错误都进行检查，可能会有延迟，准确性和网络带宽等问题。也就是说检测机制本身不是非常完善的。

例如，一些 Captive Portal 会拒绝 HTTPS 连接，与丢弃数据包导致 HTTPS 超时不同，这种场景下 Chrome 直接产生 `ERR_CONNECTION_REFUSED` 错误，而不会进行 Captive Portal 检测。

另外有一个 [bug](https://bugs.chromium.org/p/chromium/issues/detail?id=801734)，当用户从连接提示页（`chrome://interstitials/captiveportal`）导航进入 Captive Portal 登录页，而登陆页本身有 SSL/TLS 问题（如自签名证书）时，由于该问题会触发 Chrome 检测 Captive Portal，会再次进入连接提示页，而非标准的 SSL/TLS 错误。由于没有忽略按钮，最终造成死循环

## 安全考虑

看完这套复杂的处理机制，你可能会问：为什么不干脆一些，检测到处在 Captive Portal 下时就忽略 SSL/TLS 错误，直接进入登录页面，或者给用户一个风险提示，提供“继续连接”按钮呢？

这涉及安全。

HSTS 规范要求浏览器访问网站时始终采用 HTTPS 发起连接，前面我们还说过，配置了 HSTS 的网站在发生 SSL/TLS 错误时浏览器必须禁止用户忽略错误继续访问。从安全角度看，Captive Portal 使用的技术和黑客技术（中间人攻击）本质上是一模一样的，浏览器（或操作系统）本身无法鉴别所处的网络环境是正常的 Captive Portal 还是一次恶意攻击，明白这一点你就能理解为什么在 HTTPS 场景下 Chrome 不直接忽略这种错误而引入一个全新的 Captive Portal 检测服务。

## 参考  

- [Captive portal handling for HTTPS requests](https://docs.google.com/document/u/0/d/1k-gP2sswzYNvryu9NcgN7q5XrsMlUdlUdoW9WRaEmfM/mobilebasic)
