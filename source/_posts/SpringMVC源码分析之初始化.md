---
title: springMVC源码分析之初始化
date: 2016-09-04 21:41:17
tags: springMVC,源码
---

springMVC属于SpringFrameWork家族中的一员，目前已经是最流行的和效率最好的MVC框架了，特别是配合spring的bean管理容器使用能让你很优雅的开发WEB应用。从这篇开始本人会从springMVC的初始化、springMVC的Controller调用过程、springMVC的视图解析过程、springMVC拦截器原理来分析源码，而今天就让我们先来领教一下springMVC的初始化吧。

#### springMVC配置

在开始源码分析前，我们先来看看springMVC的配置文件和web.xml的配置方便接下来de分析。首先是spring-servlet.xml（配置文件什么名字都可以）
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">

    <context:annotation-config/>
    <context:component-scan base-package="com.ifenqu.controller"/>
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" p:suffix=".jsp">
    </bean>
    </beans>


```
<!--more-->
以上是一个非常简单的demo配置，包括第一个``<context:annotation-config>``用来声明允许使用注解，第二个``<context:component-scan base-package="com.ifenqu.controller"/>``来自动扫描需要注入的bean，第三个``<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" p:suffix=".jsp"></bean>``为返回的view自动加上后缀。
接下来是web.xml文件

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" id="WebApp_ID" version="3.0">
  <display-name>Archetype Created Web Application</display-name>
  <servlet>
    <servlet-name>springMVC</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>springMVC</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring-*.xml</param-value>
  </context-param>
</web-app>

```
web.xml没什么好说的，就是将springMVC在web.xml中注册加入各配置文件。
接下来看看controller

```
package com.ifenqu.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * Created by LAIYAO on 2016/9/4.
 */
@Controller
public class HelloController {
    @ResponseBody
    @RequestMapping(value="/hello")
    public String hello(){
        return "hello";
    }
}
```


#### 初始化

springMVC中最核心的就是``DispatcherServlet``这个类，该类充当前端控制器控制着request的Mapping、handle、viewResolver等。我们看看这个DispatcherServlet类继承体系。
![DispatcherServlet继承体系][1]
如图所示，DispatcherServlet继承于FrameworkServlet，而FrameworkServlet继承于HttpServletBean，最终继承于HttpServlet。

当系统启动时，在web.xml配置的各配置文件被读取最终组成``RootWebApplicationContext``并初始化IoC容器，紧接着会执行DispatcherServlet持有的IoC容器的初始化，DispatcherServlet初始化包括两个部分，第一，DispatcherServlet持有的子上下文初始化（该上下文对应的是Servlet的上下文，与Web应用的servletContext上下文呈父子关系）和DispatcherServlet其他部分如strategy初始化等，接下来我们看看springMVC的init方法，如下图所示：
![springMVC的init方法][2]
其中这个PropertyValues是DispatcherServlet的内部静态类,在这一步主要是用来读入从web.xml配置的文件如下图所示：
![pvs][3]
接下来将DispatcherServlet封装了一层并且初始化好资源加载器resourceLoader，这个resourceLoader初始化之后主要是有类似于getResourceAsStream、getRealPath等资源加载的方法的一个Map和classLoader。

接下来就是初始化这个BeanWrapper，然后执行initServletBean方法，这个方法中主要是初始化``webApplicationContext``。webApplicationContext的初始化是要以RootWebApplicationContext作为参数，通过反射来创建webApplicationContext对象，实例化结束后需要给上下文设置一下基本配置如bean定义的配置文件位置等，双亲上下文（实例化后子上下文会被setAttribute到根上下文，当要获取bean时先进根上下文获取，再去子上下文找），最后通过调用DispatcherServlet的IoC容器的refresh方法完成strategy的初始化，也就是各种解析器、HandlerMapping、适配器等的初始化，为什么是strategy呢，因为在这里使用了策略模式，springMVC中有很多解析器、映射器和适配器以适应不同场合的调用，所以这里使用使用策略模式。如下图所示：
![initStrategy方法][4]
我们挑这其中的initHandlerMappings和initHandlerAdapters方法来看看。首先initHandlerMappings，方法如下图所示：
![initHandlerMappings方法][5]
初始化HandlerMapping首先会从ApplicationContext中找到所有的HandlerMappings，为保证至少有一个HandlerMapping注册，如果在ApplicationContext中没有找到HandlerMapping，系统会提供默认的HandlerMapping，即``BeanNameUrlHandlerMapping``和``DefaultAnnotionHandlerMapping``
而initHandlerAdapters方法和initHandlerMappings差不多，或者说是简直一模一样（moji笑哭脸），不过话说回来，HandlerMapping和HandlerAdapter本来就是相同的用途的类用在不同作用域上，HandlerMapping是找到需要请求的bean，而HandlerAdapter则是找到该bean的具体方法并调用，在最初的初始化时期当然是差不多的逻辑。
在这一系列的各模块的初始化之后，WebApplicationContext算是初始化结束，同时FrameworkServlet初始化结束。


总结起来，springMVC的初始化过程就三步，第一步：加载配置文件读取配置文件属性，第二步：将配置文件属性和读取servletContext的组成自己的上下文，第三步，调用refresh方法初始化各strategy组件（各种解析器、适配器）,完成WebApplicationContext的初始化，最终完成DispacherServlet持有的IoC容器的初始化。


  [1]: /images/hierarchy.jpg "DispatcherServlet继承体系"
  [2]: /images/init.jpg "springMVC的init方法"
  [3]: /images/pvs.jpg "pvs"
  [4]: /images/initStrategy.jpg "initStrategy方法"
  [5]: /images/initHandlerMappings.jpg "initHandlerMappings方法"