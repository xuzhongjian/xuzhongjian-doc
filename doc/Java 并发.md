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

# 双重校验的单例模式 #

## 懒汉 - 线程不安全单例模式 ##

```java
package com.xuzhongjian.model;

import java.util.Objects;

/**
 * 懒汉 - 线程不安全 单例模式
 *
 * @author zjxu97 at 12/31/20 2:53 PM
 */
public class Instance {

    private static Instance instance;

    private Instance() {

    }

    public static Instance getInstance() {
        if (Objects.isNull(instance)) {
            instance = new Instance();
        }
        return instance;
    }
}
```

## 懒汉 - 线程安全单例模式 ##

上面是一个懒汉式线程不安全的单例模式，其中的静态方法 getInstance 可能会被多个方法同时调用。然后多个线程对 instance 这个静态变量的处理可能是一致的，也就是都认为其是一个空对象，没有被初始化。使用下面的方法，可以将其编程线程安全的单例模式：

```java
package com.xuzhongjian.model;

import java.util.Objects;

/**
 * 懒汉 - 线程安全 单例模式
 *
 * @author zjxu97 at 12/31/20 2:53 PM
 */
public class Instance {

    private static Instance instance;

    private Instance() {

    }

    public synchronized static Instance getInstance() {
        if (Objects.isNull(instance)) {
            instance = new Instance();
        }
        return instance;
    }
}
```

改动的地方只有一处，那就是在静态方法上加上了一个 synchronized 。加上这个变量的作用是，该方法只能同时允许一个地方调用，不允许多个位置调用。

但是在早期的JVM中，synchronized存在巨大的性能问题，所以提出了双重校验的办法：

## 双重校验 单例模式 ##

```java
package com.xuzhongjian.model;

import java.util.Objects;

/**
 * 双重校验 单例模式
 * 存在初始化风险
 *
 * @author zjxu97 at 12/31/20 2:53 PM
 */
public class Instance {

    private static Instance instance;

    private Instance() {

    }

    public static Instance getInstance() {
        if (Objects.isNull(instance)) {
            synchronized (Instance.class) {
                if (Objects.isNull(instance)) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}

```

这样的双重校验的方法可能存在的问题是，当上一个线程还没有将实例完全初始化，然后后一个线程读取到的对象还没有完全初始化。

如果使用 volatile，将instance实例变成 volatile 字段，可以解决这个问题：

```java
/**
 * 双重校验 单例模式
 * 终极版双重校验
 *
 * @author zjxu97 at 12/31/20 2:53 PM
 */
public class Instance {

    private volatile static Instance instance;

    private Instance() {

    }

    public static Instance getInstance() {
        if (Objects.isNull(instance)) {
            synchronized (Instance.class) {
                if (Objects.isNull(instance)) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}

```

这样就可以实现线程安全的返回延迟的实例化。

## 静态内部类 ##

使用静态内部类的方式实现单例模式：

```java
package com.xuzhongjian.model;

/**
 * 静态内部类实现单例模式
 *
 * @author zjxu97 at 12/31/20 2:53 PM
 */
public class Instance {

    private static class InnerInstance {
        private static Instance instance = new Instance();
    }

    private Instance() {

    }

    public static Instance getInstance() {
        return InnerInstance.instance;
    }
}
```

利用了类加载的时候 JVM 的Class初始化锁：

![类加载锁](../imgs/class_instance.jpg)

A线程首先拿到锁之后，对 instance 的 Class 对象进行初始化，待初始化完成之后，释放锁。再由 B 获取到锁之后，就不存在并发问题了，就能直接取到InnerInstance 的其中的静态变量。

# 基础知识点 #

## 先行发生原则 ##

happens-before

1. 单线程原则
   - 单线程中的操作，一个操作happens-before其之后的操作。
2. 加锁解锁原则
   - 一个锁的加锁操作happens-before一个锁的解锁及以后的操作。
3. volatile原则
   - volatile的字段的写操作，happens-before这个字段的读操作。
