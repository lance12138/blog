---
title: 使用spring方式引入外部文件
date: 2016-11-18 23:03:23
tags: 
---

> 在很多业务场景下我们都需要引入外部文件，比如大量的静态json、业务模板、xml文件等，大量的外部文件引入如果方式不正确很有可能会导致系统性能问题，今天我们就说说如何使用优雅的方式引入外部文件。

#### 传统引入方式
一般我们引入外部文件都是通过IO流的方式引入，每次使用都要从硬盘读到内存中。代码如下：
```
public String readFile(){
        File file=new File("test.txt");
        BufferedReader reader=new BufferedReader(new InputStreamReader(new FileInputStream(file)));
        StringBuffer sb=new StringBuffer();
        String temp=null;
        while((temp=reader.readLine())!=null){
            sb.append(temp);
        }
        reader.close();

    
    
    }
```
该代码缺陷在于，每用一次就要从硬盘读取到内存，性能着急，这样的代码是不行的。
<!--more-->

#### 使用spring方式

如果系统中使用来spring框架，那么你可以优雅的使用spring来引入外部文件，spring自带有resource解析器`Resource`可以在项目启动时加载静态资源。我们可以定义一个专门的Bean`ResourceResolver`来加载外部文件，代码如下：
```
public class ResourceResolver {
    private Resource resource;
    private String transforStr;
    
    @PostConstruct
    public void readFile()throws IOException{
        File file=resource.getFile();
        BufferedReader reader=new BufferedReader(new InputStreamReader(new FileInputStream(file)));
        StringBuffer sb=new StringBuffer();
        String temp=null;
        while((temp=reader.readLine())!=null){
            sb.append(temp);
        }
        reader.close();
        this.transforStr=sb.toString();

    }

    public String getTransforStr() {
        return transforStr;
    }

    public void setTransforStr(String transforStr) {
        this.transforStr = transforStr;
    }

    public Resource getResource() {
        return resource;
    }
    public void setResource(Resource resource) {
        this.resource = resource;
    }
}

```
在spring配置文件中配置：
```
  <bean id="resourceResolver" class="com.ifenqu.bean.ResourceResolver">
        <property name="resource" value="classpath:test.txt"></property>
            </bean>
```
相对于第一种实现方式，使用spring的方式只需要在项目加载的时候执行一次从磁盘加载到内存操作就行，该方式更佳优雅。