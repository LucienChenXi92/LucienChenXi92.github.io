---
title: "spring-boot源码品读之spring-boot中篇"
catalog: true
date: 2022-03-22 23:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- Java
- spring-boot
catagories:
- spring
---

##### SpringApplication.java
`SpringApplication`可以用作一个Spring应用程序的启动器，默认情况下会经历如下步骤来启动程序：
+ 基于`classpath`创建一个适宜的ApplicationContext实例
+ 注册一个`CommandLinePropertySource`对象来暴露命令行参数作为Spring的属性
+ 刷新应用程序的context，加载所有单例的Beans
+ 调用任何`CommandLineRunner` Beans

`SpringApplication`截止目前有1294行，由于内部成员变量过多，并且大部分属于默认配置，我们先跳过不看，先来看一下构造器：

###### SpringApplication 构造器
```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 接收resourceLoader并赋值
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 接收primarySources并赋值
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断WebApplication类型，servlet，reactive 或是 NONE
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 获取应用上下文initializer集合
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 获取监听器集合
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断应用main方法的class
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

###### 一共6个静态方法：
```java
// 可用于帮助SpringApplication基于指定资源和默认配置而启动的静态方法
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {	
    return run(new Class<?>[] { primarySource }, args);
}

// 作用同上
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    // 创建SpringApplication实例并运行run方法，返回ConfigurableApplicationContext实例
    return new SpringApplication(primarySources).run(args);
}

/**
 * 默认的main方法用于启动应用程序，当application资源通过命令行参数--spring.main.sources的形式定义时，* 此方法会起作用。大多数情况下，开发人员会定义自己的main方法。
 */
public static void main(String[] args) throws Exception {
    SpringApplication.run(new Class<?>[0], args);
}

// 静态方法用于退出SpringApplication并且返回一个状态码用来表示成功与否，成功为0.
public static int exit(ApplicationContext context, ExitCodeGenerator... exitCodeGenerators) {
    Assert.notNull(context, "Context must not be null");
    int exitCode = 0;
    try {
        try {
            // 创建ExitCodeGenerators对象
            ExitCodeGenerators generators = new ExitCodeGenerators();
            // 获取应用程序上下文中的ExitCodeGenerator对象集合
            Collection<ExitCodeGenerator> beans = context.getBeansOfType(ExitCodeGenerator.class).values();
            generators.addAll(exitCodeGenerators);
            generators.addAll(beans);
            // 利用ExitCodeGenerators的getExitCode函数来推断最终exitCode
            exitCode = generators.getExitCode();
            // 如果exitCode为0则忽略，则需要向应用上下文中推送退出事件，这样可以通过自定义监听ExitCodeEvent实现拓展操作
            if (exitCode != 0) {
                context.publishEvent(new ExitCodeEvent(context, exitCode));
            }
        }
        finally {
            // 关闭ApplicationContext
            close(context);
        }
    }
    catch (Exception ex) {
        ex.printStackTrace();
        exitCode = (exitCode != 0) ? exitCode : 1;
    }
    return exitCode;
}

// 私有静态方法用于关闭应用上下文
private static void close(ApplicationContext context) {
    if (context instanceof ConfigurableApplicationContext) {
        ConfigurableApplicationContext closable = (ConfigurableApplicationContext) context;
        closable.close();
    }
}

