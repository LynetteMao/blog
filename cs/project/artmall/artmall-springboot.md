---
title: artmall:multi module sping boot搭建
date: 2018-09-06 21:22:37
tags: artmall
categories: project
---

1、artmall pom包的创建

![1](http://prdj7tprx.bkt.clouddn.com/1.png)
<!-- more -->

2、创建module，创建maven module（创建四个module：artmall-common(jar)、artmall-domain(jar)、artmall-manager(jar)、artmall-web(jar)）

![2](http://prdj7tprx.bkt.clouddn.com/2.png)
```xml
    <!--armall.pom-->
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
    
        <groupId>com.artmall</groupId>
        <artifactId>artmall</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <modules>
            <module>artmall-common</module>
            <module>artmall-manager</module>
            <module>artmall-web</module>
            <module>artmall-domain</module>
        </modules>
    		<!--添加打成什么包,除了这一个，其他都是自动生成的，其他的pom同理，只不过打成jar包-->
        <packaging>pom</packaging>   
    
        <name>artmall</name>
        <description>Demo project for Spring Boot</description>
    
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.0.4.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
```

3、包结构

全部文件要放在同一个包下，不然ArtMallWebApplication.java无法扫描到包

![3](http://prdj7tprx.bkt.clouddn.com/3.png)

![4](http://prdj7tprx.bkt.clouddn.com/4.png)

4、依赖结构

这个项目里面的依赖结构是，artmall-commom、artmall-domain独立，artmall-manager依赖artmall-common、artmall-domain，artmall-web依赖artmall-commom、artmall-domain、artmall-manager。所以要再artmall-manager和artmall-web里面添加依赖.
```xml
    		<dependency>
                <groupId>com.artmall</groupId>
                <artifactId>artmall-common</artifactId>
                <version>0.0.1-SNAPSHOT</version>
            </dependency>
    
            <dependency>
                <groupId>com.artmall</groupId>
                <artifactId>artmall-domain</artifactId>
                <version>0.0.1-SNAPSHOT</version>
            </dependency>
```

5、导入基础包，就是各种启动器，按照自己要的来配置吧，这里不详细说了。

6、发现一个很有用的东西
![5](http://prdj7tprx.bkt.clouddn.com/5.png)

7、测试一下，能不能启动哦，就在controller里面写个hello world就行了！！！