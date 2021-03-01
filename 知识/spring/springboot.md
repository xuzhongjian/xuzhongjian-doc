# springboot #

## springboot自动装配原理 ##

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import tk.mybatis.spring.annotation.MapperScan;

/**
 * @author zjxu97 at 2020-08-19 13:57
 */
@SpringBootApplication
@ComponentScan(value = {"com.xuzhongjian.base", "com.xuzhongjian.newsfeed", "com.xuzhongjian.config.db"})
@MapperScan(basePackages = {"com.xiaomi.mapper"})
public class ShortvideoApiSpringbootApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShortvideoApiSpringbootApplication.class);
    }
}
```

springboot的启动类中，最主要的就是 @SpringBootApplication 这个注解。这个注解标注在某个类上，就说明该类为 SpringBoot 的主配置类，使用这个类的 main 方法来启动 这个SpringBoot的应用。

### @SpringBootApplication 注解 ###

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)})
public @interface SpringBootApplication {
    // ... 
}
```

@SpringBootApplication 这个注解中:

1. @SpringBootConfiguration
   这个注解就是把Spring原生的 @Configuration 进行了一次封装，继承了 @Configuration 注解配置类的能力，所以使用 @SpringBootApplication 和 @SpringBootApplication 修饰的类，都可以作为Spring的配置类使用。也就是Springboot的启动类，也可以作为bean的配置类使用。

2. @EnableAutoConfiguration
   这个注解就是关于springboot自动装配原理的核心注解

### @SpringBootConfiguration ###

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
}
```

在 @SpringBootConfiguration 注解上还有一个注解为 @Conguration，即为@SpringBootConfiguration、@SpringBootApplication 都是一个配置类。 可以在启动类中配置spring bean。

### @EnableAutoConfiguration ###

@EnableAutoConfiguration 是springboot实现自动配置的核心注解，意思是开启自动配置功能。在 @EnableAutoConfiguration 内部又有两个重要的注解：

1. @AutoConfigurationPackage
   自动配置包
2. @Import(AutoConfigurationImportSelector.class)
   给容器中导入一个「自动配置导入选择器」

### @AutoConfigurationPackage ###

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

AutoConfigurationPackage 中最重要的是 @Import(AutoConfigurationPackages.Registrar.class) 这个注解，@Import 是Spring底层的一个注解，表示将注解内标示的一个组件注入到容器中。

#### AutoConfigurationPackages.Registrar ####

```java
    /**
     * {@link ImportBeanDefinitionRegistrar} to store the base package from the importing
     * configuration.
     */
    static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

        @Override
        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            register(registry, new PackageImport(metadata).getPackageName());
        }

        @Override
        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new PackageImport(metadata));
        }
    }
```

这个类是 AutoConfigurationPackages 中的一个静态内部类，作用是扫描主配置类同级目录以及子包，并将相应的组件导入到springboot创建管理的容器中。
new PackageImport(metadata).getPackageName() 运算出来的结果是 com.xuzhongjian.project 也就是SpringbootApplication这个启动类所在的包。

总结： @AutoConfigurationPackag 的作用就是将SpringBoot主配置类所在的包及其下面的所有子包里面的所有组件扫描到 Spring 容器中。

### @Import(AutoConfigurationImportSelector.class) ###

@Import 注解唯一的作用就是导入组件，这个注解需要导入的组件就是 AutoConfigurationImportSelector：

观察 AutoConfigurationImportSelector.process 方法：

```java
        @Override
        public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
            Assert.state(/* to check */);
            AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
            .getAutoConfigurationEntry(getAutoConfigurationMetadata(), data);

            this.autoConfigurationEntries.add(autoConfigurationEntry);
            for (String importClassName : autoConfigurationEntry.getConfigurations()) {
                this.entries.putIfAbsent(importClassName, annotationMetadata);
            }
        }
```

调用了getAutoConfigurationEntry()方法，该方法的作用就是告诉 Spring容器需要导入什么组件。

```java
    protected AutoConfigurationEntry getAutoConfigurationEntry(
            AutoConfigurationMetadata autoConfigurationMetadata,
            AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return new AutoConfigurationEntry(configurations, exclusions);
    }
