---
title: PKI基础知识介绍-第一篇
date: 2019-11-08 10:29:10
categories: 
- PKI
tags:
- PKI
- CA
---

> A **public key infrastructure** (**PKI**) is a set of roles, policies, and procedures needed to create, manage, distribute, use, store & revoke [digital certificates](https://en.wikipedia.org/wiki/Digital_certificates "Digital certificates") and manage public-key encryption. The purpose of a PKI is to facilitate the secure electronic transfer of information for a range of network activities such as e-commerce, internet banking and confidential email. It is required for activities where simple passwords are an inadequate authentication method and more rigorous proof is required to confirm the identity of the parties involved in the communication and to validate the information being transferred.
> 一个**公共密钥基础设施**（**PKI**）是一组角色，策略，以及创建，管理，分发所需的程序，使用，储存和吊销的[数字证书](https://en.wikipedia.org/wiki/Digital_certificates "数字证书")和管理公共密钥加密。PKI的目的是促进电子商务，网上银行和机密电子邮件等一系列网络活动的信息安全电子传输。对于简单密码认证方法不充分的活动，需要更严格的证据来确认通信中涉及的各方的身份并验证所传输的信息。
	
上文为WIKI对于PKI的解释，简单来说，PKI是一个基础设施，使用**数字证书、密钥加密等措施来保障通信中的信息安全和身份安全。**从我个人浅显的理解来说，PKI就是解决如何证明我是我、如何保证我传给A的东西其他人看不到只有A能看，而且A还能确信这是我发的。

# PKI由什么组成

### 设备的派出所——CA

> 在[加密中](https://en.wikipedia.org/wiki/Cryptography "加密")，**证书颁发机构**或**证书颁发机构**（**CA**）是[颁发数字证书](https://en.wikipedia.org/wiki/Public_key_certificate "公钥证书")的实体。数字证书通过证书的指定主题来证明公钥的所有权。这允许其他人（依赖方）依赖[签名](https://en.wikipedia.org/wiki/Digital_signature "电子签名")或关于与经认证的公钥对应的私钥的断言。CA充当[受信任的第三方 -](https://en.wikipedia.org/wiki/Trusted_third_party "值得信赖的第三方")由证书的主体（所有者）和依赖证书的一方进行信任。这些证书的格式由[X.509](https://en.wikipedia.org/wiki/X.509 "X.509")标准指定。

和PKI的定义一样，它是一个基础设施，为了解决我是我的问题，引入了数字证书的概念，数字证书包括有关密钥的信息，有关其所有者身份的信息（称为主题），以及已验证证书内容的实体（称为发行者）的[数字签名](https://en.wikipedia.org/wiki/Digital_signature "电子签名")，在我看来这就相当于现实生活中的派出所，给你颁发了一个身份证，上面有你的个人信息（所有者身份的信息），也标上是哪个派出所发的，并且做了防伪（发行者的数字签名），这样在后续和其他设备交互的时候，你就可以使用数字证书去证明你是你了。

## 设备的户籍查询窗口——RA

> 确保有效和正确注册的PKI角色称为注册机构（RA）。RA负责接受对数字证书的请求并对发出请求的实体进行身份验证。

说是户籍查询窗口并不准确，在拥有RA的PKI系统中，RA作为一个和外界交互的主要通道，负责证书的申请、审批等。在*PKI体系中也可以没有RA部分，直接使用SCEP进行申请服务（仅代表我所遇到的情况，如有补充欢迎评论）*

## 记录着不可用证书的派出所——VA

> 在[公钥基础结构中](https://en.wikipedia.org/wiki/Public_key_infrastructure "公钥基础设施")，**验证机构**（**VA**）是一个实体，它根据[X.509](https://en.wikipedia.org/wiki/X.509 "X.509")标准和[RFC ](https://en.wikipedia.org/wiki/Request_for_Comments_(identifier) "征求意见（标识符）")[5280](https://tools.ietf.org/html/rfc5280)（第69页）中描述的机制提供用于验证[数字证书](https://en.wikipedia.org/wiki/Public_key_certificate "公钥证书")有效性的服务。<sup>[[1]](https://en.wikipedia.org/wiki/Validation_authority#cite_note-1)</sup>[](https://en.wikipedia.org/wiki/X.509 "X.509")[](https://en.wikipedia.org/wiki/Request_for_Comments_(identifier) "征求意见（标识符）") [](https://tools.ietf.org/html/rfc5280)<sup>[](https://en.wikipedia.org/wiki/Validation_authority#cite_note-1)</sup>
> 用于此目的的主要方法是托管[证书吊销列表，](https://en.wikipedia.org/wiki/Certificate_revocation_list "证书撤销清单")以便通过[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol "Hypertext Transfer Protocol")或[LDAP](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol "轻量级目录访问协议")协议下载。为了减少证书验证所需的网络流量，可以使用[OCSP](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol "在线证书状态协议")协议。
> 虽然验证机构能够响应基于网络的CRL请求，但它缺乏颁发或吊销证书的能力。必须使用颁发CRL中包含的证书的[证书颁发机构的](https://en.wikipedia.org/wiki/Certificate_authority "证书颁发机构")当前CRL信息不断更新它。
> 虽然这是一个潜在的劳动密集型过程，但使用专用的验证机构可以动态验证[脱机根证书颁发机构](https://en.wikipedia.org/wiki/Offline_root_certificate_authority "脱机根证书颁发机构")颁发的[证书](https://en.wikipedia.org/wiki/Offline_root_certificate_authority "脱机根证书颁发机构")。虽然根CA本身对网络流量不可用，但始终可以通过验证机构和上述协议验证由其颁发的证书。
> 维护验证机构托管的CRL的持续管理开销通常很小，因为根CA颁发（或撤销）大量证书的情况并不常见。

这部分我了解不多，现阶段我所遇到的场景更多为**内置脱机根证书**并且使用OCSP方式查询，简单来说，VA就是为了避免出现：你的身份证被派出所取消了，我还傻傻的相信你这一情况。

本章主要是介绍PKI是什么，解决了什么问题，有哪些组成部分。后续会继续更新PKI相关的一些文章，比如SCEP是什么，OCSP，CRL又是什么，以及PKI相关技术在JAVA中的实现。