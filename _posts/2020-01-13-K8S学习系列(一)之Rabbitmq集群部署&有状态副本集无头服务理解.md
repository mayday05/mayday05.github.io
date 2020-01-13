---
layout:     post
title:      K8S学习系列(一)之Rabbitmq集群部署&有状态副本集无头服务理解
subtitle:   Rabbitmq集群部署
date:       2020-01-13
author:     Mayday
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - K8S
---

[TOC]



## 一、RabbitMQ集群K8S部署

### 1.1 文件结构和说明

分别对应文件如下

- rabbitmq-cluster-statefulset.yaml
- rabbitmq-configmap.yaml
- rabbitmq-dashboard-service.yaml
- rabbitmq-headless-service.yaml
- rabbitmq-public-namespace.yaml
- rabbitmq-pv-hostpath.yaml
- rabbitmq-pv-hostpath.yaml
- rabbitmq-rbac.yaml
- rabbitmq-secret.yaml

### 1.2 部署步骤

#### 1.2.1 创建namespace和rbac

（1） 创建namespace

```
kubectl create namespace public-service
```

 如果不使用public-service,同步以下yaml文件中修改即可。一键修改如下：

```（）
sed -i "s#public-service#YOUR_NAMESPACE#g" *.yaml
```

（2） 创建rbac

创建rbac主要包含以下几个文件



#### 1.2.2 创建持久化存储卷PV（运维人员创建）

PV通常由运维人员创建，可以有hostpath方式和使用nfs方式，如果是开发环境，使用宿主机的hostpath方式即可。

##### （1）hostpath方式

```yaml
## 预定义有状态副本集的三个PV存储快
## 注意有状态副本集使用哪个PV，按照名次+编号找的
## 注意此处使用的宿主机hostPath模式，因此需要配置affinity每个节点最多只能配置一个该有状态副本集的POD，而使用nfs等指定IP则不需要

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-1
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  ##定义存储路径，如果为nfs网络存储，则为server
  hostPath:
    path: /opt/dockerdata/mq-system/rabbitmq2/data

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-2
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  hostPath:
    path: /opt/dockerdata/mq-system/rabbitmq2/data

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-3
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  hostPath:
    path: /opt/dockerdata/mq-system/rabbitmq2/data

```

##### （2）nfs方式

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-1
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /k8s/rmq-cluster/rabbitmq-cluster-1
    # nfs real ip
    server: 192.168.147.220

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-2
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /k8s/rmq-cluster/rabbitmq-cluster-2
    # nfs real ip
    server: 192.168.147.220

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-3
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /k8s/rmq-cluster/rabbitmq-cluster-3
    # nfs real ip
    server: 192.168.147.220

