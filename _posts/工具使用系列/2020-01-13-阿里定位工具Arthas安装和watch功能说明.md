---
layout:     post
title:      阿里定位工具Arthas安装和watch功能说明
subtitle:   Arthas
date:       2020-01-13
author:     Mayday
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 工具使用
---









[TOC]



## 一、安装方式

### 1.1 本地安装

#### 使用`arthas-boot`（推荐）

下载`arthas-boot.jar`，然后用`java -jar`的方式启动：

```

curl -O https://alibaba.github.io/arthas/arthas-boot.jar

java -jar arthas-boot.jar
```

### 1.2  通过rpm/deb来安装

在releases页面下载rpm/deb包： https://github.com/alibaba/arthas/releases

#### 1.2.1 安装deb

```
sudo dpkg -i arthas*.deb
```

#### 1.2.2 安装rpm

```
sudo rpm -i arthas*.rpm
```

#### 1.2.3 deb/rpm安装的用法

在安装后，可以直接执行：

```
as.sh
```



### 1.3 如何在docker中使用

#### 1.3.1 诊断Docker里的Java进程

```
docker exec -it  ${containerId} /bin/bash -c "wget https://alibaba.github.io/arthas/arthas-boot.jar && java -jar arthas-boot.jar"
```

#### 1.3.2 诊断k8s里容器里的Java进程

```
kubectl exec -it ${pod} --container ${containerId} -- /bin/bash -c "wget https://alibaba.github.io/arthas/arthas-boot.jar && java -jar arthas-boot.jar"
```

#### 1.3.3 把Arthas安装到基础镜像里

可以很简单把Arthas安装到你的Docker镜像里。

```
FROM openjdk:8-jdk-alpine

# copy arthas
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
```

### 1.4 全量安装（离线安装时推荐使用）

#### 1.4.1 下载全量包

最新版本，点击下载：[![https://img.shields.io/maven-central/v/com.taobao.arthas/arthas-packaging.svg?style=flat-square](https://img.shields.io/maven-central/v/com.taobao.arthas/arthas-packaging.svg?style=flat-square)](http://repository.sonatype.org/service/local/artifact/maven/redirect?r=central-proxy&g=com.taobao.arthas&a=arthas-packaging&e=zip&c=bin&v=LATEST)

如果下载速度比较慢，可以尝试用[阿里云的镜像仓库](https://maven.aliyun.com/)，比如要下载`3.x.x`版本（替换`3.x.x`为最新版本），下载的url是：

https://maven.aliyun.com/repository/public/com/taobao/arthas/arthas-packaging/3.x.x/arthas-packaging-3.x.x-bin.zip

### 用as.sh启动

解压后，在文件夹里有`as.sh`，直接用`./as.sh`的方式启动：

```
./as.sh
```

打印帮助信息：

```
./as.sh -h
```

------



## 二、如何使用watch功能

![**arthas-watch1.png**](https://github.com/mayday05/mayday05.github.io/blob/master/image2/arthas-watch1.png)

### 2.1 参数说明

watch 的参数比较多，主要是因为它能在 4 个不同的场景观察对象

| 参数名称            | 参数说明                                   |
| ------------------- | ------------------------------------------ |
| *class-pattern*     | 类名表达式匹配                             |
| *method-pattern*    | 方法名表达式匹配                           |
| *express*           | 观察表达式                                 |
| *condition-express* | 条件表达式                                 |
| [b]                 | 在**方法调用之前**观察                     |
| [e]                 | 在**方法异常之后**观察                     |
| [s]                 | 在**方法返回之后**观察                     |
| [f]                 | 在**方法结束之后**(正常返回和异常返回)观察 |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配       |
| [x:]                | 指定输出结果的属性遍历深度，默认为 1       |

比如 

watch org.elasticsearch.action.index.IndexRequest validate "target"

执行后，就看到了我们希望了解到的内容。

### 2.2 注意事项

- 1、默认f参数，返回方法执行后的结果

- 2、可以同时监控方法执行前和方法执行后



## 三、常见用户案例

Github完整案例见[用户案例](https://github.com/alibaba/arthas/issues?q=label%3Auser-case),对应项目issue中使用label:user-case 搜索结果

筛选出部分和自己相关的部分案例如下

| 案例标题                                                     | 使用特性                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Arthas协助排查线上skywalking不可用问题  #Issue557](https://github.com/alibaba/arthas/issues/557) | 该案例使用了watch特性，能够动态监控指定类、指定方法、输出的指定结果 |
| [Arthas源码分析--jad反编译原理](https://github.com/alibaba/arthas/issues/763) | jad线上反编译代码                                            |
| [引发线程cpu占用率持续飙升的根因分析](https://github.com/alibaba/arthas/issues/569) | thread线程分析                                               |
| [Arthas排查Kubernetes中的应用频繁挂掉重启问题](https://github.com/alibaba/arthas/issues/561) [u](https://github.com/alibaba/arthas/issues?q=label%3Auser-case) | jvm堆分析                                                    |
| [Alibaba Arthas实践--获取到Spring Context，然后为所欲为](https://github.com/alibaba/arthas/issues/482) | tt命令结合-w参数                                             |
| [分享及其资料：当DUBBO遇上Arthas - 排查问题的实践](https://github.com/alibaba/arthas/issues/327) | watch命令<br /><br />watch com.example.UserService * -e -x 2 '{params,throwExp}' |
| .......省略                                                  |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |



## 四、附 参考文献

### 1、 [官方使用指南](https://alibaba.github.io/arthas/)

### 2、 [官方用户案例](https://github.com/alibaba/arthas/issues?q=label%3Auser-case)