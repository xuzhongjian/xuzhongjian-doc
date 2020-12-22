# 测试方式 #

```java
package com.xuzhongjian;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;


/**
 * @author zjxu97 at 2020-06-04 16:44
 */

@SpringBootApplication
public class AopApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(AopApplication.class, args);
        //BeanFactory beanFactory = context.getBeanFactory();
        //Object bean = beanFactory.getBean("s");
    }
}
```



# Component #

P36-2.2

component：组件。在类上标注@Component注解，表示这个类会作为组件类，并告知Spring需要为这个类创建bean。如果需要将bean标识为指定的名字，那么可以额外配置@Component注解的属性

```java
@Component
public class City {
}
```

```java
@Component("hometownCity")
public class City {
}
```

@Component注解默认是不开启的，需要使用@ComponentScan注解Spring寻找@Component的类，并为其创建bean。

# Named #

@Named是Java提供的依赖注入的规范中的一个注解。可以使用@Named注解指定bean的名字，Spring支持@Named注解作为@Component注解的替代方案。

```java
@Named("hometownCity")
public class City {
}
```

# ComponentScan #

P36-2.3

***SpringBoot在没配置@ComponentScan的情况下，默认只扫描和主类处于同包下的Class。***

通过创建带有@ComponentScan注解的配置类，来开启对应位置的@Component。

@ComponentScan默认会扫描和配置类同一个包以及这个包的子包下面的类，如果上述位置的类附带了@Component注解，那么Spring会将其视为组件类，并且为这个类创建bean。

@ComponentScan除了使用默认的方式来指定扫描的配置类的位置以外，还可以使用字符串来指定一个或者是多个目录位置。当然这种方式是不安全的，随着项目的重构，这样的方式指定的基础包就会出现错误。

```java
@ComponentScan("com.xuzhongjian")
@ComponentScan(basePackages = "com.xuzhongjian")
@ComponentScan(basePackages = {"com.xuzhongjian.model", "com.xuzhongjian.common"})
```

为了避免这种错误，还可以使用类来指定扫描的包的位置：

```java
@ComponentScan(basePackageClasses = User.class)
@ComponentScan(basePackageClasses = {User.class, City.class})
```

最好的方式是在需要被标记成组件类的包中，定义一个用来标记的空接口（marked interface）。这样能做到业务代码和配置的解耦。

# Configuration #

使用@Configuration来表示这个类是一个配置类，配置类中包含Spring应用context中创建bean的细节。@Configuration注解可以用Java代码的形式实现Spring中xml配置文件中配置的效果。

```java
@Configuration
public class JavaConfig { 
}
```

# Bean #

P45

@Bean注解使用在配置类中的方法上，表示这个方法返回的对象需要注册成为Spring context中的bean。默认情况下使用@Bean方法生成的bean的id和方法名是一样的，但是也可以指定为自定的id：

```java
@Configuration
public class Config{
    @Bean(name = "xuzhongjian")
    public User user(){
        return new User();
    }
}
```

# Autowired #

1. 使用在方法上
    ```java
    @Component
    public class A{
        private B b;
    
        @Autowired
        public A(B b){
            this.b = b;
        }
        
    	@Autowired
        public void setB(B b){
            this.b = b;
        }
    }
    ```
    
2. 使用在成员变量上

    ```java
    @Controller
    public class UserController{
        @Autowired
        private UserService userService;
    }
    ```

@Autowired表示由Spring传入一个在类型上符合的bean。可以使用在方法上，也可以使用在成员变量上。

# Inject #

Inject 和 Named 都是Java的依赖注入的规范，Spring支持使用这两个注解来对bean进行注入。大多数场景下，@Inject和@Autowired作用相同，可以相互替代使用。

# Primary #

当一个类存在多个符合条件的bean，然后需要对这个类进行注入的时候。Spring会由于无法选择，抛出NoUniqueBeanDefinitionException。在这种场景下，可以使用@Primary注解来设置首选的bean。

```java
@Primary
@Component
public class A implements B{
    // ...
}
```

```java
@Configuration
public class JavaConfig{
    
    @Bean
    @Primary
    public B a(){
        return new A();
    }
}
```

# Qualifier #

P79

@Qualifier注解的作用是指定一个特定的识别符号。如果一个类存在多个@Primary，那么在注入的时候也无法指定一个特定的bean。那么就能通过@Qualifier指定一个唯一的bean。

@Qualifier注解，既能在注册bean的时候使用，也能在注入bean的时候使用。也就是说，既能和@Bean、@Component搭配使用，也能和@Autowired搭配使用。这很好理解，注册bean时候就相当于是起一个名字，注入bean的时候叫这个名字，然后就能获得一个特定bean。

