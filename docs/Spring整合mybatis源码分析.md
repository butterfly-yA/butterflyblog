---
title: Spring整合mybatis分析和实现
date: '2022-09-07'
categories:   # 分类
 - Spring
tags:   # 标签
 - mybatis
 - IOC
publish: true   # 发布
---

<p style="color:#7f7f7f">Spring整合mybatis的方案，如何在IOC容器中放入bean，关键字：ImportBeanDefinitionRegistrar @Import ClassPathBeanDefinitionScanner
本篇不分析源码，只是思路分析和实现。
</p>



<!-- more -->

## 1、问题思考

咱们开篇先抛出两个问题

> 什么是整合的重点？怎么实现整合？

在spring和mybatis整合的情况下，我们对mybatis的使用一般就是通过注入mapper接口调用方法来实现查询，代码如下

```java
public class UserServiceImpl implements IUserService{
    @Autowired
    private UserMapper userMapper;
    
    public User getUserById(String id){
       return userMapper.seleteById(id);
    }
}
```

这里我们都知道使用的userMapper实际上是代理对象，那么这个代理对象实际上是spring生成的还是mybitis生成的呢？

在原生的mybatis中，对于UserMapper的使用是这样的

```java
    public static void main(String[] args) throws IOException {
        //1、加载资源
        Reader reader = Resources.getResourceAsReader("mybatisConfig.xml");
        //2、获取工厂
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        //3、sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();
        //4、获取mapper代理对象
        TestMapper testMapper= sqlSession.getMapper(TestMapper.class);
        //5、使用
        System.out.println(testMapper.getUser());
    }
```

这里我们可以知道mapper代理对象是由mybatis生成的，底层实现是jdk代理

那么怎么才能让spring来`@Autowired`注入这个对象呢？

> 整合的重点就是

把某个Mapper的代理对象作为⼀个bean放⼊Spring IOC容器中，使得能够像使⽤⼀个普通bean⼀样去使⽤这个代理对象，能被@Autowire⾃动注⼊。

## 2、整合分析

这里展现初始项目的结构，在导入spring和mybatis，并且未导入整合包的情况下

