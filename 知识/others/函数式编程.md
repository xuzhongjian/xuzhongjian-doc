# Java8 函数式编程 #

[[TOC]]

## lambda 表达式 ##

什么是lambda表达式？lambda表达式就是一个匿名函数，使用表达式的方式来提供一种简洁清晰的方式来表示接口。

```java
    Runnable noArguments = () -> System.out.println("Hello World"); //1
    ActionListener oneArgument = event -> System.out.println("button clicked"); //2
    BinaryOperator<Long> add = (x, y) -> x + y; //3
    BinaryOperator<Long> addExplicit = (Long x, Long y) -> x + y; //4
```

上面都是一些lambda表达式。
上面的lambda表达式中，1、2和3都是通过编译器自己判断出来的数据的类型。但是在一些场景下，显式的指定参数的类型会比较合适。

### 引用值而不是变量 ###

在 lambda 表达式内，传入的参数必须是 final 类型的。在传入到 lambda 表达式中的参数，即使不是显式的 final 值，也必须是事实上的 final 值。如：

```java
    // 可以
    String name = getUserName();
    button.addActionListener(event -> System.out.println("hi " + name));
```

```java
    // 不可以
    String name = getUserName();
    name = formatUserName(name);
    button.addActionListener(event -> System.out.println("hi " + name));
```

第二串代码不可以，是因为 name 这个参数是一个变量，在传入 lambda 之前，哪怕在形式上，都没有实现final 。所以认为 name 没有实现形式上的 final 。

“Lambda 表达式中引用的局部变量必须是 final 或既成事实上的 final 变量。”

### 函数接口 ###

函数接口是只有一个抽象方法的接口，用作 Lambda 表达式的类型。
比如：

```java
    public interface ActionListener extends EventListener {
        public void actionPerformed(ActionEvent event);
    }
```

lambda 就是用一种简单的方式来实现函数接口。

### 类型推断 ###

和泛型的菱形符合一样，lambda 也会进行类型推断。

```java
    Predicate<Integer> atLeast5 = x -> x > 5;

    // 接口的源码
    public interface Predicate<T> {
        boolean test(T t);
    }
```

可以对传入的类型 T 进行判断。根据菱形符号中的类型，来自行判断传入的参数的类型。
但是如果省略了泛型（菱形符号），那么就没有办法进行类型推断，就会出错。

```java
    // 正确的是这样：BinaryOperator<Long> addLongs = (x, y) -> x + y;
    BinaryOperator add = (x, y) -> x + y;

    // Operator '& #x002B;' cannot be applied to java.lang.Object, java.lang.Object.
```

## 流 ##

Stream 类中的一组方法，是 Java8 函数式编程的核心点之一。

### 内部迭代 ###

```java
    int count = 0;
    for (Artist artist : allArtists) {
        if (artist.isFrom("London")) {
            count++;
        }
    }
```

上面是一个使用for循环的方式，来实现对列表的遍历。其实 for 循环的本质就是使用 Iterator 迭代器完成的一个迭代，将 for 循环展开的代码，就如下所示。

```java
    int count = 0;
    Iterator<Artist> iterator = allArtists.iterator();
    while(iterator.hasNext()) {
        Artist artist = iterator.next();
        if (artist.isFrom("London")) {
            count++;
        }
    }
```

上面是一个外部迭代的例子，使用 Iterator 迭代器是外部迭代。

下面这串代码就是使用内部迭代进行迭代。

```java
    long count = allArtists.stream()
    .filter(artist -> artist.isFrom("London"))
    .count();
```

上面的操作分为两步：
    1. 挑选出来自伦敦的艺术家
    2. 计数
上面的两个步骤，第一步叫做惰性求值，第二步叫做及早求值。惰性求值的返回依旧是一个 Stream 对象。而及早求值的返回不是 Stream 对象，而是其他的类型。
Stream 是用函数式编程方式在集合类上进行复杂操作的工具。

### 常用的流操作 ###

#### 惰性求值 ####

##### map #####

##### filter #####

##### flatMap #####

```java
    List<Integer> together = Stream.of(asList(1, 2), asList(3, 4))
                                    .flatMap(numbers -> numbers.stream())
                                    .collect(toList());
    assertEquals(asList(1, 2, 3, 4), together);
```

#### 及早求值 ####

##### collect(toList()) #####

这个操作是将一个 Stream 的类型的对象，直接转化成一个 List 的操作。

```java
    List<String> collect = hashSet.stream().collect(Collectors.toList());
```

##### reduce ######

