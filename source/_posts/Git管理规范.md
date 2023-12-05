---
title: Git管理规范
date: 2023-12-05 08:55:43
tags:
categories:
---

## 分支管理

**master/main**  分支为开发稳定分支，用来合并多人的请求，可以稳定部署在 dev 环境中，需要提交 merge request 进行合并分支，同时应当组织 code review；
**feature-_/dev-_** 这类分支为开发分支，从 master checkout 出来，在部署 dev 环境之前，建议 merge master 到自己开发分支，本地自测，保证不影响其他人开发。模块功能开发完成后应该合并进 master 分支中。
**release 分支**    制品仓库中所有制品应该来源于 release 仓库，在 master 测试过，bug 修改完成后，合并进 release 分支。对于 master 后续小的修改或者进 master 没有 review 的需要进行 review，并且只有部分人员有 merge 权限。可以直接部署到线上环境中。
**hotfix-yyyymmdd**  分支    线上环境紧急 bug 修复版本，分支从 release 或者 tag 中 checkout 出来，进行的修改需要同步 merge 进入 master/release 分支。master  分支使用 cherrypick 从 hotfix 中拿取指定的 commit。

**tag:**  对于每次版本发布进行 tag 标记，merge 进 release 时，应当自动 tag，记录为版本号

**开发流程示例：**

1. 接到新需求，从最新的 master  分支上 checkout 出新的分支，命名为 feature-xx
2. 【复杂功能】功能设计 review 后再进行开发。
3. 开发功能，自测。
4. 【可选】自测完成后，可以 merge master 最新的 code，本地验证不影响主要功能后部署到 dev 环境和前端联调
5. 联调成功后，提交 merge request 进行 code 合并，线上或者线下 review。
6. 在同一个版本所有人功能都测试完成后，合并到 master 后，进行最后的联调测试
7. 出现 bug，从最新的 master 上 checkout 新分支例如 feature-xx-bugfix ，修改问题，部署测试
8. bug 全部修复后，从 master 上提交 merge request 到 release 分支，同时打 tag 记录版本。部署 prod 环境，进行 prod 测试。
9. 产线运行出现了影响功能的紧急 bug 后，从 release 分支 checkout Hotfix-yyyymmdd 格式的分支，提交代码进行测试
10. 分支测试完成后，确保已经修复并且不影响其他功能后。将修改的 commit cherry pick 到 master 分支上，并提交 merge request 到 release 分支。

推荐设置：

```shell
git config --global pull.rebase true
```

并且使用 mr 的 fastforward 模式，保证 log 中没有 commit  节点，方便后续问题排查。

## commit 要求

核心要求：

1. 一次提交的内容尽可能单一，能够用一两句话描述清楚，养成功能完整提交的习惯。
2. 不要提交无关的或者本地配置文件，可以使用  idea 的多个 Local Changes  或者  git stash/git stash apply  将需要提交和本地修改隔离

commit message 供参考：
<type>(<scope>): <subject>

**type(必须)**
用于说明 git commit 的类别，只允许使用下面的标识。

feat：新功能（feature）
fix/to：修复 bug，可以是 QA 发现的 BUG，也可以是研发自己发现的 BUG。 - fix：产生 diff 并自动修复此问题。适合于一次提交直接修复问题 - to：只产生 diff 不自动修复此问题。适合于多次提交。最终修复问题提交时使用 fix
docs：文档（documentation）。
style：格式（不影响代码运行的变动）。
refactor：重构（即不是新增功能，也不是修改 bug 的代码变动）。
perf：优化相关，比如提升性能、体验。
test：增加测试。
chore：构建过程或辅助工具的变动。
revert：回滚到上一个版本。
merge：代码合并。
sync：同步主线或分支的 Bug。
scope(可选)
scope 用于说明  commit  影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。
例如在 Angular，可以是 location，browser，compile，compile，rootScope， ngHref，ngClick，ngView 等。如果你的修改影响了不止一个 scope，你可以使用\*代替。

subject(必须)
subject 是 commit 目的的简短描述，不超过 50 个字符。
建议使用中文（感觉中国人用中文描述问题能更清楚一些）。 - 结尾不加句号或其他标点符号。 - 根据以上规范 git commit message 将是如下的格式：

```
fix(DAO):用户查询缺少username属性
feat(Controller):用户查询接口开发
```

## 环境说明

**dev**：稳定开发环境，可以提供给前端联调，项目内测试功能，部署的应该是包括 master 中所有功能的代码版本。根据实际情况考虑是否需要多个 dev 环境
**test**：从 master 分支部署的代码，能够提测的版本，目前由开发人员自测。
**prod**：从 release 部署的生产环境，不能随意部署，需要服务可用性监控，稳定为最优先

**推荐阅读：代码开发规范**

Java 参考阿里巴巴规范，安装插件：[阿里巴巴 Java 规范](https://developer.aliyun.com/special/tech-java) [安装插件](https://developer.aliyun.com/article/224817)

Vue 参考： [Vue 规范](https://v2.cn.vuejs.org/v2/style-guide/index.html)

附录：
[Git 练习网站](https://learngitbranching.js.org/?tdsourcetag=s_pctim_aiomsg&locale=zh_CN)
