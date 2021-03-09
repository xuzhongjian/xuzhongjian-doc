# Java 集合 #

### HashMap的源码，实现原理，JDK8中对HashMap做了怎样的优化。 ###

JDK8中将JDK7中原有的链表+数组的结构，改变成了链表+数组+红黑树的形式。

在原有的发生hash冲突，使用拉链法将发生冲突的元素变成链表的基础上，当链表的长度达到8之后，将原有的链表转化成红黑树。

还有一个区别是，JDK7使用的是头插法，JDK8使用的是尾插法。在JDK7中并发插入会存在的问题是链表会成环状，然后导致读取的时候卡死。

在JDK8中使用的是尾插法，解决了JDK7中的问题，但是在并发插入的时候还是会存在数据丢失的问题。

### HaspMap扩容是怎样扩容的，为什么都是2的N次幂的大小。 ###

在超过了【负载因子 * capacity】的时候进行扩容。对原有的capacity翻倍，然后对老的数组中数据进行遍历，然后计算它们在新的数组中的位置

在计算桶的位置的时候，使用的方法是：hashCode & (length-1) ，这个操作的方法类似于取余操作，这个操作如果在n为2的N次幂的情况下是等同于 hash % n 取余数的值。 & 运算的效率要高于hash % n取余的运算。

### HashMap，HashTable，ConcurrentHashMap的区别。 ###

并发：hashmap不支持并发，其余的两个支持并发。

单线程性能：hashmap > concurrenthashmap > hashtable

在并发条件下：concurrenthashmap > hashtable

### 极高并发下HashTable和ConcurrentHashMap哪个性能更好，为什么，如何实现的。 ###

ConcurrentHashMap性能比hashtable好，锁的粒度：

hashTable，使用synchornized保证线程安全，线程竞争，竞争激烈的情况下，效率低下。当一下线程访问hashTable方法的时候，其他的线程会进入轮询或者阻塞的情况。如果线程1是用put方法添加元素，线程2不能put元素也不能get元素，所以竞争越激烈，所以效率越低。

ConcurrentHashMap使用锁分段技术，容器对数据进行分段（segment），每一段都加入分配一把锁，当一个线程访问其中一个段数据的时候，只会锁住当前这个段内的数据。其他的数据段能被访问，不会被锁住（JDK7中）。ConcurrentHashMap在JDK8中使用的是Node，每一个k-v对单独加锁，锁的粒度小，对并发的支持更好。

一句话解释就是，锁的粒度决定了性能。

https://blog.csdn.net/programmer_at/article/details/79715177

### HashMap在高并发下如果没有处理线程安全会有怎样的安全隐患，具体表现是什么。 ###

在JDK7中，HashMap在高并发下，使用put会导致链表成环。而后造成put和get操作死循环。

在JDK8中，HashMap虽然通过将头插法改为尾插法，修复了链表成环的问题，从而避免了死循环，但是存在在并发插入的情况下，数据丢失的问题。

### java中四种修饰符的限制范围。 ###

| 修饰符        | 解释         |
| ------------- | ------------ |
| public        | 所有的都可见 |
| default（无） | 同一个包内   |
| protected     | 子类         |
| private       | 仅自己       |

### Object类中的方法。 ###

| 方法名                  | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| hashCode                | 计算这个对象的hash值，默认的实现是一个native方法，依赖于虚拟机的实现。 |
| equals                  | 比较这两个对象是不是相等。默认比较的是这两个对象的物理地址是不是一致。也就是在堆内是不是同一个地址。 |
| toString                | 转化成字符串。默认的实现是对象名+hashCode的一个编码。通常在开发中会将这个方法使用Gson重写为json字符串。 |
| wait、notify、notifyAll | 和多线程相关的方法。调wait方法的线程进入waiting状态，会释放锁。只有等到这个对象调用了notify或notifyAll，才会解除waiting的状态，并继续执行下去。 |
| clone                   | 浅克隆。                                                     |



### 接口和抽象类的区别。 ###

