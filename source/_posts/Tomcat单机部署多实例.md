---
title: Tomcat单机部署多实例
date: 2015-07-01 00:00:00
categories: Tomcat
tags: Tomcat
---

## 单Tomcat部署多实例有以下优点：

 1、所有项目只应用单一Tomcat，对于项目启动、Tomcat升级可一次性解决，不需要更改过多配置

 2、单一Tomcat日志存储不再是问题，可统一跟踪及处理

 3、多实例无依赖，可做到单实例下线或维护，不影响其它实例运行，方便管理

 4、对多实例间单例不会造成任何影响

 5、可实现自定义单一实例热加载热部署，不会对其它实例造成影响

<!--more-->

这里需要说明的两个变量CATALINA_HOME、CATALINA_BASE，其中CATALINA_HOME指定的是tomcat主目录，CATALINA_BASE指定的是tomcat实例的目录，CATALINA_BASE默认和CATALINA_HOME相同。

在/etc/profile最后加入CATALINA_HOME环境变量

```
export CATALINA_HOME=/home/liuxin/tomcat/apache-tomcat-7.0.47
```
将tomcat主目录的conf、logs、webapp、temp、work目录拷贝到新的目录下作为一份新的tomcat实例，注意修改conf/server.xml文件中的各个端口号防止端口占用问题

编写启动脚本和关闭脚本，启动多实例时注意修改CATALINA_BASE为指定实例的目录

```
#! /bin/sh
export CATALINA_BASE=/home/liuxin/tomcat/tomcat1
sh "$CATALINA_HOME"/bin/startup.sh
```
```
#! /bin/sh
export CATALINA_BASE=/home/liuxin/tomcat/tomcat1
sh "$CATALINA_HOME"/bin/shutdown.sh
```
接下来就可以新建几个实例出来，启动多实例了，这样很多人用同一台服务器时可以各自跑各自的没什么影响