```

在这个方法里面调用了 getCandidateConfigurations 方法，这个方法用来获取候选的配置。

```java
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
            AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
                getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        Assert.notEmpty(configurations,
                "No auto configuration classes found in META-INF/spring.factories. If you "
                        + "are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

这个方法中的 SpringFactoriesLoader.loadFactoryNames 整个自动配置的核心，getSpringFactoriesLoaderFactoryClass() 方法的返回值固定为 EnableAutoConfiguration.class。
后面 loadFactoryNames 方法继续调用 loadSpringFactories 。

loadSpringFactories 中的 urls 是指的各个依赖包里面的 META-INF/spring.factories 的地址。用 while 来遍历，获取每个包中的相关的配置。PropertiesLoaderUtils.loadProperties 使用这个方法将每个单独的 spring.factories 加载成一个 map ，再使用for循环将其加载到 result 中。

```java
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
    }

    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }

        try {
            Enumeration<URL> urls = (classLoader != null ?
                    classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                    ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            result = new LinkedMultiValueMap<>();
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    String factoryClassName = ((String) entry.getKey()).trim();
                    for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                        result.add(factoryClassName, factoryName.trim());
                    }
                }
            }
            cache.put(classLoader, result);
            return result;
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load factories from location [" +
                    FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
    }
```

## springboot的启动流程 ##

![spring启动](https://upload-images.jianshu.io/upload_images/6912735-51aa162747fcdc3d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

```java

@SpringBootApplication
public class ShortvideoApiSpringbootApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShortvideoApiSpringbootApplication.class);
    }
}
```

### new ###

在org.springframework.boot.SpringApplication类中的方法，其中的 SpringApplication 的创建方法都在其中：

```java
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        return run(new Class<?>[] { primarySource }, args);
    }

    // run

    public static ConfigurableApplicationContext run(Class<?>[] primarySources,
            String[] args) {
        return new SpringApplication(primarySources).run(args);
    }

    // new

    public SpringApplication(Class<?>... primarySources) {
        this(null, primarySources);
    }

    // this

    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        // 默认为空
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        // 将 ShortvideoApiSpringbootApplication 这个主启动类保存到一个hashSet中
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 判断这个application的类别，是reactive还是servlet还是none
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 从类路径下找到 META‐INF/spring.factories 配置的所有 ApplicationContextInitializer
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 从类路径下找到 META‐INF/spring.factories 配置的所有 ApplicationListener
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 通过抛出异常的方法，从方法的栈中，找到有main方法，也就是主配置类
        this.mainApplicationClass = deduceMainApplicationClass();
    }
```

进入了这个this的构造方法才进入了application的主要初始化的类中，其中最重要的两个方法就是：setInitializers 和 setListeners，其中的第一个方法就是对应用上下文的进行初始化的类，第二个方法对应用关闭、刷新、开启、停止等动作。

### run ###

![run方法](https://img-blog.csdnimg.cn/20200722161850507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzY5MTcyMw==,size_16,color_FFFFFF,t_70)

```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        // 申明一个ioc容器，用于返回的结果
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        
        // Headless模式是系统的一种配置模式。在系统可能缺少显示设备、键盘或鼠标这些外设的情况下可以使用该模式。一般是在程序开始激活headless模式，告诉程序，现在你要工作在Headless mode下，就不要指望硬件帮忙了，你得自力更生，依靠系统的计算能力模拟出这些特性来。
        configureHeadlessProperty();

        // 和新建应用的对象的过程中获取监听器的过程一致，都是从 META‐INF/spring.factories 中获取监听器
        SpringApplicationRunListeners listeners = getRunListeners(args);

        // 遍历上一步获取的所有 SpringApplicationRunListener，调用其 starting 方法
        listeners.starting();


        try {
            // 应用的参数，一般情况为为空
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

            // 准备环境，把上面获取到的listeners传过去
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            configureIgnoreBeanInfo(environment);
            Banner printedBanner = printBanner(environment);

            // 根据应用的类别：servlet、reactive、none，创建上下文
            context = createApplicationContext();

            // 从类路径下 META‐INF/spring.factories 获取 SpringBootExceptionReporter
            exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,new Class[] { ConfigurableApplicationContext.class }, context);

            // 准备容器
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);

            // 刷新容器
            refreshContext(context);

            // 空方法
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass)
                        .logStarted(getApplicationLog(), stopWatch);
            }

            // 调用所有的监听器的 start 方法
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            // 调用所有的监听器的 running 方法
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
```

### refresh ###

透过方法的调用栈，最后看到了 org.springframework.context.support.AbstractApplicationContext#refresh 方法。


```java
    // org.springframework.boot.SpringApplication#refreshContext

    private void refreshContext(ConfigurableApplicationContext context) {
        refresh(context);
        if (this.registerShutdownHook) {
            try {
                context.registerShutdownHook();
            }
            catch (AccessControlException ex) {
                // Not allowed in some environments.
            }
        }
    }

    // org.springframework.boot.SpringApplication#refresh

    protected void refresh(ApplicationContext applicationContext) {
        Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
        ((AbstractApplicationContext) applicationContext).refresh();
    }

    // org.springframework.context.support.AbstractApplicationContext#refresh

    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // 准备刷新
            prepareRefresh();

            // 创建BeanFactory工厂 告诉子类去刷新内部BeanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 准备BeanFactory 进行一些设置
            prepareBeanFactory(beanFactory);

            try {
                // BeanFactory准备完成后 后置处理器增强 默认为空
                postProcessBeanFactory(beanFactory);

                // 实例化实现了 BeanFactoryPostProcessor 接口的类中构造方法
                invokeBeanFactoryPostProcessors(beanFactory);

                // 注册实现了 bean 
                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // 增强空方法
                onRefresh();

                // 注册监听器
                registerListeners();

               /**
                 * 实例化剩下的所有的单例 bean （除了懒加载以外的）
                 * 执行构造方法1
                 * setter方法2
                 * 实现了BeanNameAware接口的setBeanName方法3
                 * 实现了BeanFactoryAware接口的 setBeanFactory(BeanFactory beanFactory)方法4
                 * 实现了ApplicationContextAware接口的setsetApplicationContext 方法5
                 * 实现了BeanPostProcessor接口的postProcessBeforeInitialization方法6 ，AOP原理
                 * 标有PostConstruct注解的方法 7
                 * 实现了InitializingBean接口的afterPropertiesSet方法8
                 * 调用配置了init-method方法 9
                 * 实现了BeanPostProcessor接口的postProcessAfterInitialization方法 10
                 */
                finishBeanFactoryInitialization(beanFactory);

                // 完成刷新Context,主要调用org.springframework.context.LifecycleProcessor接口onRefresh方法,发布事件ContextRefreshedEvent事件
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```

## 实例化SpringApplication时做了什么？ ##

1. 推断WebApplicationType，主要思想就是在当前的classpath下搜索特定的类
2. 搜索META-INF\spring.factories文件配置的ApplicationContextInitializer的实现类
3. 搜索META-INF\spring.factories文件配置的ApplicationListener的实现类
4. 通过创建一个异常，从堆栈信息中，通过main方法获取主应用类

## SpringApplication的run方法做了什么？ ##

1. 创建一个 StopWatch 并执行 start 方法，这个类主要记录任务的执行时间
2. 配置 Headless 属性，Headless 模式是在缺少显示屏、键盘或者鼠标时候的系统配置
3. 在文件 META-INF\spring.factories 中获取 SpringApplicationRunListener 接口的实现类 EventPublishingRunListener ，主要发布springboot应用的事件，用来定义监听器的时间点：
   - org.springframework.boot.SpringApplicationRunListeners
   - 1、starting 在一切准备就绪，准备启动时。
   - 2、environmentPrepared 在prepareEnvironment方法内部调用。
   - 3、contextPrepared 在prepareContext的开始阶段被调用。
   - 4、contextLoaded 在prepareContext的最后一步被调用。
   - 5、started 在所有执行完成，ApplicationRunner和CommandLineRunner回调之前。
   - 6、running 在run方法最后单独使用try catch执行，只要上面没有异常，项目已经启动完成。那么running回调异常也不能影响正常流程。
   - 7、failed 在handleRunFailure异常处理中被调用。
4. 监听器调用 starting 方法
5. 把输入参数转成 DefaultApplicationArguments 类
6. 创建Environment并设置比如环境信息，系统熟悉，输入参数和profile信息
7. 打印Banner信息
8. 创建Application的上下文，根据WebApplicationTyp来创建Context类，如果非web项目则创建AnnotationConfigApplicationContext，在构造方法中初始化AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner
9. 在文件META-INF\spring.factories中获取SpringBootExceptionReporter接口的实现类FailureAnalyzers
10. 准备application的上下文
    - 初始化ApplicationContextInitializer
    - 执行Initializer的contextPrepared方法，发布ApplicationContextInitializedEvent事件
    - 如果延迟加载，在上下文添加处理器LazyInitializationBeanFactoryPostProcessor
    - 执行加载方法，BeanDefinitionLoader.load方法，主要初始化了AnnotatedGenericBeanDefinition
    - 执行Initializer的contextLoaded方法，发布ApplicationContextInitializedEvent事件
11. 刷新上下文，在这里真正加载bean到容器中。如果是web容器，会在onRefresh方法中创建一个Server并启动。

