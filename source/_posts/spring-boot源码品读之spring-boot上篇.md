---
title: "spring-boot源码品读之spring-boot上篇"
catalog: true
date: 2022-03-22 23:38:24
subtitle: ""
header-img: "/img/default.png"
tags:
- Java
- spring-boot
catagories:
- spring
---

#### Overall
spring-boot作为整个框架最核心的模块，目前目录下共有包含admin,ansi,builder等28个文件夹和`SpringApplication`，`SpringBootBanner`等28个java文件。我决定分上、中、下篇来对他们一一解读。上篇来根路径下除了`SpringApplication`之外的接口和类。由于`SpringApplication`作为应用的启动器，内容相对复杂太多，单独放在中篇中。下篇则是对其他28个文件夹及其内部类进行介绍。

#### 根目录下Java文件

##### WebApplicationType.java
`WebApplicationType`是枚举类型，主要有三个值`NONE`，`SERVLET`，`REACTIVE` 用来表示spring-boot所支持的Web应用程序的类型。

|Item|Desc|
|--|--|
|NONE|The application should not run as a web application and should not start an embedded web server.|
|SERVLET|The application should run as a servlet-based web application and should start an embedded servlet web server.|
|REACTIVE|The application should run as a reactive web application and should start an embedded reactive web server.|

两个静态方法`deduceFromClasspath`和`deduceFromApplicationContext` 分别用具有标志型的Class类和ApplicationContext两种方式来推断当前Application属于那种Web类型。

+ deduceFromClasspath 推断的优先顺序为REACTIVE > NONE > SERVLET. 
+ deduceFromApplicationContext 推断优先顺序为SERVLET > REACTIVE > NONE.

##### StartupInfoLogger.java
从class命名上我们就可以知道此class是用来记录应用程序启动时的日志的。私有静态成员变量有两个`logger`，`HOST_NAME_RESOLVE_THRESHOLD`。私有实例成员变量一个`sourceClass`。

两个无返回值方法`logStarting`，`logStarted`，作为Spring-boot程序启动时开始打印日志的入口。
+ logStarting 应该在程序启动时被调用。参数applicationLog不能为空。applicationLog会记录两个级别的log。Info级别记录`getStartingMessage`内容，debug级别记录`getRunningMessage`。
+ logStarted 应该在程序完成启动过程后被调用。当日志的输出级别为Info时会记录下`getStartedMessage`的日志内容。

三个主要的输出日志的私有方法`getStartingMessage`，`getRunningMessage`，`getStartedMessage`用来组织日志内容，采用建造者设计模式，将日志按照类别拆分若干个私有方法去各自实现，值得注意的是这三个方法的返回值类型都是`CharSequence`。

##### SpringBootVersion.java
该类用来推断和返回SpringBoot的版本。final class，无法被继承。对外暴露的静态方法`getVersion`，内部的具体实现则放在私有静态方法`determineSpringBootVersion`中。推断的主要是基于jar文件manifest中的`IMPLEMENTATION_VERSION`属性。

##### SpringBootExceptionReporter.java 
这是一个回调接口用来将应用程序启动时的错误暴露给用户。函数式接口（@FunctionalInterface）只允许有一个抽象方法既`reportException`。

##### SpringBootExceptionHandler.java 
在一般Java程序中，遇到异常一般都会终止程序并退出JVM。但是对于spring-boot应用程序而言，随随便便就退出JVM是无法保证应用程序的稳定性的，因此必须进行特殊处理。该类实现了`UncaughtExceptionHandler`接口，并需要重写`uncaughtException(Thread thread, Throwable ex)`方法，该方法会在当给定线程因为给定的异常被终止时触发，可以进行抑制处理。

`SpringBootExceptionHandler`成员变量有：
+ private static final Set<String> LOG_CONFIGURATION_MESSAGES
+ private static LoggedExceptionHandlerThreadLocal handler
+ private final UncaughtExceptionHandler parent
+ private final List<Throwable> loggedExceptions
+ private int exitCode = 0

