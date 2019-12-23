

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

#### 安装deb

```
sudo dpkg -i arthas*.deb
```

#### 安装rpm

```
sudo rpm -i arthas*.rpm
```

#### deb/rpm安装的用法

在安装后，可以直接执行：

```
as.sh
```



### 1.3 如何在docker中使用

#### 诊断Docker里的Java进程

```
docker exec -it  ${containerId} /bin/bash -c "wget https://alibaba.github.io/arthas/arthas-boot.jar && java -jar arthas-boot.jar"
```

#### 诊断k8s里容器里的Java进程

```
kubectl exec -it ${pod} --container ${containerId} -- /bin/bash -c "wget https://alibaba.github.io/arthas/arthas-boot.jar && java -jar arthas-boot.jar"
```

#### 把Arthas安装到基础镜像里

可以很简单把Arthas安装到你的Docker镜像里。

```
FROM openjdk:8-jdk-alpine

# copy arthas
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
```

## 二、如何使用watch功能

![**arthas-watch1.png**](https://github.com/mayday05/mayday05.github.io/blob/master/image2/arthas-watch1.png)

#### 参数说明

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

## 三、常见用户案例

Github完整案例见[用户案例](https://github.com/alibaba/arthas/issues?q=label%3Auser-case),对应项目issue中使用label:user-case 搜索结果

筛选出部分和自己相关的部分案例如下

| 案例标题                                                     | 使用特性                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Arthas协助排查线上skywalking不可用问题  #Issue557](https://github.com/alibaba/arthas/issues/557) | 该案例使用了watch特性，能够动态监控指定类、指定方法、输出的指定结果 |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |
|                                                              |                                                              |



# 





参考文献

1、 