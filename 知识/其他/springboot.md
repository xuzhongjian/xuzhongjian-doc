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