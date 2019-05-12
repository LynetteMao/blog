---
title: spring boot打包为jar包部署关于静态资源的处理
date: 2018-10-23 18:35:07
tags: artmall;问题解决
categories: project
---

## 问题描述

spring boot打包为jar包，部署至linux服务器，自动创建了一个和jar包同级的image文件用来存放上传的图片。

图片和文件可以上传至此文件夹，但是当前端调用API获取到要展示图片的url时，却通过这个url无法获取到图片，一开始是被拦截下来，没有权限，当我对权限进行开放后，显示的404.

## 基础知识

- 静态资源处理


早期的spring mvc对静态资源的请求需要在DispatcherServlet的请求映射中，往往要加上`*.do和*.xhtml`等方式，这就决定了请求URL必须是一个带后缀的URL。

现在处理方式是：

允许静态资源放在任何地方，通过location属性指定静态资源位置，由于location属性是Resource属性，因此可以使用`classpath:`等资源前缀的指定资源位置。

详细基础知识:https://blog.csdn.net/xichenguan/article/details/52794862?utm_source=blogxgwz0

- spring boot对静态资源的处理

sping boot进行了自动配置，我们打开`WebMvcAutoConfiguration`

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                logger.debug("Default resource handling disabled");
            } else {
                Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
                CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
                if (!registry.hasMappingForPattern("/webjars/**")) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

                String staticPathPattern = this.mvcProperties.getStaticPathPattern();
                if (!registry.hasMappingForPattern(staticPathPattern)) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations())).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

            }
        }
```





## 实现

### 配置

因为我的url映射是`/image/**`，然后映射到文件夹`file:/root/artmall/image/`,一开始使用的是`classpath:/image/`发现映射不了，就使用的绝对路径来进行映射，暂时还没找到解决方法。

实现其实就一步，重写addResourceHandlers

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Override
    public void  addResourceHandlers(ResourceHandlerRegistry registry) {
        registry
                .addResourceHandler("/image/**")
                .addResourceLocations("file:/root/artmall/image/");
// /root/artmall/image/*
    }

}
```

## 问题

- 问题一：网上的资料大部分都是继承WebMvcConfigurerAdapter，但是我这显示的是

```
'org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter' is deprecated less... (Ctrl+F1) 
This inspection reports where deprecated code is used in the specified inspection scope.
```

通过查阅api发现,可以通过直接implemented WebMvcConfigurer就行了

> as of 5.0 [`WebMvcConfigurer`](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html) has default methods (made possible by a Java 8 baseline) and can be implemented directly without the need for this adapter

- 问题二：前面整合了shiro，每次发送静态资源的请求的时候都会被拦截下来

  解决方案：在shiro的拦截链上对这个url进行放行

  ```java
          filterRuleMap.put("/*/login","anon");
  
          filterRuleMap.put("/business/register/**","anon");
          filterRuleMap.put("/show/**","anon");
          filterRuleMap.put("/image/**","anon");
          filterRuleMap.put("401","anon");
          filterRuleMap.put("/**","jwt");
  ```


### 测试

配置完了之后，我们可以在启动程序中发现，已经配置好了

```java
2018-10-24 09:18:53.670  INFO 10648 --- [  restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2018-10-24 09:18:53.670  INFO 10648 --- [  restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2018-10-24 09:18:53.670  INFO 10648 --- [  restartedMain] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/image/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]

```

启动服务器，在浏览器上输入url，即可以显示图片！！！