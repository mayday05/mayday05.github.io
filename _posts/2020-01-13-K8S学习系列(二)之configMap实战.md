---
layout:     post
title:      K8S学习系列(二)之configMap实战
subtitle:   configMap实战
date:       2020-01-13
author:     Mayday
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - K8S
---

# k8s -- ConfigMap

[TOC]

# 一、ConfigMap概念

​        在几乎所有的应用开发中，都会涉及到配置文件的变更，比如说在web的程序中，需要连接数据库，缓存甚至是队列等等。而我们的一个应用程序从写第一行代码开始，要经历开发环境、测试环境、预发布环境只到最终的线上环境。而每一个环境都要定义其独立的各种配置。如果我们不能很好的管理这些配置文件，你的运维工作将顿时变的无比的繁琐。为此业内的一些大公司专门开发了自己的一套配置管理中心，如360的Qcon，百度的disconf等。kubernetes也提供了自己的一套方案，即ConfigMap。kubernetes通过ConfigMap来实现对容器中应用的配置管理。 



​       ConfigMap顾名思义，是用于保存配置数据的键值对，可以用来保存单个属性，也可以保存配置文件。Secret可以为Pod提供密码、Token、私钥等敏感数据；对于一些非敏感数据，比如应用的配置信息，则可以使用ConfigMap。

```zh
区别：
ConfigMap的创建和使用方式与Secret非常类似，主要的不同是以明文的形式存放
```

# 二、ConfigMap如何创建

推荐根据yaml描述文件创建，其他方式暂不介绍。

## 根据yaml描述文件创建

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-conf
  namespace: public-service
  labels:
    app: redis
data:
  redis.conf: |-
    port 6379
    timeout 3000
    tcp-keepalive 0
    databases 99
```

应用configMap

```shell
$ kubectl create  -f  config.yaml
configmap "redis-conf" created
```

# 三、ConfigMap的使用

## 3.1  三种使用方式说明和注意事项

Pod可以通过三种方式来使用ConfigMap，分别为：

- 将ConfigMap中的数据设置为环境变量
- 将ConfigMap中的数据设置为命令行参数
- 使用Volume将ConfigMap作为文件或目录挂载

**注意！！**

- ConfigMap必须在Pod使用它之前创建
- 使用envFrom时，将会自动忽略无效的键
- Pod只能使用同一个命名空间的ConfigMap

## 3.2  方式一：用作环境变量

首先直接使用`kubectl create configmap`命令创建两个ConfigMap，分别名为special-config和env-config：

```shell
$ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm

$ kubectl create configmap env-config --from-literal=log_level=INFO

```

环境变量引用方式如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config ## 引用configMap文件名
              key: special.how ## 引用上述configMap中的key值
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
            name: env-config
  restartPolicy: Never
```



## 3.3 方式二：用作命令行参数

将ConfigMap用作命令行参数时，需要先把ConfigMap的数据保存在环境变量中，然后通过$(VAR_NAME)的方式引用环境变量。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ] ## 使用环境变量
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never

```

当pod运行结束后，它的输出如下：

```shell
$very charm
```

## 3.4 方式三：使用volume挂载

使用volume将ConfigMap作为文件或目录直接挂载，对应如下：

### （1） 默认生成所有key-value

 将创建的ConfigMap直接挂载至Pod的/etc/config目录下，其中每一个key-value键值对都会生成一个文件，key为文件名，value为内容。 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/special.how" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume ## 对应volumeMounts名称
      configMap:
        name: special-config ## 对应configMap名称，不指定key时，生成所有
  restartPolicy: Never

```



### （2） 相对路径以及生成指定key

将创建的ConfigMap中special.how这个key挂载到/etc/config目录下的一个**<u>相对路径/keys/special.level</u>**。如果存在同名文件，直接覆盖。其他的key不挂载。

即

- path可以指定相对路径
- items可以指定生成key，其他key不生成（暂时没用）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys/special.level" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items: ## 指定key生成
        - key: special.how
          path: keys/special.level ## 指定相对路径，即“/etc/config” + “keys/special.level”
  restartPolicy: Never

```



### （3） 不覆盖挂载路径其他文件(推荐)

在一般情况下 configmap 挂载文件时，会先覆盖掉挂载目录，然后再将 configmap 中的内容作为文件挂载进行。（即会将指路径文件全部删除后替换为当前文件夹或者文件，因此对于原路径下不是所有文件都要替换时，使用这种方式）

如果想不对原来的文件夹下的文件造成覆盖，只是将 configmap 中的每个 key，按照文件的方式挂载到目录下，可以使用 subpath 参数。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      command: ["/bin/sh","-c","sleep 36000"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/nginx/special.how ## 配合suPath使用时，special.how为文件名，即pod容器中只生成了/etc/nginx/目录，目录之下为文件，只有一个名为special.how的文件（subPath筛选只挂载app.properties文件）
        subPath: special.how ## 语意：即只覆盖指定文件，不会覆盖掉整个nginx下面文件夹其他文件
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.how
          path: special.how
  restartPolicy: Never

```

举例如下：

```shell
root@dapi-test-pod:/# ls /etc/nginx/
conf.d    fastcgi_params    koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params    special.how  uwsgi_params  win-utf
-----即special.how文件存在，该nginx下面的其他文件也存在

root@dapi-test-pod:/# cat /etc/nginx/special.how
Never
```



# 附 参考文献

- [kubernetes之configmap,深度解析mountPath,subPath,key,path的关系和作用](https://www.cnblogs.com/pu20065226/p/10690628.html)
- [k8s -- ConfigMap](https://www.jianshu.com/p/b1d516f02ecd)