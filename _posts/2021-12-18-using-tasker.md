---
layout: post
title: 使用 Tasker
categories: [Tutorial, Tasker]
---

## 介绍

Tasker 是一款功能强大的 Android 自动化 APP。本文记录了 Tasker 常用基础知识以及一些自动化场景，文章内容后续会不断更新。

## Tasker 基础

### Tasker 基本组件

- Project：将 Tasker profiles, tasks, scenes 和 variables（全局变量） 进行逻辑分组，以方便管理。可以给 Project 设置直观的图标。
- Profile：触发 Task 的环境上下文，比如时间、地点、事件、应用等等，必须关联到 Task。
- Task：Task 代表一个具体的任务，由一至多个具体的 Action 组成，Task 可类比于函数，Action 则是组成函数的一行行代码（具体指令）。
- Action：Tasker 任务的最小执行单位，包括通知、文件读写、HTTP 请求、系统配置甚至第三方插件等等。值得注意 If/Else 等条件分支语句也是一类 Action。丰富的 Action 是 Tasker 强大功能的基石。

### 变量

Tasker 用 「%变量名」表示一个变量，其中

- 全局变量：变量名含一个或多个大写字母，比如 %Car，%SMS_BODY，作用域是整个 Tasker 程序。在 Tasker 主页 - VARS 页签定义全局变量。
- 局部变量：全部为小写字母，作用域是 Task。在 Task 内使用 Variable Set Action 来定义。

### 变量处理

#### 字符串匹配和提取

按一定的模式处理字符串变量，用来提取字符串中的关键词，支持正则表达式。

- [simple match/regex 简单匹配/正则](https://tasker.joaoapps.com/userguide/en/help/ah_match_regex.html)

简单匹配例子：

```
my name is $name, #old years old
```

上面的公式 $ 表示任意字符串 # 表示任意数字，匹配的结果将生成对应的 %name 和 %old 两个局部变量，用于后续访问。因为一个 pattern 可能匹配多次，可以用 name(数字)  来访问指定第 n 个匹配结果。

正则表达式例子：

```
(?<variable>reg) 
```

上面的例子使用命名捕获组，匹配结果将会存在 %variable 变量

- search and replace

比较少用，容易误用：

1. 不支持正则表达式的捕获组，比如表达式  (group1)(group2) ，实际会忽略 `()`；
2. `Store Matches In Array` 的意思是将捕获的结果存储在数组变量中，ret1 是访问匹配到的第一个结果，而不是正则表达式中的捕获组 1。

### 变量结构化解析

对于结构化的变量，比如来自 web 的 HMTL、JSON 响应，如果要解析内容，办法之一是用前面提到的模式匹配，但存在不直观、仅适用于简单情形的问题。

新版本 Tasker 开始支持 JSON 等结构化变量解析，语法十分简单，直接参考[官方文档](https://tasker.joaoapps.com/userguide/en/variables.html#json)即可。

### Task 参数和返回值

前文提到 Task 类比于函数，函数有入参和返回值，Tasker  Task 之间可以互相调用，同样也有参数和返回值。这套机制使得 Task 功能更多样以及更容易复用。

Tasker 为每一个 Task 隐式创建了 %part1、%par2 两个局部变量，用来保存该 Task 的入参，调用者 Task 通过 Perform Task Action 来调用被调 Task 时，可以指定是否传递参数，以及指定用于接收返回值的变量，而被调用 Task 则一方面直接通过 %part1、%part2 使用入参，另一方面通过 Return Action 来传递回返回值。

具体做法：

## Tasker 应用场景

### HTTP 请求

Tasker 的 HTTP Request Action 可以用来实现自动签到等功能，关于 HTTP Request 总结如下：

- 该 Action 用于发起 HTTP 请求，可定制 Method（GET/POST）、Headers 等参数
- 执行后会输出若干变量，其中最重要的是用于存放请求响应的 %http_data。
- Task 在处理响应时候，可以用[变量处理](#变量处理)的若干方法来提取有用的信息，特别是新版的结构化变量读取功能，可以方便地解析 JSON/HTML 等格式响应信息。注意：变量结构化解析依赖 HTTP Request 打开 `Structure Output(JSON, etc)` 选项。
- 如果需要 Tasker 管理 Cookie，而不是请求参数中手动设置，勾选 `User Cookies` 选项。

### Wake On LAN

自动化 Wake On LAN 是一个非常实用的场景，指定手机打开特定应用时自动唤醒 NAS、PC 等局域网设备。

Tasker 本体并不支持 WOL，但调用插件/三方应用来扩展实现，推荐用 Google Play 的 [Wake On Lan](https://play.google.com/store/apps/details?id=co.uk.mrwebb.wakeonlan)。具体方法是：先配置好可用的 WOL 目标，然后进入 Tasker 创建关联的 WOL Task 调用（Plugin Action），最后新建一个 `Event - Application` Profile，将 Profile 绑定到 WOL task 即可。如果不想每次启动应用都触发 WOL，可以长按 Profile，设置一下 Cooldown Time 参数。

### 自动化 Shadowsocks 连接

对于家里有 Shadowsocks 透明代理环境的，回家时一般会手动关闭手机上的 Shadowoscks，出门时又要手动开启。这一场景可通过 Tasker 可以实现自动化：根据目标 WiF 连接状态i，来开启关闭手机上的 Shadowsocks 连接。具体过程：

1. 分别创建两个 Shadowsocks Plugin Action Task，其中一个 Task 开启 Shadowsocks，另一个是关闭。
2. 新建一个  State  Profile，选择 Net - Wifi Connected，其中 SSID 填写 WiFi 的 SSID，Active 选择 Any。
3. 将 Profile 分别关联到先前创建的两个 Action，其中关闭 Shadowsocks 作为 Exit Task 即可