我们先来看一下私有静态内部类`LoggedExceptionHandlerThreadLocal`，继承Threadlocal，用来附加和跟踪处理程序。而`LoggedExceptionHandlerThreadLocal`作为`SpringBootExceptionHandler`的静态成员变量在类加载时就被初始化了，因此每一个Spring-boot应用程序线程中都会带有SpringBootExceptionHandler的实例，可以通过静态方法`forCurrentThread`方法获取。
```java
private static class LoggedExceptionHandlerThreadLocal 
    extends ThreadLocal<SpringBootExceptionHandler> {
    @Override
    protected SpringBootExceptionHandler initialValue() {
        SpringBootExceptionHandler handler = new SpringBootExceptionHandler(
            Thread.currentThread().getUncaughtExceptionHandler());
        Thread.currentThread().setUncaughtExceptionHandler(handler);
        return handler;
    }
}
// SpringBootExceptionHandler 构造器，接收参数为相同接口父级handler
SpringBootExceptionHandler(UncaughtExceptionHandler parent) {
    this.parent = parent;
}
// 获取当前线程的SpringBootExceptionHandler实例
static SpringBootExceptionHandler forCurrentThread() {
    return handler.get();
}
```

接下来我们再来看一下最核心的`uncaughtException`方法，如果处理两个事情，第一是要判断是否要交给父级处理器处理，第二是要判断程序是否需要终止并退出JVM。
```java
@Override
public void uncaughtException(Thread thread, Throwable ex) {
    try {
        if (isPassedToParent(ex) && this.parent != null) {
            this.parent.uncaughtException(thread, ex);
        }
    }
    finally {
        this.loggedExceptions.clear();
        if (this.exitCode != 0) {
            System.exit(this.exitCode);
        }
    }
}
```
+ 是否要交给父级处理器的判断依据有两方面，如果异常是日志配置异常，或者并未注册到成员变量`loggedExceptions`中，则需要交给父级处理器处理。由此同时可见，`SpringBootExceptionHandler`有着比父级处理器更高的优先级。
+ 程序退出则要看exitCode是否为0，如果为0则可以忽略。如果不为0，则说明程序遇到错误（error），需要终止程序并退出。

最后有两个实例方法`registerLoggedException`，`registerExitCode`分别来维护成员变量`loggedExceptions` 和 `exitCode`的值。

##### SpringBootConfiguration.java
`@SpringBootConfiguration`注解同`@Configuration`注解一样，可以标注当前类是配置类，并会将当前类内的方法注册成bean加入到spring-boot容器内。

##### SpringBootBanner.java
spring-boot标志话的启动横幅，没啥值得多说的。。。

##### SpringApplicationRunListeners.java
`SpringApplicationRunListeners`本质上是`SpringApplicationRunListener`的集合。对外一共暴露7个方法，`starting`，`environmentPrepared`，`contextPrepared`，`contextLoaded`，`started`，`running`，`failed` 均是遍历集合中`SpringApplicationRunListener`对象，并调用对应的方法。

##### SpringApplicationRunListener.java
`SpringApplicationRunListener` 接口主要是为了让Spring Boot在启动初始化的过程中可以通过`SpringApplication`接口回调来让用户在启动的各个流程中实现自定义逻辑。

|Item|Desc|
|--|--|
|starting|Called immediately when the run method has first started. Can be used for very early initialization.|
|environmentPrepared|Called once the environment has been prepared, but before the ApplicationContext has been created.|
|contextPrepared|Called once the ApplicationContext has been created and prepared, but before sources have been loaded.|
|contextLoaded|Called once the application context has been loaded but before it has been refreshed.|
|started|The context has been refreshed and the application has started but CommandLineRunner CommandLineRunners and ApplicationRunner ApplicationRunners have not been called.|
|running|Called immediately before the run method finishes, when the application context has been refreshed and all CommandLineRunner CommandLineRunners and ApplicationRunner ApplicationRunners have been called.|
|failed|alled when a failure occurs when running the application.|

##### Banner.java
函数式接口，内部枚举类`Mode`共有三个值：`OFF`，`CONSOLE`，`LOG`。对外需要实现的方法`printBanner`用于将banner打印到给定的`PrintStream`中。
```java
void printBanner(Environment environment, Class<?> sourceClass, PrintStream out);
```

