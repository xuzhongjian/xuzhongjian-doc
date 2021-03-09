# 面向切面编程 #

熟悉Spring的人，对面向切面编程一定不陌生。面向切面编程的缩写是AOP。在web服务中，会存在大量的类似的行为需要在不同的地方执行。比如对用户登陆状态的检查、检查用户是不是在黑名单中、对请求的参数或响应的结果进行记录等。这些业务如果单独的写，需要很多的重复的代码。但是我们能够使用AOP思想，将各个业务抽象成一个切面。自动的处理上述的逻辑。

## AOP术语 ##

AOP的术语并不直观，但是依然需要掌握：

### advice ###

切面执行的操作，叫做advice（通知）。比如我们需要对一些方法的请求参数进行记录，那么在一个切面中对参数进行记录的动作就是一个通知。按照通知执行的时机，将通知分为5种类型：

1. 前置通知（Before）：在目标方法执行之前执行的通知。
2. 后置通知（After）：在目标方法执行之后执行的通知，并不关心方法执行的结果。
3. 返回通知（After-returning）：在目标方法成功执行之后执行的通知。
4. 异常通知（After-throwing）：在方法因为执行过程中抛出异常而执行的通知。
5. 环绕通知（Around）：通知包括了被通知的方法，在方法的前后等位置自定义行为。

### join point ###

join point是连接点。连接点是程序运行的过程中，切面可以插入的一个时机点。这个点可以是方法调用时、抛出异常时，切面代码利用这些点插入正常的流程之中，并添加新的行为。一切可以被选作增加动作的时机点，都是连接点。

### pointcut ###

pointcut：切点。切点是从连接点的集合中选择的一个或多个连接点，在这些连接点上添加advice，对需要执行的方法进行增强。

### aspect ###

aspect，切面。切面是pointcut和advice的集合。

### introduction ###

introduction，引入：引入允许我们想现有的添加新的方法或者是属性。

### weaving ###

weaving，织入：织入是将切面引入到通常的对象中并且创建新的代理对象的过程。切面在制定的连接点被引入到指定的目标对象中，引入的时机有：

1. 编译期：切面在目标类的编译期就织入到目标对象中，这种织入方式需要特殊的编译器。AspectJ就是使用的这种方法。
2. 类加载期：切面在目标类加载到JVM的时候被织入，这种织入方式需要特殊的类加载器。
3. 运行期：切面在应用程序运行的时候被织入。在织入切面时，会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面的。

## SpringAOP ##

Spring的通知（增强）都是由Java编写的。

Spring在运行时将通知（增强）织入到Spring管理的bean中。

Spring只支持方法级别的连接点。

## AspectJStyle ##

Spring借助AspectJ的切点表达式来定义Spring的AOP。使用AspectJ风格的标记方式，并不代表使用的就是AspectJ的AOP方法，只是表示在语言上类似于AspectJ，其本质还是Spring实现的面向切面。

### 对连接点的选择 ###

+ execution：用于匹配方法的连接点
+ within：指定的类中的所有方法及其子类中从该类中继承而得到的方法
+ this：指定的类中的所有方法，而不包括父类中未被重写的方法
+ target：指定类、类的子类或接口的实现中全部的方法（包括继承得来的方法）
+ args：指定的传入参数的方法
+ @within：用指定的注解标注的类中的自有方法（不包括继承得来的方法）以及被子类继承的这些方法
+ @target：指定注解标注的类中的全部方法（包括继承得来的方法）
+ @args：将带有指定注解的对象作为参数的方法
+ @annotation：带有指定注解的方法
+ bean：Spring AOP扩展的，AspectJ没有对于指示符，用于匹配特定名称的Bean对象的执行方法

note：

1. @within：当前类有注解，则对该类对父类重载的方法和自有方法产生效果，其他的方法均无效果。
   若子类无注解，且子类调用的是该类的方法（即子类未对该方法进行重载），则有效果。
