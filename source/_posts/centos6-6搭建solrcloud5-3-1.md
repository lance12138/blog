---
title: centos6.6搭建solrcloud5.3.1
date: 2016-07-31 17:52:13
tags: centos,solrcloud 
---

### **1.搭建前准备**
去官网分别下载以下压缩包：
```
jdk-8u45-linux-x64.tar.gz

zookeeper-3.4.6.tar.gz

solrcloud-5.3.1.tar.gz
```
#### 1.1安装配置jdk

```
tar xvzf /home/workspaces/jdk-8u45-linux-x64.tar.gz
```
配置环境变量，进入profile文件
```
vim /etc/profile
```
追加以下内容
```
JAVA_HOME=/usr/java/jdk1.8.45
JRE_HOME=/usr/java/jdk1.8.45/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```
<!--more-->
### 2安装配置开启zookeeper集群

```
tar xf /home/workspace/soft/zookeeper-3.4.6.tar.gz   
```
复制zookeeper默认配置文件
```
cd zookeeper-3.4.6/conf  
cp zoo_sample.cfg zoo.cfg 
```
将需要搭建成集群的服务器添加进去
```
vim zoo.cfg  
  
tickTime=2000  
 initLimit=10  
 syncLimit=5  
 dataDir=/path/to/zookeeper/data  
 clientPort=2181  
 server.1=192.168.156.121:2888:3888   
server.2=192.168.156.122:2888:3888  
 server.3=192.168.156.123:2888:3888 
```
将服务器编号写入每一台服务器
```
# 注意每台机器上的不一样


echo"1">myid#在solr1上


echo"2">myid#在solr2上


echo"3">myid#在solr3上
```
其中solr1、solr2、solr3分别是我三台不同服务器名
在上面配置了三台服务器集群，要在三台服务器分别如上配置并且开启zookeeper服务器
```
zookeeper-3.4.6/bin ./zkServer.sh start  
```
### 3.将solr安装为服务
```
tar xf /home/workspace/soft/solrcloud-5.3.1.tar.gz 
```
创建两个文件夹solr、data
```
mkdir -p /solrcloud/{data,solr}
```
设置服务名称、端口等
```
cd solr-5.3.1/bin  
./install_solr_service.sh /home/wokspace/soft/solr-5.3.1.tgz  -d /solrcloud/data/ -i /solrcloud/solr/ -s solrcloud -u root -p 8080  
```
这里面-d是放solr的data数据，-i是放solr文件，-s是服务名 -u是用户 -p是实用端口，默认是8983

修改solrcloud的data文件
```
cd /home/workspace/solrcloud/data  
ls  
data  log4j.properties  logs  solr-8983.pid  solr.in.sh  
vim solr.in.sh  
  
# Set the ZooKeeper connection string if using an external ZooKeeper ensemble  
# e.g. host1:2181,host2:2181/chroot  
# Leave empty if not using SolrCloud  
ZK_HOST="192.168.156.121:2181,192.168.156.122:2181,192.168.156.123:2181"
```
启动solrcloud
```
service solrcloud restart  
```
### 4.更新配置文件创建collection
```
cd solr-5.3.1  
./server/scripts/cloud-scripts/zkcli.sh -zkhost localhost:2181 -cmd upconfig -confname demo-conf -confdir server/solr/configsets/basic_configs/conf/  
  
./server/scripts/cloud-scripts/zkcli.sh -zkhost localhost:2181 -cmd  linkconfig -collection demo -confname demo-conf  
  
curl 'http://192.168.156.121:8080/solr/admin/collections?action=CREATE&name=demo&numShards=1&replicationFactor=1'  
   
```
 
其中-upconfig 是更新配置文件，-linkconfig是创建链接  
最后curl 提交请求创建collection，collection名称为demo，一个collection创建一个shard，一个shard创建一个replica
### 5.添加文件数据索引
```
bin/post -c demo -p 8080 /home/workspace/solrcloud/example/exampledocs/books.json  
  
以上 -c是指定要上传数据给哪个collection，-p是端口号 
```
如果想自定义field字段，进入schema.xml添加或修改field，修改之后需要再次更新配置文件和第四步的更新配置文件一样

### 7.查询
```
curl 'http://192.168.219.128:8080/solr/demo/select?wt=json&indent=true&q=cat:book&fl=name'  
```
响应
```
{  
  "responseHeader":{  
    "status":0,  
    "QTime":6,  
    "params":{  
      "q":"cat:book",  
      "indent":"true",  
      "fl":"name",  
      "wt":"json"}},  
  "response":{"numFound":4,"start":0,"docs":[  
      {  
        "name":["The Lightning Thief"]},  
      {  
        "name":["The Sea of Monsters"]},  
      {  
        "name":["Sophie's World : The Greek Philosophers"]},  
      {  
        "name":["Lucene in Action, Second Edition"]}]  
  }}  
```
