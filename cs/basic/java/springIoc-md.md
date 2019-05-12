---
title: IOC容器
date: 2018-07-31 17:06:49
tags: java
categories: basic
---

开始的项目是使用ssm结构，就涉及到了很多关于Spring的知识，但是一直都是在谷歌上查找零碎的资料，也就只知道大概怎么用，然而还是不理解其原理及实现。最近实习的时候，要我们开始学习SpringBoot，为了理解更为透彻，我就找了一本书《精通Spring 4.x企业应用开发实战》，不得不说还是看书这种更为体系的学习更适合我。以下就是学习IoC容器后的总结。   
<!-- more -->
## 基础知识   
没有理解反射机制，就很难理解IoC，万丈高楼平地起！
### 类装载器（ClassLoader）   
   这个涉及到将类装载到虚拟机上，为了防止黑客的攻击，JVM装载类的时候使用的是“双亲委派模型”，也就是说，当底层的ClassLoader要装载的类了，先查看自己的命名空间里有没有这个类。如果没有就去找到这个类，但是他不能直接装载，他要打个电话问一下他的上级，一个接一个的问上级（loader2问loader1,loader1问系统类加载器，系统类加载器问根类加载器），上级说他加载不了，loader1的ClassLoader找到了这个类，上级说我加载不了，你来加载吧，于是loader1
   加载Sample类，并返回一个引用给loader2。   
   >"你不知道，说来话长， 我们之前出现过事故，有个黑客写了个类java.lang.String,  和我们老板手下有一个干活最卖力的员工名字一模一样，只是这个黑客类里边竟然有格式化硬盘的代码，我们的小兵Classloader 不明就里，就把这个黑客类给先装载了，也执行了， 最后的结果，唉，很惨的... "   
   "那后来怎么办？"    
   "后来我们老板就定下了规矩：他的骨干员工像String, ArrayList等只能由他自己的心腹去装载， 我听说老板的心腹都是分层级的，像传销一样， 每个都有上线， 最顶层的叫Bootstrap Classloader , 下一次级叫Extension Classloader， 现在开车的这位其实叫App Classloader，位于最底层，  咱这位Classloader 在装载一个类之前，一定要问一问这几位权利极高的大爷，请他们先装载，这几位爷装载不了，才由我们这些小兵来出马。“   
   "这能避免黑客攻击？"   
   "能啊！ 你想想， 那个黑客写了个攻击的java.lang.String,  我们在装载之前，肯定要请示Extension, Bootstrap这些大爷先来装载， 由于String是老板的核心员工，肯定会他们先装载啊， 这些大爷把String 直接就给我们了， 我们就不会装载黑客类了"     ----刘欣《码农翻身》
- 注：如果报Java.lang.NoSuchMethodError的错，一般都是在类路径上放置了多个不同版本的的类包，正好加载的那个类包没有这个方法。

<br>

### 反射   
- 场景1:   
    - 问题：一个已经发布了可独立运行的程序（如tomcat），但是如果客户想加入想拓展功能，可以对外提供接口进行功能的拓展。拓展的功能用一个类实现接口来实现这个功能，但是这个类产生的对象无法被这个可独立运行的程序运行（因为要在此应用程序new出来才能，但是没有源代码，不能new）
    - 解决办法：在应用程序中对外提供接口的时候提供配置文件，应用程序读取配置文件即可。客户只要实现接口，并填写配置文件（包含类名）。
    - 原理： 根据配置文件里的文件名，找到这个文件，并拿到这个文件（只要拿到这个字节码文件就干什么都可以），如果相对指定名称的字节码文件进行加载并获取其中的内容并调用，就要用到了反射技术。

- 概念: 动态获取类中的信息，就是java反射，对类的解剖。
- 作用： 
    - 早期new对象，先根据被new的类的名称找寻该类的字节码文件，加载进内存，并创建该字节码文件的对象，并接着创捷该字节文件的对应的Person对象。
    - 现在通过反射机制，只需要把名字的接口暴漏出去，都可以进行实例。
- 主要反射类及实现
  反射相关的类一般都在java.lang.relfect包里   
  - 获取Class对象   
    ```java
    public static Class<?> forName(String className) throws ClassNotFoundException
    //返回与带有给定字符串名的类或接口相关联的Class对象，相当于：
    Class.forName(className, true, currentLoader)
    //其中 currentLoader 表示当前类的定义类加载器
    ```    
    
- 创建实例
    -   使用Class对象的newInstance(),如果反射的类是空参，就可以直接用这个，class类里一定要有一个空构造函数。
    -   或者构造器getConstructor()

<br>

## 概述

### IoC的概念
IoC(Inverse of Control) 控制翻转，是一种思想，

### IoC在反射中的应用   
```java
<bean id="Foo" class="com.smart.Foo"></bean>
```   
在Spring里经常出现这样的配置，通过反射，Spring再通过这个配置文件进行对象的实例化。   
Spring通过解析XML文件，获取其中的内容。   
 
```java
 //解析<property .../>元素的name属性得到该字符串值为“courseDao”
 String nameStr = "courseDao";
 //解析<property .../>元素的ref属性得到该字符串值为“courseDao”
 String refStr = "courseDao";
 //生成将要调用setter方法名
 String setterName = "set" + nameStr.substring(0, 1).toUpperCase()
 + nameStr.substring(1);
 //获取spring容器中名为refStr的Bean，该Bean将会作为传入参数
 Object paramBean = container.get(refStr);
 //获取setter方法的Method类，此处的cls是刚才反射代码得到的Class对象
 Method setter = cls.getMethod(setterName, paramBean.getClass());
 //调用invoke()方法，此处的obj是刚才反射代码得到的Object对象
 setter.invoke(obj, paramBean); 
```  