4. 传递性
   - happens-before具有传递性。
5. start
   - A线程启动B线程，A的启动操作happens-before B线程中的所有操作。
6. join
   - A线程调用B线程的join方法，B线程中剩余的操作happens-before A线程的join方法的返回。

## Java内存模型抽象 ##

![jmm内存模型](../imgs/jmm.png)

1. 整个Java进程有一个主内存，这个内存区域中存放所有的共享变量。

2. 除了主内存之外，每一个线程都会对应一个本地内存的区域。

3. 当线程需要取用变量的时候，从主内存中取得变量的副本，放置在本地内存中。

## 内存屏障 ##

为了在JMM体系中从底层保持内存可见性，Java代码在编译成底层指令的时候会添加内存屏障。

![内存屏障](../imgs/barriers.jpg)

# 多线程 #

## 线程和进程 ##

### 线程 ###

### 进程 ###

### 使用多线程的原因 ###

1. 更多的处理器核心数量，更加适合多线程处理。使用单线程是对资源的浪费。

2. 数据一致性不强的操作，可以使用多线程来处理，缩响应时间。

## 线程的创建 ##

1. 新建一个类，继承Thread类，重写其中的run方法。然后实例化这个类，直接使用这个类的start方法，即可启动这个线程。

    ```java
    package com.xuzhongjian.thread.model;

    /**
    * @author zjxu97 at 1/2/21 11:04 AM
    */
    public class TestThread extends Thread {
        @Override
        public void run() {
            super.run();
        }
    }
    ```

    ```java
    package com.xuzhongjian.thread;

    import com.xuzhongjian.thread.model.TestThread;

    /**
    * @author zjxu97 at 1/3/21 1:11 AM
    */
    public class Main {
        public static void main(String[] args) {
            TestThread thread = new TestThread();
            System.out.println("this is main thread");
            thread.start();
        }
    }
    ```

2. 实现runnable接口，实现其中的run方法，将这个runnable接口的实现作为Thread类的构造方法的参数，对Thread类进行实例化，调用Thread对象的start方法。

    ```java
    package com.xuzhongjian.thread;

    /**
    * @author zjxu97 at 1/2/21 11:00 AM
    */
    public class Main {
        public static void main(String[] args) {
            Thread thread = new Thread(() -> {
                System.out.println("this is a new thread");
            });
            System.out.println("this is main thread");
            thread.start();
        }
    }
    ```

3. 实现callable接口，将实现的callable接口的实现作为FeatureTask类的构造方法，构造一个FeatureTask类的实现，然后将这个实现作为Thread类的构造方法的，然后启动这个线程。

```java
package com.xuzhongjian.thread.model;

import java.util.concurrent.FutureTask;

/**
 * @author zjxu97 at 1/3/21 1:13 AM
 */
public class FutureTaskMain {
    public static void main(String[] args) {
        try {
            FutureTask<String> futureTask = new FutureTask<>(() -> {
                Thread.sleep(100);
                return "futureTask done";
            });

            Thread thread = new Thread(futureTask);
            thread.start();
            while (!futureTask.isDone()) {
                Thread.sleep(10);
                System.out.println("main waiting");
            }
            System.out.println(futureTask.get());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

```java
// sout:
main waiting
main waiting
main waiting
main waiting
main waiting
main waiting
main waiting
main waiting
main waiting
futureTask done
```

## 线程的状态 ##

![Java线程状态表](../imgs/thread_status.jpg)

![线程转化图](../imgs/thread_tans.jpg)

1. 初始化

2. 就绪

3. 运行中
   - 在Java线程中，其中就绪和运行中统一被称为可运行状态。

4. 等待

5. 计时等待

6. 阻塞

7. 终止

## 中断 ##

关于中断的三个方法

1. interrupt()
    - Thread 类的实例方法：将某个线程标记成中断中断状态。
2. interrupted()
    - Thread 类的静态方法：重置调用该方法的线程的中断状态。
3. isInterrupted()
    - Thread 类的实例方法：判断线程有没有中断状态。

“许多声明抛出InterruptedException的方法（例如Thread.sleep(long millis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。”

Excerpt From: 方腾飞,魏鹏,程晓明 著. “Java并发编程的艺术 (Java核心技术系列).” Apple Books.

## 线程间通信 ##

如果每个线程都单独的运行，线程和线程之间没有相互配合和沟通，这样的多线程是价值是很低的。

# 等待通知 #

等待 / 通知方法是所有的Java对象都具备的，在 Object 类中定义的基本方法可以实现这个功能的：

![object通知](../imgs/object_notify.jpg)

例子：

```java
package com.xuzhongjian.thread;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

