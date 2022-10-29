---
title: Nacos的基本使用
date: '2021-08-07'
categories:   # 分类
 - SpringCloud
tags:   # 标签
 - Nacos
 - 注册中心
 - SpringCloud
publish: true   # 发布
prev: ./Eureka的基本原理和基本使用
---



# Nacos的使用

[Nacos](https://nacos.io/)是阿里巴巴的产品，现在是[SpringCloud](https://spring.io/projects/spring-cloud)中的一个组件。相比[Eureka](https://github.com/Netflix/eureka)功能更加丰富，在国内受欢迎程度较高。

![image-20221029095822023](https://i0.hdslb.com/bfs/album/9071ef00f63a99549b03476b1126172b5bbd1c24.png)

## 1、Windows安装

开发阶段采用单机安装即可。

### 1.1、下载安装包

在Nacos的GitHub页面，提供有下载链接，可以下载编译好的Nacos服务端或者源代码：

GitHub主页：https://github.com/alibaba/nacos

GitHub的Release下载页：https://github.com/alibaba/nacos/releases

如图：

![image-20221029100409396](https://i0.hdslb.com/bfs/album/0cc1696f7b852d71d731112937e9890fe7154c99.png)

本文采用1.4.1.版本的Nacos---`nacos-server-1.4.1.zip`，下载解压安装即可，注意路径不能有中文



1.2、目录说明：

- bin：启动脚本
- conf：配置文件

![image-20221029100553554](https://i0.hdslb.com/bfs/album/f8539cf33de369ff2ab38a0d8febd0cdb7fe3822.png)

### 1.2、端口配置

Nacos的默认端口是8848，如果你电脑上的其它进程占用了8848端口，`8848钛金手机`，请先尝试关闭该进程。

**如果无法关闭占用8848端口的进程**，也可以进入nacos的conf目录，修改配置文件中的端口：

![image-20221029100609351](https://i0.hdslb.com/bfs/album/f6c8c5ca5c559f98ab4357e58d31f7ee4ef6b68a.png)



### 1.3、.启动

启动非常简单，进入bin目录，结构如下：

![image-20221029100624459](https://i0.hdslb.com/bfs/album/bab2430e28cb5b99192ffd3adc8cd685f3e1f562.png)

然后执行命令即可：

- windows命令：

  ```
  startup.cmd -m standalone
  ```


执行后的效果如图：

![image-20221029100636931](https://i0.hdslb.com/bfs/album/602bd92b69f3d252eae88fa6d83aeb2e7be76ea8.png)

>Nacos本身也是一个SpringBoot项目，所以就不再需要去额外搭建一个像 Eureka 的注册中心的服务端



### 1.4、访问

在浏览器输入地址：http://127.0.0.1:8848/nacos即可：

![image-20221029100704797](https://i0.hdslb.com/bfs/album/8f0340f818fbb7b882f86022c998ba4eacab2d7b.png)

默认的账号和密码都是nacos，进入后：

![image-20221029101453276](https://i0.hdslb.com/bfs/album/091f7b34ee8e0060db867aeecf795f7631dcdf39.png)



## 2、Nacos的使用

>基于Eureka一文中提到的项目，注释掉所有Eureka的相关代码

父工程：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.5.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

客户端：

```xml
<!-- nacos客户端依赖包 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

```

### 2.1、初始化

#### 2.1.2、配置文件

>注释Eureka的地址

```yml
spring:    # 注册的nacos地址，多个,隔开
  cloud:
    nacos:
      server-addr: localhost:8848
```

**重启服务，可以看到已经注册到nacos**

![image-20221029104742282](https://i0.hdslb.com/bfs/album/ee0d39c12a0b9526de4e0df7325f1f0153a9f217.png)



## 3、服务分级存储模型

一个**服务**可以有多个**实例**，例如我们的user-service，可以有:

- 127.0.0.1:8081
- 127.0.0.1:8082
- 127.0.0.1:8083

假如这些实例分布于全国各地的不同机房，例如：

- 127.0.0.1:8081，在上海机房
- 127.0.0.1:8082，在上海机房
- 127.0.0.1:8083，在杭州机房

Nacos就将同一机房内的实例 划分为一个**集群**。

也就是说，user-service是服务，一个服务可以包含多个集群，如杭州、上海，每个集群下可以有多个实例，形成分级模型，如图：

![image-20221029105436634](https://i0.hdslb.com/bfs/album/6246959a682e03a3d5ed9de33224af3124e8ec85.png)

微服务互相访问时，应该尽可能访问同集群实例，因为本地访问速度更快。当本集群内不可用时，才访问其它集群。例如：

![image-20221029105450914](https://i0.hdslb.com/bfs/album/ad35aad770d9355af745a8e34e108499d8ac4746.png)

杭州机房内的order-service应该优先访问同机房的user-service。



### 3.1、给user-service配置集群

#### 3.1.1、修改配置文件

```yml
# user-service的application.yml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: HZ # 集群名称
# order-service
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: HZ # 集群名称
```

>在新开一个user-service实例，端口8083，集群为上海

![image-20221029111652092](https://i0.hdslb.com/bfs/album/c994367bbd8fddcc0fd3460bbcc5c9f8970a3b83.png)

>查看Nacos控制台，存在两个集群

![image-20221029111737493](https://i0.hdslb.com/bfs/album/88baf4e066b34807de9f9cbab3a4bd8c646852c2.png)

>发送测试请求，来个十几次，http://localhost:8080/order/101

可以发送IDEA控制台打印并没有实现优先同一集群访问，因为默认的负载均衡方案，`ZoneAvoidanceRule`并不能实现根据同集群优先

因此Nacos中提供了一个`NacosRule`的实现，可以优先从同集群中挑选实例。

#### 3.1.2、配置oder-service负载均衡策略

>如果采用@bean注入，那么这个策略将全局启用，不需要启用使用配置文件

```yml
userservice:
  ribbon:
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule # 负载均衡规则 
```

>再次请求测试，发现所有请求都打到了同集群的服务上

![image-20221029112449022](https://i0.hdslb.com/bfs/album/471b371608eea70356b7e1ef40f94abee3c58881.png)

那么如果同集群的服务都挂了咋办？他会自动访问其他集群的服务

## 4、权重配置

实际部署中会出现这样的场景：

服务器设备性能有差异，部分实例所在机器性能较好，另一些较差，我们希望性能好的机器承担更多的用户请求。

但默认情况下NacosRule是同集群内随机挑选，不会考虑机器的性能问题。



因此，Nacos提供了权重配置来控制访问频率，权重越大则访问频率越高。



在nacos控制台，找到user-service的实例列表，点击编辑，即可修改权重：

>**注意**：如果权重修改为0，则该实例永远不会被访问

![image-20221029112706179](https://i0.hdslb.com/bfs/album/56ccec9e0cb1ed5405ee1d8941ccfceae5156e3d.png)

## 5、环境隔离

Nacos提供了namespace来实现环境隔离功能。

- nacos中可以有多个namespace
- namespace下可以有group、service等
- 不同namespace之间相互隔离，例如不同namespace的服务互相不可见

![image-20221029114111074](https://i0.hdslb.com/bfs/album/764b2ab91940f9a239a8a84b9c9c52fe302fe86c.png)

### 5.1、创建namespace

默认情况下，所有service、data、group都在同一个namespace，名为public：

![image-20221029114155446](https://i0.hdslb.com/bfs/album/215c8d4a128acbae5ecb1d9a7c41f7a056b5a55f.png)

>这里新建一个命名空间

![image-20221029114244497](https://i0.hdslb.com/bfs/album/0a7505495c8cffed1761bc8c94fa9293728317f4.png)



5.2、配置命名空间

### 5.2、给微服务配置namespace

给微服务配置namespace只能通过修改配置来实现。

例如，修改order-service的application.yml文件：

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: HZ
        namespace: e4eae97c-6230-4271-840b-a1e198ed9f80 # 命名空间，填ID
```



重启order-service后，访问控制台，可以看到下面的结果：

![image-20221029114327199](https://i0.hdslb.com/bfs/album/e2db6a916d60a6be4d14176bc146d9ac852d329a.png)



![image-20221029114335878](https://i0.hdslb.com/bfs/album/eeb543ce52044c2df5fbebb28f4ab6c33376c792.png)

此时访问order-service，因为namespace不同，会导致找不到userservice，控制台会报错：

![image-20221029114342301](https://i0.hdslb.com/bfs/album/05c49366cf268f41cd111991e8692bc78b1f3cf9.png)



## 6.Nacos与Eureka的区别

Nacos的服务实例分为两种l类型：

- 临时实例：如果实例宕机超过一定时间，会从服务列表剔除，默认的类型。

- 非临时实例：如果实例宕机，不会从服务列表剔除，也可以叫永久实例。



配置一个服务实例为永久实例：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        ephemeral: false # 设置为非临时实例
```





Nacos和Eureka整体结构类似，服务注册、服务拉取、心跳等待，但是也存在一些差异：

![image-20221029114401682](https://i0.hdslb.com/bfs/album/c66c88dadac8ae8605cf9769c5ed5c49eb5a3181.png)



- Nacos与eureka的共同点
  - 都支持服务注册和服务拉取
  - 都支持服务提供者心跳方式做健康检测

- Nacos与Eureka的区别
  - Nacos支持服务端主动检测提供者状态：临时实例采用心跳模式，非临时实例采用主动检测模式
  - 临时实例心跳不正常会被剔除，非临时实例则不会被剔除
  - Nacos支持服务列表变更的消息推送模式，服务列表更新更及时
  - Nacos集群默认采用AP方式，当集群中存在非临时实例时，采用CP模式；Eureka采用AP方式





