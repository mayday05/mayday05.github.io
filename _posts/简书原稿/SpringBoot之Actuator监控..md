

## 一、Spring Boot Actuator作用

### 1.1 Spring Boot Actuator作用：

- [ ] 健康检查
- [ ] 审计
- [ ] 统计
- [ ] 监控
- [ ] HTTP追踪

​         Actuator同时还可以与外部应用监控系统整合，比如 [Prometheus](https://prometheus.io/), [Graphite](https://graphiteapp.org/), [DataDog](https://www.datadoghq.com/), [Influx](https://www.influxdata.com/), [Wavefront](https://www.wavefront.com/), [New Relic](https://newrelic.com/)等。这些系统提供了非常好的仪表盘、图标、分析和告警等功能，使得你可以通过统一的接口轻松的监控和管理你的应用。

​        Actuator使用[Micrometer](http://micrometer.io/)来整合上面提到的外部应用监控系统。这使得只要通过非常小的配置就可以集成任何应用监控系统。

​        以下是一些非常有用的actuator endpoints列表，可以在[official documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)上面看到完整的列表。

| Endpoint ID    | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| auditevents    | 显示应用暴露的审计事件 (比如认证进入、订单失败)              |
| info           | 显示应用的基本信息                                           |
| health         | 显示应用的健康状态                                           |
| metrics        | 显示应用多样的度量信息                                       |
| loggers        | 显示和修改配置的loggers                                      |
| logfile        | 返回log file中的内容(如果logging.file或者logging.path被设置) |
| httptrace      | 显示HTTP足迹，最近100个HTTP request/repsponse                |
| env            | 显示当前的环境特性                                           |
| flyway         | 显示数据库迁移路径的详细信息                                 |
| liquidbase     | 显示Liquibase 数据库迁移的纤细信息                           |
| shutdown       | 让你逐步关闭应用                                             |
| mappings       | 显示所有的@RequestMapping路径                                |
| scheduledtasks | 显示应用中的调度任务                                         |
| threaddump     | 执行一个线程dump                                             |
| heapdump       | 返回一个GZip压缩的JVM堆dump                                  |



### 1.2  打开和关闭Actuator Endpoints

**默认，只有`health`和`info`通过HTTP暴露了出来**

默认，上述所有的endpoints都是打开的，除了`shutdown` endpoint。

你可以通过设置`management.endpoint..enabled to true or false`(`id`是endpoint的id)来决定打开还是关闭一个actuator endpoint。

举个例子，要想打开`shutdown` endpoint，增加以下内容在你的`application.properties`文件中：

```properties
management.endpoint.shutdown.enabled=true
```



### 1.3  暴露Actuator Endpoints

默认，素偶偶的actuator endpoint通过JMX被暴露，而通过HTTP暴露的只有`health`和`info`。

以下是你可以通过应用的properties可以通过HTTP和JMX暴露的actuator endpoint。

- 通过HTTP暴露Actuator endpoints。

  ```properties
  # Use "*" to expose all endpoints, or a comma-separated list to expose selected ones
  management.endpoints.web.exposure.include=health,info 
  management.endpoints.web.exposure.exclude=
  
  ```

  

- 通过JMX暴露Actuator endpoints。

  ```properties
  # Use "*" to expose all endpoints, or a comma-separated list to expose selected ones
  management.endpoints.jmx.exposure.include=*
  management.endpoints.jmx.exposure.exclude=
  
  ```

  

### 1.4 使用Spring Security来保证Actuator Endpoints安全

Actuator endpoints是敏感的，必须保障进入是被授权的。如果Spring Security是包含在你的应用中，那么endpoint是通过HTTP认证被保护起来的。

如果没有， 你可以增加以下以来到你的应用中去：

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>

```

接下去看一下如何覆写spring security配置，并且定义你自己的进入规则。下面的例子展示了一个简单的spring security配置。它使用叫做[`EndPointRequest`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/actuate/autoconfigure/security/servlet/EndpointRequest.html)的`ReqeustMatcher`工厂模式来配置Actuator endpoints进入规则。

```java
package com.example.actuator.config;

import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.boot.actuate.context.ShutdownEndpoint;
import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {

    /*
        This spring security configuration does the following

        1. Restrict access to the Shutdown endpoint to the ACTUATOR_ADMIN role.
        2. Allow access to all other actuator endpoints.
        3. Allow access to static resources.
        4. Allow access to the home page (/).
        5. All other requests need to be authenticated.
        5. Enable http basic authentication to make the configuration complete.
           You are free to use any other form of authentication.
     */

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .requestMatchers(EndpointRequest.to(ShutdownEndpoint.class))
                        .hasRole("ACTUATOR_ADMIN") // 角色
                    .requestMatchers(EndpointRequest.toAnyEndpoint())
                        .permitAll()
                    .requestMatchers(PathRequest.toStaticResources().atCommonLocations())
                        .permitAll()
                    .antMatchers("/")
                        .permitAll()
                    .antMatchers("/**")
                        .authenticated()
                .and()
                .httpBasic();
    }
}

```

为了能够测试以上的配置，你可以在`application.yaml`中增加spring security用户。

```yaml
# Spring Security Default user name and password
spring:
  security:
    user:
      name: actuator
      password: actuator
      roles: ACTUATOR_ADMIN

```

完整代码见 [Github](https://github.com/bigjar/actuator-demo)

## 二、如何使用

启用 Actuator 最简单方式是添加 **spring-boot-starter-actuator** ‘Starter’依赖。 

注意：不同版本对应的yaml或者properties参数不同。

###   1、pom.xml

```
<!-- 2、监控 —— Actuator插件 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

###   2、application.yml

spring-boot-starter-actuator对应2.0版本以上

```yaml
management:
  endpoints:
    # 暴露 EndPoint 以供访问，有jmx和web两种方式，exclude 的优先级高于 include
    jmx:
      exposure:
        exclude: '*'
        include: '*'
    web:
      exposure:
      # exclude: '*'
        include: ["health","info","beans","mappings","logfile","metrics","shutdown","env"]
      base-path: /actuator  # 配置 Endpoint 的基础路径
      cors: # 配置跨域资源共享
        allowed-origins: http://example.com
        allowed-methods: GET,POST
    enabled-by-default: true # 修改全局 endpoint 默认设置
  endpoint:
    auditevents: # 1、显示当前引用程序的审计事件信息，默认开启
      enabled: true
      cache:
        time-to-live: 10s # 配置端点缓存响应的时间
    beans: # 2、显示一个应用中所有 Spring Beans 的完整列表，默认开启
      enabled: true
    conditions: # 3、显示配置类和自动配置类的状态及它们被应用和未被应用的原因，默认开启
      enabled: true
    configprops: # 4、显示一个所有@ConfigurationProperties的集合列表，默认开启
      enabled: true
    env: # 5、显示来自Spring的 ConfigurableEnvironment的属性，默认开启
      enabled: true
    flyway: # 6、显示数据库迁移路径，如果有的话，默认开启
      enabled: true
    health: # 7、显示健康信息，默认开启
      enabled: true
      show-details: always
    info: # 8、显示任意的应用信息，默认开启
      enabled: true
    liquibase: # 9、展示任何Liquibase数据库迁移路径，如果有的话，默认开启
      enabled: true
    metrics: # 10、展示当前应用的metrics信息，默认开启
      enabled: true
    mappings: # 11、显示一个所有@RequestMapping路径的集合列表，默认开启
      enabled: true
    scheduledtasks: # 12、显示应用程序中的计划任务，默认开启
      enabled: true
    sessions: # 13、允许从Spring会话支持的会话存储中检索和删除(retrieval and deletion)用户会话。使用Spring Session对反应性Web应用程序的支持时不可用。默认开启。
      enabled: true
    shutdown: # 14、允许应用以优雅的方式关闭，默认关闭
      enabled: true
    threaddump: # 15、执行一个线程dump
      enabled: true
    # web 应用时可以使用以下端点
    heapdump: # 16、    返回一个GZip压缩的hprof堆dump文件，默认开启
      enabled: true
    jolokia: # 17、通过HTTP暴露JMX beans（当Jolokia在类路径上时，WebFlux不可用），默认开启
      enabled: true
    logfile: # 18、返回日志文件内容（如果设置了logging.file或logging.path属性的话），支持使用HTTP Range头接收日志文件内容的部分信息，默认开启
      enabled: true
    prometheus: #19、以可以被Prometheus服务器抓取的格式显示metrics信息，默认开启
      enabled: true
```

​        这大抵就是全部默认的 Endpoint  的配置了，怎么样？强大吧！之前做了一个网络监控的项目，就是能够实时查看服务器的 CPU、内存、磁盘、IO 这些（基于 sigar.jar  实现），然后现在发现 Spring-Boot 就这样轻松支持了，还更强大，更简便......

​         默认的 Endpoint 映射前缀是 **/actuator**（2.0版本以上已经默认没有base-path为"/"了，注意），可以通过如上 base-path 自定义设置。

​         每个 Endpoint 都可以配置开启或者禁用。但是仅仅开启 Endpoint 是不够的，还需要通过 jmx 或者 web 暴露他们，通过 exclude 和 include 属性配置。

## 三、深入学习资料

### 1、官网地址

Spring Boot Actuator: Production-ready Features 见 https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-endpoints

2、[Spring Boot Actuator: Production-ready features](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready.html)

3、[Micrometer: Spring Boot 2’s new application metrics collector](https://spring.io/blog/2018/03/16/micrometer-spring-boot-2-s-new-application-metrics-collector)



## 附 参考文章:

1、https://www.jianshu.com/p/d5943e303a1f

2、[Spring Boot Actuator: Health check, Auditing, Metrics gathering and Monitoring](https://www.callicoder.com/spring-boot-actuator/)