##### SpringApplicationBannerPrinter.java
这个类用来打印应用程序的启动横幅，在spring-boot应用程序启动时会默认打印Spring-boot默认的Banner，但其实用户也可以通过定制化Banner的路径，打印属于自己的横幅。`SpringApplicationBannerPrinter`支持两种输入方式，文本和图片。

我们先来看一下成员变量和构造器情况：
```java
// 静态变量 - Banner在配置Bean中的属性名
static final String BANNER_LOCATION_PROPERTY = "spring.banner.location";
static final String BANNER_IMAGE_LOCATION_PROPERTY = "spring.banner.image.location";
// 默认Banner的路径
static final String DEFAULT_BANNER_LOCATION = "banner.txt";
// 所支持的图片Banner文件类型
static final String[] IMAGE_EXTENSION = { "gif", "jpg", "png" };
// Spring-boot默认Banner
private static final Banner DEFAULT_BANNER = new SpringBootBanner();
// 私有实例变量 - 资源加载器和备胎Banner
private final ResourceLoader resourceLoader;
private final Banner fallbackBanner;

SpringApplicationBannerPrinter(ResourceLoader resourceLoader, Banner fallbackBanner) {
    this.resourceLoader = resourceLoader;
    this.fallbackBanner = fallbackBanner;
}
```

其次是私有静态内部类`PrintedBanner`，利用装饰者模式对已经打印过的Banner进行封装，后续重复使用时不再需要制定源文件。
```java
private static class PrintedBanner implements Banner {
    private final Banner banner;
    private final Class<?> sourceClass;

    PrintedBanner(Banner banner, Class<?> sourceClass) {
        this.banner = banner;
        this.sourceClass = sourceClass;
    }

    @Override
    public void printBanner(Environment environment, Class<?> sourceClass, PrintStream out) {
        sourceClass = (sourceClass != null) ? sourceClass : this.sourceClass;
        this.banner.printBanner(environment, sourceClass, out);
    }
}
```

另一个私有静态内部类`Banners`其实就是一组Banner的集合，同样实现了`Banner`接口，其实现则是依次将集合中的Banner进行打印。

接下来看一下主要的两个`print`方法，返回值都是已被打印了的`PrintedBanner`对象，便于调用端后续操作。方法内的实现主要是三步：加载横幅内容，打印内容，返回已打印的横幅对象。
```java
// 供日志形式打印横幅
Banner print(Environment environment, Class<?> sourceClass, Log logger) {
    Banner banner = getBanner(environment);
    try {
        logger.info(createStringFromBanner(banner, environment, sourceClass));
    }
    catch (UnsupportedEncodingException ex) {
        logger.warn("Failed to create String for banner", ex);
    }
    return new PrintedBanner(banner, sourceClass);
}
// 供输出流形式打印横幅
Banner print(Environment environment, Class<?> sourceClass, PrintStream out) {
    Banner banner = getBanner(environment);
    banner.printBanner(environment, sourceClass, out);
    return new PrintedBanner(banner, sourceClass);
}
```

##### ResourceBanner.java
`ResourceBanner`同样实现了`Banner`接口，用来打印原文本的资源。先来看一下属性和构造器，
```java
private static final Log logger = LogFactory.getLog(ResourceBanner.class);
private Resource resource;

public ResourceBanner(Resource resource) {
    // 验证了Resource的有效性
    Assert.notNull(resource, "Resource must not be null");
    Assert.isTrue(resource.exists(), "Resource must exist");
    this.resource = resource;
}
```

最核心的`printBanner`方法如下：
```java
@Override
public void printBanner(Environment environment, Class<?> sourceClass, PrintStream out) {
    try {
        // 加载Banner资源
        String banner = StreamUtils.copyToString(this.resource.getInputStream(),
        environment.getProperty("spring.banner.charset", Charset.class, StandardCharsets.UTF_8));
        // 从运行环境中获取环境，版本，ansi，标题等信息并注入到横幅中
        for (PropertyResolver resolver : getPropertyResolvers(environment, sourceClass)) {
            banner = resolver.resolvePlaceholders(banner);
        }
        out.println(banner);
    }
    catch (Exception ex) {
        logger.warn(LogMessage.format("Banner not printable: %s (%s: '%s')", 
        this.resource, ex.getClass(), ex.getMessage()), ex);
    }
}
```