```java
        HashSet<String> hashSet = new HashSet<>();
        hashSet.add("xuzhongjian");
        hashSet.add("xiaoshuaige");
        List<String> list = hashSet.stream().collect(Collectors.toList());
        String result = list.stream().reduce("begin", (x, y) -> x + ":" + y);
        System.out.println(result);

        // output:
        // begin::xiaoshuaige:xuzhongjian
```

reduce 表示将一个 Stream 对象中的所有元素，通过设定好的计算方式，最终只取得一个值。因为只取得一个值，所以这是一个及早求值的操作。

##### max 和 min #####

max 和 min 都是基于 reduce 而来的方法。

```java
    Integer reduce = intList.stream().reduce(Integer.MAX_VALUE, (x, y) -> x > y ? y : x);
```

上面这个是使用 reduce 生成一个求最小值的方法，求最大值也是类似的思路。

### 高阶函数 ###

高阶函数是指接受另外一个函 数作为参数，或返回一个函数的函数。

```java
    java.util.stream.Stream#reduce(T, java.util.function.BinaryOperator<T>)
    java.util.concurrent.ExecutorService#submit(java.lang.Runnable)
```

比如上面的 reduce 这个方法。几乎所有的 lambda 表达式的方法都是高阶函数。再比如，常用的线程池任务提交也是一个高阶函数。

## 类库 ##

在类库中，主要有两个方法。

### Stream.of(T ...) ###

```java
    List<Integer> idList = Stream.of(1, 2, 3, 4, 5).collect(Collectors.toList());

    List<Integer> together = Stream.of(asList(1, 2), asList(3, 4))
                                    .flatMap(numbers -> numbers.stream())
                                    .collect(toList());
    assertEquals(asList(1, 2, 3, 4), together);
```

使用这个方法，能够快速的构建一个 list 。

### Optional ###

Optional 是为核心类库新设计的一个数据类型，用来替换 null 值。

```java
    Optional<String> a = Optional.of("a");
    assertEquals("a", a.get());

    Optional emptyOptional = Optional.empty();
    Optional alsoEmpty = Optional.ofNullable(null);

    assertFalse(emptyOptional.isPresent());
    assertTrue(a.isPresent());
    assertEquals("b", emptyOptional.orElse("b"));
    assertEquals("c", emptyOptional.orElseGet(() -> "c"));
```

使用 Optional 对象的时候，先使用 isPresent() 方法检查对象是否存在。或者使用 orElse 或者是 orElseGet 方法，当 Optional 对象为空的时候，可以提供一个备选值。orElse 和 orElseGet 的区别在于，orElseGet 传入的是一个接口，支持更复杂的操作。而 orElse 可能只能传入一个简单的值。

## 高级集合类和收集器 ##

### 方法引用 ###

```java
    artist -> artist.getName()
    Artist::getName
    // ------------------------------
    (name, nationality) -> new Artist(name, nationality)
    Artist::new
```

使用方法引用，可以当参数是毋庸质疑的情况，可以使用方法引用。当传入的参数和方法的需要的参数完全匹配的情况，就可以使用方法引用。最简单的例子就是，当方法没有参数需要传入，比如 bean 的 getter。

### 使用收集器 ###

#### 转换成其他的集合 ####

如果在使用 stream.collect() 的时候需要指定生成的集合的类型，而不是生成一个模糊的集合类型。那么可以使用这个方法：

```java
    Stream<String> stream = null;
    TreeSet<String> collect = stream.collect(Collectors.toCollection(TreeSet::new));
```

上面的例子中，使用 Collectors.toCollection(TreeSet::new) 指定了生成一个 TreeSet。如果使用 Collectors.toSet()，那么没用办法指定生成一个TreeMap，只能生成一个Set。

#### 转换成值 ####

```java
    int averagingNum = list.stream().collect(Collectors.averagingInt(album -> album.getTrackList().size()));
```

还可以使用 maxBy 和 minBy 找出最大和最小的值。

#### 数据分块 ####

数据分块指的是，按照一个 boolean 属性，来将一个 stream 划分为 true 和 false 两个 list。
将艺术家组成的流分成乐队和独唱歌手两部分。

```java
    public Map<Boolean, List<Artist>> bandsAndSolo(Stream<Artist> artists) {
        return artists.collect(partitioningBy(artist -> artist.isSolo()));
    }
```

#### 数据分组 ####

数据分组，是数据分块的进一步，数据分块可以理解成是数据分组的一个特殊形式，即，只有两个值的数据分组就是数据分块，数据分组是可以有多个值的数据分块。

```java
    public Map<Artist, List<Album>> albumsByArtist(Stream<Album> albums) {
        return albums.collect(Collectors.groupingBy(Album::getMainMusician));
    }
```
