---
title: "梳理Spring-boot知识点"
catalog: true
date: 2020-04-29 10:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- 后端
- Java
catagories:
---

昨天去一家外资银行面试，面试官问了我Spring-boot如何启动，这个问题一下给我问蒙了，一方面没有想到会问这种基本的问题，每天敲 mvn spring-boot:run命令行已经敲成肌肉记忆了，一时间被问是什么，我脑子一下卡住了，反应了几秒才回答上来。结果整个面试不是很顺利，面试官扣了很多细节，甚至怀疑我写不写代码。。。哎，今天就梳理一下Spring-boot这块的知识点吧。

### 什么是Spring-boot
Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。
Spring Boot可以理解为是Spring的一个封装和增强，同时又是组成Spring cloud微服务生态的最基本单元，对其他框架和设施有很好的兼容。是目前企业中Java开发不可缺少的框架之一。

### Spring-boot 核心知识点
+ IOC
+ AOP
+ 配置注入

### IOC(控制反转)

#### 什么是控制反转
在最原始的代码中，比如我们Service层中需要一个Dao层的实现类一般会直接声明：

```java
private IDao dao = new DaoMysqlImpl();
```
这种写法其实是程序代码自己去控制实例的生成，而且，在Service层中需要指明Dao层的具体哪个实现类，将Service 和 DaoMysqlImpl这两个class耦合了起来，一旦需要变更实现，程序的Service层必须进行修改，不符合程序开发中的低耦合原则。
Spring 提供了一个Bean的容器来解决这个问题，实例的初始化不再由程序代码来实现，而是被动的由Spring容器提供，具体所需的实例可以通过配置文件进行指定。这样一旦出现实现的变更，只要添加新的Dao层的实现类，并且在配置文件中更新注入的Bean即可，更符合开闭思想。

```java
@Autowired
private IDao dao;
```
#### 如何让Spring帮我们注入bean
我们首先定一个Bean User.class如下：

```java
package com.lucien.myweb.model;

public class User {

    private String name;
    private int age;
    private User partner;

    public User() {}

    public User(String name) {
        this.name = name;
    }

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setPartner(User partner) {
        this.partner = partner;
    }

    public String getInfo() {
        return "name: " + this.name + ", age: " + this.age + ", partner: " 
                + this.partner == null ? "unknown" : this.partner.getName();
    }
}
```

在resources中创建beans.xml文件，并且在beans中声明需要Spring容器创建的对象，属性以及各种依赖关系。
创建对象方式有无参构造，有参构造，工厂类构造三种主要方式。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--无参构造-->
    <bean id="boy" class="com.lucien.myweb.model.User" scope="singleton">
        <property name="age" value="20"></property>
        <property name="name" value="Lucien"></property>
    </bean>
    <!--有参构造-->
    <bean id="girl" class="com.lucien.myweb.model.User">
        <constructor-arg name="name" value="Fish"></constructor-arg>
        <constructor-arg name="age" value="18"></constructor-arg>
        <property name="partner" ref="boy"></property>
    </bean>
    <!--工厂类-->
    <bean id="factory" class="com.lucien.myweb.model.UserFactory"></bean>
    <bean id="li" factory-bean="factory" factory-method="getInstance"></bean>
    <!--工厂静态方法-->
    <bean id="hua" class="com.lucien.myweb.model.UserFactory" factory-method="getInstanceStatic"></bean>
</beans>
```
最后随便在Controller层写了个接口，调用了一下，通过Debug我们可以看到xml中定义的两个Bean已经成功注入了。

```java
@Controller
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    @ResponseBody
    public String hello() {
        ClassPathXmlApplicationContext ac = new ClassPathXmlApplicationContext("beans.xml");
        String response1 = ((User) ac.getBean("boy")).getInfo();
        String response2 = ((User) ac.getBean("girl")).getInfo();
        return response2;

    }
}
```
![/img/spring_ioc1.png](/img/spring_ioc1.png)

#### 如何让Spring-boot帮我们注入bean
上面是比较老的一种实现，通过xml文件去定义bean的属性。当项目中的配置信息非常逐渐庞大，xml这种写法就会让人很头疼，在spring-boot中可以通过定义Java bean的形式避免使用xml。
@Configuration 注解相当于beams标签内部区域，每个@Bean注解相当于一条的记录。

```java
package com.lucien.myweb.configuration;