```java
@Component
public class Cake implements Dessert{ ... }

@Component
@Qualifier("iceCream")
public class IceCream implements Dessert{ ... }

@Component
public class Cookies implements Dessert{ ... }

@Controller
class AaController{
    @Autowired
    @Qualifier("iceCream")
    private Dessert dessert;
}
```

当一个Qualifier没办法满足的时候，我们尝试用两个@Qualifier来解决这个问题，但是同一个位置不能添加两个相同的注解，那么这个时候我们创建两个不同的注解来实现这个问题：

```java
@Target({ElementType.CONSTRUCTOR,ElementType.METHOD,
         ElementType.FIELD,ElementType.Type})
@Retention(RetentionPlicy.RUNTIME)
@Quailfier
public @interface Cold{}
```
```java
@Target({ElementType.CONSTRUCTOR,ElementType.METHOD,
         ElementType.FIELD,ElementType.Type})
@Retention(RetentionPlicy.RUNTIME)
@Quailfier
public @interface Creamy{}
```

通过上面的方法就能实现两个配置就能同时使用两个@Quailfier注解。

# Conditional #

Spring4引入了一个新的注解：Conditional。

```java
@Configuration
public class MagicConfig {
    @Bean
    @Conditional(MagicExistsCondition.class)
    public MagicBean magicBean() {
        return new MagicBean();
    }
}
```

```java
public class MagicExistsCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        return env.containsProperty("magic");
    }
}
```

设置给@Conditional的类必须是实现了Condition接口的类。Condition接口定义了一个方法：matches，这个方法返回一个布尔值。如果是true，那么就确定生成这个bean，如果是false，那么就忽略这个bean。

```java
public interface Condition {
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

其中matches方法的传入值有两个：

```java
public interface ConditionContext {
	BeanDefinitionRegistry getRegistry();
	ConfigurableListableBeanFactory getBeanFactory();
	Environment getEnvironment();
	ResourceLoader getResourceLoader();
	ClassLoader getClassLoader();
}
```

```java
public interface AnnotatedTypeMetadata {
	boolean isAnnotated(String annotationType);
	Map<String, Object> getAnnotationAttributes(String annotationType); getAnnotationAttributes(String annotationType, boolean classValuesAsString);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationType);
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationType, boolean classValuesAsString);
}
```

通过传入的这两个对象，可以获取这个环境、上下文、注解的相关的信息，然后自定义返回一个boolean，告知Spring是否需要注册这个bean。

# Profile #

Spring引入了profile 的功能。使用profile可以指定由@Configuration修饰的配置类属于的环境，也可以在配置类中指定@Bean对应的环境。

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // ...
    }

    @Bean
    @Profile("qa")
    public DataSource qaDataSource() {
        // ...
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        // ...
    }
}
```

激活profile

Spring在确定哪个profile处于激活状态时，需要依赖两个独立的属性

+ spring.profiles.active
+ spring.profiles.default

如果设置了spring.profiles.active属性的话，那么就会激活对应的profile。但如果没有设置spring.profiles.active属性的话，那么会查找spring.profiles.default的值。如果两者均没有设置的话，那就没有激活的profile，就只会创建那些没有定义在 profile中的bean。

有多种方式来设置这两个属性:

1. 作为DispatcherServlet的初始化参数
2. 作为Web应用的上下文参数
3. 作为JNDI条目
4. 作为环境变量
5. 作为JVM的系统属性
6. 在集成测试类上，使用@ActiveProfiles注解设置
7. ***Springboot在yml文件中设置这两个属性。***

@Profile实现方法：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {
    String[] value();
}
```

```java
class ProfileCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        if (context.getEnvironment() != null) {
            MultiValueMap<String, Object> attrs 
                = metadata.getAllAnnotationAttributes(Profile.class.getName());
            if (attrs != null) {
                for (Object value : attrs.get("value")) {
                    if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                        return true;
                    }
                }
                return false;
            }
        }
        return true;
    }
}
```

# Scope #

在默认情况写，Spring context中的作用域默认为单例。也就是说，在默认情况下，无论一个bean注入其他的bean多少次，使用的都是同一个实例对象。

如果有一些实例会保持一些状态，那么如果再使用单例的bean，就会产生数据上的错误。Spring定义了四个作用域：

+ 单例（Singleton）在整个应用中，只创建一次bean的实例。
+ 原型（Prototype）每次注入或通过context获取时，都创建一个bean的实例。
+ 会话（Session）在web应用中，为每个会话创建一个bean的实例。
+ 请求（Request）在web应用中，为每个请求创建一个bean的实例。

设定方式：

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_SESSION)
public class Notebook{ ... }
```

```java
@Configuration
public class SpringConfig{
    
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_REQUEST)
    public B b(){
        return new B();
    }
}
```
