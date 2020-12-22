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
