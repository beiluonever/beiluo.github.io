---
title: Maven常用命令
date: 2019-12-16 15:04:48
tags:
- Maven
categories:
- Maven
---
## 基础操作
maven的生命周期操作：
maven clean package install 等，后续可能会补充一下

## 稍微复杂操作
-DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。

-Dmaven.test.skip=true 不执行测试用例，也不编译测试用例类。

mvn -pl moduleA -am install 多模块项目时，打包指定一个moduleA，会自动打包这个模块所依赖的模块