/**
 * @author zjxu97 at 1/5/21 12:17 AM
 */
public class WaitNotify {
    private static final Object lock = new Object();
    private static Boolean flag = true;

    public static void main(String[] args) throws Exception {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
    }

    static class Wait implements Runnable {
        public void run() {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 当条件不满足时，继续wait，同时释放了lock的锁
                while (flag) {
                    try {
                        System.out.println(Thread.currentThread() + " flag is true. wait @ " + new SimpleDateFormat(" HH:mm:ss ").format(new Date()));
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 条件满足时，完成工作
                System.out.println(Thread.currentThread() + " flag is false. running @ " + new SimpleDateFormat(" HH: mm: ss ").format(new Date()));
            }
        }
    }

    static class Notify implements Runnable {
        public void run() {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 获取lock的锁，然后进行通知，通知时不会释放lock的锁，
                // 直到当前线程释放了lock后，WaitThread才能从wait方法中返回
                System.out.println(Thread.currentThread() + " hold lock. notify @ " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 再次加锁
            synchronized (lock) {
                System.out.println(Thread.currentThread() + " hold lock again. sleep @ " + new SimpleDateFormat(" HH: mm: ss ").format(new Date()));
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 三个线程轮番打印 ##

```java
package com.xuzhongjian.thread;

/**
 * @author zjxu97 at 1/5/21 8:48 PM
 */
public class ThreeThread {

    public static void main(String[] args) {
        Object a = new Object();
        Object b = new Object();
        Object c = new Object();

        new DemoThread(a, b, "a").start();
        new DemoThread(b, c, "b").start();
        new DemoThread(c, a, "c").start();
    }

    static class DemoThread extends Thread {

        private final Object cur;
        private final Object next;

        DemoThread(Object cur, Object next, String name) {
            this.cur = cur;
            this.next = next;
            this.setName(name);
        }


        @Override
        public void run() {
            while (true) {
                try {
                    synchronized (next) {
                        synchronized (cur) {
                            System.out.println(Thread.currentThread().getName());
                            cur.notify();
                        }
                        Thread.sleep(10);
                        next.wait();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

        }
    }
}
```

## note ##

![notify转化图](../imgs/notify_status.jpg)

1. 使用wait、notify、notifyAll方法需要在这个对象的synchronized中，就是要先持有这个对象的监视器。

2. 调用wait方法之后，线程的状态由 RUNNING 变为 WAITING，会释放锁，但是wait方法会被阻塞。

3. wait调用后，等待这个对象的notify方法被调用，并且调用notify的方法从synchronized代码块退出之后，wait方法才会执行结束再返回。

4. notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED。

## 经典范式 ##

### 等待方 ###

```java
    synchronized(o){
        // 条件不满足
        while(!condition){
            // 被通知之后还是需要检查状态
            o.wait();
        }
        // TODO something
    }
```

### 通知方 ###

```java
    synchronized(o){
        // 改变 condition
        o.notify();
    }
```

# 线程间通信 #

## volatile ##

关键字volatile可以用来修饰字段（成员变量），就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。

volatile 的两个作用：

1. 保证内存可见性
   - 保证变量的修改，每次的写入都是直接写入到主内存中；每次对变量的读取都是从主内存中读取；
     ![jmm内存模型](/Users/zjxu97/Github/xuzhongjian-doc/old/knowledge/Java-base/imgs/jmm.png)
2. 防止指令重排
   - 在volatile操作前后添加上内存屏障，禁止指令重排。

### volatile的内存屏障 ###

1. 在每个volatile写操作的前面插入一个StoreStore屏障。
2. 在每个volatile写操作的后面插入一个StoreLoad屏障。
3. 在每个volatile读操作的后面插入一个LoadLoad屏障。
4. 在每个volatile读操作的后面插入一个LoadStore屏障。

## synchronized ##

使用synchronized：

1. 修饰实例方法

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

2. 修饰静态方法

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

3. 修饰对象

   1. 修饰this：修饰this，同一个实例，只能有一个线程执行这个方法。
   2. 修饰一个对象：这个用法和后面的notify，wait联合使用，可以实现基本的线程间通信。

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块或者同步方法，而没有获取到监视器（执行该方法）的线程将会被阻塞在同步块和同步方法的入口处，进入BLOCKED状态。**表面上看，synchronized是加锁，但是实际上是对某个对象的监视器的竞争。**

## 几个方法 ##



# 线程池源码分析 #

线程池的核心代码在 ThreadPoolExecutor 类。

## 几个参数 ##

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

corePoolSize：线程池的基本大小，用于执行的运行中的线程的池子。

maximumPoolSize：最大线程数量，线程池允许创建的最大线程数。
    如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果。

keepAliveTime & unit：线程池的工作线程空闲后，保持存活的时间。
    如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

workQueue：等待队列，在核心池线程运行中的线程达到指定的数量的时候，将后面提交的任务存储到这个队列中。

threadFactory：线程工厂，用来设置创建线程的策略。
    使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字。

handler：拒绝策略。
    AbortPolicy：直接抛出异常。
    CallerRunsPolicy：只用调用者所在线程来运行任务。
    DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
    DiscardPolicy：不处理，丢弃掉。

## 提交一个任务 ##

java.util.concurrent.AbstractExecutorService#submit(java.lang.Runnable)

```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
```

java.util.concurrent.AbstractExecutorService#newTaskFor(java.lang.Runnable, T)

```java
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```

java.util.concurrent.ThreadPoolExecutor#execute

```java
    /**
     * 在未来的某个时刻执行给定的任务。
     * 这个任务可能在一个新线程也有可能在一个已经存在的线程里被执行。
     * 
     * 如果这个线程无法被提交、执行。或者是这个线程池已经被关闭，或者容量已经到了。
     * 那么这个任务就会按照当前的拒绝策略来处理。
     *
     * @param command 被执行任务。
     * @throws RejectedExecutionException 拒绝执行异常
     * @throws NullPointerException 如果给出的任务为空
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. 如果核心池的容量还没到（运行中的线程数量），尝试去开启一个新的线程。
         * 将这个任务作为这个新线程的第一个任务。
         *
         * 2. 如果这个任务能被成功的加入队列中，我们仍然需要双重校验一下，我们是否需要
         * 添加一个新的线程，如果之前的线程死了在上次检查之后，或者这个线程池关闭了。
         * 所以我们二次校验一下。我们新建一个线程或者回退这次的进入队列。
         *
         * 3. 如果无法加入到队列中，尝试去创建一个新的线程，如果失败了，就拒绝这个任务。
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 运行的线程数量已经饱和了 && 成功的向阻塞队列中添加了这个任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 线程池状态不是RUNNING状态，说明执行过shutdown命令，需要对新加入的任务执行reject()操作
            // 这里需要recheck，是因为任务入队列前后，线程池的状态可能会发生变化
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

![线程池](../imgs/thread_pool.jpg)

java.util.concurrent.ThreadPoolExecutor#addWorker

```java

    /**
     * 根据当前的线程池的状态、核心或最大容量来判断一个新的工作线程能否被添加到
     * 当前的线程池。如果可以的话，相应的调整线程数量的计数值，如果可能，将创建
     * 并启动一个新的worker，并将firstTask作为其第一个任务运行。如果池已停止
     * 或有资格关闭，则此方法返回false。如果线程工厂在被请求时未能创建线程，它
     * 也会返回false。如果出现线程工厂返回null，或者因为异常，会快速回滚。
     *
     * @param 这个线程应该首先运行的任务。worker是用一个初始的第一个任务（在
     * execute方法中）创建的，当线程少于corePoolSize时（在这种情况下，我们总
     * 是启动一个线程），或者当队列已满时（在这种情况下，我们必须绕过队列）。最
     * 初空闲线程通常是通过prestartCoreThread创建的，或者用来替换其他正在消
     * 亡的工作线程。
     *
     * @param core如果为true，则使用corePoolSize，否则使用maximumPoolSize。
     * （这里使用的是布尔指示符，而不是值，以确保在检查其他池状态后读取新值）。
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        //1. 自增workerCounter
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 异常情况的判断：线程池状态、任务、队列
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
                return false;

            for (;;) {
                // workercount
                int wc = workerCountOf(c);
                //core true代表是往核心线程池中增加线程 false代表往最大线程池中增加线程
                if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // cas的形式自增线程池的worker的数量
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();
                // 判断线程池的 worker 的数量是否发生的变化
                if (runStateOf(c) != rs)
                    continue retry; // 通常执行这个之后就返回false退出了
            }
        }
        
        //2. 新建线程，并加入到线程池workers中
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                // 对workers的锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 获取runState
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 获取锁之后，添加新的worker到线程池中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    // 解锁
                    mainLock.unlock();
                }
                // 启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

调用t.start()之后，实质上调用的是Worker类的run()方法:
java.util.concurrent.ThreadPoolExecutor.Worker#run

```java
        /** 将主运行循环委托给外部runWorker。  */
        public void run() {
            runWorker(this);
        }
```

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // 允许中断
        boolean completedAbruptly = true;
        try {
            // 此处循环获得任务，执行任务
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 按照条件设置线程的中断状态
                if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() &&  runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted()){
                    wt.interrupt();
                }
                
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    // 完成的任务数量增加、解锁
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

java.util.concurrent.ThreadPoolExecutor#getTask

```java
    private Runnable getTask() {
        boolean timedOut = false; // 上次的 poll 是否超时了

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 仅在必要时检查队列是否为空。
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // workers 会被淘汰吗？
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // wc过大，相对的wq过小，减少ctl的workerCount字段。
            if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c)){
                    return null;
                }
                continue;
            }

            try {
                // 取得任务，如果在 keepAliveTime 时间内还没有取得任务，那么在上面减少运行中的线程的时间。
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null){
                    return r;
                }
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

# 并发面试题

## Synchronized

### 问题一：Synchronized用过吗？其原理是什么？
synchronized是一个互斥同步锁，synchronized是基于每个对象的监视器来实现的（monitor）。基于底层的monitorenter和monitorexit来实现的。在加锁的代码开始处，插入monitorenter语句，如果没有线程获取了这个对象的监视器，那么当前这个线程取得这个线程的监视器。在加锁的代码的结尾处，插入monitorexit语句，表示释放这个对象的锁。

### 问题二：synchronized对象的锁 ，这个“锁”到底是什么？如何确定对象的锁？
锁就是监视器。

### 问题三：什么是可重入性 ， 为什么说Synchronized是可重入锁？
可重入性：当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功，否则阻塞。

Synchronized配合对象的监视器使用一个计数器来实现可重入性。如果计数器的值为零，那么状态为没有线程持有这个对象的锁。

### 问题四：JVM对Java的原生锁做了哪些优化？
1. 偏向锁		大多数情况下，锁都是由同一个线程获得的。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。
2. 轻量级锁	在偏向锁cas替换对象头中的mark word的时候，如果替换失败，那么表示这次锁的争夺失败。线程进入自旋状态。在同步代码块执行的速度快的场景下具有很高的效率。
3. 重量级锁	在轻量级锁尝试自旋获取锁时候失败，就是阻塞同步锁。

### 问题五：为什么说Synchronized是非公平锁？
非公平主要表现在获取锁的行为上，并非是按照申请锁的时间前后给等待线程分配锁的，每当锁被释放后，任何一个线程都有机会竞争到锁，这样做的目的是为了提高执行性能，缺点是可能会产生线程饥饿现象。

### 问题六：什么是锁消除和锁粗化 ？
锁消除和锁粗化都是编译器级别对锁的优化。
锁消除：没有必要加锁的代码，但是确加了锁，在编译器编译的时候，会对加的锁删除，将锁消除。
锁粗化：锁粗化是跟锁消除的一个另一面：当需要加锁，但是锁加的粒度太细了。在编译器编译阶段，会将粒度太细没必要的锁，粗化成一个或多个大的锁。

### 问题七：为什么说Synchronized是一个悲观锁？乐观锁的实现原理又是什么？什么是CAS，它有什么特性？
不管是否会产生竞争，任何的数据操作都必须要先加锁、后操作。这样的策略就是悲观锁。synchronized的枷锁策略就是，在代码前后插入monitorenter、monitorexit。所以很明显，这是一个悲观锁。
乐观锁的核心算法是cas compare and swap。

### 问题八：乐观锁一定就是好的吗？

乐观锁，cas。存在三个问题：
1. ABA问题
2. 自旋开销过大
3. 只能针对单个变量进行cas

## 锁

https://blog.csdn.net/weixin_41566126/article/details/106563405

### 问题一： 跟 Synchronized相比 ，可重入锁ReentrantLock其实现原理有什么不同？
基于 AQS ，底层使用的是cas。Synchronized是基于对象的监视器。
### 问题二：那么请谈谈AQS框架是怎么回事儿？
AQS（AbstractQueuedSynchronizer类）是一个用来构建锁和同步器的框架，各种Lock包中的锁（常用的有ReentrantLock、ReadWriteLock），以及其他如Semaphore、CountDownLatch，甚至是早期的FutureTask等，都是基于AQS来构建：
1. AQS在内部定义了一个volatileintstate变量，表示同步状态：当线程调用lock方法时，如果state=0，说明没有任何线程占有共享资源的锁，可以获得锁并将state=1；如果state=1，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待。
2. AQS通过Node内部类构成的一个双向链表结构的同步队列，来完成线程获取锁的排队工作，当有线程获取锁失败后，就被添加到队列末尾。
	+ Node类是对要访问同步代码的线程的封装，包含了线程本身及其状态叫waitStatus（有五种不同取值，分别表示是否被阻塞，是否等待唤醒，是否已经被取消等），每个Node结点关联其prev结点和next结点，方便线程释放锁后快速唤醒下一个在等待的线程，是一个FIFO的过程。
	+ Node类有两个常量，SHARED和EXCLUSIVE，分别代表共享模式和独占模式。所谓共享模式是一个锁允许多条线程同时操作（信号量Semaphore就是基于AQS的共享模式实现的），独占模式是同一个时间段只能有一个线程对共享资源进行操作，多余的请求线程需要排队等待（如ReentranLock）。
3. AQS通过内部类ConditionObject构建等待队列（可有多个），当Condition调用wait()方法后，线程将会加入等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移动同步队列中进行锁竞争。
4. AQS和Condition各自维护了不同的队列，在使用Lock和Condition的时候，其实就是两个队列的互相移动。

### 问题三：请尽可能详尽地对比下Synchronized 和 ReentrantLock的异同

1. 实现：synchronized是由jvm实现的，可重入锁是由jdk实现的。
2. 公平性：synchronized是非公平锁，ReentrantLock可以选择公平或者不公平。
3. 超时：synchronized没有办法设置超时时间，如果没有办法获得，那么会一直阻塞，re可以设置超时等待时间。
4. 显式释放：synch不需要显式释放锁，但是re需要显式释放锁

### 问题四： ReentrantLock 是如何实现可重入性的？

根据AQS中state状态的值。

### 问题五： 除了ReetrantLock，你还接触过JUC中的哪些并发工具？

略

### 问题六： 请谈谈ReadWriteLock 和 StampedLock。

略

### 问题七： 如何让Java的线程彼此同步？你了解过哪些同步器？请分别介绍下 。

 CyclicBarrier 和 CountDownLatch。

### 问题八： CyclicBarrier 和 CountDownLatch 看起来很相似，请对比下呢？

**CyclicBarrier：**字面意思是“可循环使用的屏障”。

它的作用是：让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活，线程进入屏障通过CyclicBarrier的await()方法。

```java
package com.zjxu9;

import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author zjxu97 at 2/24/21 11:29 PM
 */
public class CyclicBarrierTest {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();

        for (int i = 0; i < 7; i++) {
            executorService.submit(new Run());
        }
    }
}

class Run implements Runnable {
    private static final CyclicBarrier c = new CyclicBarrier(4, () -> System.out.println("barrierAction: 召唤神龙!"));

    @Override
    public void run() {
        Random random = new Random();
        int threadId = random.nextInt(100);
        try {
            System.out.println("thread run1 now, " + threadId);
            c.await();
            System.out.println("thread run2 now, " + threadId);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
//-------//-------//-------//-------//-------//-------//-------
thread run1 now, 70
thread run1 now, 97
thread run1 now, 72
thread run1 now, 24
thread run1 now, 61
thread run1 now, 94
thread run1 now, 95
barrierAction: 召唤神龙!
thread run2 now, 24
thread run2 now, 70
thread run2 now, 72
thread run2 now, 97
```





**CountDownLatch**：是一种多线程控制工具类，被称为到计数器，这个工具通常用来控制线程等待，它可以让某一个线程等待直到倒计数结束再开始执行。

```java
package com.zjxu9;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author zjxu97 at 2/24/21 11:48 PM
 */
public class CountDownLatchTest implements Runnable {
    private static final CountDownLatch end = new CountDownLatch(10);

    @Override
    public void run() {
        try {
            Thread.sleep(2000);
            System.out.println("模拟任务");
            //减少数值1
            end.countDown();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    public static void main(String[] args) throws InterruptedException {
        ExecutorService exec = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            exec.submit(new CountDownLatchTest());
        }
        //数值减少为0时唤醒所有等待线程执行操作
        end.await();
        //执行任务
        System.out.println("执行任务");
        exec.shutdown();
    }
}
//-----------//-----------//-----------//-----------
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
模拟任务
执行任务
```



## 线程池 ##

### 问题一：Java中的线程池是如何实现的？ ###



### 问题二：创建线程池的几个核心构造参数？ ###

1. 核心池容量
2. 总容量
3. 空闲单位
4. 空闲时间
5. 阻塞队列
6. 拒绝策略
7. 线程工厂

### 问题三：线程池中的线程是怎么创建的？是一开始就随着线程池的启动创建好的吗？ ###

不是，在添加的时候，新建的

### 问题四：既然提到可以通过配置不同参数创建出不同的线程池，那么Java中默认实现好的线程池又有哪些呢？请比较它们的异同 。 ###

缓存线程池、单线程池、固定容量线程池、定时线程池

### 问题六：如何在Java线程池中提交线程？ ###

实现 runnable。

```java
	executorService.submit(() -> System.out.println("hello world!"));
```





##  Java 内存模型相关问题 ##


### 问题一：什么是Java的内存模型，Java中各个线程是怎么彼此看到对方的变量的？
### 问题二：请谈谈volatile有什么特点，为什么它能保证变量对所有线程的可见性？
### 问题三：既然volatile能够保证线程间的变量可见性，是不是就意味着基于volatile变量的运算就是并 发安全的 ？
### 问题四：请对比下volatile对比Synchronized的异同

### 问题五：很多人都说要慎用ThreadLocal，谈谈你的理解，使用ThreadLocal需要注意些什么？