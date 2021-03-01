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