2. @target：若当前类有注解，则对该类继承、自有、重载的方法有效。若子类无注解，则无效果。

e.g.

+ this & target 比较

  + ```java
    @Pointcut(value = "this(com.xx.Father)")
    @Pointcut(value = "target(com.xx.Father)")
    ```

  + father类继承了grandfather中的一个say()方法，son类只是继承father，未重写say()方法。当调用father对象的say方法时，this和target都会匹配到，调用son对象的say方法时，因为未重写，this不匹配，而target会匹配到。

  + father类继承了grandfather中的一个say()方法，son类重写了say()方法。当调用father对象的say方法时，this和target都会匹配到。调用son对象的say方法时，因为重写了，所以this和target也都会匹配到。

### execution参数 ###

```java
@Pointcut(value = "execution(注解？修饰符？返回类型 方法所在类？方法名（参数）异常？)")
```

+ 注解（可选）：带有指定的注解的，比如：@override
+ 修饰符（可选）：指定访问修饰符的，比如：public、protected等
+ 返回类型：比如：java.lang.String，字符串类型；* 所有类型
+ 方法所在类（可选）：方法所在的类的全限定路径，比如：com.xuzhongjian.service.UserService
+ 方法名：比如：*Check（以Check结尾的方法）
+ 参数：比如：()代表是没有参数；(..)代表是匹配任意数量、任意类型的参数；(java.lang.String)表示一个String类型的参数；(java.lang.String..)表示任意数量的String类型参数；(*,java.lang.String)第一个是任意类型，第二个是String的参数的方法
+ 异常（可选）：语法："throws 任意异常类型"，可以是多个，用逗号分隔，例如：throws java.lang.IllegalArgumentException, java.lang.ArrayIndexOutOfBoundsException

### 注解：获取参数 ###

使用注解对连接点的选择的时候，可以附带额外的参数，对于不同的参数进行不同的advice操作。

```java
package com.xuzhongjian.aop;

import com.xuzhongjian.aop.annotation.AnnotationAnnotation;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

/**
 * @author zjxu97 at 2020-06-05 10:06
 */
@Slf4j
@Aspect
@Component
public class AnnotationAspect {
    @Around("@annotation(com.xuzhongjian.aop.annotation.AnnotationAnnotation)")
    public Object aroundPjp(ProceedingJoinPoint pjp) throws Throwable {
        MethodSignature signature = (MethodSignature) pjp.getSignature();// #1
        Method method = signature.getMethod();// #2
        AnnotationAnnotation thisAnnotation 
            = method.getAnnotation(AnnotationAnnotation.class);// #3
        int value = thisAnnotation.value();// #4
        String param1 = thisAnnotation.param1();
        String param2 = thisAnnotation.param2();
        String param3 = thisAnnotation.param3();
        log.info(value + " " + param1 + " " + param2 + " " + param3);// #5
        return pjp.proceed();
    }
}
```

1. 从pjp中获取带有指定注解的方法的签名
2. 从签名中获取对应的方法
3. 从方法中获取需要的格式的注解的对象
4. 从对象中获取对应的字段的值
5. 得到每个字段对应的值之后，进行不同的操作

***注解的参数只能从@Around方式传入的ProceedingJoinPoint对象中才能获取到***



# bean #

## 测试方式 ##

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

## Component ##

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

## Named ##

@Named是Java提供的依赖注入的规范中的一个注解。可以使用@Named注解指定bean的名字，Spring支持@Named注解作为@Component注解的替代方案。

```java
@Named("hometownCity")
public class City {
}
```

## ComponentScan ##

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

## Configuration ##

使用@Configuration来表示这个类是一个配置类，配置类中包含Spring应用context中创建bean的细节。@Configuration注解可以用Java代码的形式实现Spring中xml配置文件中配置的效果。

```java
@Configuration
public class JavaConfig { 
}
```

## Bean ##

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

