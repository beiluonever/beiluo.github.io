---
title: 记录一次去除SpringSecurity BasicAuthorizationFilter的过程
date: 2019-11-14 09:18:50
tags:
- SpringSecurity
- BasicAuthorizationFilter
- OAuth2
categories: 
- Java
typora-root-url: ..\images
---
## 现象描述
前提：一个使用SpringBoot1.5.6+OAth2 (2.0版本)验证的权限系统
现象：在登录过程中，输入错误的用户名会导致弹出www basic的验证框，如下图

![alert-baic-auth](https://blog.ohmyssh.top/images/alert-baic-auth.png)

