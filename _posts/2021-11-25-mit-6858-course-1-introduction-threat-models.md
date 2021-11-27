---
layout: post
title: MIT 6.858 Course 1 - Introduction, Threat Models 
categories: [MIT 6.858, Course]
---

## 安全分为 Policy、Threat Model、Mechanism 三部分。

Policy 错误例子：Gmail 的密码重置需要辅助邮箱，辅助邮箱提供商的邮箱重置有问题，导致Gmail 不行。

Threat Model：对攻击者的假设（assumption）。比如对于web服务器，假设就是攻击者无法物理接触，但可以发送任何规格的数据包。错误例子：在 80 年代 Mit Kerberos 认为 DES  56bit 就是安全的，但是这个假设随着时间很快改变了，还有就是假设 SSL 的所有 CA 都是安全的，然而 CA 上百家，各个国家（中国、印度等等都有）很容易签发任意域名的证书。

Mechanism : 让政策和假设得到遵循。错误的例子（相比于 Mechnism 和 Policy 特定于场景，较难分析，机制错误相对容易）：

- 比如 Apple 的防暴力破解，在 Find my iphone 这个接口就忘记部署了；
- citi bank 的某个 web 接口没有认证，直接输入 id 就能取到信息；
- Android 比特币使用 SecureRandom()，然而这个方法其实只是用一个真随机数导入 PRNG 来持续生成随机数，这种方式是 OK 的，但是该 Java 库有 bug，在某些场景忘记使用随机种子，而使用全 0  的种子，这样 PRNG 生成的随机数就能被预测测
- 浏览器用 c 语言节码 SSL 证书，在读取 string 时以 \0 中断，这样可以构造 amazon.com\0.evil.com 的证书来欺骗。

## 缓冲区溢出基础原理

课程后半讲了一个最基础的缓冲区溢出实例，并实操演示，是下节课的准备。