## Autowired ##

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

## Inject ##

Inject 和 Named 都是Java的依赖注入的规范，Spring支持使用这两个注解来对bean进行注入。大多数场景下，@Inject和@Autowired作用相同，可以相互替代使用。

## Primary ##

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

## Qualifier ##

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

## Conditional ##

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

## Profile ##

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

## Scope ##

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

# Spring Security #

基于Spring的DI和AOP的特性，Spring Security能提供对Spring服务的安全保护。

## 配置 ##

### 开启配置 ###

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

/**
 * @author zjxu97 at 2020-06-06 14:38
 */
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
}
```

#### @EnableWebSecurity ####

在一个由 @Configuration 标注的配置类上，使用 @EnableWebSecurity 注解，表示开启 spring-security。

#### WebSecurityConfigurerAdapter ####

Security 相关的配置必须配置在一个实现 了 WebSecurityConfigurer 的 bean 中。虽然在Spring应用上下文中任何实现了WebSecurityConfigurer 的bean都可以用来配置Security。但最简单的方法就是扩展 WebSecurityConfigurerAdapter，并将其设置成 @Configuration 配置类。

### 内存的用户验证 ###

```java
package com.xuzhongjian.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.PasswordEncoder;

/**
 * @author zjxu97 at 2020-06-06 14:38
 */
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private PasswordEncoder encoder;

    @Autowired
    public void setEncoder(PasswordEncoder encoder) {
        this.encoder = encoder;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication().passwordEncoder(encoder)
                .withUser("admin").password("admin").roles("USER", "ADMIN")
                .and().withUser("xuzhongjian").password("19970701").roles("USER");
    }
}
```

#### 重写configure ####

SecurityConfig 继承了 WebSecurityConfigurerAdapter 类，开启用户验证的方法就是重写其中的configure方法。

#### inMemoryAuthentication ####

调用传入的 auth 对象的 inMemoryAuthentication() 方法，就能将用户相关的信息存储到内存中。但是开启了这个配置之后，并没有存储相关的用户和对应的密码。可以使用构造器风格的代码方式，定义内存中存储的用户的用户名和密码。这个构造器方式的方法有以下这些：

| 方法名                                        | 解释                   |
| --------------------------------------------- | ---------------------- |
| withUser(String)                              | 创建用户并设置用户名   |
| password(String)                              | 设置用户的密码         |
| roles(String...)                              | 授予用户一个或多个角色 |
| authorities(GrantedAuthority...)              | 授予用户一个或多个权限 |
| authorities(List<? extends GrantedAuthority>) | 授予用户一个或多个权限 |
| authorities(String...)                        | 授予用户一个或多个权限 |
| accountExpired(boolean)                       | 定义账号是否过期       |
| accountLocked(boolean)                        | 定义账号是否被锁定     |
| credentialsExpired(boolean)                   | 定义凭证是否过期       |
| disabled(boolean)                             | 定义账号是否被禁用     |
| and()                                         | 连接两个用户配置       |

其中roles(String...)方法是authorities(String...)方法的简写形式。roles(String...)方法所给定的值都会添加一个“ROLE_”前缀，并将其作为权限授予给用户。

在我的代码实例中，需要通过 passwordEncoder(PasswordEncoder) 方法指定一个 PasswordEncoder 的实例，作为从内存中校验密码的过程中的一个加密过程。

#### PasswordEncoder ####

PasswordEncoder 是一个编码规则的类，作用是进行编码和匹配：

```java
package org.springframework.security.crypto.password;

public interface PasswordEncoder {

    /**
     * 对原始密码进行编码。通常，一个良好的编码算法使用 SHA-1 算法或优秀的八位hash算法或优秀的椒盐噪声算法。
     */
    String encode(CharSequence rawPassword);

