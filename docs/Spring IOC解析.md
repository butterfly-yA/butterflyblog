---
title: Spring IOC浅析
date: '2022-08-26'
categories:   # 分类
 - Spring
tags:   # 标签
 - IOC
publish: true   # 发布
---

<p style="color:#7f7f7f">Spring IOC是什么，BeanFactory和ApplicationContext，BeanFactory和FactoryBean，Bean的生命周期以及提供的扩展点，BeanDefinition</p>

<!-- more -->
## 0、说明

这篇文章只是第一篇，但是可能不会再出第二篇了，下面的参考原文其实已经把IOC描述的很清晰了。

参考文章，原文更精彩。

[ Spring IOC详解 以及 Bean生命周期详细过程_java叶新东老师的博客-CSDN博客](https://blog.csdn.net/qq_27184497/article/details/117173285)

[IOC（一）BeanFactory和BeanDefinition - 掘金 (juejin.cn)](https://juejin.cn/post/6844903846867632136)

[Spring（一）开篇 - 龙四丶 - 博客园 (cnblogs.com)](https://www.cnblogs.com/loongk/p/12194690.html)

[Spring IoC 源码分析 (基于注解) 一 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1495512)

[Spring IoC之BeanDefinitionReader - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/107839916)

## 1、IOC简介

`spring版本5.1.3.RELEASE`

IOC即控制反转，也可以称为依赖注入（`DI`），实际上依赖注入是IOC的别名。

IOC和AOP（切面）是Spring的两大特点。

### 1.1、什么是控制反转

在原来都是程序员自个儿构建对象，通过new 或者 反射等方式，那么控制反转就是由IOC容器来创建和销毁对象，程序员无需关心这些个流程，只重点关注业务即可

![image-20221115190803755](http://img.sybutterfly.top/img202211192334719.png)

### 1.2、什么是依赖注入

依赖注入就是给予调用方它所需要的事物，是指在 Spring IOC 容器创建对象的过程中，将所依赖的对象通过配置进行注入，比如如下代码，一个玩家类需要一把武器，这里直接参数传递了一个剑给他，但是这样使用玩家类，如果以后需要将武器换为枪，那么所有涉及到玩家初始化的地方都需要更改，这样就是紧耦合

```java
class Player{  
    Weapon weapon;  
    Player(Sword sword){  
        // 与 Sword类紧密耦合
        this.weapon = sword;  
    }  
    public void attack() {
        weapon.attack();
    }
}  
```

那么基于Spring的依赖注入，我们就可以这样写

```java
class Player{  
    @Autowired     //自个儿去IOC容器中找这个类型的对象进行注入
    Weapon weapon;  
    public void attack() {
        weapon.attack();
    }
} 
```

### 1.2.1、@Autowired和@Resource的区别

```xml
<bean id="userService" class="com.test.UserServiceImpl"></bean> 
```

**@Autowired：**这是spring的注解，默认按照类型注入byType，注入的对象需要在IOC容器中存在，否则需要加上属性required=false,匹配的是xml中bean的class

**@Resource：**这是Java的注解，按照对象名注入，byName，匹配的是xml中bean的id，同时@Resource还有两个重要的属性：**name和type，用来显式指定byName和byType方式注入**

### 1.2.2、依赖注入的三种方式

#### setter注入

使用set方法进行依赖注入的时候必须给需要注入的对象创建set方法

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
     <!--配置UserServiceImpl对象-->
     <!--bean中的name属性可以有多个值，多个值可以使用逗号分隔开.(name属性不是必须的)-->
    <bean id="userService" name="name1,name2,name3" class="com.yanga.service.impl.UserServiceImpl"/>
</beans>
```

#### 构造注入

需要对应的参数构造

```xml
<!--通过 构造方法方法注入 (需要为注入的成员变量提供 Set 方法。)-->
    <bean id="userDao2" class="com.yanga.dao.impl.UserDaoImpl"/>
    <bean id="userService2" class="com.yanga.service.impl.UserServiceImpl">
        <constructor-arg name="userDao">
            <ref bean="userDao2"/>
        </constructor-arg>
        <!--（index:根据参数位置识别参数）-->
        <!--<constructor-arg index="0" ref="userDao2"/>-->
        <!--（name:根据参数类型识别参数）-->
        <!--<constructor-arg type="com.yanga.dao.UserDao" ref="userDao2"/>-->
    </bean>
```

#### 注解注入

@Resource和@Autowired

## 2、IOC加载过程

![image-20221116201207338](http://img.sybutterfly.top/img202211192334296.png)

### 2.1、IOC容器

Spring 容器负责实例化，配置和装配 Spring beans，**BeanFactory这个类是spring中所有bean工厂**，也就是俗称的IOC容器的祖宗，各种IOC容器都只是它的实现或者为了满足特别需求的扩展实现。

#### 2.1.1、BeanFactory接口

　　BeanFactory，以Factory结尾，表示它是一个工厂类(接口)， 它负责生产和管理bean的一个工厂。在Spring中，BeanFactory是IOC容器的核心接口，它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

​    BeanFactory只是个接口，并不是IOC容器的具体实现，但是Spring容器给出了很多种实现，如 DefaultListableBeanFactory、XmlBeanFactory、ApplicationContext等，其中XmlBeanFactory就是常用的一个，该实现将以XML方式描述组成应用的对象及对象间的依赖关系。XmlBeanFactory类将持有此XML配置元数据，并用它来构建一个完全可配置的系统或应用。

#### 2.1.2、ApplicationContext接口

它由BeanFactory接口派生而来，ApplicationContext包含BeanFactory的所有功能，通常建议比BeanFactory优先。

ApplicationContext以一种更向面向框架的方式工作以及对上下文进行分层和实现继承，ApplicationContext包还提供了以下的功能：

- MessageSource, 提供国际化的消息访问 
- 资源访问，如URL和文件 
- 事件传播 
- 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层; 

#### 2.1.3、对BeanFactory的思考

一个工厂如果想拥有这样的功能，那么它一定需要以下几个因素：

1. 需要持有各种bean的定义，否则无法正确的完成bean的实例化。
2. 需要持有bean之间的依赖关系，否则在bean实例化的过程中也会出现问题。例如，A包含了B的实例。但是在A初始化之后，调用B实例的方法时，就会报空指针异常，因为A并没有被真正的正确初始化。
3. 以上两种都要依赖于我们所写的依赖关系的定义，暂且认为是XML文件（其实可以是各种各样的），那么我们需要一个工具来完成XML文件的读取。

我目前想到的，只需要满足以上三种条件，便可以创建一个bean工厂，来生产各种bean。当然，spring肯定有更高级的做法，以上只是我直观的去想如何实现IOC。

那么从上面的描述中，我又引申出了另一个核心问题，Bean的定义是什么，他的依赖关系又如何描述？

那么答案是，一个祖宗级别的接口，来看BeanDefinition。



### 2.2、BeanDefinition

**1、**首先，获取BeanDefinition类，这个类的作用实际上就描述其他类的构成，比如User类的属性，构造方法，实现接口等等，都会被他的BeanDefinition记录下来，这里可能有一个问题为什么要生成BeanDefinition类，而不直接从具体的类去构建对象。

>分析

首先spring的bean定义是多种多样的，假如一个bean在xml中，那么直接生成的逻辑就是通过IO边解析边生成，这样的话每次都需要IO吗？那么还有的bean是注解形式，这样又要增加扫描注解的逻辑来找寻类。这样的设计在实际使用中肯定是不可能的。

这样在引入BeanDefinition后结构就变为

![image-20221116210601117](http://img.sybutterfly.top/img202211192334297.png)

第二点，那就是直接通过类的class对象来生成bean，因为bean是封闭的，你是没办法来对里面的成员变量（对象）来进行依赖注入的，而BeanDefinition中恰恰就定义了一个类的全部信息。这样不就可以了？

这大概就是BeanDefinition的优点。



**2、**在 `BeanDefinition` 和 完整`BeanDefinition` 中间通过一个后置增强器，可以对bean的定义信息进行统一修改，只需要实现 `BeanFactoryPostProcessor` 接口即可，这个后置增强器是可以有多个的，你只要在不同的类实现多个 `BeanFactoryPostProcessor` 接口就会执行多次，就像这样：

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;
/**
 * 扩展方法--后置增强器（可修改bean的定义信息）
 */
@Component
public class ExtBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
//        BeanDefinition studentService = beanFactory.getBeanDefinition("studentService");
        System.out.println("扩展方法--可进行修改beanDefinition的定义信息");
    }
}
```

3、得到完整`BeanDefinition`之后就可以进行创建对象了，这整个过程被称为 bean 的生命周期，也就是从实例化到销毁的过程；

### 2.3、BeanDefinition是怎么生成的？

对于xml、properties、注解，这三个都是有不同的策略，大概都是通过`BeanDefinitionRegistry`来对BeanDefinition 的注册、移除、查询等一系列的操作，BeanDefinitionRegistry 接口一次只能注册一个 BeanDefinition，而且只能自己构造 BeanDefinition 类来注册。`BeanDefinitionReader` 解决了这些问题，它一般可以使用一个 BeanDefinitionRegistry 构造，然后通过 loadBeanDefinitions()等方法，把 Resources 转化为多个 BeanDefinition 并注册到 BeanDefinitionRegistry。

具体的内容可以查看开篇的原文连接，原文更精彩。

## 3、Bean的生命周期

下面是详细的生命周期

![72677c123f5e41b3b8498654acac8fe0_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0](http://img.sybutterfly.top/img202211192334298.jpg)

接下来我们要将1、3、4 放到一起讲，是因为它们是在同一个接口里面的，实现`InstantiationAwareBeanPostProcessor`接口即可

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.stereotype.Component;
 //这个类用于测试1、3、4
@Component
public class MyInstantiationAwareBeanPostProcessorTest implements InstantiationAwareBeanPostProcessor {
 
    // 实例化前置
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        
        System.out.println("postProcessBeforeInstantiation被调用了----在对象实例化之前调用-----beanName:" + beanName);
        // 默认什么都不做，返回null
        return null;
    }
 
    // 实例化后置
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInstantiation被调用了---------beanName:" + beanName);
        //默认返回true，什么也不做，继续下一步
        return true;
    }
    
    // 属性修改
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        System.out.println("postProcessPropertyValues被调用了---------beanName:"+beanName);
        // 此方法可对bean中的属性值进行、添加、修改、删除操作；
        // 对属性值进行修改，如果postProcessAfterInstantiation方法返回false，该方法可能不会被调用，
        return pvs;
    }
}
```

#### 1、实例化前置

实例化前置使用的是 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(Class<?> beanClass, String beanName) 方法，方法里有2个参数，分别是beanClass和beanName，顾名思义，就是对在对象实例化之前对bean对象的class信息进行修改或者扩展，以达到我们想要的功能，它的底层是动态代理AOP技术实现的；且是bean生命周期中最先执行的方法；

返回非空：返回值是Object类型，这意味着我们可以返回任何类型的值，由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成对象的目标对象的实例，也就是说，如果返回了非空的值，那么以后我们需要用到这个bean的时候，拿到的就现在返回的对象了，也就不会去走第二步去实例化对象了；

返回空（null）值：默认也是返回null值的，那么就直接返回，接下来会调用doCreateBean方法来实例化对象；

#### 2、实例化对象

doCreateBean方法创建实例，用反射技术创建，这个没什么好说的，只是相当于new了一个对象出来而已，但需要注意的是，这个时候只是将对象实例化了，对象内的属性还未设置；

#### 3、实例化后置

方法名称： InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(Object bean, String beanName)

在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null。因为他的返回值是决定要不要调用postProcessPropertyValues方法中的一个因素(因为还有一个因素是mbd.getDependencyCheck());

返回false ：如果该方法返回false，并且不需要check，那么postProcessPropertyValues就会被忽略不执行；

返回true ： 如果返回true，postProcessPropertyValues就会被执行

#### 4、属性修改

方法名称 ：InstantiationAwareBeanPostProcessor.PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)

此方法可对属性值进行修改，修改范围包括添加、修改、删除操作；，如果实例化后置 postProcessAfterInstantiation() 方法返回false，那么该方法不会被调用；

#### 5、给用户属性赋值

用户属性指的是用spring 的人自定义的bean对象属性，像 User、Student、Teacher 、UserService、IndexService 这类的对象都是自定义bean对象，第5步主要给这类属性进行赋值操作，使用的是 AbstractAutowireCapableBeanFactory.populateBean() 方法进行赋值；

#### 6、给容器属性赋值

容器属性其实就是容器自带的属性，这些属性都是spring本来就有的；可以肯定的是，它们都是 Aware 接口的实现类，主要有以下实现类，我已经将它们的执行顺序都排列好了，

![9bb85e96039f4692b7d5e344fadad816_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0](http://img.sybutterfly.top/img202211192334299.jpg)

我们先看看怎么用，然后再来讲解每个Aware的作用；上代码

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanClassLoaderAware;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.context.*;
import org.springframework.context.annotation.ImportAware;
import org.springframework.context.weaving.LoadTimeWeaverAware;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.instrument.classloading.LoadTimeWeaver;
import org.springframework.stereotype.Component;
import org.springframework.util.StringValueResolver;
import org.springframework.web.context.ServletContextAware;
import javax.servlet.ServletContext;
 
@Component
public class AllAwareInterface  implements BeanNameAware, BeanClassLoaderAware,
        BeanFactoryAware, EnvironmentAware, EmbeddedValueResolverAware,
        ResourceLoaderAware, ApplicationEventPublisherAware, MessageSourceAware,
        ApplicationContextAware, ServletContextAware, LoadTimeWeaverAware, ImportAware {
 
    @Override
    public void setBeanName(String name) {
        // BeanNameAware作用：让Bean对Name有知觉
        //这个方法只是简单的返回我们当前的beanName,听官方的意思是这个接口更多的使用在spring的框架代码中，实际开发环境应该不建议使用
        System.out.println("1 我是 BeanNameAware 的 setBeanName 方法  ---参数：name，内容："+ name);
    }
    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("2 我是 BeanClassLoaderAware 的 setBeanClassLoader 方法");
    }
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        // 注意： 如果使用 @Configuration 注解的话，setBeanFactory方法会执行2次，
        System.out.println("3 我是 BeanFactoryAware 的 setBeanFactory 方法");
    }
    @Override
    public void setEnvironment(Environment environment) {
        System.out.println("4 我是 EnvironmentAware 的 setEnvironment 方法");
    }
    @Override
    public void setEmbeddedValueResolver(StringValueResolver stringValueResolver) {
        System.out.println("5 我是 EmbeddedValueResolverAware 的 setEmbeddedValueResolver 方法");
    }
    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        System.out.println("6 我是 ResourceLoaderAware 的 setResourceLoader 方法");
    }
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        System.out.println("7 我是 ApplicationEventPublisherAware 的 setApplicationEventPublisher 方法");
    }
    @Override
    public void setMessageSource(MessageSource messageSource) {
        System.out.println("8 我是 MessageSourceAware 的 setMessageSource 方法");
    }
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("9 我是 ApplicationContextAware 的 setApplicationContext 方法");
    }
    @Override
    public void setServletContext(ServletContext servletContext) {
        System.out.println("10 我是 ServletContextAware 的 setServletContext 方法");
    }
    @Override
    public void setLoadTimeWeaver(LoadTimeWeaver loadTimeWeaver) {
        //LoadTimeWeaver 简称LTW，LTW是AOP的一种实现方式，此方法是为了获取Aop织入的对象，使用的织入方式是：类加载期织入，
        // 一般的aop都是运行期织入，就是在运行的时候才进行织入切面方法，但是LTW是在类加载前就被织入了，也就是class文件在jvm加载之前进行织入切面方法
        // 只有在使用 @EnableLoadTimeWeaving 或者存在 LoadTimeWeaver 实现的 Bean 时才会调用，顺序也很靠后
        System.out.println("11 我是 LoadTimeWeaverAware 的 setLoadTimeWeaver 方法");
    }
    @Override
    public void setImportMetadata(AnnotationMetadata annotationMetadata) {
        //只有被其他配置类 @Import(XX.class) 时才会调用，这个调用对 XX.class 中的所有 @Bean 来说顺序是第 1 的。
        System.out.println("12 我是 ImportAware 的 setImportMetadata 方法");
    }
}
```

启动spring后的控制台打印的部分结果如下：

![d213706e41734f6795cb6dc60bf92a38_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0](http://img.sybutterfly.top/img202211192334301.png)

可以看到它们的输出结果按照顺序依次排列打印出来了，这就是它的标准顺序了；接下来我们了解下它们的具体作用

##### 6.1 BeanNameAware.setBeanName()

这个方法只是简单的返回我们当前的beanName,听官方的意思是这个接口更多的使用在spring的框架代码中，实际开发环境应该不建议使用

##### 6.2 BeanClassLoaderAware.setBeanClassLoader() 

获取Bean的类装载器，

##### 6.3 BeanFactoryAware.setBeanFactory()

获取bean工厂，beanFactory让你可以不依赖注入方式，随意的读取IOC容器里面的对象，不过beanFactory本身还是要注入的。

需要注意的是，一般情况下我们都用 @Component 注解，如果使用 @Configuration 注解的话，setBeanFactory方法会执行2次；

##### 6.4 EnvironmentAware.setEnvironment()

实现了EnvironmentAware接口重写setEnvironment方法后，在工程启动时可以获得application.properties 、xml、yml 的配置文件配置的属性值。

其他的就不讲了，更多可以查看`原文`

#### 7、初始化前置

方法名称： BeanPostProcessor.postProcessBeforeInitialization()

在每一个 Bean 初始化之前执行的方法（有多少 Bean 调用多少次）

注意 ： 启用该方法后，标注了@PostConstruct注解的方法会失效

#### 8、初始化后置

方法名称： BeanPostProcessor.postProcessAfterInitialization()

在每一个 Bean 初始化之后执行的方法（有多少 Bean 调用多少次）

初始化前置和初始化后置的实现代码如下

```java
package com.Spring.Boot.init;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;
 
@Component
public class ExtBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 在每一个 Bean 初始化之前执行的方法（有多少 Bean 调用多少次）
        // 注意 ： 启用该方法后，标注了@PostConstruct注解的方法会失效
        System.out.println("初始化前置方法");
        return null;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
         在每一个 Bean 初始化之后执行的方法（有多少 Bean 调用多少次）
        System.out.println("初始化后置方法");
        return null;
    }
}
```

#### 9、执行初始化方法

初始化方法有三个，分别是 添加了@PostConstruct 注解的方法、实现InitializingBean接口、在@bean注解上添加 initMethod属性；我们一个个讲

#### 10、初始化方法一：@PostConstruct

在bean对象内添加@PostConstruct 注解后即可实现初始化的功能，被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。 有多个则会执行多次；

注意： 如果spring中一个类实现了 BeanPostProcessor接口的postProcessBeforeInitialization() 方法，也就是12的初始后置方法，那么该类的@PostConstruct注解会失效；

```java
package com.Spring.Boot.init;
import org.springframework.stereotype.Component;
import javax.annotation.PostConstruct;
 
// @PostConstruct注解
@Component
public class ExtPostConstruct {
 
    /**
     * 被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。如果有多个则会执行多次
     * 注意： 如果spring 实现了 BeanPostProcessor接口的postProcessBeforeInitialization方法，该@PostConstruct注解会失效
     */
    @PostConstruct
    public void init() {
        System.out.println("第一个init...");
    }
 
    // 有多个会执行多次
    @PostConstruct
    public void init1() {
        System.out.println("第二个init1...");
    }
 
}
```

#### 11、InitializingBean.afterPropertiesSet()

spring 初始化方法之一，作用是在BeanFactory完成属性设置之后,执行自定义的初始化行为。

执行顺序：在initMethod之前执行，在@PostConstruct之后执行

```java
package com.Spring.Boot.init;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;
 
@Component
public class ExtInitializingBean implements InitializingBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        // 一个 InitializingBean 执行一次
        // spring 初始化方法，作用是在BeanFactory完成属性设置之后,执行自定义的  初始化行为.
        // 执行顺序：在initMethod之前执行，在@PostConstruct之后执行
        System.out.println("InitializingBean");
    }
}
```

#### 12、init-method

bean 配置文件属性 init-method 用于在bean初始化时指定执行方法，用来替代继承 InitializingBean接口,

注意的一点是只有一个类完整的实例被创建出来后，才能走初始化方法。

示例代码，先定义一个类： BeanTest.java ，在类中定义一个初始化方法 initMethod_1()

```java
package com.Spring.Boot.init.bean;
 
public class BeanTest {
    
    // 将要执行的初始化方法
    public void initMethod_1(){
        System.out.println("我是beanTest的init方法");
    }
}
```

#### 13、使用中

到这一步，bean对象就已经完全创建好了，是一个完整对象了，并且正在被其他对象使用了；

#### 14、销毁流程

在这里需要先说一下，被spring容器管理的bean默认是单例的，默认在类上面有个 @Scope注解，也就是这样的

```java
package com.Spring.Boot.init;
 
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
 
@Component()
@Scope(value = ConfigurableBeanFactory.SCOPE_SINGLETON)
// @Scope(value = "singleton")  // 也可以这样写
public class InitMethod  {
  // methods....
}
```

如果要设置成多例，只需要把@Scope的属性值改一下就行，就像这样，**多例模式也叫原型模式，它底层不是重新创建一个bean对象出来，而是使用深拷贝技术实现的，就是复制一个对象出来进行使用**

```java
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
// @Scope(value = "prototype") // 也可以这样写
```

为什么要介绍单例和多例呢？ 因为啊，销毁流程的走向就跟你是单例还是多例有关；

如果是单例模式，会先执行 DisposableBean.destroy()方法，然后在执行 destroy-Method 方法；

##### 14.1 DisposableBean.destroy()

单例模式的销毁方式，示例代码

```java
package com.Spring.Boot.init.destroy;
 
import org.springframework.beans.factory.DisposableBean;
import org.springframework.stereotype.Component;
 
/**
 * 销毁方法
 */
@Component
public class ExtDisposableBean implements DisposableBean {
    @Override
    public void destroy() throws Exception {
        System.out.println("我被销毁了");
    }
}
```

控制台打印如下

![b30a038a797b4b258000b99be5efa05f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0](http://img.sybutterfly.top/img202211192334302.jpg)

##### 14.2 destory-method方法

还是拿 第11 个流程的例子来讲，只不过这次我们在@Bean注解里加上 destroyMethod属性，指向销毁方法 ：destroyMethod_1()

```java
package com.Spring.Boot.init;
 
import com.Spring.Boot.init.bean.BeanTest;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
 
@Component()
public class InitMethod  {
 
    // 在@Bean注解上添加initMethod属性，指向类中的 initMethod_1 执行初始化方法
    // 在@Bean注解上添加destroyMethod属性，指向类中的 destroyMethod_1 执行销毁方法
    @Bean(initMethod = "initMethod_1",destroyMethod = "destroyMethod_1")
    public BeanTest getBeanTest(){
        return new BeanTest();
    }
}

BeanTest.java
package com.Spring.Boot.init.bean;
public class BeanTest {
    // 将要执行的初始化方法
    public void initMethod_1(){
        System.out.println("我是beanTest的init方法");
    }
    // 将要执行的销毁方法
    public void destroyMethod_1(){
        System.out.println("我是beanTest的init方法");
    }
}
```

#### 15、返回bean给用户，剩下的生命周期由用户控制

因为多例模式下，spring无法进行管理，所以将生命周期交给用户控制，用户用完bean对象后，java垃圾处理器会自动将无用的对象进行回收操作；  
