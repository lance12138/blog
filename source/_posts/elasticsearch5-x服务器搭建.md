---
title: elasticsearch5.x服务器搭建
date: 2017-06-21 21:49:55
tags: elasticsearch
---
### 引言

之前与搜索有关的需求都是使用solrCloud实现的，最近公司在做日志监控的时候使用并没有用solr而是使用了好评更甚的elasticsearch。之前本来就一直想了解es，现在刚好有机会学习，特此记录下学习的过程。
![elasticsearch][1]
<!--more-->

### what is elasticsearch
elasticsearch（以下使用缩写es代替）是一个基于lucene的分布式、近实时的全文搜索引擎。它还是一个分布式非关系型数据库，被es存储的json数据每个字段都会被索引且可被搜索，并且es可以轻松的存储和处理达pb级别的数据。

### 服务环境搭建

首先准备好elasticsearch按照包我用的是当时最新版5.3.0。
```
elasticsearch-5.3.0.tar.gz 
```
解压，因本人机器性能有限，并没有用上集群，使用默认设置直接启动。
**需要注意的是elasticsearch不允许root权限进行启动**，所以需要创建一个单独用户来运行。
```
adduser elastic -g elastic -p elastic

groupadd elastic

tar -zvxf elasticsearch-5.3.0.tar.gz 
chown -R elastic:elastic elasticsearch-5.3.0

```
使用默认配置启动elasticsearch

```
./elasticsearch-5.3.0/bin/elasticsearch

```
当看到以下日志时就说明启动成功
```
[2017-06-21T04:42:29,857][INFO ][o.e.n.Node               ] initialized
[2017-06-21T04:42:29,857][INFO ][o.e.n.Node               ] [vQCqom0] starting ...
[2017-06-21T04:42:30,305][INFO ][o.e.t.TransportService   ] [vQCqom0] publish_address {192.168.10.133:9300}, bound_addresses {[::]:9300}
[2017-06-21T04:42:30,333][INFO ][o.e.b.BootstrapChecks    ] [vQCqom0] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-06-21T04:42:33,430][INFO ][o.e.c.s.ClusterService   ] [vQCqom0] new_master {vQCqom0}{vQCqom0yQMypwf2VdeS4cw}{8_zAvAFORVK2GCS-3N0CNQ}{192.168.10.133}{192.168.10.133:9300}, reason: zen-disco-elected-as-master ([0] nodes joined)
[2017-06-21T04:42:33,508][INFO ][o.e.h.n.Netty4HttpServerTransport] [vQCqom0] publish_address {192.168.10.133:9200}, bound_addresses {[::]:9200}
[2017-06-21T04:42:33,521][INFO ][o.e.n.Node               ] [vQCqom0] started
[2017-06-21T04:42:33,651][INFO ][o.w.a.d.Monitor          ] try load config from /elastic/elasticsearch-5.3.0/config/analysis-ik/IKAnalyzer.cfg.xml
[2017-06-21T04:42:33,653][INFO ][o.w.a.d.Monitor          ] try load config from /elastic/elasticsearch-5.3.0/plugins/ik/config/IKAnalyzer.cfg.xml
[2017-06-21T04:42:35,023][INFO ][o.e.m.j.JvmGcMonitorService] [vQCqom0] [gc][young][5][9] duration [985ms], collections [1]/[1.1s], total [985ms]/[4.2s], memory [76.5mb]->[65.7mb]/[1.9gb], all_pools {[young] [49mb]->[221.7kb]/[66.5mb]}{[survivor] [8.3mb]->[8.3mb]/[8.3mb]}{[old] [19.1mb]->[57.4mb]/[1.9gb]}
[2017-06-21T04:42:35,023][WARN ][o.e.m.j.JvmGcMonitorService] [vQCqom0] [gc][5] overhead, spent [985ms] collecting in the last [1.1s]
[2017-06-21T04:42:35,209][INFO ][o.w.a.d.Monitor          ] [Dict Loading] custom/mydict.dic
[2017-06-21T04:42:35,236][INFO ][o.w.a.d.Monitor          ] [Dict Loading] custom/single_word_low_freq.dic
[2017-06-21T04:42:35,261][INFO ][o.w.a.d.Monitor          ] [Dict Loading] custom/ext_stopword.dic
[2017-06-21T04:42:35,538][INFO ][o.e.g.GatewayService     ] [vQCqom0] recovered [2] indices into cluster_state
[2017-06-21T04:42:36,324][INFO ][o.e.c.r.a.AllocationService] [vQCqom0] Cluster health status changed from [RED] to [YELLOW] (reason: [shards started [[megacorp][2]] ...]).
[2017-06-21T04:43:04,255][INFO ][o.e.m.j.JvmGcMonitorService] [vQCqom0] [gc][young][34][10] duration [728ms], collections [1]/[1.1s], total [728ms]/[4.9s], memory [131.6mb]->[90.2mb]/[1.9gb], all_pools {[young] [65.9mb]->[145kb]/[66.5mb]}{[survivor] [8.3mb]->[8.3mb]/[8.3mb]}{[old] [57.4mb]->[81.8mb]/[1.9gb]}
[2017-06-21T04:43:04,255][WARN ][o.e.m.j.JvmGcMonitorService] [vQCqom0] [gc][34] overhead, spent [728ms] collecting in the last [1.1s]

```
### 错误集锦与处理

**ERROR: bootstrap checks failed**

具体报错信息为：max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]
max number of threads [1024] for user [lishang] likely too low, increase to at least [2048]

意思就是文件描述符至少需要65536个，线程数至少需要2048个。

解决方法：切换到root用户，进入到security目录下的limits.conf
```
vim /etc/security/limits.conf
```

文件末尾追加一下数据
```
* soft nofile 65536

* hard nofile 131072

* soft nproc 2048

* hard nproc 4096
```
重启生效

**max number of threads [1024] for user [lish] likely too low, increase to at least [2048]**

解决方法：切换到root用户，进入limits.d目录下修改配置文件。
```
vim /etc/security/limits.d/90-nproc.conf
将这一行
* soft nproc 1024
改为
* soft nproc 2048
```
**max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]**

解决方法：切换到root用户修改配置sysctl.conf
```
vim /etc/sysctl.conf 
```
文件后面追加一下配置：
```
vm.max_map_count=655360
```
执行命令
```
sysctl -p
```

 [1]: /images/elastic.jpg "elasticsearch"