    /**
     * 从存储中获取的编码过的密码，对提交的原始密码进行编码，将这两个密码进行匹配。
     * 如果密码匹配，则返回true；否则，返回false。
     * 这个过程中，存储的密码本身永远不会被解码。
     * 
     * @param rawPassword 输入的原始密码
     * @param encodedPassword 加密之后的真实密码
     * @return 将rawPassword加密之后，和encodedPassword进行匹配，返回T/F
     */
    boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

### 数据库表的用户认证 ###

```java
package com.xuzhongjian.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.password.PasswordEncoder;

import javax.sql.DataSource;

/**
 * @author zjxu97 at 2020-06-06 14:38
 */
@Slf4j
@Configuration
@EnableWebSecurity
public class DatabaseSecurityConfig extends WebSecurityConfigurerAdapter {
    
    // 查询用户账号密码
    private static final String QUERY_USER = "select user_name,password,enabled from user where user_name = ?";
    // 查询用户权限
    private static final String QUERY_AUTH = "select user_name,authority from authorities where user_name = ?";

    private PasswordEncoder encoder;

    private DataSource dataSource;

    @Autowired
    public void setEncoder(PasswordEncoder encoder, DataSource dataSource) {
        this.encoder = encoder;
        this.dataSource = dataSource;
    }

    /**
     * 数据库的用户表在sql/user.sql中，数据库的配置的位置在application.yaml中，都在resources目录下
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        auth.jdbcAuthentication().dataSource(dataSource)
                .usersByUsernameQuery(QUERY_USER)
                .authoritiesByUsernameQuery(QUERY_AUTH)
                .passwordEncoder(encoder);
    }
}

```

#### jdbcAuthentication ####

用户数据通常会存储在关系型数据库中，并通过JDBC进行访问。为了配置Spring Security使用以JDBC为支撑的用户存储，我们可以使用jdbcAuthentication()方法，来配置关系型数据库中的用户表和权限表。

#### dataSource ####

数据源配置，在本例子中，使用set方式的 autowired 方法注入到 dataSource 字段中。而后将这个 dataSource 配置到auth 的 datasource()方法中。

关于这个datasource，可以通过以下方式生成一个bean：

```java
package com.xuzhongjian.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;

/**
 * @author zjxu97 at 2020-06-06 18:47
 */
@Configuration
public class BeanConfig {

    @Bean
    @Primary
    public DataSource dataSource() {
        DriverManagerDataSource driverManagerDataSource = new DriverManagerDataSource();
        driverManagerDataSource.setDriverClassName("com.mysql.jdbc.Driver");
        driverManagerDataSource.setUrl("jdbc:mysql://39.97.xxx.xxx:3306/users");
        driverManagerDataSource.setUsername("root");
        driverManagerDataSource.setPassword("19970701Xzj.");
        return driverManagerDataSource;
    }
}
```

但是更加简单的方法是：引入的 spring-boot-starter-jdbc 和 mysql-connector-java 的依赖，然后在application.yaml文件中配置 spring.datasource 相关的值，那么Springboot会自动自动的配置 datasource 的 bean，然后在 set 方法上面的@Autowired 可以直接引入这个bean。

```yaml
# DataSource Config
spring:
  datasource:
    url: jdbc:mysql://39.97.xxx.xxx:3306/users?serverTimezone=UTC&characterEncoding=UTF-8&useSSL=false
    username: root
    password: 19970701Xzj.
```

#### usersByUsernameQuery ####

设置用来查询刚才设定好的数据库中的用户的用户名和id的sql语句：

```java
    /**
     * 设置用来查询用户的用户名和密码的sql语句，比如：
     * 
     * <code>
     *     select username,password,enabled from users where username = ?
     * </code>
     * @param 用于选择用户名、密码以及'是否通过用户名启用用户'的查询。用户名必须包含一个参数。
     * @return 一个用于查询用户相关的信息的配置类
     * @throws Exception 抛出异常
     */
    public JdbcUserDetailsManagerConfigurer<B> usersByUsernameQuery(String query)
        throws Exception {
        getUserDetailsService().setUsersByUsernameQuery(query);
        return this;
    }
```

从示例sql

```sql
select username,password,enabled from users where username = ?
```

中，可以发现，查询的语句应该包含三个字段：username、password和enabled。那么我们自己定义的用户表和sql语句也应该包含上面的三个字段。

#### authoritiesByUsernameQuery ####

同样的，使用authoritiesByUsernameQuery方法来设置用户对应的权限：

```java
    /**
     * 设置用于通过用户名查找用户权限的sql语句。比如:
     *
     * <code>
     *     select username,authority from authorities where username = ?
     * </code>
     *
     * @param query 用于选择用户名（按用户名授权）的查询。
     * @return 一个用于查询用户权限相关的信息的配置类
     * @throws Exception 抛出异常
     */
    public JdbcUserDetailsManagerConfigurer<B> authoritiesByUsernameQuery(String query)
    throws Exception {
                getUserDetailsService().setAuthoritiesByUsernameQuery(query);
        return this;
    }
```

从

```sql
    select username,authority from authorities where username = ?
```

可以发现需要返回的字段有username和authority，同上，我们自己定义的权限表和sql语句也应该包含username和authority两个字段。

## 拦截 ##

前面设置了内存和数据库两种方式的请求拦截，在上面默认的情况下，会对所有的请求进行拦截。在进行实际的项目中，并不需要对所有的接口进行一视同仁的拦截。

默认的拦截配置：

```java
    protected void configure(HttpSecurity http) throws Exception {
        logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

        http
            .authorizeRequests()
                .anyRequest().authenticated() // 所有请求都需要进行登陆授权
                .and()
                .formLogin().and()// 表单登陆
            .httpBasic();
    }
```

对每个请求进行细粒度安全性控制的关键在于重载 configure(HttpSecurity) 方法，而这个 configure 方法还有很多相关的方法：

| 方法名                     | 解释                                              |
| -------------------------- | ------------------------------------------------- |
| authorizeRequests()        | 进行对接口访问的控制                              |
| antMatchers(String...)     | 设定的路径支持ant风格的通配符                     |
| access(String)             | 如果给定的SpEL表达式计算结果为true，就允许访问    |
| anonymous()                | 允许匿名用户访问                                  |
| authenticated()            | 允许认证过的用户访问                              |
| denyAll()                  | 无条件拒绝所有访问                                |
| fullyAuthenticated()       | 如果用户是完整认证的话(非remember-me)，就允许访问 |
| hasAnyAuthority(String...) | 如果用户具备给定权限中的某一个的话，就允许访问    |
| hasAnyRole(String...)      | 如果用户具备给定角色中的某一个的话，就允许访问    |
| hasAuthority(String)       | 如果用户具备给定权限的话，就允许访问              |
| hasIpAddress(String)       | 如果请求来自给定IP地址的话，就允许访问            |
| hasRole(String)            | 如果用户具备给定角色的话，就允许访问              |
| not()                      | 对其他访问方法的结果求反                          |
| permitAll()                | 无条件允许访问                                    |
| rememberMe()               | 如果用户是通过Remember-me功能认证的，就允许访问   |

** role 前加上 ROLE_ 就是权限*

### antMatchers ###

1. 接口一定要以`/`开头
2. 可以使用`/doc.html`来控制对swagger文档的访问
3. `/security/**` ：匹配`/security`下的所有http请求
4. `/security/*` ：匹配`/security`下的一级http请求，匹配不到`/security/test`这个请求
5. 根据 3 & 4 ，`/*/test`既能匹配到`/security/test`，也能匹配到`/security-else/test`

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/security/**","/*/test").authenticated()
                .and().formLogin()
                .and().httpBasic();
    }