import com.lucien.myweb.model.User;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BeanConfiguration {

    @Bean
    public User getBoy() {
        User boy = new User();
        boy.setAge(20);
        boy.setName("Lucien");
        return boy;
    }

    @Bean
    public User getGirl() {
        User girl = new User();
        girl.setAge(18);
        girl.setName("Fish");
        girl.setPartner(getBoy());
        return girl;
    }
}

@Controller
public class HelloController {


    @Autowired
    @Qualifier("getGirl")
    private User girl;


    @Autowired
    @Qualifier("getBoy")    // 通过@Qualifier指定需要注入的Bean
    private User boy;

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    @ResponseBody
    public String hello() {
        System.out.println(girl.getInfo());     // name: Fish, age: 18, partner: Lucien
        System.out.println(boy.getInfo());      // name: Lucien, age: 20, partner: unknown
        return girl.getInfo();
    }
}
```

值得注意的一点，在xml中可以直接在bean标签上指定Bean的作用域，如下:

```xml 
<bean id="boy" class="com.lucien.myweb.model.User" scope="singleton">
    <property name="age" value="20"></property>
    <property name="name" value="Lucien"></property>
</bean>
```

在Java class声明的方式下，需要用@scope注解来表示作用域，如下：

```java
@Bean
@Scope("singleton")
public User getBoy() {
    User boy = new User();
    boy.setAge(20);
    boy.setName("Lucien");
    return boy;
}
```

#### 待思考问题
1. Spring 是如何基于配置信息去生成Bean的？
2. 对象容器是如何管理生成的对象的？

### AOP(面向切面编程)

#### 什么是AOP
AOP 全称为Aspect Oriented Programming，通过预编译方式和运行期间动态代理实现程序功能的统一维护的一种技术。Spring中广泛使用AOP技术，来实现日志记录，功能增强等功能。

#### AOP的好处
使得真实角色处理的业务更加存粹，不再去关注一些公共的事情。
公共的业务由代理来完成，公共业务发生扩展时变得更加集中和方便。

#### AOP的原理
之前有整理过一个动态代理的专题：设计模式 - 代理模式，在此不再赘述。

代理一般分为动态代理和静态代理两大类：

+ 静态代理：委托类和代理类同时继承某一接口，在代码通过构造函数把委托类的实例传入代理类来创建委托类实例，然后调用代理类实例的方法去执行。(在Java 多线程中，实现Runnable接口的MyThread类和Thread其实就是静态代理关系)
+ 动态代理：当需要代理的类很多时，每有一个委托类就会需要写一个对应的代理类，代码量巨大。这种情况下我们需要使用动态代理帮我们解决。声明一个动态代理器就可以应对各种委托类。而动态代理有两种实现方式：JDK和CGLIB。
值得注意，基于JDK实现的动态代理其实利用Java的反射机制实现的，导入的包都是在java.lang.reflect下面。

#### 如何实现AOP
比如我们想要实现一个日志的切面可以通过Spring AOP Api来实现。在实现之前我们需要先记住几个概念：

+ 连接点（Joinpoint）: 表示需要在程序中插入横切关注点的扩展点，连接点可能是类初始化、方法执行、方法调用、字段调用或处理异常等等，Spring只支持方法执行连接点；在AOP中表示为“在哪里干”；
+ 切入点（Pointcut）: 选择一组相关连接点的模式，即可以认为连接点的集合，Spring支持perl5正则表达式和AspectJ切入点模式，Spring默认使用AspectJ语法；在AOP中表示为“在哪里干的集合”；
+ 通知（Advice）: 在连接点上执行的行为，通知提供了在AOP中需要在切入点所选择的连接点处进行扩展现有行为的手段；包括前置通知（before advice）、后置通知(after advice)、环绕通知（around advice），在Spring中通过代理模式实现AOP，并通过拦截器模式以环绕连接点的拦截器链织入通知；在AOP中表示为“干什么”；
+ 切面（Aspect）：横切关注点的模块化，比如日志组件。可以认为是通知、引入和切入点的组合；在Spring中可以使用Schema和@AspectJ方式进行组织实现；在AOP中表示为“在哪干和干什么集合”；
+ 引入（Introduction）: 也称为内部类型声明，为已有的类添加额外新的字段或方法，Spring允许引入新的接口（必须对应一个实现）到所有被代理对象（目标对象）；在AOP中表示为“干什么（引入什么）”；
+ 目标对象（Target Object）:需要被织入横切关注点的对象，即该对象是切入点选择的对象，需要被通知的对象，从而也可称为“被通知对象”；由于Spring AOP 通过代理模式实现，从而这个对象永远是被代理对象；在AOP中表示为“对谁干”；
+ AOP代理（AOP Proxy）: AOP框架使用代理模式创建的对象，从而实现在连接点处插入通知（即应用切面），就是通过代理来对目标对象应用切面。在Spring中，AOP代理可以用JDK动态代理或CGLIB代理实现，而通过拦截器模型应用切面。
+ 织入（Weaving）: 织入是一个过程，是将切面应用到目标对象从而创建出AOP代理对象的过程，织入可以在编译期、类装载期、运行期进行。组装方面来创建一个被通知对象。这可以在编译时完成（例如使用AspectJ编译器），也可以在运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入。
> 
> 以上对概念的解释来自于博客： Spring boot之AOP面向切面编程 - 行者小朱
> 

首先，引入如下Spring AOP的包：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

然后，我们写一个切面类Logger, 值得注意的是切面类上要同时有@Component 和 @Aspect两个注解，缺少哪个都不会生效，缺少前者Spring不知道这个类需要被加载，缺少后者Spring不知道这是个切面类。

```java
ppackage com.lucien.myweb.log;
 