```

#### 1.3 创建持久化声明PVC

讲到创建持久化存储卷声明时，不一定需要单独写一个声明文件，依据部署文件的不同可以选择不单独声明，直接在部署文件中声明即可，见下文使用PV方式二。

##### （1）使用PV方式一

方式一为部署模板时，定义容器存储卷volumes时通过**persistentVolumeClaim**字段指明PVC名称。

比如下面：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: mq-system
spec:
  serviceName: rabbitmq-headless   # 必须与headless service的name相同，用于hostname传播访问pod
  selector:
    matchLabels:
      app: rabbitmq # 在apps/v1中，需与 .spec.template.metadata.label 相同，用于hostname传播访问pod，而在apps/v1beta中无需这样做
  replicas: 1
  template:
    metadata:
      labels:
        app: rabbitmq  # 在apps/v1中，需与 .spec.selector.matchLabels 相同
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      ## 设置Affinity
      affinity:
       #如果某个maste节点上已有rabbitmq pod，则不在这个master节点上建rabbitmq pod。确保3个rabbitmq pod分别在3个master节点上创建
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
                 - rabbitmq
            topologyKey: kubernetes.io/hostname
      ## 选择在master上部署
      nodeSelector:
        role: master
      containers:        
      - name: rabbitmq
        image: rabbitmq:3.7.12
        # 定义拉取策略为本地不存在才拉取
        imagePullPolicy: IfNotPresent
        # 定义容器环境变量
        env:
          - name: HOSTNAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          # 若在ConfigMap中设置了service_name，则此处无需再次设置，注意环境变量声名的顺序
          - name: K8S_SERVICE_NAME
            value: "rabbitmq-headless"
          - name: RABBITMQ_NODENAME
            value: "rabbit@$(HOSTNAME).$(K8S_SERVICE_NAME)"
          - name: K8S_HOSTNAME_SUFFIX
            value: ".$(K8S_SERVICE_NAME)"
          # 将Rabbitmq的集群ErlangCookie环境变量必须一致，设置相同，此处直接明文，建议使用secret的Opaque类型密文形式
          - name: RABBITMQ_ERLANG_COOKIE
            value: "mycookie" 
          # - name: RABBITMQ_LOGS
            # value: "/var/log/rabbitmq/node@$(HOSTNAME).log"
          # - name: RABBITMQ_UPGRADE_LOG
            # value: "/var/log/rabbitmq/node@$(HOSTNAME).log"
        ports:
          - name: http
            protocol: TCP
            containerPort: 15672
          - name: amqp
            protocol: TCP
            containerPort: 5672
        livenessProbe:
          exec:
            command: ["rabbitmqctl", "status"]
          initialDelaySeconds: 60
          periodSeconds: 60
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["rabbitmqctl", "status"]
          initialDelaySeconds: 20
          periodSeconds: 60
          timeoutSeconds: 5
        # 配置容器中挂载卷
        volumeMounts:
          - name: config-volume
            mountPath: /etc/rabbitmq
          - name: timezone
            mountPath: /etc/localtime
          - name: log
            mountPath: /var/log/rabbitmq
          - name: rabbitmq-data
            mountPath: /var/lib/rabbitmq/mnesia
      ## 定义存储卷，分别对应上述volimeMounts
      volumes:
        ## 定义rabbitmq配置文件为指定config-map的文件
        - name: config-volume
          configMap:
            name: rabbitmq-config
            items:
            - key: rabbitmq.conf
              path: rabbitmq.conf
            - key: enabled_plugins
              path: enabled_plugins
        ## 定义rabbitmq的数据文件，为采用PVC关联的PV的存储卷，即使用名为XXX的PVC指定的PV存储空间
        - name: rabbitmq-data
          persistentVolumeClaim:
            claimName: rabbitmq-data-claim
        ## 定义时区映射文件为宿主机路径
        - name: timezone
          hostPath:
            path: /etc/localtime
        ## 定义日志文件对应宿主机路径
        - name: log
          hostPath:
            path: /opt/dockerdata/mq-system/rabbitmq/log

```

上面部署文件指定的PVC如下：

```yaml
## PVC 描述的，则是 Pod 所希望使用的持久化存储的属性。比如，Volume 存储的大小、可读写权限等等
## PVC 可以理解为持久化存储的“接口”，它提供了对某种持久化存储的描述，但不提供具体的实现
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  ## 声名一个名为rabbitmq-data-claim的PVC
  name: rabbitmq-data-claim
  namespace: mq-system
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rabbitmq-storageClass
  resources:  
    requests:
      storage: 10Gi
  selector:
    ## 其关联PV的字段为release，内容为rabbitmq-data
    matchLabels:
      release: rabbitmq-data


```



##### （2）使用PV方式二（不单独写PVC文件）

