---
title: SpringCloud Zuul网关在Filter中添加或修改Request参数
date: 2019-11-21 09:16:25
tags:
- SpringCloud
- Zuul
- SpringBoot1.X
categories:
- Java
- SpringCloud
---
## SpringCloud Zuul网关在Filter中添加或修改Request参数
### 使用目的
在网关中经常会处理一些Token或者鉴权信息，在处理完后，将Token处理后，把用户信息放到Request中，方便后续系统的时候。我这次使用的场景有些特别，由于项目需要兼容用户使用RequestBody 传递JSON参数，但是由于当前系统没有对RequestBody做兼容，导致数据获取不到，因此在Zuul中把参数处理完成后传递给后面的应用。
### 使用环境
```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.6.RELEASE</version>
        <relativePath/>
    </parent>
	
	<spring-cloud.version>Dalston.SR2</spring-cloud.version>
```
<!-- more -->

### 实现方式
1. 在RequestContext中添加QueryParams
```.java
    /**
     * 设置参数给后续服务使用
     * @param reqJson 请求的RequestBody参数
     */
    private void setRequestContextParams(JSONObject reqJson) {
        //设置参数 给后面的请求使用
        RequestContext ctx = RequestContext.getCurrentContext();
        Map<String, List<String>> params = new HashMap<>(16);

        String strategyId =  reqJson.getString(Constants.STRATEGY_ID);

        params.put(Constants.SIGN, Collections.singletonList(sign));
        params.put(Constants.CLIENT_ID, Collections.singletonList(clientId));
        params.put(Constants.AUTHORIZATION, Collections.singletonList(userId));
        params.put(Constants.STRATEGY_ID,Collections.singletonList(strategyId));
        ctx.setRequestQueryParams(params);
    }
```
2. 使用RequestContext的setRequest()使用RequestWrapper将当前的Request更新掉
	这个没有编写代码，大概流程如下
	1. 编写一个类继承HttpServletRequestWrapper(request) 
	2. 重写其中的方法，将自己想要的实现放置进去
	3. 设置到RequestContext中去

参考博客链接：
[springcloud 中 zuul 如何修改请求参数](https://blog.csdn.net/kysmkj/article/details/79092781)
[springcloud zuul重写/修改请求参数](http://www.wekri.com/2018/07/18/springcloud/springCloudZuul-rewrite-request-parameters/)