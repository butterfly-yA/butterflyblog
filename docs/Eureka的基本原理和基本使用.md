---
title: Eureka的基本使用和Ribbon的原理浅析
date: '2022-08-07'
categories:   # 分类
 - SpringCloud
tags:   # 标签
 - Eureka
 - Ribbon
 - 注册中心
 - SpringCloud
publish: true   # 发布
next: ./Nacos的基本使用
---


<!-- more -->


# Eureka的基本原理和基本使用

## 1、简介

Eureka是Netflix(网飞)开源的一个spring cloud组件，主要功能是注册中心。

>关于注册中心

springcloud是一个非常优秀的微服务框架，要管理众多的服务，就需要对这些服务进行治理，也就是我们说的`服务治理`,`服务治理`的作用就是在传统的rpc远程调用框架中，管理每个服务与每个服务之间的依赖关系，可以实现服务调用、负载均衡、服务容错、以及服务的注册与发现。
 ​       如果微服务之间存在调用依赖，就需要得到目标服务的服务地址，也就是微服务治理的`服务发现`。要完成服务发现，就需要将服务信息存储到某个载体，载体本身即是微服务治理的`服务注册中心`，而存储到载体的动作即是`服务注册`。
 ​       springcloud支持的注册中心有`Eureka`、`Zookeeper`、`Consul`、`Nacos`

>springcloud支持的注册中心

| 组件名称  | 所属公司  | 组件简介                                                     |
| :-------: | --------- | ------------------------------------------------------------ |
|  Eureka   | Netflix   | springcloud最早的注册中心，目前已经进入`停更进维`了          |
| Zookeeper | Apache    | zookeeper是一个分布式协调工具，可以实现注册中心功能          |
|  Consul   | Hashicorp | Consul 简化了分布式环境中的服务的注册和发现流程，通过 HTTP 或者 DNS 接口发现。支持外部 SaaS 提供者等。 |
|   Nacos   | Alibaba   | Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。 |

## 2、基本使用

>注意本文基于2.3.9.RELEASE的springboot和Hoxton.SR10的springcloud

### 2.1、项目初始化