##### ImageBanner.java
`ImageBanner`同样实现了`Banner`接口，用来实现图片资源转换ASCII艺术图形，并打印成横幅。
具体实现后面有时间单独出一篇日志研究吧，把图片转换成字符码还是挺有意思的。

##### LazyInitializationExcludeFilter.java
`LazyInitializationExcludeFilter`被注解标注为函数式接口，源码注释说是用来过滤掉那些被`LazyInitializationBeanFactoryPostProcessor`在`AbstractBeanDefinition`中设置了`setLazyInit(boolean) lazy-init`的beans definitions。翻译成人话：当Spring Application中设置了全局懒加载时，通过这个filter可以指定哪些beans可以在项目启动时强制加载。

方法`isExcluded`，满足方法Bean definitions会被强制加载。

##### LazyInitializationBeanFactoryPostProcessor.java
`LazyInitializationBeanFactoryPostProcessor`实现了`BeanFactoryPostProcessor`接口，可以在Spring加载Bean definitions之后，beans创建之前，修改beans的属性。整个processor的作用就是当设置全局懒加载时，根据配置和定义的一个或多个`LazyInitializationExcludeFilter`来更新bean懒加载的属性值。
```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    // Take care not to force the eager init of factory beans when getting filters
    Collection<LazyInitializationExcludeFilter> filters = beanFactory
        .getBeansOfType(LazyInitializationExcludeFilter.class, false, false).values();
    for (String beanName : beanFactory.getBeanDefinitionNames()) {
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
        if (beanDefinition instanceof AbstractBeanDefinition) {
            postProcess(beanFactory, filters, beanName, (AbstractBeanDefinition) beanDefinition);
        }
    }
}

private void postProcess(ConfigurableListableBeanFactory beanFactory,
    Collection<LazyInitializationExcludeFilter> filters, String beanName,
    AbstractBeanDefinition beanDefinition) {
    Boolean lazyInit = beanDefinition.getLazyInit();
    if (lazyInit != null) {
        return;
    }
    Class<?> beanType = getBeanType(beanFactory, beanName);
    // 如果所有filter都没有将该bean排除掉，则设置为懒加载
    if (!isExcluded(filters, beanName, beanDefinition, beanType)) {
        beanDefinition.setLazyInit(true);
    }
}
```

##### ExitCodeExceptionMapper.java
策略接口，可用于在异常和exit code之间提供映射。
```java
// 传入异常，返回应用程序应返回的exit code
int getExitCode(Throwable exception);
```

##### ExitCodeGenerator.java
函数式接口，用于通过执行命令行的形式生成`exit code`。
```java
// 返回应用程序应返回的exit code
int getExitCode();
```

##### ExitCodeGenerators.java
`ExitCodeGenerators`维护了一组`ExitCodeGenerator`实例，并且允许返回计算后的`exit code`。

先来看一下私有静态内部类`MappedExitCodeGenerator`，通过构造器来绑定exception和`ExitCodeExceptionMapper`，并且采用适配器模式，将`ExitCodeExceptionMapper`中的有参的`getExitCode`适配成无参。
```java
private static class MappedExitCodeGenerator implements ExitCodeGenerator {

    private final Throwable exception;
    private final ExitCodeExceptionMapper mapper;

    MappedExitCodeGenerator(Throwable exception, ExitCodeExceptionMapper mapper) {
        this.exception = exception;
        this.mapper = mapper;
    }

    @Override
    public int getExitCode() {
        return this.mapper.getExitCode(this.exception);
    }
}
```

基于集合中的所有`ExitCodeGenerator`来计算并返回最终的exit code，从实现逻辑上看，返回结果为集合中最后的绝对值相对较大值。但为什么这样设计？TODO 
```java
int getExitCode() {
    int exitCode = 0;
    for (ExitCodeGenerator generator : this.generators) {
        try {
            int value = generator.getExitCode();
            if (value > 0 && value > exitCode || value < 0 && value < exitCode) {
                exitCode = value;
            }
        }
        catch (Exception ex) {
            // 如果exitCode 不为0则返回exitCode，为0则返回1
            exitCode = (exitCode != 0) ? exitCode : 1;
            ex.printStackTrace();
        }
    }
	return exitCode;
}
```