![image-20221119095039033](http://img.sybutterfly.top/img202211192251664.png)

### xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url"
                          value="jdbc:mysql://localhost:3306/springmybatis?useUnicode=true&amp;characterEncoding=utf8"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <package name="com.yanga.mapper"/>
    </mappers>
</configuration>
```

### Mapper

```java
public interface TestMapper {

    @Select("select 'user'")
    String getUser();

}
```

### Service

```java
@Component
public class TestService {
    @Autowired
    TestMapper testMapper;

    public String getUser(){
//       return "sss";
        return testMapper.getUser();
    }
}
```

### Test

```java
public class Test01 {
    @Test
    public void testService(){
        AnnotationConfigApplicationContext context=new AnnotationConfigApplicationContext(AppConfig.class);
        TestService testService = (TestService) context.getBean("testService");
        System.out.println(testService.getUser());
    }
}
```

启动项目Test，可以发现启动报错，因为找不到TestMapper的实例，如果放入IOC容器一个该类型的实例，是不是就可以了呢？如果你尝试在mapper接口上加入`@Component`注解来实现，显然这是不行的，因为spring的默认扫描机制是不是将接口放入IOC容器的

那么关键问题就呼之欲出了，怎么生成这个代理类的对象，并将它放到IOC，代理类其实就是通过mybatis自个儿生成的，那么就只剩怎么放入了

## 3、代理类放入IOC

### 3.1、@Bean

> 配置类修改,思路参照AB

```java
@ComponentScan("com.yanga")
@Configuration
public class AppConfig {
//@Bean注入容器

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws IOException {
        //1、加载资源
        Reader reader = Resources.getResourceAsReader("mybatisConfig.xml");
        //2、获取工厂
        return new SqlSessionFactoryBuilder().build(reader);
    }

    @Bean                       //B.bean注入的参数默认是autowired注入，这个参数也需要在IOC容器
    public TestMapper testMapper(SqlSessionFactory sqlSessionFactory){
        //A.这里是接口，肯定是不能直接new的，直接new也不能查询数据库
        //需要将mybatis的实现方案在这里使用

        //3、sqlSession   //4、获取mapper代理对象
        return sqlSessionFactory.openSession().getMapper(TestMapper.class);
    }
}
```

> 再次测试，可以发现已经可以调用mapper了

![image-20221119101014819](http://img.sybutterfly.top/img202211192251667.png)

> 这样看起来肯定是可以用的，但是如果mapper有几十个呢？每个都得配置？这样配置文件也太臃肿了

### 3.2、FactoryBean

FactoryBean在实例化的时候，就会调用它的 getObject方法获取对象，也就是说虽然存入IOC的是FactoryBean，但实际上获取的是它的 getObject方法的返回对象

```java
@Component  //放入的TestFactoryBean但实际上是两个方法逻辑上产生的对象
public class MyFactoryBean implements FactoryBean {
    @Override
    public Object getObject() throws Exception {
        //1、加载资源
        Reader reader = Resources.getResourceAsReader("mybatisConfig.xml");
        //2、获取工厂
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        //3、sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();
        //4、获取mapper代理对象
        TestMapper testMapper= sqlSession.getMapper(TestMapper.class);
        return testMapper;
    }

    @Override
    public Class<?> getObjectType() {
        return TestMapper.class;
    }
}
```

> 这样的逻辑还是存在问题，一个FactoryBean只能生成一个bean对象（在getObjectType方法中已经确定了），还是得面临构造多个的问题，当然我们可以通过抽离mapper，然后构造赋值的方式来使我们的factoryBean变得更灵活

```java
public class MyFactoryBean implements FactoryBean {

    private Class mapperInterface;

    public MyFactoryBean() {
    }
    
    public TestFactoryBean(Class mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    @Override
    public Object getObject() throws Exception {
        //1、加载资源
        Reader reader = Resources.getResourceAsReader("mybatisConfig.xml");
        //2、获取工厂
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        //3、sqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();
        //4、获取mapper代理对象
        return sqlSession.getMapper(mapperInterface);
    }

    @Override
    public Class<?> getObjectType() {
        return mapperInterface;
    }
}
```

>但是谁来为factoryBean构造赋值呢？

### 3.3、BeanDefinition

在IOC构建单例Bean的时候，不是直接基于class对象，而是基于BeanDefinition，也就是说一个BeanDefinition就是一个Bean对象，只要有BeanDefinition就可以构建出对应的Bean对象。

实际上MyFactoryBean也是⼀个Bean，我们也可以通过⽣成⼀个BeanDefinition来⽣成⼀个MyFactoryBean，并给构造⽅法的参数设置不同的值

那么我们要怎么构建一个BeanDefinition呢？这就需要`ImportBeanDefinitionRegistrar`和`@Import注解`了。

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        //1.BeanDefinition构建者
        BeanDefinitionBuilder builder=BeanDefinitionBuilder.genericBeanDefinition();
        //2.构建BeanDefinition
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        //3.属性填充
        beanDefinition.setBeanClass(MyFactoryBean.class);
        //4.构造赋值
     beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(TestMapper.class);
        //5.添加beanDefinition
        beanDefinitionRegistry.registerBeanDefinition(TestMapper.class.getSimpleName(), beanDefinition);
    }
}
```

> AppConfig上添加@Import注解

```java
@ComponentScan("com.yanga")
@Configuration
@Import(MyImportBeanDefinitionRegistrar.class)
public class AppConfig {
}
```

这样在启动Spring时就会扫描到@Import执行`MyImportBeanDefinitionRegistrar`#`registerBeanDefinitions`逻辑，新增⼀个BeanDefinition，该BeanDefinition会⽣成⼀个MyFactoryBean对象，并且在⽣成MyFactoryBean对象时会传⼊TestMapper.class对象，通过MyFactoryBean内部的逻辑，相当于会⾃动⽣产⼀个TestMapper接⼝的代理对象作为⼀个bean。

![image-20221119111050370](http://img.sybutterfly.top/img202211192251668.png)

这样其实还不够完善，有个优化点，我们可以指定扫描的包路径，让容器自动扫描然后获取到所有的mapper接口，生成BeanDefinition，获取到代理对象

## 4、优化

> 定义扫描包注解@MyMapperScan，并在AppConfig类上使用

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import(MyImportBeanDefinitionRegistrar.class)
public @interface MyMapperScan {
    String value();
}
```

> 修改MyImportBeanDefinitionRegistrar类，注意spring默认的扫描器是不会将接口扫描成BeanDefinition的，所以我们要重写扫描的逻辑

![image-20221119140704502](http://img.sybutterfly.top/img202211192251669.png)

> 自定义MyBeanDefinitionScanner

这里的扫描逻辑是直接是指定包下的接口都会被扫描，也可以是@MyMapper注解标注的被扫描

**说明一下：**

在Spring中ClassPathBeanDefinitionScanner是扫描所有类的，在容器启动时，这个类会扫描`@ComponentScan`注解的包路径，默认的扫描规则是扫描@Component与@Service等Spring支持的注解，然后将这些儿类生成对应的BeanDefinition，然后通过BeanDefinition来生产Bean对象，注意这里默认的扫描规则是不会扫描接口的，所以需要重写方法来自定义扫描规则，当然我们也不能将接口直接注册为BeanDefinition，因为这样在通过BeanDefinition创建Bean对象时，肯定是报错的，接口不能new对象是吧？

```java

/**
 * @author yangA
 * @date 2022/11/19 15:15
 * 自定义spring扫描器，扫描mapper接口，生成BeanDefinition
 */

public class MyBeanDefinitionScanner extends ClassPathBeanDefinitionScanner {

  private BeanDefinitionRegistry registry;

    public MyBeanDefinitionScanner(BeanDefinitionRegistry registry) {
        //默认只扫描@Component与@ManagedBean建议取消默认的
        super(registry, false);
        this.registry=registry;
        //重新添加过滤规则  类似注解组件扫描

        //过滤规则是含有@MyMapper类的注解才会被扫描
        //addIncludeFilter(new AnnotationTypeFilter(MyMapper.class));

        //无过滤规则，只有满足 isCandidateComponent() 就会被扫描
        addIncludeFilter(new TypeFilter() {
            @Override
            public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
                return true;
            }
        });
    }

    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> definitionHolders = super.doScan(basePackages);
        if (definitionHolders.size() == 0) {
            System.out.println("没有mapper接口被扫描");
        } else {
            System.out.println(definitionHolders);
        }
        //对扫描结果进行拦截处理,在这里进行代理
        handleBeanDefinitionHolder(definitionHolders);
        return definitionHolders;
    }

    /**
     * 生成mapper代理类对应的MyFactoryBean
     * @param definitionHolders
     */
    private void handleBeanDefinitionHolder(Set<BeanDefinitionHolder> definitionHolders) {
        for (BeanDefinitionHolder holder : definitionHolders) {
            AbstractBeanDefinition beanDefinition = (AbstractBeanDefinition) holder.getBeanDefinition();
            // 1.获取扫描出的接口的beanClassName
            String beanClassName = beanDefinition.getBeanClassName();
            // 2.设置BeanClass为MyFactoryBean
            beanDefinition.setBeanClass(MyFactoryBean.class);
            // 3.添加构造方法的参数,MyFactoryBean的构造方法需要Class对象,这里设置的是string的参数,spring会使用Class.forName加载
            // 成MyFactoryBean中构造方法需要的类参数
            beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);
            System.out.println("-----------------");
            System.out.println(beanDefinition);
        }
    }

    @Override//对扫描对象的条件限制  是不是接口且是独立的bean(不依赖其他bean)
    protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
        return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
    }
}
```

> 为什么要使用FactoryBean呢？

因为实际上你要生成的代理类只能是某个接口或者某个类的子类对象，显然这里是没有其他的父类的，而接口是不能设置为BeanDefinition的beanClass的，原因如下，他并不是一个实际的类型，spring不能根据这个类反射构建出对象，所以咱们需要FactoryBean，这个家伙存入IOC后，构建的实际上是他`getObject()`方法返回的对象

![image-20221119171431484](http://img.sybutterfly.top/img202211192251670.png)

> 测试

这里新建一个测试用的mapper接口，测试是否能够生成多个代理类并使用，结构与`TestMapper`一致

```
public interface TestMapper2 {
    @Select("select 'user2'")
    String getUser();
}
```

> 运行，可以看到成功输出

![image-20221119173939281](http://img.sybutterfly.top/img202211192251671.png)

## 5、总结

> 所以，到此为⽌，Spring整合Mybatis的核心原理就结束了，再次总结⼀下：

定义⼀个MyFactoryBean，⽤来将Mybatis的代理对象⽣成⼀个bean对象

定义⼀个MyImportBeanDefinitionRegistrar，⽤来⽣成不同Mapper对象的MyFactoryBean

定义⼀个@MyMapperScan，⽤来在启动Spring时执⾏MyImportBeanDefinitionRegistrar的逻辑，并

指定包路径

定义一个@MyMapper，标注节接口，表明这是一个mapper接口

定义一个MyBeanDefinitionScanner，用于扫描生成MyFactoryBean的BeanDefinition

> 以上这个几个要素分别对象org.mybatis.spring中的

MapperFactoryBean

MapperScannerRegistrar

@MapperScan

@Mapper

ClassPathMapperScanner

>那么到这里其实自定义注解被spring容器扫描和扫描接口进行动态代理的步骤也就有了，自己动手实现一下吧~~~~~~~~~~~~~~