1. 多继承：一个类，对接口可以实现多个，但是只能继承一个抽象类。
2. 字段：接口中的所有字段是也只能是 `public static final`属性的字段；但是抽象类中字段没有这个限制，与普通的类一致。
3. 方法：接口的方法必须为`public abstract` ，是抽象方法，不能有方法的实现体。抽象类中可以有抽象方法，也可以有与普通的类一致的普通方法。JDK8后接口可以有default修饰的默认实现。

关于选择：

1. 如果需要创建不带有任何通用方法和成员变量的基类，选择接口。
2. 在必须要有方法定义和成员变量的时候才选择成为一个抽象类。



### 动态代理的两种方式，以及区别。 ###

+ JDK动态代理：利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。
+ CGlib动态代理：利用ASM（开源的Java字节码编辑库，操作字节码）开源包，将代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。
+ 区别：
  + JDK代理只能对**实现接口的类**生成代理；
  + CGlib是针对类实现代理，对指定的类生成一个子类，并覆盖其中的方法，这种通过继承类的实现方式，不能代理**final修饰的类**。

### Java序列化的方式。 ###

 Java对象的序列化有两种方式。

1. 是相应的对象实现了序列化接口Serializable，这个使用的比较多，对于序列化接口Serializable接口是一个空的接口，它的主要作用就是标识这个对象时可序列化的，jre对象在传输对象的时候会进行相关的封装。

2. b.实现序列化的第二种方式为实现接口Externalizable:

   ```java
   void writeExternal(ObjectOutput out) throws IOException;
   
   void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
   ```

   实现其中的两个抽象方法就能实现完成自动的序列化。

### 传值和传引用的区别，Java是怎么样的，有没有传值引用。 ###

Java对于基本数据类型是值传递，对于对象是引用传递。

### 一个ArrayList在循环过程中删除，会不会出问题，为什么。 ###

会。会产生并发修改异常`ConcurrentModificationException`  。在所有的集合内都有一个modCount字段，modcount的意思就是修改次数。所有存在modcount的集合都是线程不安全的。modcount只有在本数据结构对应迭代器中才使用。

在一个迭代器初始的时候会赋予它调用这个迭代器的对象的mCount，如何在迭代器遍历的过程中，一旦发现这个对象的mcount和迭代器中存储的mcount不一样那就抛异常。

快速失败策略：

我们知道 java.util.HashMap 不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。

这一策略在源码中的实现是通过 modCount 字段，modCount 顾名思义就是修改次数，对HashMap 内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的 expectedModCount。在迭代过程中，判断 modCount 跟 expectedModCount 是否相等，如果不相等就表示已经有其他线程修改了 Map：注意到 modCount 声明为 volatile，保证线程之间修改的可见性。

所以，遍历那些非线程安全的数据结构时，尽量使用迭代器。

### Java 集合类框架的基本接口有哪些？ ###