import org.aspectj.lang.JoinPoint; 
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;
 
@Component
@Aspect
public class Logger {
 
    // 声明切入点，针对那些类中的那些方法要进行增强    
    @Pointcut("execution(public * com.lucien.myweb.controller.*.*(..))")
    public void executeRequest() {}
 
    // 声明前置通知
    @Before("executeRequest()")
    public void doBefore(JoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        System.out.println(signature.getDeclaringTypeName() + " 中的 " +signature.getName() + " 被调用");
    }
 
    // 声明后置通知
    @After("executeRequest()")
    public void doAfter(JoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        System.out.println(signature.getDeclaringTypeName() + " 中的 " +signature.getName() + " 被调用完毕");
    }
 
    // 声明环绕通知
    @Around("executeRequest()")
    public void doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        try {
            Object proceed = proceedingJoinPoint.proceed();
            Thread.sleep(1000);
        } catch (Throwable throwable) {
            System.out.println(throwable.getStackTrace());
        } finally {
            long endTime = System.currentTimeMillis();
            System.out.println("共执行：" + (endTime - startTime) + "ms");
        }
    } 
}
```

然后我们写一个一个Controller 和 service，里面简单声明几个方法：

```java
package com.lucien.myweb.controller;

import com.lucien.myweb.Service.Service;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class HelloController {


    @Autowired
    private Service service;

    @RequestMapping(value = "/add", method = RequestMethod.GET)
    @ResponseBody
    public void add() {
        service.add();
    }

    @RequestMapping(value = "/update", method = RequestMethod.GET)
    @ResponseBody
    public void update() {
        service.update();
    }

    @RequestMapping(value = "/search", method = RequestMethod.GET)
    @ResponseBody
    public void search() {
        service.search();
    }

    @RequestMapping(value = "/delete", method = RequestMethod.GET)
    @ResponseBody
    public void delete() {
        service.delete();
    }
}

package com.lucien.myweb.Service;

import org.springframework.stereotype.Component;

@Component
public class Service {

    public void add() {
        System.out.println("add");
    }

    public void update() {
        System.out.println("update");
    }

    public void search() {
        System.out.println("search");
    }