```

除了 String 类型的参数以外，可以在第一位添加一个HTTP请求的类型，用来指定匹配的HTTP请求方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers(HttpMethod.GET, "/security/**", "/*/test").authenticated()
                .and().formLogin()
                .and().httpBasic();
    }
```

### access ###

access方法，将SpEL表达式扩展开，使用SpEL的风格来描述安全相关的规则，如下：

| Expression                                                   | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `hasRole(String role)`                                       | 如果用户拥有指定的角色那么返回 `true` 。比如，`hasRole('admin')`。默认的，如果指定的角色不是以“ROLE_”，那么就会默认的添加上。这个设置可以通过修改`DefaultWebSecurityExpressionHandler`中的`defaultRolePrefix`。 |
| `hasAnyRole(String… roles)`                                  | 多个参数的`hasRole(String role)`。                           |
| `hasAuthority(String authority)`                             | 如果当前的请求具有指定的权限，返回 `true` 。比如，`hasAuthority('read')`。 |
| `hasAnyAuthority(String… authorities)`                       | 多个参数的`hasAuthority(String authority)`。                 |
| `principal`                                                  | 用户的principal对象                                          |
| `authentication`                                             | 用户的认证对象                                               |
| `permitAll`                                                  | 允许所有。                                                   |
| `denyAll`                                                    | 拒绝所有。                                                   |
| `isAnonymous()`                                              | 匿名的用户的请求返回 `true` 。                               |
| `isRememberMe()`                                             | `RememberMe`的用户的请求返回 `true` 。                       |
| `isAuthenticated()`                                          | 认证过的用户请求返回 `true` 。                               |
| `isFullyAuthenticated()`                                     | 既不是`RememberMe`，也不是匿名的用户的请求返回 `true` 。     |
| `hasPermission(Object target, Object permission)`            | 如果用户可以访问给定权限的给定目标，则返回`true` 。比如：`hasPermission(1, 'com.example.domain.Message', 'read')` 。 |
| `hasPermission(Object targetId, String targetType, Object permission)` | 如果用户可以访问给定权限的给定目标，则返回`true` 。比如：`hasPermission(1, 'com.example.domain.Message', 'read')` 。 |

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers(HttpMethod.GET, "/*/test").access("hasAnyRole('USER') || hasIpAddress('127.0.0.1')")
                .and().formLogin()
                .and().httpBasic();
    }