![](https://img-blog.csdn.net/20180607181446785?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JfZXZhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

1. iterator
2. collection
3. queue
4. list
5. set
6. sortedset
7. map
8. sortedmap

### HashSet 和 TreeSet 有什么区别？ ###

1. hashset可以放入空值，treeset不行。
2. treeset是一个排序的set，可以按照默认顺序或者自定义顺序排序。元素放入自动排序。而hashmap是无序的。
3. 识别上，treeset根据自然升序或者自定义的比较器来排序。hashmap通过hashcode来选择元素的位置。

### HashSet 的底层实现是什么? ###

基于hashmap，hashmap的kv对结构中，将v设置成空，即可实现hashset。

### LinkedHashMap 的实现原理? ###

LinkedHashMap，在hashMap的基础上，增加了一个双链表，来维护插入顺序或者是访问顺序。默认情况下是插入顺序，在构造LinkedHashMap的时候，可以使用使用boolean accessOrder来控制是否是访问顺序。

### 使用linkedHashMap来实现LRU。 ###

```java
package com.zjxu9;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * @author zjxu97 at 2/22/21 8:43 PM
 */
public class LURTest {
    private static final int SIZE = 5;

    public static void main(String[] args) {
        /*
         * 16       初始容量
         * 0.75f    HashMap扩容的负载因子
         * false    访问顺序 true / 添加顺序 false
         * lru 使用true
         */
        LinkedHashMap<String, String> map = new LinkedHashMap<String, String>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry eldest) {
                return this.size() > SIZE;
            }
        };
        for (int i = 0; i <= 13; i++) {
            map.put("" + i, "" + (i * 3 - 5));
            if (i == 10) {
                map.get("" + 8);
            }
        }
        System.out.println("size: " + map.size());
        map.forEach((a, b) -> System.out.println(a + " " + b));
    }
}
```

removeEldestEntry方法，是在put方法执行完毕之后，对put方法的一个增强。是hashMap中保留的一个扩展点，而后LinkedHashMap对这个方法进行了实现。

### 为什么集合类没有实现 Cloneable 和 Serializable 接口？ ###

克隆(cloning)或者是序列化(serialization)的语义和含义是跟具体的实现相关的。因此，应该由集合类的具体实现来决定如何被克隆或者是序列化。

### 数组 (Array) 和列表 (ArrayList) 有什么区别？什么时候应该使用 Array 而不是 ArrayList？ ###

确定不扩容的数据使用array。后续存在扩容的可能的时候使用arraylist。

### Set 里的元素是不能重复的，那么用什么方法来区分重复与否呢？是用 == 还是 equals()？它们有何区别？ ###

使用的equals方法来判断是否重复的。区别在于，equals()方法比较的是两个对象中具体的值，但是 == 是将两个对象的地址相比较。

### Comparable 和 Comparator 接口是干什么的？列出它们的区别 ###

这两个都是接口：

Comparable中的CompareTo方法：将此对象与指定的对象进行比较。当此对象小于、等于或大于指定对象时，返回负整数、零或正整数。

表示是一种可以排序的能力，比如Number类下的所有的数字类都实现了这个接口。实现Comparable接口的对象列表（和数组）可以通过 Collections.sort（和 Arrays.sort）进行自动排序。实现此接口的对象可以用作有序映射中的键或有序集合中的元素，无需指定比较器。

Comparator中的compare方法：比较其两个参数的顺序。当第一个参数小于、等于或大于第二个参数时，返回负整数、零或正整数。是一种比较器，这是一个函数式接口。通常使用lambda表达式来表示。使用场景最多的位置是stream。

### Collection 和 Collections 的区别。 ###

一个是接口，一个是对于这个接口提供的工具类。s





# HashMap #

[[TOC]]

HashMap 是通过hash的方式，来存储 k-v 对的一个对象。简单来说，就是在 k 上面添加索引，通过 k 上的索引能够快速的找到 v 。因为通常 k-v 是一个整体的数据结构。这样能将 list 遍历查找的时间复杂度从O(n) 降低到 O(1)。

## 基本数据结构 ##

在 Java 1.7 中，HashMap 是由数组和链表组装而成的。而在 Java 1.8 中，在原来的数据结构的基础上，添加了红黑树，结构更加复杂了，但是效率也变高了。

## 数据插入的基本流程 ##

当尝试着插入一个新的k-v（也就是Map.Entry<K,V>对象）时。

1. 计算传入的 k 的 hash 值 = hashK。

2. 使用 (table.length - 1) & hashK，计算出这个 k-v 对应的数组的位置 = p。

3. 执行插入：
    1. 如果 table[p] 为空，那么直接将新插入的 k-v 插入到 table[p] 处即可。

    2. 如果 table[p] 已经有值了，那么还需要讨论：

        1. 如果 k.hash = table[p].hash，那么认为这是同一个 k-v，然后将 table[p] 更新。

        2. 如果 table[p] 是二叉树的头节点，那么调用二叉树的put方法，将其put进二叉树里面。

        3. 如果上面的两点都不满足，那么数组的这个位置就是一个链表。先直接遍历，获取到队尾，然后在队尾插入我们需要插入的新的k-v。

            1. 如果插入新的 k-v 对之后的新的列表的长度，达到了8（树化阈值TREEIFY_THRESHOLD），那么进行树化。

## 几个阈值 ##

```java
    /**
     * 默认的初始容量 - 必须是2的n次方
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * 最大容量，如果某个具有参数的构造函数隐式指定了更高的值，则使用该值。
     * 必须是2的n次方。
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 构造函数中未指定时，使用的负载因子。
     * size / capacity > DEFAULT_LOAD_FACTOR 那么resize。
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 存储的结构从list转化成tree的阈值。从list转化成tree的时候，应该大于8。
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 存储的结构从tree转化成list的阈值。从tree转化成list的时候，应该小于等于8。
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 可对存储的元素进行树型化的最小容量。
     * 没达到这个容量的话，会调整数组的大小。
     * 至少是4倍的树化阈值，才能避免调整数组的大小和树化阈值之间的冲突。
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
```

## 几个字段 ##

```java

    /**
     * 数组。
     */
    transient Node<K,V>[] table;

    /**
     * 用来存储k-v对的set。
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * 已经存储的k-v对的数量。
     */
    transient int size;

    /**
     * hashMap 发生结构性变化的次数。
     */
    transient int modCount;

    /**
     * 下一次的resize发生在这个值这里。
     * threshold = load factor * capacity
     * 
     * @serial
     */
    int threshold;

    /**
     * 负载因子，默认为0.75
     *
     * @serial
     */
    final float loadFactor;
```

## 构造方法 ##

### 默认的构造方法 ###

```java
    /**
     * 构造一个空的 HashMap，使用默认的容量16，和默认的负载因子0.75。
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // all other fields defaulted
    }
```

### 指定参数的构造方法 ###

```java
    /**
     * 构造一个空的 hashMap ，使用指定的初始容量和负载因子。
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException 如果初始容量为负数或者负载因子非正数。
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

```java
    /**
     * 返回大于等于cap的最小的2的n次方。
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

## 计算应在数组中的位置 ##

### k.hash ##

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

主要分为3个步骤：

1. 取对象的 hashCode 方法 = h。

2. 取 h 高位，也就是将 h 的前16高位，移动到后16低位上 = hash。

3. return h ^ hash;

这样做的目的是，尽量的让 hashCode 的高位也参与计算。因为计算在 table 中的位置是通过低位来计算的。

### hash & (table.length - 1) ###

```java
    // ...
    // int n = tab.length;
    if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
    // ...
```

上面计算出来的 hash 值，通过和 table.length 取与操作，可以达到index = hash % (table.length) 的效果。这样可以取得一个范围为 [0 , table.length-1] 的随机数，并将这个随机数作为在数组中的位置。（当然，以这种方式计算的速度要比 % 操作快一些。）

## 插入 ##

插入方法是整个 HashMap 中的核心方法之一。

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // tab 是node数组，p是计算出来的那个数组的位置的node，n是数组的长度，i是p的index
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            // 如果数组为空，那么新建一个数组。
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 如果计算出来的数组的位置为空，那么直接吧新的值set进去即可。
            tab[i] = newNode(hash, key, value, null);
        else {
            // 计算出来的位置已经有了值，也就是发生了hash冲突。
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // p 和插入进来的 key 的 hash 值相同，那么认为 p.key 和 key 是同一个值，那么将 p 覆盖。
                e = p;
            else if (p instanceof TreeNode)
                // 如果 p 是一个数的节点，那么认为数组的这个位置已经是一个红黑树了。直接使用树的插入方法插入即可。
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 除了上面两个可能性以外就是链表了。
                for (int binCount = 0; ; ++binCount) {
                    // 找到链表的尾节点
                    if ((e = p.next) == null) {
                        // 并且把新插入的值 set 到尾节点上
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            // 如果达到了树化的阈值，进行树化，将这个链表转化成一个红黑树。
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果在遍历链表的过程中，发现了 hash 值相同的，那么判定成是同一个 k，将原来的值覆盖即可。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // 已经存在了，直接覆盖。
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 如果超过 负载因子 * capcity，那么对数组进行扩容。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

## 扩容 ##

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 达到最大的容量之后，直接返回，没有操作
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 容量 * 2，并且阈值也 * 2
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        // 在其他条件下进行初始化
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 计算新的阈值：新的容量 * 负载因子
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        // 老的数组中的这个位置只有一个元素
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 老的数组中的这个位置是一个红黑树
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
                        // 老的数组中的这个位置是一个链表的头
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 取决于某一位是不是为 0 
                            // 比如 oldCap 为16
                            // 那么 就是 xxxx xxxx & 10000
                            // 所以取决于第五位是 0 还是 1
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // 扩容之后的 index = index + oldCap
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```