---
title: SpringBoot项目中解决上传文件乱码
date: 2019-11-12 11:07:14
tags: 
- SpringBoot
- UploadFileEncode
- 上传文件乱码
categories: 
- Java
---

## SpringBoot项目中MultipartFile接受数据文件乱码原因

MultipartFile 在接收数据时会以系统默认的编码解码上传数据，如果系统默认的编码和页面编码不一致就会乱码。乱码其实就是当接收方和发送方的编码不一致就会出现。

## 单个SpringBoot项目解决方案

1. 修改配置文件

   ```.yml
   spring:
     http:
       encoding:
         charset: UTF-8
         force: true
         enabled: true
   server:
     tomcat:
       uri-encoding: UTF-8
   ```

   

2. 添加配置类

   ```.java
   @Configuration
   public class ConfigurerAdapter extends WebMvcConfigurerAdapter {
   
       @Bean
       public HttpMessageConverter<String> responseBodyConverter() {
           StringHttpMessageConverter converter = new StringHttpMessageConverter(
                   Charset.forName("UTF-8"));
           return converter;
       }
   
       @Override
       public void configureMessageConverters(
               List<HttpMessageConverter<?>> converters) {
           super.configureMessageConverters(converters);
           converters.add(responseBodyConverter());
       }
   
       @Override
       public void configureContentNegotiation(
               ContentNegotiationConfigurer configurer) {
           configurer.favorPathExtension(false);
       }
   }
   ```

   

3. 获取文件名时添加编码转换

## SpringCloud（使用Zuul做网关项目)解决方案
在使用上面几步都解决不了的情况下，你最需要考虑下你当前项目的请求流程，此处提供对zuul组件上传文件乱码的解决方案。
官网说明：
> 18.8 Uploading Files through Zuul
If you use @EnableZuulProxy, you can use the proxy paths to upload files and it should work, so long as the files are small. For large files there is an alternative path that bypasses the Spring DispatcherServlet (to avoid multipart processing) in "/zuul/*". In other words, if you have zuul.routes.customers=/customers/**, then you can POST large files to /zuul/customers/*. The servlet path is externalized via zuul.servletPath. If the proxy route takes you through a Ribbon load balancer, extremely large files also require elevated timeout settings, as shown in the following example:
```
application.yml

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 60000
ribbon:
  ConnectTimeout: 3000
  ReadTimeout: 60000
```
>Note that, for streaming to work with large files, you need to use chunked encoding in the request (which some browsers do not do by default), as shown in the following example:
```
$ curl -v -H "Transfer-Encoding: chunked" \
    -F "file=@mylarge.iso" localhost:9999/zuul/simple/file
```
[zuul uploading files](https://cloud.spring.io/spring-cloud-static/Greenwich.SR3/multi/multi__router_and_filter_zuul.html#_uploading_files_through_zuul)

```.java
/**
 * Path to install Zuul as a servlet (not part of Spring MVC). The servlet is more
 * memory efficient for requests with large bodies, e.g. file uploads.
 */
private String servletPath = "/zuul";
```

按照官网说明，可以将请求发送到/zuul/项目原转发路径/*，但是一般项目后期，如果要修改请求路径较为复杂。这里可以将所有请求都走此过滤。设置如下：

```.yaml
zuul:
	servlet-path: /
```