```

## 安全 ##

### HTTPS ###

使用HTTP发送一些稍微敏感的信息，是一个具有一定风险的事情。通过HTTP发送的数据如果没有经过加密，存在着敏感信息被拦截的风险。在对敏感的数据传输的过程中，可以通过spring-security将其设置成http请求的方式：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers(HttpMethod.GET, "/*/test").hasRole("admin")
                .antMatchers(HttpMethod.POST, "/security/access").authenticated()
                .anyRequest().permitAll()
                .and()
                .requiresChannel()
                .antMatchers(HttpMethod.GET, "/*/test").requiresSecure();// 强制使用HTTPS请求
    }
```

requiresChannel() + requiresSecure()方法会为选定的URL强制使用HTTPS。在任何时候，符合条件的SpEL的请求都会被强制使用HTTPS的方式请求。

同样的，也会有一些场景不需要使用到HTTPS：

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http.requiresChannel()
                .antMatchers(HttpMethod.GET, "/").requiresInsecure();
    }
```

使用 requiresChannel() + requiresInsecure() 可以指定的接口的请求重定向到非HTTPS的请求，也就是普通的HTTP请求。

### CSRF ###

跨站请求攻击：简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：**简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的**。*[ 来自百度百科 ]*

Spring-Security实现CSRF跨域的方法是在发送请求的时候，在表单内置一个csrf.token字段，这个字段从表单中提交过来，和Spring中内置的 csrf.token 字段相比较。如果是一致的，那么才能进行下面的验证以及一系列操作。

从Spring-Security 3.2 开始，默认开启CSRF跨域功能。通过下面的设置可以关闭默认开启的CSRF验证：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
    }
```

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