方式二为部署模板时，直接通过volumeClaimTemplates指定方式一中声明的PVC相关参数，包括存储大小、写入模式、存储类名称等。

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: rmq-cluster
  name: rmq-cluster
  namespace: public-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rmq-cluster
  ## 定义无头服务名称
  serviceName: rabbitmq-headless # 必须与headless 服务的name相同，用于hostname传播访问pod，且注意有状态副本集为了互相感知节点存在RABBITMQ_NODENAME名称也会使用无头服务的DNS
  template:
    metadata:
      labels:
        app: rmq-cluster
    spec:
      serviceAccountName: rmq-cluster
      terminationGracePeriodSeconds: 30
      ## 设置Affinity
      affinity:
       #如果某个maste节点上已有rabbitmq pod，则不在这个master节点上建rabbitmq pod。确保3个rabbitmq pod分别在3个master节点上创建
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
                 - rmq-cluster
            topologyKey: kubernetes.io/hostname
      containers:
      - args:
        - -c
        - cp -v /etc/rabbitmq/rabbitmq.conf ${RABBITMQ_CONFIG_FILE}; exec docker-entrypoint.sh
          rabbitmq-server
        command:
        - sh
        env:
        ## 用户名，从密件文件中读取
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              key: username
              name: rmq-cluster-secret
        ## 密码，从密件文件中读取
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              key: password
              name: rmq-cluster-secret
        ## ERlang Cookie,从密件文件中读取
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              key: cookie
              name: rmq-cluster-secret
        ## POD IP
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ## POD 名称
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ## POD 命名空间
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ## 是否使用长节点名称，有点的就是长节点名称，必须配置
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        
        # 若在ConfigMap中设置了service_name，则此处无需再次设置，注意环境变量声名的顺序
        - name: K8S_SERVICE_NAME
          value: rabbitmq-headless
        
        ## 节点名称命名规则，注意此处节点名称必须符合无头服务的命名规则，比如此处为rmq-cluster.rabbitmq
        ## 注意之前随意改变了有状态副本集的名称serviceName，并且同步修改了无头服务的对应字段，但是没有修改此处的$(K8S_SERVICE_NAME)名称，造成提示
        ## "ERROR: epmd error for host rmq-cluster-0.rmq-cluster.public-service.svc.cluster.local: nxdomain (non-existing domain)"错误
        - name: RABBITMQ_NODENAME
          value: rabbit@$(POD_NAME).$(K8S_SERVICE_NAME).$(POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_CONFIG_FILE
          value: /var/lib/rabbitmq/rabbitmq.conf
        image: rabbitmq:3.7-management
        imagePullPolicy: IfNotPresent
        ## 配置探针
        livenessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 30
          timeoutSeconds: 10
        ## 容器组名称
        name: rabbitmq
        ports:
        - containerPort: 15672
          name: http
          protocol: TCP
        - containerPort: 5672
          name: amqp
          protocol: TCP
        ##　探针
        readinessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 10
          timeoutSeconds: 10
        ## 容器存储声明，此处两个容器存储都必须通过volumes或者volumeClaimTemplates指定
        volumeMounts:
        ## 定义配置文件，容器内映射路径为XXX
        - name: config-volume
          mountPath: /etc/rabbitmq          
          readOnly: false
        ## 定义数据存储，容器内映射路径为XXX
        - name: rabbitmq-storage
          mountPath: /var/lib/rabbitmq          
          readOnly: false
      ##　外部存储绑定
      volumes:
      - name: config-volume ## 名称和volumeMounts中某个name相同
        configMap:
          name: rmq-cluster-config ## 名称需和对应的configMap名称相同
          items:
          - key: rabbitmq.conf
            path: rabbitmq.conf
          - key: enabled_plugins
            path: enabled_plugins          
  volumeClaimTemplates:
  - metadata:
      ## 名称需和volumeMounts中某个name相同
      name: rabbitmq-storage
    spec:
      accessModes:
      - ReadWriteMany
      storageClassName: "rmq-storage-class"
      resources:
        requests:
          storage: 4Gi


```

#### 1.4 创建configMap(根据需求)

通常容器部署过程中，部分文件不喜欢写死，因此通过configMap方式动态配置是大多数人的选择，如果是用户名、密码这种，建议使用secret这种加密存储方式。

如下，动态指定开启的插件，以及rabbitmq服务端的配置（比如配置集群信息等）。

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rmq-cluster-config
  namespace: public-service
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
    rabbitmq.conf: |
      loopback_users.guest = false

      ## Clustering
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      #################################################
      # public-service is rabbitmq-cluster's namespace#
      #################################################
      cluster_formation.k8s.hostname_suffix = .rmq-cluster.public-service.svc.cluster.local
      cluster_formation.node_cleanup.interval = 10
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      ## queue master locator
      queue_master_locator=min-masters

```

#### 1.5 Secret创建Erlang对应的Cookie

Erlang对应的Cookie信息比较敏感，因此需通过Secret方式创建如下：

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: rmq-cluster-secret
  namespace: public-service
stringData:
  cookie: ERLANG_COOKIE
  password: RABBITMQ_PASS
  url: amqp://RABBITMQ_USER:RABBITMQ_PASS@rmq-cluster-balancer
  username: RABBITMQ_USER
type: Opaque ## 模糊

```

#### 1.6 创建部署文件

部署文件在1.3中使用PVC已经写明，不再重复说明。

#### 1.7 定义headless服务

定义headless(又称无头)服务前，须明白无头服务的作用，见第三章节。

```yaml
## 注意无头服务必须创建在有状态副本集前面，否则无法利用无头服务的DNS特性
## 详细参考文章说明https://www.jianshu.com/p/a6d8b28c88a2
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
  name: rabbitmq-headless ## 必须与有状态副本集的的serviceName相同，用于hostname传播访问pod
  namespace: public-service
spec:
  clusterIP: None
  ports:
  - name: amqp
    port: 5672
    targetPort: 5672
  selector:
    app: rmq-cluster

```



#### 1.8 定义对外暴露服务

用户根据需求对外暴露部分服务，比如dashboard服务端口，或者直接暴露rabbitmq的broker端口。

```yaml
# 用于暴露dashboard到外网
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
    type: LoadBalancer
  name: rabbitmq-service
  namespace: public-service
spec:
  type: NodePort  # 注意如果你想在外网下访问mq，需要增配nodeport
  ports:
  - name: http
    port: 15672
    protocol: TCP
    targetPort: 15672 # 注意k8s默认情况下，nodeport要在30000~32767之间，可以自行修改
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: 5672
  selector:
    app: rmq-cluster



```



## 二、有状态副本集作用和组成

### 2.1 statefulset作用

  在k8s中，statefulset主要管理一下特效的应用：

> ​    a)、每一个Pod稳定且有唯一的网络标识符；--------- 即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现 
>
> ​    b)、稳定且持久的存储设备；  ---------即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现 
>
> ​    c)、要求有序、平滑的部署和扩展； ------- （即从0到N-1） 
>
> ​    d)、要求有序、平滑的终止和删除； ------- （即从N-1到0） 
>
> ​    e)、有序的滚动更新，应该先更新从节点，再更新主节点； 



### 2.2 statefulset三要素

 statefulset由三个组件组成

> ​    a)  headless service（无头的服务，即没名字）；-------- 用于定义网络标志（DNS domain） 
>
> ​    b)  statefulset控制器 ；                                         -------- 定义具体应用 
>
> ​    c)  volumeClaimTemplate（存储卷申请模板，因为每个pod要有专用存储卷，而不能共用存储卷） --- 用于创建PersistentVolumes 
>
> 如何理解无头服务和普通服务的区别

### 2.3 statefulSet中DNS格式什么样子

StatefulSet中每个Pod的DNS格式为`statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`，其中` `

- `statefulSetName`为StatefulSet的名字
- `0..N-1`为Pod所在的序号，从0开始到N-1
- `serviceName`为Headless Service的名字
- `namespace`为服务所在的namespace，Headless Servic和StatefulSet必须在相同的namespace
- `.cluster.local`为Cluster Domain



## 三、 关于headless服务

### 3.1  headless服务概念

> ### headless service
>
> headless  service是一个特殊的ClusterIP类service，这种service创建时不指定clusterIP(--cluster-ip=None)，因为这点，kube-proxy不会管这种service，于是node上不会有相关的iptables规则。
>
> 当headless service有配置selector时，其对应的所有后端节点，会被记录到dns中，在访问service domain时kube-dns会将所有endpoints返回，选择哪个进行访问则是系统自己决定；

### 	  3.2  nslookup理解无头服务

关于无头服务理解，可以阅读下面这篇文章，普通服务DNS查询时只能获取服务ClusterIP，而headless服务可以获取pod的真实IP地址。即

> 参考文章[K8S容器编排之Headless浅谈](https://www.jianshu.com/p/a6d8b28c88a2) -- 用nslookup实战说明无头服务到底提供了什么功能
>
>  1、`dns`查询普通clusterIP类型的服务时只会返回`Service`的地址。具体`client`访问的是哪个`Real Server`，是由`iptables`来决定的 
>
> 2、 `dns`查询headless服务时会如实的返回2个真实的`endpoint` 

### 3.3   Headless Service有什么使用场景呢？

参考如下理解：

> `Headless Service`就是没头的`Service`。有什么使用场景呢？
>
> - 第一种：自主选择权，有时候`client`想自己来决定使用哪个`Real Server`，可以通过查询`DNS`来获取`Real Server`的信息。
> - 第二种：`Headless Services`还有一个用处（PS：也就是我们需要的那个特性）。`Headless Service`的对应的每一个`Endpoints`，即每一个`Pod`，都会有对应的`DNS`域名；这样`Pod`之间就可以互相访问。我们还是看上面的这个例子。

### 3.4 无头服务声明注意事项

最后强调一点就是，无头服务名称需与有状态副本集的的serviceName相同，用于hostname传播访问pod

且如果有状态副本集不同节点需要互通时，需要使用`当headless service有配置selector时，其对应的所有后端节点，会被记录到dns中`特性，因此其无头服务必须声明。

以上文中环境变量为例

```yaml
         # 若在ConfigMap中设置了service_name，则此处无需再次设置，注意环境变量声名的顺序
        - name: K8S_SERVICE_NAME
          value: rabbitmq-headless
        
        ## 节点名称命名规则，注意此处节点名称必须符合无头服务的命名规则，比如此处为rmq-cluster.rabbitmq
        ## 注意之前随意改变了有状态副本集的名称serviceName，并且同步修改了无头服务的对应字段，但是没有修改此处的$(K8S_SERVICE_NAME)名称，造成提示
        ## "ERROR: epmd error for host rmq-cluster-0.rmq-cluster.public-service.svc.cluster.local: nxdomain (non-existing domain)"错误
        - name: RABBITMQ_NODENAME
          value: rabbit@$(POD_NAME).$(K8S_SERVICE_NAME).$(POD_NAMESPACE).svc.cluster.local
```



即以下几个变量名称必须相同：

- 1、无头服务的name字段

- 2、有状态副本集的serviceName字段

- 3、有状态副本集中容器对应的K8S_SERVICE_NAME环境变量



***本文到底结束，如有描述不当，还望谅解指正！***

源文件对应github地址待整理更新。



## 附 参考文献

- [浅析k8s service的应用](https://segmentfault.com/a/1190000013840788)---说明K8S服务类型和区别
- [K8S容器编排之Headless浅谈](https://www.jianshu.com/p/a6d8b28c88a2) -- 用nslookup说明无头服务
- [k8s部署kafka集群](https://www.cnblogs.com/cuishuai/p/10243291.html) -- 描述了Kafka集群K8S部署文件和有状态副本集的应用场景和组成