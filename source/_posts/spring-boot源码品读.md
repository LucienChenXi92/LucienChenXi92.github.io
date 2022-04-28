---
title: "spring-boot源码品读"
catalog: true
date: 2022-03-11 22:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- Java
- spring-boot
catagories:
- spring
---
#### Overall

spring-boot是当今Java最主流的开发框架，研究其设计和实现非常有价值。我愿意出一系列的笔记来记录品读过程中的所见所得，当然，可以预见到这个周期不会太短。为了避免spring-boot版本更新而造成信息不一致，我将spring-boot 4.0.0 版本代码fork到了自己的github repository下面。

我预想的学习路线为：
1. 熟悉Spring-boot各核心模块及其作用
2. 分模块深入研究源码设计及其实现
3. 探究模块之间的关系
***

#### Modules
先来看一下官方readme中给spring-boot各模块的介绍：
##### spring-boot
核心库，同时支持Spring Boot其他模块功能，包括：
+ `SpringApplication`类，提供静态、便捷的方法以能够快速构建一个独立的Spring应用。它唯一的任务就是创建和刷新一个合适的Spring容器（`ApplicationContext`）
+ 内置可选配的web应用（Tomcat，Jetty or Undertow）
+ 一流的外部配置支持
+ 便捷的`ApplicationContext`初始器，包括支持记录默认值（TODO: 这里不确定自己理解和翻译的对不对。。。）

##### spring-boot-autoconfigure
Spring Boot可以根据分类路径的内容来为常见应用程序装载大部分配置，一个简单的`@EnableAutoConfiguration`注解会自动触发Spring 容器的配置自动装配。 自动配置尝试推导户可能会用到的Bean。 例如，如果`HSQLDB`在资源路径（classpath）下，而且用户还没有配置数据库连接，那么他将可能会想要声明一个基于内存的数据库。当用户开始自己定义Bean时，自动配置永远都会自动回退掉它自己的Bean。

##### spring-boot-starters
启动器是一系列的依赖描述（dependency descriptors），你可以把他们引用到自己的应用程序中，借此，你不再需要搜寻示例代码和复制依赖描述就可以一站式的获取所有Spring以及相关的技术。例如，如果你想用Spring和JPA来实现数据库访问，那么你可以简单的引入`spring-boot-starter-data-jpa`就好了。

##### spring-boot-cli
Spring 命令行应用编译并运行Groovy Source，使其超级易于编写最简洁的代码让你的应用程序跑起来。Spring CLI同时可以监视文件，当发生变化时自动重新编译并应用。

##### spring-boot-actuator
执行期（actuator）的endpoint可以实现对应用程序的监控和交互。Spring Boot提供了的执行器endpoint所需的基础架构。它包含对执行器endpoint的注解支持。开箱即用，该模块提供了很多endpoint，诸如`HealthEndpoint`, `EnvironmentEndpoint`, `BeansEndpoint`等等。

##### spring-boot-test
该模块包含可以帮助测试应用程序的核心项目和注解。

##### spring-boot-loader
Spring Boot加载器允许你构建可以使用`Java -jar`命令启动的简单的jar文件，一般来说，你不需要直接使用`spring-boot-loader`, 二是使用Gradle或Maven插件。
                                                                                    
##### spring-boot-devtools
spring-boot-devtools模块提供了额外的开发时功能，例如自动重启，以获得更加流畅的应用程序开发体验。运行完全打包的应用程序时，开发工具会自动禁用。