最后有4个不同参数的`addAll`方法和两个`add`方法，用于将新的`ExitCodeGenerator`注入集合。

##### ExitCodeEvent.java
继承`ApplicationEvent`对象，当应用程序从`ExitCodeGenerator`获取到exit code后将会触发这个事件。
```java
private final int exitCode;

/**
 * Create a new {@link ExitCodeEvent} instance.
 * @param source the source of the event
 * @param exitCode the exit code
 */
public ExitCodeEvent(Object source, int exitCode) {
    super(source);
    this.exitCode = exitCode;
}

/**
 * Return the exit code that will be used to exit the JVM.
 * @return the exit code
 */
public int getExitCode() {
    return this.exitCode;
}
```

##### EnvironmentConverter.java
`EnvironmentConverter`是用来转换`Environment`类型的工具类。先来看一下成员变量和构造器：
```java
private static final String CONFIGURABLE_WEB_ENVIRONMENT_CLASS = "org.springframework.web.context.ConfigurableWebEnvironment";
private static final Set<String> SERVLET_ENVIRONMENT_SOURCE_NAMES;

static {
    Set<String> names = new HashSet<>();
    names.add(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME);
    names.add(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME);
    names.add(StandardServletEnvironment.JNDI_PROPERTY_SOURCE_NAME);
    SERVLET_ENVIRONMENT_SOURCE_NAMES = Collections.unmodifiableSet(names);
}

private final ClassLoader classLoader;

EnvironmentConverter(ClassLoader classLoader) {
	this.classLoader = classLoader;
}
```

对外暴露的方法为`convertEnvironmentIfNecessary`，用来将给定的`ConfigurableEnvironment`对象转换成指定类型环境，前提指定环境必须属于`StandardEnvironment`的子类。如果给定的环境对象已经是相通的类型，既不会进行转换操作。
```java
StandardEnvironment convertEnvironmentIfNecessary(ConfigurableEnvironment environment,
        Class<? extends StandardEnvironment> type) {
    if (type.equals(environment.getClass())) {
        return (StandardEnvironment) environment;
    }
    return convertEnvironment(environment, type);
}

private StandardEnvironment convertEnvironment(ConfigurableEnvironment environment,
        Class<? extends StandardEnvironment> type) {
    StandardEnvironment result = createEnvironment(type);
    result.setActiveProfiles(environment.getActiveProfiles());
    result.setConversionService(environment.getConversionService());
    copyPropertySources(environment, result);
    return result;
}
```

##### DefaultApplicationArguments.java
`ApplicationArguments`的默认实现，实现接口包括`getSourceArgs`，`getOptionNames`，`containsOption`，`getOptionValues`，`getNonOptionArgs`。

##### CommandLineRunner.java
函数式接口，用于标志着SpringApplication中的Bean应如何执行。可以在同一个应用上下文中定义多个`CommandLineRunner`并用`order`注解来定义加载顺序。
```java
/**
 * Callback used to run the bean.
 * @param args incoming main method arguments
 * @throws Exception on error
 */
void run(String... args) throws Exception;
```

##### ClearCachesApplicationListener.java
`ClearCachesApplicationListener`用于清除上下文中的缓存，实现`ApplicationListener`接口，当SpringApplication触发`ContextRefreshedEvent`事件时会分别清理掉通过反射获取到的类方法和属性缓存，同时清除掉类加载器的缓存。
```java

@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 清除ReflectionUtils中declaredMethodsCache和declaredFieldsCache
    ReflectionUtils.clearCache();
    // 清除类加载器缓存
    clearClassLoaderCaches(Thread.currentThread().getContextClassLoader());
}

private void clearClassLoaderCaches(ClassLoader classLoader) {
    if (classLoader == null) {
        return;
    }
    try {
        Method clearCacheMethod = classLoader.getClass().getDeclaredMethod("clearCache");
        clearCacheMethod.invoke(classLoader);
    }
    catch (Exception ex) {
        // Ignore
    }
    // 递归清除掉所有父级类加载器缓存
    clearClassLoaderCaches(classLoader.getParent());
}
```

