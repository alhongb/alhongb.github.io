---
title: 深入理解 OAuth 2.0
categories: [Web]
tags: [OAuth 2.0]
layout: post
seo:
  date_modified: 2020-03-17 14:34:48 +0800
---

OAuth 2.0 RFC（[The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)）篇幅很长、内容详实，覆盖了包括原理概念、工作流程、报文格式、安全性、拓展性等等诸多方面，直接阅读十分不易。因此，本文结合博主的背景知识，提取、翻译规范主干内容并深入解读 OAuth 2.0，意图帮助读者避免一开始就陷入 RFC 的繁枝末节中。

## 整体介绍

在传统的客户端/服务器（`client-server`）认证模型中，客户端直接使用 `resource orwner`（资源所有者）的凭据（`credentials`，例如账户和口令）来请求访问服务器上的 `protected resource`（受保护资源）。在这种模型下，想要提供第三方访问 `protected resource` 的 能力，`resource orwner` 就需要将其凭证分享给第三方。这就引入一些问题与局限：

1. 第三方应用（`third-party application`）需要存储 `resource owner` 的凭据以供后续使用，通常会是明文的口令。
2. 服务器需要支持口令认证，尽管这种认证方式有固有的弱点。
3. 第三方应用能够访问 `resource owner` 过于广泛的受保护资源，而 `resource owner` 缺乏任何限制能力，包括限制访问的时间周期、范围。（解读：这是因为用于认证 `resource owner` 身份的原始凭证已经泄露给三方了）
4. `resource owner` 无法吊销特定三方的访问，而只能撤销所有三方的访问权限，而且要做的这点，必须通过修改密码的方式。
5. 任何一个第三方应用的失陷（`compromise`）都会造成最终用户的口令泄露，也即意味着所有受此口令保护的资源将沦陷。

解读：传统授权模型具有上述缺陷的根本原因在于 `client` 和 `resource owner` 之间没有区分开：当 `client` 获取到原始授权凭据后，它实际就成为了 `resouce owner`。

OAuth 2.0 授权框架通过在 `client` 和 `resource owner` 之间引入一个 `authorization layer`（这是一个抽象的层，`abstraction layer`）并将角色 `client` 从 `resource owner` 中分离来解决上述问题：`client` 不再使用 `resource owner` 自身的凭据来访问 `resource owner` 控制的、由 `resource server` 托管的 `resource`。

`client` 不使用 `resource  owner` 自身凭据来访问资源，而是使用 `access token` —— 一个可以表示访问的范围，生命周期，及其他属性的字符串。`access token` 在 `resource owner` 的同意下由 `authorization server` 颁布给第三方客户端（`client`），接着 `client` 用 `access token` 来访问托管于 `resource server` 的受保护资源。

举例来说：一个最终用户（`resource owner`) 可以授权某个打印机服务（`client`）访问她存储于照片分享服务提供者（`resource server`）上的受保护照片，而不用和打印机服务分享她自己的用户名和口令。取代之的是，她直接向照片分享服务提供者信任的服务器认证（`authorization Server`），然后服务器向打印机服务发布委托专用（`delegation-specific`）凭据（`access token`）。

解读：「委托专用」意味该凭据只能用于委派访问场景，而不可用于其他的，如认证场景。传统认证授权模式下客户端直接使用用户「身份」，即使后续不保存用户的凭证（账户和口令），也使用了 token、session ID 等身份认证凭据。这是两种模式的本质区别，也是「授权」和「认证」的区别。

### 角色

OAuth 2.0 授权模型定义了四种角色

- resource owner

资源所有者，是能够授予受保护资源访问权的实体。特别地，当资源所有者是一个人时，称之为 `end-user`（最终用户）

- resource server

资源服务器，托管受保护资源的服务器，能勾接收和响应受保护资源请求（通过 `access token`）

- client

客户端，一个在 `resource owner` 的授权下代其请求受保护资源的应用。RFC6749 特别指明「client」这一术语没有任何具体技术实现的约束，所以不论是在服务器、桌面平台还是终端设备上运行的各类程序，都是可以的。

- authorization server

授权服务器，在成功认证 `resource owner` 并获其授权（`authorization`）后颁发 `access token` 的服务器

OAuth 2.0 也没有规定 `authorization server` 和 `resource server` 之间应该如何交互：根据不同的实现，二者可以同时部署在一个服务器上，也可以分属于不同的实体。而且，单个 `authorization Server` 也可能签发被多个 `resource server` 接受的 `access token`。

### Protocol Flow

<pre><font face="monospace">     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+

                  图 1：逻辑工作流程
</font></pre>

OAuth 2.0 定义了多种工作模式以应用不同场景，但逻辑上它们的工作流程可以统一地抽象为上图，此图描述了各角色间如何交互：

  (A)   `client` 请求来自 `resource owner` 的授权。授权请求可以直接地发往 `resouce owner`（正如图中所示），也可以间接地通过 `authorization server` 作为中介来处理。

  (B)   `client` 获得 `resource owner` 的授权许可（`authorization grant`）。这里许可模式（`grant type`）取决于 `client` 请求中的参数以及 `authorization server` 侧是否支持，可以是 RFC 中定义的四种之一，也可以是自定义扩展方式。详见后文。