// 私有静态方法用于转换集合类型成为LinkedHashSet
private static <E> Set<E> asUnmodifiableOrderedSet(Collection<E> elements) {
    List<E> list = new ArrayList<>(elements);
    list.sort(AnnotationAwareOrderComparator.INSTANCE);
    return new LinkedHashSet<>(list);
}
```

从上面几个静态方法我们可以捋出来，`main`方法作为程序入口，基于指定资源和默认配置够建`SpringApplication`实例，然后调用实例方法`run`来构造`ConfigurableApplicationContext`实例，正戏正式开演。

###### 正戏开始，run方法
```java
public ConfigurableApplicationContext run(String... args) {
    // 构造StopWatch对象用于计时
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    // 声明最终返回的ConfigurableApplicationContext实例
    ConfigurableApplicationContext context = null;
    // 声明收集SpringBootExceptionReporter的集合
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    // 获取所有监听器对象
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 触发所有监听器的starting方法
    listeners.starting();
    try {
        // 构造默认应用参数对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 构造ConfigurableEnvironment对象
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 将environment实例中spring.beaninfo.ignore配置值应用到系统层面
        configureIgnoreBeanInfo(environment);
        // 构造printedBanner对象用于打印启动横幅
        Banner printedBanner = printBanner(environment);
        // 构造ApplicationContext对象
        context = createApplicationContext();
        // 通过getSpringFactoriesInstances方法加载所有SpringBootExceptionReporter实例
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
        // 准备Context
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新Context
        refreshContext(context);
        // 在刷新完context之后触发afterRefresh，预留的扩展方法
        afterRefresh(context, applicationArguments);
        // 终止倒计时
        stopWatch.stop();
        // 基于logStartupInfo属性值
        if (this.logStartupInfo) {
            // 构建启动日志对象并打印启动日志
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 触发所有监听器的started方法
        listeners.started(context);
        // 获取并调用所有ApplicationRunner和CommandLineRunner的run方法
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 程序启动后调用所有监听器的running方法
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```


###### 高频动作getSpringFactoriesInstances
在我们开始继续深入看其他方法之前我们先来看一个出现多次的私有方法`getSpringFactoriesInstances`，在构造中出现过两次，分别用于获取`listener`和`initializers`集合，在启动过程中又用于获取`SpringBootExceptionReporter`集合。从方法名上来看是获取Spring工厂实例，传一个Class的类型，返回对应类型实例集合，那这些实例都是从哪里来的呢？
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 获取一个非null的类加载器
    ClassLoader classLoader = getClassLoader();
    // 把类加载器交给SpringFactoriesLoader来获取所有对应工厂类型的实现类名称
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 利用传进来的工厂类类型，参数的类型，参数值，工厂实现的类名等信息构建实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

@SuppressWarnings("unchecked")
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
    ClassLoader classLoader, Object[] args, Set<String> names) {
    // 按照实现类名称集合长度声明对应实例数组，避免扩容带来性能消耗
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            // 根据实现类名加载类实例
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            // 获取构造器对象
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            // 使用BeanUtils工具类提供的实例构造方法获取实例，相比直接使用Constructor的newInstance，有更详细的异常处理
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```
至此，我们梳理出来`getSpringFactoriesInstances`这个方法作用是获取并创建Spring框架自己所需的工厂实例，主要组件既`SpringFactoriesLoader`，深入探究，其作用如下：
+ 从`META-INF/spring.factories`文件中把属性全部加载到一个Map<String, List<String>>结构的缓存中
+ key为factoryTypeName，value为factoryImplementationNames
+ 最后基于给定的工厂类型返回对应实现类的名称列表（相同类型可能有多种实现）

将`META-INF/spring.factories`内容贴在下面供参考：
```java
# Logging Systems
org.springframework.boot.logging.LoggingSystemFactory=\
org.springframework.boot.logging.logback.LogbackLoggingSystem.Factory,\
org.springframework.boot.logging.log4j2.Log4J2LoggingSystem.Factory,\
org.springframework.boot.logging.java.JavaLoggingSystem.Factory

# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader

# ConfigData Location Resolvers
org.springframework.boot.context.config.ConfigDataLocationResolver=\
org.springframework.boot.context.config.ConfigTreeConfigDataLocationResolver,\
org.springframework.boot.context.config.StandardConfigDataLocationResolver

# ConfigData Loaders
org.springframework.boot.context.config.ConfigDataLoader=\
org.springframework.boot.context.config.ConfigTreeConfigDataLoader,\
org.springframework.boot.context.config.StandardConfigDataLoader

# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers

# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.env.EnvironmentPostProcessorApplicationListener

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor,\
org.springframework.boot.env.RandomValuePropertySourceEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.DebugAgentEnvironmentPostProcessor

# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.context.config.ConfigDataNotFoundFailureAnalyzer,\
org.springframework.boot.context.properties.IncompatibleConfigurationFailureAnalyzer,\
org.springframework.boot.context.properties.NotConstructorBoundInjectionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanCurrentlyInCreationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanDefinitionOverrideFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BeanNotOfRequiredTypeFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.BindValidationFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.UnboundConfigurationPropertyFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.ConnectorStartFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoSuchMethodFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.NoUniqueBeanDefinitionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.PortInUseFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.ValidationExceptionFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyNameFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.InvalidConfigurationPropertyValueFailureAnalyzer,\
org.springframework.boot.diagnostics.analyzer.PatternParseFailureAnalyzer,\
org.springframework.boot.liquibase.LiquibaseChangelogMissingFailureAnalyzer

# Failure Analysis Reporters
org.springframework.boot.diagnostics.FailureAnalysisReporter=\
org.springframework.boot.diagnostics.LoggingFailureAnalysisReporter

# Database Initializer Detectors
org.springframework.boot.sql.init.dependency.DatabaseInitializerDetector=\
org.springframework.boot.flyway.FlywayDatabaseInitializerDetector,\
org.springframework.boot.jdbc.AbstractDataSourceInitializerDatabaseInitializerDetector,\
org.springframework.boot.jdbc.init.DataSourceScriptDatabaseInitializerDetector,\
org.springframework.boot.liquibase.LiquibaseDatabaseInitializerDetector,\
org.springframework.boot.orm.jpa.JpaDatabaseInitializerDetector,\
org.springframework.boot.r2dbc.init.R2dbcScriptDatabaseInitializerDetector

# Depends On Database Initialization Detectors
org.springframework.boot.sql.init.dependency.DependsOnDatabaseInitializationDetector=\
org.springframework.boot.sql.init.dependency.AnnotationDependsOnDatabaseInitializationDetector,\
org.springframework.boot.jdbc.SpringJdbcDependsOnDatabaseInitializationDetector,\
org.springframework.boot.jooq.JooqDependsOnDatabaseInitializationDetector,\
org.springframework.boot.orm.jpa.JpaDependsOnDatabaseInitializationDetector
```


###### 监听器的准备工作
接着我们按run方法中的执行顺序，依次看看各个私有方法的作用及实现，首先是`getRunListeners`方法，用于获取获`SpringApplicationRunListeners`对象，在上一篇中我们知道，`SpringApplicationRunListeners`其实就是对一组`SpringApplicationRunListener`集合的封装，对外暴露的方法和`SpringApplicationRunListener`相同，通过遍历的形式来触发集合内所有实例对应的方法。
```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    // 构造SpringApplicationRunListener构造器对应的参数数组
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    // 获取Spring工厂实例中SpringApplicationRunListener的实现集合，既EventPublishingRunListener，最后封装成SpringApplicationRunListeners对象返回
    return new SpringApplicationRunListeners(logger,
        getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

###### 配置环境的准备工作

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
    // 获取环境对象，如果当前没有，则基于webApplicationType类型创建对应的标准环境对象
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置propertySources和profile
    configureEnvironment(environment, applicationArguments.getSourceArgs());

    ConfigurationPropertySources.attach(environment);
    // 触发所有监听器的environmentPrepared函数，标志环境已准备好
    listeners.environmentPrepared(environment);
    // 
    bindToSpringApplication(environment);
    // 是否为定制化环境，默认为false，当setEnvironment方法被调用时会被置成true
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
	}
    ConfigurationPropertySources.attach(environment);
    return environment;
}

// 获取或创建新的环境实例
private ConfigurableEnvironment getOrCreateEnvironment() {
    // 如果当前SpringApplication环境属性不为空，则直接拿来用
    if (this.environment != null) {
        return this.environment;
    }
    // 基于webApplicationType的值创建对应的环境对象实例
    switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
    }
}

// 基于webApplicationType的值推断StandardEnvironment的具体类型
private Class<? extends StandardEnvironment> deduceEnvironmentClass() {
    switch (this.webApplicationType) {
        case SERVLET:
            return StandardServletEnvironment.class;
        case REACTIVE:
            return StandardReactiveWebEnvironment.class;
        default:
            return StandardEnvironment.class;
    }
}


/**
 * 用于为环境实例配置profile和配置属性来源的模版方法
 */
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    // 增加类型转换服务，默认为true
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    // 配置属性来源
    configurePropertySources(environment, args);
    // 配置环境profile
    configureProfiles(environment, args);
}

/**
 * 增加，移除或调整应用环境中的PropertySource
 */
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    // 增加命令行属性，默认为true
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        // 如果sources中已包含CommandLinePropertySource，则进行替换
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(
                new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        // 如果不包含则添加
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}

/**
 * 为应用环境配置哪些profiles为active，主要依据配置系统中或命令行中的spring.profiles.active的属性值。
 */
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```