##### BeanDefinitionLoader.java
Spring boot最重要的组件之一，从底层XML和JavaConfig中加载bean definitions。作为`AnnotatedBeanDefinitionReader`, `XmlBeanDefinitionReader`和`ClassPathBeanDefinitionScanner`的一个简单的门面。

先来看一下成员变量和构造器
```java
// sources 来源于构造器可变参数
private final Object[] sources;
// 三种主要的bean definition加载方式的reader
private final AnnotatedBeanDefinitionReader annotatedReader;
private final XmlBeanDefinitionReader xmlReader;
private BeanDefinitionReader groovyReader;

private final ClassPathBeanDefinitionScanner scanner;
private ResourceLoader resourceLoader;

/**
 * Create a new {@link BeanDefinitionLoader} that will load beans into the specified
 * {@link BeanDefinitionRegistry}.
 * @param registry the bean definition registry that will contain the loaded beans
 * @param sources the bean sources
 */
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    this.xmlReader = new XmlBeanDefinitionReader(registry);
    if (isGroovyPresent()) {
        this.groovyReader = new GroovyBeanDefinitionReader(registry);
	}
    this.scanner = new ClassPathBeanDefinitionScanner(registry);
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```

三个对外暴露的set方法`setBeanNameGenerator`，`setResourceLoader`，`setEnvironment`供外部注入`beanNameGenerator`，`resourceLoader`，`environment`到底层的哥哥loader和reader中。
```java
/**
 * Set the bean name generator to be used by the underlying readers and scanner.
 * @param beanNameGenerator the bean name generator
 */
void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
    this.annotatedReader.setBeanNameGenerator(beanNameGenerator);
    this.xmlReader.setBeanNameGenerator(beanNameGenerator);
    this.scanner.setBeanNameGenerator(beanNameGenerator);
}

/**
 * Set the resource loader to be used by the underlying readers and scanner.
 * @param resourceLoader the resource loader
 */
void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
    this.xmlReader.setResourceLoader(resourceLoader);
    this.scanner.setResourceLoader(resourceLoader);
}

/**
 * Set the environment to be used by the underlying readers and scanner.
 * @param environment the environment
 */
void setEnvironment(ConfigurableEnvironment environment) {
    this.annotatedReader.setEnvironment(environment);
    this.xmlReader.setEnvironment(environment);
    this.scanner.setEnvironment(environment);
}
```

加载资源的入口在无参的`load`方法中，遍历所有sources对象并返回被加载的bean的总数。
```java
/**
 * Load the sources into the reader.
 * @return the number of loaded beans
 */
int load() {
    int count = 0;
    for (Object source : this.sources) {
        count += load(source);
    }
    return count;
}
```

在`load(source)`中将非null的source的类型划分为四类：`Class<?>`，`Resource`，`Package`，`CharSequence`。之后再基于不同的资源类型调用不同的加载器进行加载。例如对待`Class<?>`类型，会用`annotatedReader`来进行加载，对于`Resource`类型的资源，优先考虑使用groovyReader进行加载，如果不是`groovy`的资源则使用xmlReader进行加载。值得注意的是`CharSequence`相对会复杂一些，需要对内容进行推断。
```java
private int load(Object source) {
    Assert.notNull(source, "Source must not be null");
    if (source instanceof Class<?>) {
        return load((Class<?>) source);
	}
    if (source instanceof Resource) {
        return load((Resource) source);
    }
    if (source instanceof Package) {
        return load((Package) source);
    }
    if (source instanceof CharSequence) {
        return load((CharSequence) source);
    }
    throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

##### ApplicationRunner.java
函数式接口，用于表明bean在应在`SpringApplication`中运行，多个`ApplicationRunner`可以在同个应用上下文中被定义，并用`ordered`接口或`@order`注解来定义执行顺序。
```java
/**
 * Callback used to run the bean.
 * @param args incoming application arguments
 * @throws Exception on error
 */
void run(ApplicationArguments args) throws Exception;
```

##### ApplicationArguments.java
接口，用于提供对所启动的`SpringApplication`参数的访问。方法包括`getSourceArgs`，`getOptionNames`，`getOptionValues`等等。