注解：在不同工作模式下，`resource owner` 可以使用其凭证直接响应 `client` 的访问请求（这将暴露其凭证给 `client` 所属的第三方或所在的应用程序环境），也可以转而在独立的 `authorization server` （即所谓的抽象授权层）处处理访问请求。显然，这两种方式有着不同的安全等级。

  (C)  `client` 向 `authorization server` 认证自己的身份并展示（`resouce owner` 的） 授权许可，以获取 `access token`。

  (D)  `authorization server` 认证 `client` 并校验授权许可，如果合法，颁发 `access token`。

  (E) `client` 展示 `access token` 来认证，请求访问 `resource server` 上的受保护资源

  (F) `resource server` 校验  `access token` ，如果合法，正常响应资源请求

### Authorization Grant

`authorization grant` （授权许可），是一个代表 `resource owner` 对受保护资源进行了授权的凭据，被 `client` 使用来获取访问资源的 `access token`。OAuth 2.0 定义了四种类型（`type`）的 `authorization grant`，包括：`authorization code`、`implicit`、`resource owner password credentials` 和 `client credentials`。字面上看，它们的区别在于有的直接使用 `resource owner` 的账户口令来授权，有的采用间接方式授权，而有的只用 `client` 身份即可获得访问授权。此外，OAuth 2.0 还支持[扩展的许可类型](https://tools.ietf.org/html/rfc6749#section-4.5)，受限篇幅本文不做介绍。

回顾逻辑工作流图可以看到，`authorization grant` 被用于 `(B)`、`(C)` 两个步骤，实际工作流中则分别对应有四种「flow」。其中有些工作流程简单、有些相对复杂，有些具有 OAuth 2.0 全部安全属性而有些则安全性欠佳。这部分内容在章节[「四种授权模式及工作流」](#四种授权模式及工作流)会详细展开介绍。

### Access Token

`access token` 是访问受保护资源的凭据，是一段颁布给 `client`、代表授权的字符串。具体而言，`token` 表达了 `resource owner` 授予访问的特定范围（`scope`）、持续时间，并且由 `resource server` 和 `authorization server` 落实前述限制。通常，`access token` 字符串（的含义）对 `client` 不透明。

有两种类型的 `token` 实现：1. 作为授权信息的索引，以获取实际的授权信息；2. 自包含（`self-contain`）授权信息，并且可执行校验，例如一串包含数据和数字签名的字符串（一个典型的例子是 JSON Web Token，缩写 JWT）。实践中，`client` 使用 `access token` 可能还需要其他认证凭据，这点不在 OAuth 2.0 规范范围内。

前文提到 OAuth 2.0 引入了一个授权抽象层，`access token` 就是实现抽象的关键措施：它替换了原有的授权结构，相比用户名、密码等传统授权方式，使用 `access token` 授权可以实行更多的约束机制（原文：「This abstraction enables issuing access tokens more restrictive than the authorization grant used to obtain them」）。另外，这使得授权协议更加简洁：`resource server` 只需理解 `access token` ，而不需要理解各式各样的认证方式。

`access token` 可以有不同的格式、结构、以及采取的措施（例如密码学属性），取决于服务端的安全需求。`access token` 具备哪些属性、如何用它来访问受保护资源方法的定义超出了本规范的范畴，在多个配套的 RFC 定义：如 [The OAuth 2.0 Authorization Framework: Bearer Token Usage - RFC6450](https://tools.ietf.org/html/rfc6750) 定义了 目前 OAuth 2.0 主流使用 access token 的方式（称之为 Bearer Token），包括定义 HTTP 报文格式等，Bearer Token 的格式在这个规范中依然没有定义，可以是简短的字符串，也可以是 JWT）；[OAuth 2.0 Token Introspection - RFC 7662](https://tools.ietf.org/html/rfc7662) 则定义了一个协议，包括  access token 的属性规格（例如有效性，允许的 scope 上下文等）以及 resource owner 如何与 authorization server 通信来获取 access token 的属性信息。

###  Refresh Token

`refresh token` 是获取 `access token` 的凭据。`refresh token` 由 `authorization server` 颁布给 `client`，用于获取新的 `access token` ——当前的 `access token` 已失效或过期、为了获取额外的 `scope` 等价或收窄的 `access token`（`access token` 可能具有更短的生命周期和更少的用户授权）。是否签发 `refresh token` 取决于 `authorization server` 的权衡，如果签发，将随 `access token` 一同发布（如图 1 的 `(D)` 步骤）

同样，`refresh token` 也是一段代表 `resource owner` 对 `client` 授权的字符串，且该字符串也是对 client 不透明。`token` 是授权信息的标识（索引），以供检索授权信息。与 `access token` 不同，`refresh token` 仅适用于 `authorization server`，而且绝不会发送给 `resource server`。

<pre><font face="monospace">  +--------+                                           +---------------+
  |        |--(A)------- Authorization Grant --------->|               |
  |        |                                           |               |
  |        |<-(B)----------- Access Token -------------|               |
  |        |               & refresh token             |               |
  |        |                                           |               |
  |        |                            +----------+   |               |
  |        |--(C)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
  | Client |                            |  Server  |   |     Server    |
  |        |--(E)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(F)- Invalid Token Error -|          |   |               |
  |        |                            +----------+   |               |
  |        |                                           |               |
  |        |--(G)----------- refresh token ----------->|               |
  |        |                                           |               |
  |        |<-(H)----------- Access Token -------------|               |
  +--------+           & Optional refresh token        +---------------+

               图 2：刷新过期的 Access Token
</font></pre>

图 2 所示的 `access token` 刷新流程包括以下步骤:

(A)   `client` 使 ` resouce owner` 的  `authorization grant` 作为认证，向 `authorization server` 请求  `access token`

(B)  `authorzation server` 认证 ` client`  并校验 `authorization grant`，如果合法，签发 `access token`  和 `refresh token`。

(C)  `client` 使用 `access token` 向 `resource server` 请求访问受保护资源

(D)  `resource server` 校验 `access token` ，如果合法，服务本次请求。

(E)  重复步骤 (C) 和 (D) 直到  `access token`  过期。如果 `client` 发现 `access token` 过期，跳到步骤 (G)；否则发起下一次受保护资源请求

(F)  由于 `access token` 非法，`resource server` 返回一个 token 无效的错误

(G)   `client` 向 `authorization server` 认证身份并使用 `refresh token` 申请一个新的 `access token` 。`client` 认证要求取决于client的类型以及 `authorization server` 的策略。

(H) `authorization server` 认证 `client` 并校验 `refresh token`，如果合法，签发一个新的 `access token` （可选地，一个新的 `refresh token`）。

步骤 `(C)`，`(D)`，`(E)`， 和 `(F)` 的实现细节超出了 OAuth 2.0 规范的范畴、

注：规范对 `refresh token` 在不同工作模式的约束如下

| 工作模式 | Refresh Token |
| -------- | ------------- |
| code     | 可选          |
| implicit | 禁用          |
| password | 可选          |
| client   | 不应该使用    |

## Client

### Client Registration

接入 OAuth 2.0 协议之前，要向 `authorization Server` 注册 `client`，使 `authorization Server` 能提前获知 `clients` 的信息以做更细粒度的访问控制，至于通过何种渠道注册并不在规范范畴内定义，通常情况是用户在 HTML 表单中提交注册操作。而注册的内容，OAuth 2.0 认为**应该**包括：

- 指定 `client type`，正如本文下一节 [「Client Type」](#client-type-and-client-authentication) 所述的
- 提供 `client` 的 `redirection URI`

重定向 URI（`redirection URI`）是标识可以接受响应的一个地址，无论是 `client`、`user-agent` 还是 `authorization server`，都会使用这种地址跳转技术来发送或接受消息，这种行为类似于软件开放中的 callback：请求者留下一个重定向地址，响应者则通过这一地址发送准备好的响应。在 OAuth 2.0 你将多次看到它。详细内容参考 [rfc6749#section-3.1.2](https://tools.ietf.org/html/rfc6749#section-3.1.2)

- `authorization server` 要求的任何其他信息，如应用名称、网站地址、描述、logo 图片和接受的法律条款等等

### Client Type and Client Authentication

OAuth 2.0 身份信息是否机密两种类型的 `client`

#### confidential client

具备安全地保管客户端凭证的能力的 `client` ，例如由安全服务器实现的程序。一个 `client` 是 `confidential ` 的，意味着 `authorization server` 出于安全需求，会（且[必须](https://tools.ietf.org/html/rfc6749#section-3.2.1)）对 `client` 执行满足其安全要求的认证措施，而 `client` 用于认证的身份凭证可以是口令、公私钥对等等。

你可能会问，为什么要对 `client` 进行认证，有什么好处？

- 将颁布给 `client` 的 `refresh token` 和 `authorization code` 和它绑定起来，并能校验，尤其是在不安全传输通道传递或重定向 URI（指向 `redirection endpoint` 即 `client`）在注册时并不完整的情况下。
- 可以对 compromised 的 `client ` 进行吊销，从而避免攻击者滥用盗取的 `refresh token`，至于如何吊销，禁用或改变某个 `client` 的身份凭证均可。很明显，禁用/改变单个 `client` 的身份凭证比吊销一整套 `refresh token` 要来得快捷。
- 带来认证管理的最佳实践，因为认证管理一般要求定期轮转身份凭证（`periodic credential rotation`）。定期轮转一整套的 `refresh token` 要比轮转单独的 `client` 身份凭证要复杂得多。

#### public client

缺乏安全保管凭证的能力的 `client`，例如安装到最终用户设备的软件。authorization server 可选（`MAY`）和 `public client` 建立身份认证措施。然而，此时 `authorization server` 一定不能（`MUST NOT`) 为了辨识 `client` 身份而信赖 `public client` 的认证结果。

解读：这是很显然的，`public client` 不具备安全的认证能力

### 典型的 client 实现

有 3 种典型实现：

- web application

使用诸如 PHP、Java、Python、Ruby 和 ASP.NET 这样的语言和相应框架开发的运行在服务器上的程序（也叫后端），`resource owner` 在设备上通过 user-agent（一般就是浏览器）访问其提供的 HTML 用户界面（也叫前端）。这种架构下，`client` 的身份凭据以及任何被颁予的 `access token`/`refresh token` 都存储在后端，即不会暴露给 `resource owner`。所以有前后端之分的 `web application` 是 `confidential` 的，应使用最完整、最安全的 `authorization code` 授权模式（`grant type`）。实际例子： Google OAuth 2.0 之 [Web Server Applications](https://developers.google.com/identity/protocols/OAuth2WebServer)。

- user-agent-based application

顾名思义，这是一种代码是从 Web 服务器上下载然后运行在用户设备上的 user-agent（例如浏览器）之上的 `client`。它没有执行代码的后端服务器，只有负责托管前端资源服务器，因此只能借助 user-agent 的能力来调用 `resource server`、`authorization server` 的接口。也因此 OAuth 2.0 工作流程中的数据流、凭证&授权信息都将很容易就能被 `resource owner` 访问（通常也是可见的）。所以，它使用 `code` 模式将没有意义，不仅没有安全效益，还会降低性能。显然 `user-agent-based application` 是 `public` 的，应使用 `implicit` 授权模式。实际例子：Google OAuth  2.0 之 [Client-side Web Applications](https://developers.google.com/identity/protocols/OAuth2UserAgent)（也有称为 `Single-Page Apps` 的）。

- native application

一种在最终用户设备上安装和运行（比如桌面应用，手机原生应用）、并被 `resource owner` 使用的 `client`，与  `user-agent-based application`  一样，数据流、凭证&授权信息也是可以被  `resource owner` 访问的。`native application` 是 `public` 的，可以使用 `authorization code` 模式或 `implicit` 模式，但由于 `native application` 不具维持身份凭证机密性的能力，因而使用 `authorization code` 授权模式将达不到预期的安全性，所以理论上不应该执行 `client credentials` 验证。但在 [OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252) 和
[Proof Key for Code Exchange by OAuth Public Clients (PKCE)](https://tools.ietf.org/html/rfc7636) 两个 `native application` 实践规范中提出了改进的使用 `authorization code` 授权模式方法，读者可自行前往了解。实际的例子有：Google OAuth 2 之 [Mobile & Desktop Apps](https://developers.google.com/identity/protocols/OAuth2InstalledApp)

到这里你可能会疑惑，`native application` 与前述 `user-agent-based application` 有什么区别呢？

`user-agent-based application` 本身就运行在 user-agent（如浏览器）内部，所以它很容易就能利用 user-agent 的能力来执行授权请求（想想看，授权系统大部分是构建在 HTTP Web 上，并使用 user-agent 提供用户界面），而 `native application` （注意区别于 user-agent，二者概念上是分离的）：它需要额外借助一个 `external user-agent` （例如：独立的浏览器应用）或 `embedded user-agent`（例如：嵌入原生应用的 WebView）来执行授权请求，所以相比 `user-agent-based application`，`native application` 要特别地考虑安全性、操作系统特性以及整体的用户体验。

另外，按使用不同形式的 user-agent ，`native application` 接收 `authorization server` 响应的授权信息（` token` 或 `code`）的方式不同：

External user-agent - `native application` 向操作系统预先注册 URI scheme（例如 [Android Deep Link](https://developer.android.com/training/app-links)），然后由 user-agent 重定向此 URI 来接收响应（参考 [Chrome 的实现](https://developer.chrome.com/multidevice/android/intents)）；手动复制粘贴授权信息；启动一个本地的 web 服务器；安装一个浏览器插件/扩展；或如果有 `client` 可控的服务器，也可以重定向响应到该服务器，由该服务转递给 `native application` 使用；

Embedded user-agent - `native application` 通过直接通信从 `embedded user-agent` 处接受响应，所谓直接通信，包括监视资源加载时导致的状态变化或访问 user-agent 的 cookie 存储。

（造成上述二者区别的原因：技术上前者通常和 `native application` 是不同应用程序，有不同进程边界，而后者则或运行在 `native application` 的进程空间或属于 `native application` 的一部分）

扩展解读：距离 RFC6749 发布近 8 年，现代浏览器的技术实现细节发生了很多变化，规范早期定义的 user-agent 的具体形态也随之调整。例如 Android 平台 Chrome 浏览器的 [Custom Tab](https://developer.chrome.com/multidevice/android/customtabs) 特性，提供了一种不离开应用（in-app）使用浏览器的方式，这种情景下 user-agent 虽然「嵌入」到应用内，却具有隔离的运行空间和上下文，应用无法访问 Tab 内的 Web 内容。因此术语「external」应当引申为具有独立隔离的运行、安全环境，而不局限于字面意义上的「外部」。

## Protocol Endpoints

先介绍 `endpoint` 概念:

>在 Web 服务领域，`endpoint`是代表该服务对外暴露的一个或多个可以接受消息的最终端点（endpoint），即可被调用者引用的入口、处理器或资源，使得来自外部的 Web 消息可以寻址到该最终端点。`endpoint` 传达了寻址 Web 服务所需的信息，客户端需要先了解此信息，然后才能访问服务。

一句话解释：Web 服务用 `endpoint` 对外描述自己提供的接口的信息，使得客户端能够参考这些信息决定使用哪个 `endpoint` 的接口来访问服务，Web 服务可以按业务逻辑需要划分一个或多个 `endpoint`

OAuth 2.0 也使用 `endpoint` 来描述不同角色的不同功能接口，整个 OAuth 2.0 授权流程涉及的 `endpoint` 包括：

### Authorization Server 的两个 Endpoint

- Authorization endpoint - 被 `client` 用来获取 `resource owner` 的授权，借助于 `user-agent` 的重定向机制。
- Token endpoint - 被 `client` 用来将（用户的）授权兑换为一个 `access token`，通常伴随着对 `client` 的认证。

### Client 的一个 Endpoint

- Redirection endpoint - 被 `authorization server` 用来返回包含授权凭证的响应，借助 `resource owner` 的 `user-agent`

这些 `endpoint` 在四种工作模式中会或全部或部分的参与使用

### Access Token Scope

`scope` 是在 `client` 发起 `authorization request` 时可选的参数，使 `client` 可以指定访问资源的范围，`authorization server` 在 acccess token 中响应的 `scope` 必须是以下情况之一

- 实际的 `scope`（如果认为 `client` 指定的范围太大）
- 默认的 `scope`
- `scope`非法

`scope` 的格式是  用空格拼接的一个或多个字符串，字符串内容由 `authorization server` 自行定义，OAuth 2.0 规范中不做约束。

更多关于 `Protocol Endpoints` 的介绍请参考 [rfc6749#section-3](https://tools.ietf.org/html/rfc6749#section-3)

## 四种授权模式及工作流

前文提到 `resource owner` 有四种授权模式（`grant type`）对 `client` 进行 `authorization grant`，对应的，有四种工作流程，本章逐一介绍。

拓展阅读：其实，OAuth 的 1.0 版本期望用全面的单一协议流程来囊括所有应用场景（OAuth 1.0 早期本来也是分离的三种流程，但后来各方讨论后合并为单一种），然而实践中发现单一流程虽能比较好地适用于基于 Web 的应用程序，但在其他方面却提供了较差的体验，而单一但复杂的流程也造成了集成困难和混乱。OAuth 2.0 版本，终于回归细分场景适配不同流程的策略。关于这部分内容，参见 [User Experience and Alternative Token Issuance Options](https://www.oauth.com/oauth2-servers/differences-between-oauth-1-2/user-experience-alternative-token-issuance-options/)。

### Authorization Code

此模式下 `resource owner` 以间接的方式完成授权许可：`client` 不直接请求 `resource owner`，而是将 `resource owner` 引导到 `authorization server`，由 `authorization server` 来处理授权请求。如果`resource owner` 批准了请求，`authorization server` 将反过来引导 `resource owner` 回到 `client`，并且伴随一个 `authorization code`。最终，`client` 凭借该 Code 请求 `authorization server` 来获取 Access Token。

`authorization server` 决定返回 `authorization code` 之前， 会对 `resource owner` 进行身份认证，并且该身份认证过程仅发生在 `authorization server` 处，因此其认证凭据任何时候都不会分享给 `client`。

同时，`authorization code` 还有一些安全上的裨益，例如 `authorization server` 能够认证 `client` （`client` 请求 `access token` 时）；`access token` 直接传递给 `client`，不会经由 `resource owner` 使用的 user-agent，从而避免 `access token` 暴露出去，包括暴露给 `resource owner`。

`authorization code` 模式是为私密型 `client` 定制的，流程如下图所示

<pre><font face="monospace">     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)

    注：由于要穿过 user-agent，步骤（A），（B）和（C）的线间断为两部分。

                  图 3：Authorization Code Flow
</font></pre>

图 3 描述的工作流程包含以下步骤：

  (A)  `client` 引导 `resource owner` 的 user-agent 到  `authorization server` 的 `authorization endpoint` 上。OAuth 2.0 定义的 `client` 包括其 `client` 标识，请求的 `scope`，`local state`，和一个一旦访问被批准时 `authorization server` 将 user-agent 重定向回去的 URI。

  (B)  `authorization server` 认证 `resource owner`（通过 user-agent）确定 `resource owner` 许可还是拒绝 `client` 的访问请求。

  (C)  如果 `resource owner` 许可访问，`authorization server` 使用之前提供的重定向 URI（请求内或 `client` 注册阶段）将 user-agent 重定向回 `client`，重定向的URI的参数包含 `authorization code` 以及`client`之前提供的 `local state`。

   (D)  `client` 使用上一步获取的 `authorization code` 向 `authorization server` 的 `token endpoint` 请求 `access token`。这个过程，`client` 会向 `authorization server` 发起身份认证。`client` 包含用于获取 `authorization code` 的重定向 URI，将用于验证。

  (E)  `authorization server` 认证 client，校验 `authorization code` 是否合法，并确保接收的重定向 URI 与步骤(C)  中用于重定向 `client` 的URI一致（在这里，URI将用于返回 `access token`，需确保 `authorization code` 被对应的合法 `client` 使用）。如无问题，`authorization server` 以 `access token` 以及可选的 `refresh token` 作为响应。

注：`(E)` 步骤，为了避免错误地使用颁发给其他 `client` 的 `authorization code`，`client` 必须发送 `client_id` 到 `authorization endpoint`，以避免 `authorization code` 冒用（但此举并不会给 `protected resource` 带来额外的安全性）。

authorization code 模式不适合 Public 型 Client 使用，但在 [OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252) 和 [Proof Key for Code Exchange by OAuth Public Clients (PKCE)](https://tools.ietf.org/html/rfc7636) 两个 `native application` 实践规范中提出了改进的使用 `authorization code` 授权模式方法。

### Implicit

注：在 OAuth 2.0 RFC 发布之时 `implicit` 模式其实是专为运行在浏览器上用 JavaScript 等前端语言实现的 `client` 而优化的，因为这种 `client` 如果使用 `authorization code` 模式将存在两个矛盾：

1. `client` 需要多次跳转（重定向）才能获取到 `access token`，响应慢，效率低。
2. 浏览器内应用（in-browser application）没有服务器后端承载业务功能，「前端先获取 `authorization code`，后端再换取 `token`」 没有意义：不仅不会带来安全增益，反而降低效率。

在 `implicit` 模式， `resource owner` 许可访问后， `client` 会被直接授予一个 `access token` 而不是“中间码”（`authorization code`）。这是和 `authorization code` 模式相比最大的区别。

<pre><font face="monospace">     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier     +---------------+
     |         -+----(A)-- & Redirection URI --->|               |
     |  User-   |                                | Authorization |
     |  Agent  -|----(B)-- User authenticates -->|     Server    |
     |          |                                |               |
     |          |<---(C)--- Redirection URI ----<|               |
     |          |          with Access Token     +---------------+
     |          |            in Fragment
     |          |                                +---------------+
     |          |----(D)--- Redirection URI ---->|   Web-Hosted  |
     |          |          without Fragment      |     Client    |
     |          |                                |    Resource   |
     |     (F)  |<---(E)------- Script ---------<|               |
     |          |                                +---------------+
     +-|--------+
       |    |
      (A)  (G) Access Token
       |    |
       ^    v
     +---------+
     |         |
     |  Client |
     |         |
     +---------+
     注：由于要穿过 user-agent，步骤（A）和（B）的线间断为两部分。

                       图 4: Implicit Grant Flow
</font></pre>

注：由于没有用 `authortization code` 交换 `token` 这一步骤，`client` 不和 `authortization server` 直接交互；`implicit` 模式下 `authorization server` 不认证  `client`，而是仅用重定向 URI 来标识 `client` 身份，因此确保 URI 的完整性是 `implicit` 模式的安全关键。

### Resource Owner Password Credentials

不同于 `authortization code` 和 `implicit` 授权模式在 `authorizaiton server` 处间接授权 `client`，`resource owner password credentials` 授权模式是 `resouce owner` 直接将自身的口令（例如用户名和口令)）作为授权凭据，让 `client` 获取 `access token`。显而易见，这种授权模式仅适用于 `resource owner` 和 `client` 之间存在有高度的信任关系（比如 `client` 是操作系统的一部分或高度特权的应用），且其他的授权许可类型不可用时，意味着 `auhtorization server` 需要谨慎支持这种模式。

尽管该授权模式需要 `client` 直接接触 `resource owner` 的凭据（意味者存在凭据泄露或身份被仿冒的可能），但实际上 `resource owner` 的凭据仅在请求时使用一次，最终还是会被转换成 `access token`。因此，password 模式可用于需要规避 `client` 为了后续使用而存储 `resouce owner` 凭据的场景（比如 `HTTP Basic`、`HTTP Digest` 认证），其核心安全收益在于将凭据替换成了长期的 `access token` 或 `refresh token`。

<pre><font face="monospace">     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          v
          |    Resource Owner
         (A) Password Credentials
          |
          v
     +---------+                                  +---------------+
     |         |>--(B)---- Resource Owner ------->|               |
     |         |         Password Credentials     | Authorization |
     | Client  |                                  |     Server    |
     |         |<--(C)---- Access Token ---------<|               |
     |         |    (w/ Optional Refresh Token)   |               |
     +---------+                                  +---------------+

           图 5: Resource Owner Password Credentials Flow
</font></pre>

图 5 描述的工作流程包含以下步骤：

  (A)  `resource owner` 向 `client` 提供自己的用户名和口令。

  (B)  `client` 用上一步获取的凭据向 `authorization server` 的 `token endpoint` 请求 `access token`。

  (C)  `authorization server` 认证 `client` 和 `resource owner` 的凭证，如果合法，颁发 `access token`。

### Client Credentials

`client` 可以仅使用其 `client credentials` （或其他支持的认证措施）就可请求 `access token`，这适用于被请求的 `protected resource` 属于 `client` 控制的场景（注解：代表 `client` 自己访问资源而用户），或是 `authorization server` 事先安排的、其他 `resource owner` 的` protected resource`。

`client credentials` 授权模式必须仅用于 `confidential` 类型的 `client`。（因为这种授权模型，`protected resource` 的安全性依赖于对 `client` 的认证的安全性，`public client` 显然是不妥的）

<pre><font face="monospace">     +---------+                                  +---------------+
     |         |                                  |               |
     |         |>--(A)- Client Authentication --->| Authorization |
     | Client  |                                  |     Server    |
     |         |<--(B)---- Access Token ---------<|               |
     |         |                                  |               |
     +---------+                                  +---------------+

                   图 6: Client Credentials Flow
</font></pre>

图 6 描述的工作流程包含以下步骤：

   (A)  `client` 向 `authorization server` 的 token endpoint 发起认证并请求一个 `access token`。

   (B)  `authorization server` 认证 `client`，如果合法，颁发一个 `access token`。

解读：`client credentials` 是四种授权模式中唯一不需要 `resource owner` 参与的，严格来说 `client credentials` 授权模式并不属于 OAuth 2.0 要解决的典型场景（用户向三方授权），但它提供了一种解决方案，使 OAuth 2.0 授权框架更完整，适用于更宽泛的场景。实际的应用例子可以参考 Keycloak 的 [Service Account](https://github.com/keycloak/keycloak-documentation/blob/master/server_admin/topics/clients/oidc/service-accounts.adoc)。

## 常见问题

### 如何理解 Redirection 和 Redirection URI？

在 `authorization code` 和 `implict` 授权模式，`redirection` 的目的和作用都是将 payload （credentials，例如 `authorization code` 和 `access token`）传递给 `redirection URI` 指定的实体。

在 `authorization code` 模式，有两次 `redirection`，一次是 `authorization server` 命令 user-agent 重定向到指定的 `client` URI ，来传递 `authorization code` 到 `authorization code` 授权模式的 `client`（一般在云端）；一次是 `client` 指定 `authorization server` 重定向到 `client` 提供的 `redirection URI`，来接收 `access token`。因此两次的 `redirection URI` 应当保持一致，且应是在预期的值，以确保安全。

在 `implict` 模式，只有一次 `redirection`，`authorization server` 命令 user-agent 重定向到指定的 `web-hosted client`  提供的 `redirection URI`（实践中可以是系统 `app-name://` 或 `localhost`），此时 user-agent 访问此 URI，但不会传递 `access token`（因为是 URI fragment 形式），`web-hosted client` URI 返回特定的 HTML/JS 页面，取出 `access token`，最终传递给 `implict` 模式的 `client`（一般在本地）。

### 为什么 Access token 之外还要引入 Authorization Code？

因为不希望授权（`token`）被暴露到不安全的环境。按照 OAuth 2.0 的模型，用户的认证是在 user-agent 处完成的，当实际使用 `access token` 的 `client` 和用户使用的 user-agent （如浏览器）之间存在安全保护等级差异时，那么就引入一个中间态授权码（`intermediate credentials` ，来源于 [rfc6749#section-1.3.2](https://tools.ietf.org/html/rfc6749#section-1.3.2)，在OAuth 2.0 中就是 `authorization code` ），使得 `resource owner` 所处的 user-agent 仅接触一次性的 `authorization code`，而永远不会接触到真正的授权 `access token`（即便是 `resource owner` 也不可见）。最终的 `access token` 则由更加可信的 `client`（例如 `confidential client`）保管，这在安全上就有意义：可以防止 `access token` 被非法使用（而 `authorization code` 仅一次有效，且需要合法 `client` 才能兑换为 `access token`）。例如：具有后端服务器 `web application` 或 `native application`，就可以使用 `authorization code` 模式来间接安全地获得 `access token`，而显而易见 `user-agent-based application` 所有实现都位于 user-agent（浏览器）内，不适合使用 `authorization code` 。

利用 `authorization server` 间接处理授权请求才是安全最佳的方式，才是完全符合 OAuth 2.0 将用户授权从`client` 中分离出来这一核心理念的工作流程。

### client 到底应该如何使用 access token 来请求资源？ resource server 又该如何校验 access token？

你可能注意到 RFC 6749 并没有定义到底如何使用 access token，包括 client 以什么请求规格来使用 access token，更重要的 resource server 如何校验 access token 合法性、识别 scope，实际上规范甚至连 access token 的详细规格都没有定义：只说是一个代表权限被授予给 client 的字符串（"a string representing an access
   authorization issued to the client"），而非身份凭据。事实上 RFC 6749 只是抽象地说明了工作流程，而 access token 的规格和使用细节则超出规范范畴。（当然，这些都有配套 RFC 来规定）

前一个问题在 Bearer Token RFC 6750 中定义，你可搜索本文查到。

至于另外一个问题，有两种不同的思路的办法：其中很自然就想到的一种是 token 本身只是索引，resource server 通过查询 authorization server 来获取存储在数据库中的 access token 关联的元信息，这在 [OAuth 2.0 Token Introspection - RFC 6742](https://tools.ietf.org/html/rfc7662) 中给出了详细的说明。

另一种则是本文提到的 self-container 即 token 自身包含结构化信息，典型实现是 JWT。在 RFC 7662 也提到了这一点。

### 和 OAuth 1.0的差别？

<https://www.oauth.com/oauth2-servers/differences-between-oauth-1-2/>

### 业界最佳实践？

<https://developers.google.com/identity/protocols/OAuth2>

### 简单说下 OAuth 2 

<https://aaronparecki.com/oauth-2-simplified/#single-page-apps>（仅供参考）

### 什么时候用OAuth 2.0？

<https://stackoverflow.com/questions/40956418/is-oauth2-only-used-when-there-is-a-third-party-authorization>

### 为什么OAuth 2 在 Access Token 之外还要引入 Refresh Token？

<https://stackoverflow.com/questions/3487991/why-does-oauth-v2-have-both-access-and-refresh-tokens>

### OAuth和 Open ID，认证和授权，到底什么区别？

参阅「Access Token 可以代表用户（认证）吗？」问题，正因为 `access token` 无法用于认证用户，而又有引入中间层（用户不直接在 `client` 处认证）的需求，因此诞生了 `Open ID`，使得 `client` 可以通过信任的另外的一个 `authentication server` 来认证某个用户：用户在这个 `authentication server`上使用凭据进行认证，然后 `authentication server` 告诉 `client ` 用户真实的身份。

参考 [Difference Between OAUTH, OpenID and OPENID Connect in very simple term?](https://security.stackexchange.com/questions/44611/difference-between-oauth-openid-and-openid-connect-in-very-simple-term)

### Access Token 可以代表用户（认证）吗？

`access token` 表征用户授权给特定的 `client`，而 `client` 用 `access token` 来在 `resource server` 处认证自己。换言之，如果 `access token` 用于「认证」，应当只能用于认证 `client` 自己，而不能用于认证用户。

一个可以证明将 `access token` 用来认证用户是错误的例子：假设 A、B 两个网站都依靠 C 网站的 `access token` 来认证用户，它们的做法是：如果能获取到访问 C 网站上用户 ID 的权限（`access token`），就用这个 ID 来认证用户。那么 A 网站就可以使用一个合法的 `access token` 在 B 网站上仿冒某个用户了，反之亦然。问题的根源在于网站们错误地将 `access token` 用于认证用户。这是对OAuth 2.0 的一种典型误用。

上述的回答还没有根本地回答问题，即为什么 `client` 从 `access token` 中无法获取用户的身份？仔细看 RFC6749 就可以知道，OAuth 2.0 规范并没有定义 `access token` 的规格，包括 `client` 如何解析 `access token` 等等，甚至规范认为 `access token` 作为字符串，对于 `client` 是不透明且没有语义的，`client` 只是拿着 `access token` 去访问资源，并只在 `resource server` 处产生语义（`scope` 等）。

参考 [rfc6749#section-10.16](https://tools.ietf.org/html/rfc6749#section-10.16)

### Access Token 仅代表权限，那么它如何和用户关联起来呢？换言之，一个 Access Token 的 Scope 能访问其他用户的资源么？

参考 [How can a OAuth2 resource server relate an access token to the user that authorized it to prevent unauthorized access to other user resources?](https://security.stackexchange.com/questions/199120/how-can-a-oauth2-resource-server-relate-an-access-token-to-the-user-that-authori/199178) 和 [rfc7662](https://tools.ietf.org/html/rfc7662)

### Code 模式下 Web Application 类型的 Client 将 Access Token 放置在服务器端使用，因为认为 user-agent 所在的环境不安全。但是 user-agent 和服务器端之间本身是有一层认证授权凭据（比如会话），如果这些信息容易泄露， 恶意 user-agent 同样可以使用它来操作 Access Token，那么把 Access Token 放在服务器端使用有什么安全意思呢，放置在 user-agent 处不是一样的吗？

`access token` 是 `resource owner` 的原始授权信息，而会话仅代表和 `web application` 之间建立的认证授权关系，截获了会话并不意味着能够任意操作 `access token`（必须通过有限的接口操作），这和原始 `access token` 泄露是不同的。
