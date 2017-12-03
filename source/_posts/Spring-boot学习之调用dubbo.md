---
title: Spring boot学习之调用dubbo
date: 2016-12-07 22:37:32
tags: Spring boot
---
Spring boot好早之前就有在用，但是都是用Spring boot建简单的web站点，很多Spring boot其他特性的使用都处于未知状态，最近又开始使用Spring boot开发，特此记录。
### 使用@Import注解引入配置文件
Spring boot最好用的一点就是自动配置，约定大于配置的特性让我们减少了很多配置工作，但是有很多情况下是不能自动配置的，比如dubbo就没有，所以dubbo的引入依旧需要配置文件，Spring boot要使配置文件生效需要引入配置文件，引入方式如下：
```
@SpringBootApplication
@Import("classpath:*.xml")
public class App{
    
    public static void main(String[] args){
        SpringApplication.run(App.class,args);
    }
}
```
### 使用CommandLineRunner接口

在开发时经常会有在项目启动后就初始化模块、或者执行一次的需求，这时*CommandLineRunner*就是一个很好的选择，该接口有``run()``方法当实现了*CommandLineRunner*接口,项目启动之后会首先执行run方法。例如：
```

@Component
public class StartupRunner implements CommandLineRunner {
    protected final Logger logger = LoggerFactory.getLogger(StartupRunner.class);
    @Override
    public void run(String... strings) throws Exception {
        logger.info("you are invoke this method!");
    }
}
```