    public void delete() {
        System.out.println("delete");
    }

}
```

ok, 启动项目，调用一下这几个接口看一下效果：
![/img/spring_aop1.png](/img/spring_aop1.png)

从上面的返回可以看出通知顺序为：firstly 前置通知 , then 环绕通知， finally 后置通知。以上即为一个AOP的简单Demo，可以想象一下，其实很多异常的同一处理，记录日志，记录Metrics都是可以利用AOP来实现的。代码简洁而且没有侵入性。
除了前置通知，后置通知，环绕通知之外，还有两种通知，分别是@AfterThrowing 和 @AfterReturning, 用法可以从命名上看出来，分别是报错和Return返回结果时被触发。

### 配置注入

#### 为何需要依赖注入
在项目中，我们一般会将endpoint, username, schedule job的开关等信息放在配置文件中，代码中需要的地方通过注入的形式获取对应的值，这样有利于将这些经常变动的信息同程序代码解偶，一旦需要改动时，所有注入的值会一同发生改变，以免遗漏。

#### 如何注入配置
在创建Spring-boot项目时会自动创建一个application.property的文件, 项目的配置文件可以在这个文件中以键:值的形式进行声明。 在过往的项目中，我发现使用yaml格式来记录配置信息更加直观方便，我们下面就用yaml格式来进行演示：

首先我们在resource下面创建一个配置文件application.yml, 并且创建一个配置类YmlFileConfigurationLoader来加载此文件中的配置：

```java
package com.lucien.myweb.init;

import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.io.ClassPathResource;

@Configuration
public class YmlFileConfigurationLoader {

    @Bean
    public static PropertySourcesPlaceholderConfigurer properties() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        YamlPropertiesFactoryBean yaml = new YamlPropertiesFactoryBean();
        yaml.setResources(new ClassPathResource[]{ new ClassPathResource("application.yml") });
        configurer.setProperties(yaml.getObject());
        return configurer;
    }

}
```

之后我们可以在application.yml中定义一些配置属性：
```yaml
server:
  port: 8080

elasticsearch:
  endpoint: "endpoint of ES"
  index: "my-index"
  ableToUse: true

beans:
  users:
    - age: 20
      name: "Lucien·Chen"
    - age: 18
      name: "Fish·Liu"
```

然后我们在Controller层加一个方法来返回注入到项目中的配置信息：

```java
package com.lucien.myweb.controller;

import com.lucien.myweb.configuration.BeanConfigurationV2;
import org.json.JSONObject;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.HashMap;
import java.util.Map;

@Controller
public class HelloController {


    private final String endpoint; 
    private final String index;
    private final boolean ableToUse;
    private final String osName;
    private final Map<String, Object> config;
    private final BeanConfigurationV2 bc;

    public HelloController(@Value("${elasticsearch.endpoint}") String endpoint,
                           @Value("${elasticsearch.index}") String index,
                           @Value("${elasticsearch.ableToUse}") boolean ableToUse,
                           @Value("#{systemProperties['os.name']}") String osName,
                           @Value("#{systemProperties}") Map<String, Object> config,
                           BeanConfigurationV2 bc) {
        this.endpoint = endpoint;
        this.index = index;
        this.ableToUse = ableToUse;
        this.osName = osName;
        this.config = config;
        this.bc = bc;
    }

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    @ResponseBody
    public String hello() {
        Map<String, Object> resp = new HashMap<>();
        resp.put("users", bc.getUsers());
        resp.put("config", config);
        resp.put("osName", osName);
        resp.put("endpoint", endpoint);
        resp.put("index", index);
        resp.put("ableToUse", ableToUse);
        return new JSONObject(resp).toString();
    }

}
```

我们可以看到，在配置文件中定义的一些基本类型数据可以通过@Value的方式直接注入到项目中，而类似List，Map，甚至是对象类型的数据需要通过@ConfigurationProperties的形式注入，比如我们有在配置文件中声明了一个叫users的map，我们需要创建一个component如下：

```java
package com.lucien.myweb.configuration;

import com.lucien.myweb.model.User;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
@ConfigurationProperties(prefix = "beans")
public class BeanConfigurationV2 {

    private List<User> users;

    public void setUsers(List<User> users) {
        this.users = users;
    }

    public List<User> getUsers() {
        return users;
    }
}
```

在这个component中，我们需要指定属性的前缀”beans”，然后根据配置文件中所定义的数据结构来声明一个List，每当我们需要这个List时需要引入这个component来，然后再通过getter来获取对象应信息。另外，除了这种直接在配置文件中声明配置信息的方式，还有通过直接声明Java bean的形式来声明的方式，如上面IOC那节所演示，利用@Configuration 和 @Bean 的方式，

#### 配置注入总结
好我们来梳理一下，在配置注入这里常用注解如下：

@Configuration // 声明这个类为配置类
@ConfigurationProperties(prefix = “profix”) // 声明配置section的前缀
@Value // 注入配置的value
@Bean // 声明bean
@Autowired // 自动注入
@Qualifier(“methodName”) // 指定注入的Bean