![20210901090745](http://img.sybutterfly.top/img/202211142056215.webp)

cloud-demo：父工程，管理依赖

- order-service：订单微服务，负责订单相关业务
- user-service：用户微服务，负责用户相关业务

要求：

- 订单微服务和用户微服务都必须有**各自的数据库**，相互独立
- 订单服务和用户服务**都对外暴露 Restful 的接口**
- 订单服务如果需要查询用户信息，**只能调用用户服务的 Restful 接口**，不能查询用户数据库

>这里给出初始项目地址，请拉取最初提交记录，数据库文件也在，自建数据库

[spring-cloud-learning demo](https://gitee.com/yangAgitee7/spring-cloud-learning/tree/bda59f8ad95ca43cb99fac64214b31326cb4e036/)

### 2.2、初始化运行

>IDEA中打开services方便多服务启动

![image-20221028210729372](http://img.sybutterfly.top/img/202211142056444.png)

> 启动两个项目,浏览器访问测试用例即可

![image-20221028210849428](http://img.sybutterfly.top/img/202211142056066.png)

这里我们可以看见order服务并没有查询出我们需要的user信息，那么在单体架构下，我们可以直接调用对应的userservice进行数据库查询，但在微服务架构下如何查询呢？

这里就涉及到宁一个微服务的特点了，对外暴露接口，也就是说我们可以通过接口进行服务的调用，这里我们需要一个调用服务的工具---==**RestTemplate**==类，这也是spring提供的一个http访问类

### 2.3、RestTemplate调用其他服务

> 修改order主类或者配置类，将RestTemplate注入IOC容器

```java
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }

    @Bean
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

>修改orderservice代码如下

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    //注入RestTemplate对象
    @Autowired
    private RestTemplate restTemplate;

    public Order queryOrderById(Long orderId) {
        // 1.查询订单
        Order order = orderMapper.findById(orderId);
        //2.调用restTemplate调用userService获取user信息
        String url="http://localhost:8081/user/"+order.getUserId();
        User user = restTemplate.getForObject(url, User.class);
//        //3.设置信息
        order.setUser(user);
        // 4.返回
        return order;
    }
}
```

>重启order服务后，再次测试，数据到达预期

![image-20221028211459473](http://img.sybutterfly.top/img/202211142056845.png)

但是新的问题又出现了，这里我们进行了硬编码，而且在集群的环境下，你不能这样仅仅写一个接口地址在这里，那么我们就需要一个工具帮助我们管理这些繁多的微服务，也就我们说的注册中心

![image-20221028211650858](http://img.sybutterfly.top/img/202211142056027.png)

### 2.4、Eureka Service的编写

#### 2.4.1、新建一个maven工程eureka-service,并导入依赖

```xml
<!--        euraka服务端依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

#### 2.4.2、编写配置文件

```yml
server:
  port: 10086
spring:
  application:
    name: eureka-service
eureka:
  client:
    service-url:  # 这是euraka服务地址，集群环境下，每个eureka也是需要注册的，多个地址用,隔开
      defaultZone: http://127.0.0.1:10086/eureka
```

#### 2.4.3、编写主类

```java
@SpringBootApplication
@EnableEurekaServer    //开启eureka服务端
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class,args);
    }
}
```

#### 2.4.4、启动该类，访问10086

![image-20221028212712720](http://img.sybutterfly.top/img/202211142056516.png)

>可以看到当前存在一个已注册的服务，就是它本身，UP表示存活，后面是地址

### 2.5、Euraka Client编写

>修该user和order类

#### 2.5.1、pom文件添加依赖

```xml
<!--        eureka客户端-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

#### 2.5.2、修改启动类或者配置类

添加注解`@EnableEurekaClient`声明该类是一个Eureka客户端

#### 2.5.3、修改RestTemplate

```java
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication
@EnableEurekaClient
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }

    @Bean
    @LoadBalanced   //开启负载均衡
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

#### 2.5.4、修改orderservice

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private RestTemplate restTemplate;

    public Order queryOrderById(Long orderId) {
        // 1.查询订单
        Order order = orderMapper.findById(orderId);
        //2.调用restTemplate调用userService获取user信息
//        String url="http://localhost:8081/user/"+order.getUserId();
        //这里就是服务名
        String url="http://userservice/user/"+order.getUserId();
        User user = restTemplate.getForObject(url, User.class);
//        //3.设置信息
        order.setUser(user);
        // 4.返回
        return order;
    }
}
```

#### 2.5.5、修该配置文件

```yml
# 注意两个微服务需要spring: application:name: 服务名

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10086/eureka  # 声明这个服务去哪里注册
```

>重启服务，注意先启动注册中心，在启动其他两个服务，可以看见两个服务已经注册

![image-20221028213640192](http://img.sybutterfly.top/img/202211142057268.png)

>测试，可以发现我们这样依旧能够调用服务访问数据

![image-20221028213922392](http://img.sybutterfly.top/img/202211142057913.png)

### 2.6、原理浅析

Eureka将注册我们的服务，开启` @LoadBalanced`后，spring将启用 `Ribbon` 来负责负载均衡，而这个类也负责了（通过拦截请求）服务的拉取和服务名到真实地址的解析，更多的将在下一节说明。

### 2.7、客户端集群，多实例

![image-20221029091506587](http://img.sybutterfly.top/img/202211142057794.png)

>修改配置，8082端口

![image-20221029091557806](http://img.sybutterfly.top/img/202211142057310.png)

>测试，成功，并且控制台两个服务都在打印日志，开启了负载均衡

![image-20221029091703645](http://img.sybutterfly.top/img/202211142057412.png)

## 3.工作流程浅析

![20210901090919](http://img.sybutterfly.top/img/202211142057780.webp)

**order-service 如何得知 user-service 实例地址？**

- user-service 服务实例启动后，将自己的信息注册到 eureka-server(Eureka服务端)，叫做**服务注册**
- eureka-server 保存服务名称到服务实例地址列表的映射关系
- order-service 根据服务名称，拉取实例地址列表，这个叫**服务发现**或服务拉取

**order-service 如何从多个 user-service 实例中选择具体的实例？**

order-service从实例列表中利用**负载均衡算法**选中一个实例地址，向该实例地址发起远程调用

**order-service 如何得知某个 user-service 实例是否依然健康，是不是已经宕机？**

- user-service 会**每隔一段时间(默认30秒)向 eureka-server 发起请求**，报告自己状态，称为**心跳**
- 当超过一定时间没有发送心跳时，eureka-server 会认为微服务实例故障，将该实例从服务列表中剔除
- order-service 拉取服务时，就能将故障实例排除了

# Ribbon负载均衡原理和源码跟踪

## 1、流程浅析

我们添加了 `@LoadBalanced` 注解，即可实现负载均衡功能，这是什么原理呢？

**SpringCloud 底层提供了一个名为 Ribbon 的组件，来实现负载均衡功能。**

![20210901091242](http://img.sybutterfly.top/img/202211142057532.webp)

>请求并不是直接与eureka注册中心交互，而是通过Ribbon组件，利用Ribbon实现了负载均衡和服务的拉取

## 2、源码跟踪

为什么我们只输入了 service 名称就可以访问了呢？为什么不需要获取ip和端口，这显然有人帮我们根据 service 名称，获取到了服务实例的ip和端口。它就是`LoadBalancerInterceptor`，这个类会在对 RestTemplate 的请求进行拦截，然后从 Eureka 根据服务 id 获取服务列表，随后利用负载均衡算法得到真实的服务地址信息，替换服务 id。

我们进行源码跟踪：

![20210901091323](http://img.sybutterfly.top/img/202211142057210.webp)



这里的 `intercept()` 方法，拦截了用户的 HttpRequest 请求，然后做了几件事：

- `request.getURI()`：获取请求uri，即 http://user-service/user/8
- `originalUri.getHost()`：获取uri路径的主机名，其实就是服务id `user-service`
- `this.loadBalancer.execute()`：处理服务id，和用户请求

这里的 `this.loadBalancer` 是 `LoadBalancerClient` 类型

继续跟入 `execute()` 方法：

![image-20221029094546700](http://img.sybutterfly.top/img/202211142057136.png)

- `getLoadBalancer(serviceId)`：根据服务id获取 `ILoadBalancer`，而 `ILoadBalancer` 会拿着服务 id 去 eureka 中获取服务列表。
- `getServer(loadBalancer)`：利用内置的负载均衡算法，从服务列表中选择一个。在图中**可以看到获取了8082端口的服务**

可以看到获取服务时，通过一个 `getServer()` 方法来做负载均衡:

![image-20221029094620584](http://img.sybutterfly.top/img/202211142057939.png)



我们继续跟入：

![image-20221029094637340](http://img.sybutterfly.top/img/202211142057922.png)



继续跟踪源码 `chooseServer()` 方法，发现这么一段代码：

![image-20221029094655005](http://img.sybutterfly.top/img/202211142057888.png)

我们看看这个 `rule` 是谁：

![image-20221029094705482](http://img.sybutterfly.top/img/202211142057083.png)



这里的 rule 默认值是一个 `RoundRobinRule` ，看类的介绍：

![image-20221029094717862](http://img.sybutterfly.top/img/202211142057369.png)



负载均衡默认使用了轮训算法，当然我们也可以自定义。

## 3、流程总结

SpringCloud Ribbon 底层采用了一个拦截器，拦截了 RestTemplate 发出的请求，对地址做了修改。

基本流程如下：

- 拦截我们的 `RestTemplate` 请求 http://userservice/user/1
- `RibbonLoadBalancerClient` 会从请求url中获取服务名称，也就是 user-service
- `DynamicServerListLoadBalancer` 根据 user-service 到 eureka 拉取服务列表
- eureka 返回列表，localhost:8081、localhost:8082
- `IRule` 利用内置负载均衡规则，从列表中选择一个，例如 localhost:8081
- `RibbonLoadBalancerClient` 修改请求地址，用 localhost:8081 替代 userservice，得到 http://localhost:8081/user/1，发起真实请求

![image-20221029094729593](http://img.sybutterfly.top/img/202211142057076.png)



## 4、负载均衡策略

负载均衡的规则都定义在 IRule 接口中，而 IRule 有很多不同的实现类：

![image-20221029094742846](http://img.sybutterfly.top/img/202211142057354.png)



不同规则的含义如下：

| **内置负载均衡规则类**    | **规则描述**                                                 |
| :------------------------ | :----------------------------------------------------------- |
| RoundRobinRule            | 简单轮询服务列表来选择服务器。它是Ribbon默认的负载均衡规则。 |
| AvailabilityFilteringRule | 对以下两种服务器进行忽略：（1）在默认情况下，这台服务器如果3次连接失败，这台服务器就会被设置为“短路”状态。短路状态将持续30秒，如果再次连接失败，短路的持续时间就会几何级地增加。 （2）并发数过高的服务器。如果一个服务器的并发连接数过高，配置了AvailabilityFilteringRule 规则的客户端也会将其忽略。并发连接数的上限，可以由客户端设置。 |
| WeightedResponseTimeRule  | 为每一个服务器赋予一个权重值。服务器响应时间越长，这个服务器的权重就越小。这个规则会随机选择服务器，这个权重值会影响服务器的选择。 |
| **ZoneAvoidanceRule**     | 以区域可用的服务器为基础进行服务器的选择。使用Zone对服务器进行分类，这个Zone可以理解为一个机房、一个机架等。而后再对Zone内的多个服务做轮询。 |
| BestAvailableRule         | 忽略那些短路的服务器，并选择并发数较低的服务器。             |
| RandomRule                | 随机选择一个可用的服务器。                                   |
| RetryRule                 | 重试机制的选择逻辑                                           |

默认的实现就是 `ZoneAvoidanceRule`，**是一种轮询方案**。

### 4.1、自定义策略

通过定义 IRule 实现可以修改负载均衡规则，有两种方式：

1 、代码方式在 order-service 中的 OrderApplication 类中，定义一个新的 IRule：

```java
@Bean
public IRule randomRule(){
    return new RandomRule();
}
```



2 、配置文件方式：在 order-service 的 application.yml 文件中，添加新的配置也可以修改规则：

```yml
userservice: # 给需要调用的微服务配置负载均衡规则，orderservice服务去调用userservice服务
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule # 负载均衡规则 
```

**注意**：一般用默认的负载均衡规则，不做修改。

## 5、饥饿加载

当我们启动 orderservice，第一次访问时，时间消耗会大很多，这是因为 Ribbon 懒加载的机制。

![20210901091850](http://img.sybutterfly.top/img/202211142057231.webp)



Ribbon 默认是采用懒加载，即第一次访问时才会去创建 LoadBalanceClient，拉取集群地址，所以请求时间会很长。

而饥饿加载则会在项目启动时创建 LoadBalanceClient，降低第一次访问的耗时，通过下面配置开启饥饿加载：

```yml
ribbon:
  eager-load:
    enabled: true
    clients: userservice # 项目启动时直接去拉取userservice的集群，多个用","隔开
```

