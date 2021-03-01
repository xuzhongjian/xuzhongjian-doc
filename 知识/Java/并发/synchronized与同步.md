# synchronized与同步 #

## synchronized在方法上 ##

### 静态方法 ###

所有的这个类的实例化的对象中，只能有一个线程能执行这个方法。

```java
package com.xuzhongjian.thread.model;
/**
* @author zjxu97 at 1/3/21 11:20 PM
*/
public class SynchronizedTest {

    public static synchronized void test() {
        // TODO
    }
}
```

### 实例方法 ###

同一个实例，只能有一个线程执行这个方法。

```java
package com.xuzhongjian.thread.model;

/**
 * @author zjxu97 at 1/3/21 11:17 PM
 */
public class SynchronizedTest {
    public synchronized void test() {
        // TODO
    }    
}
```

## “锁”对象 ##

使用如下的方式来进行同步就是锁对象：

```java
    synchronized(o){
        //...
    }
```

### this ###

```java
package com.xuzhongjian.thread.model;

/**
 * @author zjxu97 at 1/4/21 10:24 PM
 */
public class SynchronizedTest {
    synchronized(this){
        //...
    }
}
```

按照上面的方式，“锁”this，表示持有这个对象的锁。

```java
package com.xuzhongjian.thread.model;

/**
 * @author zjxu97 at 1/4/21 10:24 PM
 */
public class SynchronizedTest {
    public void test1(){
        synchronized(this){
            // TODO something
        }
    }
    
    public synchronized void test1(){
        // TODO something
    }
}
```

上面的两个方法等效。

this 能够保证同步的前提是，同一个对象，特殊的，这个对象就是这个方法执行的本体。也就是说，this同步的场景是，同一个对象在多个线程里面执行同一个方法，这个时候可以使用synchronized(this)来同步。

### object ###

和 this 类似的，object只能同时被一个线程持有，不能同时被多个线程持有。持有这个额对象的类才能执行方法。

## 原理 ##

Java的每一个对象都有一个监视器（monitor），通过对Java的「.class」文件信息分析可以看到，使用synchronized对代码进行同步就是执行 monitorenter 和monitorexit 指令。所以 synchronized 锁定的其实只是对象的监视器。只有获取到指定的对象的监视器的线程才能执行。

在使用 synchronized 对代码进行同步的时候，需要注意的是，目前加的这个 synchronized 关键字，所需要的是哪个对象的监视器，这个对象是不是只有一个。

比如下面就是一个错误的使用：

```java
public class Test implements Runnable{
    private String name;
//    private static MethodSync methodSync = new MethodSync();
    private MethodSync methodSync = new MethodSync();

    public Test(String name){
        this.name = name;
    }

    @Override
    public void run() {
        methodSync.method(name);
    }

    public static void main(String[] args) {
        //
        // 错误就在这个地方新建了，两个 Test 对象。
        // 而 synchronized 需要争夺的对象是 MethodSync 的对象
        // 这个对象在 Test 中是成员变量，所以两个 Test 对象会有两个 MethodSync 对象
        // 所以结果就是每个线程一个，所以不需要竞争锁，起不到同步代码的效果
        // 修改方式是 MethodSync 修饰成 static
        // 
        Thread t1 = new Thread(new Test("test 1"));
        Thread t2 = new Thread(new Test("test 2"));
        t1.start();
        t2.start();
    }
}
```

```java
public class MethodSync {

    /*
     * @Task : 测试 synchronized 修饰方法时锁定的是调用该方法的对象
     * @param name  线程的标记名称
     */
    public synchronized void method(String name){
        System.out.println(name + " Start a sync method");
        try{
            Thread.sleep(300);
        }catch(InterruptedException e){}
        System.out.println(name + " End the sync method");
    }
}
```
