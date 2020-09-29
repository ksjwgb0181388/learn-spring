# 一、JAVA8新特性

## 1.1 并行流与串行流

- **并行流**：把一个内容分成多个数据块，并用不同的线程分别处理每个数据块的流。相比较串行的流，**串行流可以最大限度的提高系统的执行效率**



java8中将并行进行了优化，通过Stream API可以声明地通过parallel（）与sequential（）在并行流与顺序流之间进行切换



## 1.2 Optinal 最大减少空指针异常



## 1.3 Nashorm引擎，允许在JVM上运行JS程序





# 二、Lambda表达式

## 2.1 使用举例

~~~java
Runnable r1 = new Runnable(){
  @override
  public void run(){
      sout("123");
};
    
//Lambda 简化
Runnable r2 = ()->sout("123");
    

//Lambda表达式    
Comparator<Integer> com = (o1,o2) -> Integer.compare(o1,o2);
int result = com.compare(11,22);
    
//方法引用
Comparator<Integer> com2 = Integer :: compare;
int result = com2.compare(11,22);    
    
~~~



## 2.2 表达式解析

1. ->：lambda 操作符 或 箭头操作符
   1. 左边：形参列表（原来接口的形参）
   2. 右边：lambda体（就是重写的抽象方法的方法体）
2. 本质：作为接口的实例（就是对象）



### 基本使用

1. 无参返回值

   ~~~java
   Runnable r1 = new Runnable(){
     @override
     public void run(){
         sout("123");
   };
       
   //Lambda 简化
   Runnable r2 = ()->sout("123");
   ~~~

2. 一个参数，但没有返回值

   ~~~java
   Concumer<String> con = new Concumer<String>(){
       public void accept(String s){
           sout(s);
       }
   }
   con.accept("123");
   
   
   Concumer<String> con2 = s-> sout(s);
   con2.accept("我是你爷");
   
   ~~~

3. 两个或以上的参数，多条执行语句，还有返回值

   ~~~java
   //Lambda表达式    
   Comparator<Integer> com = (o1,o2) -> Integer.compare(o1,o2);
   int result = com.compare(11,22);
   ~~~



# 三、函数式接口

如果一个接口中只声明了一个抽象方法，则此接口就是函数式接口

可以在一个接口上添加注解 **@FunctionalInterface**，这样可以检查它是否是一个函数式接口



## 3.1 JAVA内置四大核心函数式接口

| 函数式接口                                                   | 参数类型                  | 返回类型                  | 用途                                                         |
| ------------------------------------------------------------ | ------------------------- | ------------------------- | ------------------------------------------------------------ |
| **Consumer<T> 消费型接口**                                   | T                         | void                      | 对类型为T的对象应用操作，包含方法 **void accept(T t)**       |
| **Suppiler<T> 供给型接口**                                   | 无                        | T                         | 返回类型为T的对象，包含方法。 **T get()**                    |
| **Function<T,R> 函数式接口**                                 | T                         | R                         | 对类型为T的对象应用操作，并返回结果。结果是R类型的对象。方法 **Rapply(T t)** |
| **Predicate<T> 断定型接口**                                  | T                         | boolean                   | 确定类型T的对象是否满足某约束，并返回布尔值。包含方法 **boolean test(T t)** |
| BiFunction<T,U,R>                                            | T,U                       | R                         | 对类型为 T,U参数应用操作，返回R类型的结果。方法 **R apply(T t,U u)** |
| UnaryOperator<T>  (Function的子接口)                         | T,T                       | T                         | 对类型为T的对象进行一元计算，并返回T类型的结果。包含方法为 **T apply(T t1,T t2)** |
| BiConsumer<T,U>                                              | T,U                       | void                      | 对类型T,U参数应用操作。包含方法：**void accept(T,t,U,u)**    |
| BiPredicate<T,U>                                             | T,U                       | boolean                   | 包含方法：**boolean test（T,t，U,u）**                       |
| ToIntFunction<T><br />ToLongFuncition<T><br />ToDoubleFunction<T> | T                         | int<br />long<br />double | 分别计算int、long、double值的函数                            |
| intFunction<R><br />LongFunction<R><br />DoubleFunction<R>   | int<br />long<br />double | R                         | 参数分别为int、long、double类型的函数                        |



### Consumer

~~~java
public void contextLoads() {
    happy(400,money-> System.out.println(money));
}

public void happy(double money, Consumer<Double> con){
    con.accept(money);
}
~~~



### Predicate

~~~Java
        List<String> list = Arrays.asList("北京", "天津", "东京", "南京","洗净");
        List<String> list1 = filterString(list, new Predicate<String>() {
            @Override
            public boolean test(String s) {
                return s.contains("京");
            }
        });


        //Lambda简化
        List<String> list2 = filterString(list, s -> s.contains("京"));
        System.out.println(list2);

    

    //根据给定的规则，过滤集合中的字符串。此规则由Predicate的方法决定
    public List<String> filterString(List<String> list, Predicate<String> predicate){

        ArrayList<String> filterList = new ArrayList<>();

        for (String s : list) {
            if (predicate.test(s)){
                filterList.add(s);
            }
        }
        return filterList;
    }
~~~





# 四、方法引用和构造器引用

## 4.1 方法引用 情况1

- 当要传递给Lambda体的操作，已经有实现的方法了，可以使用方法引用
- 方法引用看做是Lambda表达式深层次的表达。换句话说，方法引用就是Lambda表达式！也就是函数式接口的一个实例。通过方法的名字来指向一个方法，可以认定是Lambda的语法糖
- **要求**：==实现接口的抽象方法的参数列表和返回值类型，必须与方法引用的方法的参数列表和返回值一致！==
- 格式：使用操作符 ==::== 将类（或对象）与方法名隔开
- 如下主要使用情况
  - 对象::实例方法名
  - 类::静态方法名
  - 类::实例方法名



~~~java
    //使用情景
    //当要传递给Lambda体的操作，已经有实现的方法了，可以使用方法引用

    //	情况一：对象::实例方法
    //  Consumer中的void accept(T t)
    //  PrintStream中的void println(T t)
  
  Consumer<String> con = str -> sout(str);
  con.accept("123");

  //方法引用简化
  PrintStream ps = System.out;
  Consumer<String> con2 = ps :: println;


  //Emploee中的 String getName();
  //Supplier中的T get()
  Emploee emp = new Emploee();
  Supplier sup = emp :: getName;
  sout(sup.get())


~~~



**方法引用的使用要求**

- 接口中的抽象方法的形参列表和返回值类型与 方法引用的形参列表和返回值都得相同（针对情况1和2）





## 4.2 方法引用 情况2

类::静态方法

~~~java
//Comparator中的int compare(T t1,T t2)
//Integer 中的int compare(T t1,T t2)

Comparator<Integer> com1 = (t1,t2) -> Integer.compare(t1,t2);
com1.compare(11,2);

Comparator<Integer> com2 = Integer :: compare;
com2.compare(111,22);


//Function中的R apply(T t)
//Math中的Long round(Double d)
Function<Double,Long> f = d -> Math.round(d);

Function<Double,Long> f = Math::round;

~~~



## 4.3 方法引用 	情况3

类::实例方法

~~~java
//Comparator中的int compare(T t1,T t2)
//String中的int t1.compareTo(t2)
Comparator<String> com1 = (t1,t2)->t1.compareTo(t2);

Comparator<String> com2 = String :: compareTo;

//BiPredicate中的Boolean test(T t1,T t2)
//String 中的boolean t1.equals(t2)

//Function中的R apply(T t)
//Employee中的String getName()
Function fn1 = e-> e.getName();

Function fn2 = Emlpyee::getName;

~~~



## 4.4 方法引用 情况4

### 构造器引用

~~~java
//supplier中的 T get()
//Employee的空参构造器：Employee()
Supplier<Employee> emp = () -> new Employee();

Supplier<Employee> emp = Employee :: new;

~~~







# 五、StreamAPI的概念

**Stream API**：真正的将函数式编程引入到java中。

可以指定你希望对集合进行的操作，可以执行复杂的查找，过滤，映射数据等操作。

使用StreamAPI对集合数据进行操作，就类似于使用SQL执行的数据库查询



## 5.1 StreamAPI和Collection集合区别

Collection是一种静态的内存数据结构，而Stream是有关计算的。

前者主要面向内存，存储在内存中

后者主要面向CPU，通过CPU实时计算



## 5.2 什么是Stream

是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列

==集合将求数据，Stream将计算！！！==

- Stream 自己不会存储数据
- 不会改变源对象，相反，会返回一个持有结果的新Stream
- Stream操作是有延迟的。这意味着他们会等到需要结果的时候才执行



## 5.3 Stream的操作三个步骤

- 创建Stream

  一个数据源（如：集合，数组）获取一个流

- 中间操作

  一个中间操作链，对数据源的数据进行处理

- 中止操作（终端操作）

  一旦执行中止操作，就执行中间操作链，并产生结果。之后不再使用



![](C:\Users\Admin\Desktop\微信图片_20200919110024.png)





# 六、StreamAPI的实例化

## 6.1 集合创建Stream

通过集合进行创建

- default Stream<E> stream() 返回一个顺序流
- default Stream<E> parallelStream() : 返回一个并行流

~~~java
List<String> list = new ArrayList();

//返回一个顺序流
//按照集合里面的顺序去取
Stream<String> s = list.stream();

//返回一个并行流
//按照多线程方式进行获取集合数据
Stream<String> s = list.parallelStream();


~~~



## 6.2 数组创建Stream

- static<T> Stream<T> stream(T[] array)：返回一个流



重载形式，能够处理对应基本类型的数组

- public static IntStream stream(int[] array);
- public static LongStream stream(long [] array);
- public static DoubleStream stream(double[] array);



~~~java
int[] arr = {1,2,3,4};

//返回一个int类型
IntStream stream = Arrays.stream(arr); 
~~~





## 6.3 通过Stream的of()

可以调用Stream类静态方法of()，通过显示值创建一个流

可以接受任意数量的参数



- public static<T> Stream<T> of(T....values)：返回一个流



~~~java
Stream<Integer> stream = Steam.of(1,2,3,4);
~~~





## 6.4 创建无限流

- public static<T> Stream<T> iterate(final T seed,final UnaryOperator<T> f)： 迭代
- public static<T> Stream<T> generate(Supplier<T> s)：生成



~~~java
//遍历前10个偶数
Stream.iterate(0,t-> t+2).limit(10);

//生成
Stream.generate(Math::random).limit(10);
~~~



# 七、Stream中间操作（筛选与切片）

## 7.1 筛选与切片

| 方法                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| filter(Predicate p) | 接受Lambda，从流中排除某些元素                               |
| distinct()          | 筛选，通过流所生成的元素的hashCode和equals去除重复元素       |
| limit(long maxSize) | 截断流，使其元素不超过给定数量                               |
| skip(long n)        | 跳过元素，返回一个扔掉了前n个元素的流。若流中元素不足n个，则返回一个空流。<br />与limit互补 |



~~~java
public void test2(){
    List<String> list = new ArrayList<>();
    list.add("京东");
    list.add("北京");
    list.add("北京");
    list.add("京都");
    list.add("京都");
    list.add("码云");
    list.add("马虎疼");

    // filter(Predicate p)接受Lambda，从流中排除某些元素
    list.stream().filter(s -> s.contains("京")).forEach(System.out::println);
    System.out.println("***************************************");

    // distinct()筛选，通过流所生成的元素的hashCode和equals去除重复元素
    System.out.println("***************************************");
    list.stream().distinct().forEach(System.out::println);
    System.out.println("==================================");


    // limit(long maxSize)截断流，使其元素不超过给定数量
    list.stream().limit(1).forEach(System.out::println);
    System.out.println("***************************************");


    // skip(long n)跳过元素，返回一个扔掉了前n个元素的流。若流中元素不足n个，则返回一个空流。与limit互补
    list.stream().skip(2).forEach(System.out::println);
}
~~~



## 7.2 映射

|              方法               |                             描述                             |
| :-----------------------------: | :----------------------------------------------------------: |
|       **map(Function f)**       | 接受一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素 |
| mapToDouble(ToDoubleFunction f) | 接受一个函数作为参数，该函数会被应用到每个元素上，产生一个新的DoubleStream |
|    mapToInt(ToIntFunction f)    | 接受一个函数作为参数，该函数会被应用到每个元素上，产生一个新的IntStream |
|     **flatMap(Function f)**     | 接受一个函数作为参数，将流中的每个值都换成另一个流，然后把所有的流连接成一个流 |
|                                 |                                                              |



~~~java
//map(Function f)接受一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素
List<String> list = new ArrayList<>();
list.add("aaaa");
list.add("bbb");
list.add("ccc");
list.stream().map(str->str.toUpperCase()).forEach(s-> System.out.println(s));


//获取长度大于3的字符串


~~~





## 7.3 排序

- sorted() ： 产生一个新流，其中按自然顺序排序
- sorted(Comparator com) ： 产生一个新流，其中按照比较器顺序排序



~~~java
List<Integer> integers = Arrays.asList(12, 45, 14, 77, 10, -98);

integers.stream().sorted().forEach(System.out::println);
System.out.println();


//自定义排序
integers.stream().sorted((e1,e2)->{
    return Integer.compare(e2,e1);
}).forEach(System.out::println);
~~~





# 八、中止操作

## 8.1 匹配与查找

- 终端操作会从流的流水线生成结果，其结果可以是任何不是流的值，例如 list、Integer、甚至是void
- 流进行了中止操作后，不能再次使用



| 方法                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| allMatch(Predicate p)  | 检查是否匹配所有元素                                         |
| anyMatch(Predicate P)  | 检查是否至少匹配一个元素                                     |
| noneMatch(Predicate p) | 检查是否没有匹配所有元素                                     |
| findFirst()            | 返回第一个元素                                               |
| findAny()              | 返回当前流中的任意元素                                       |
| count()                | 返回流中元素总数                                             |
| max(Comparator c)      | 返回流中最大值                                               |
| min(Comparator c)      | 返回流中最小值                                               |
| forEach(Consumer c)    | 内部迭代（使用Collection接口需要用户取迭代，成为外部迭代。相反Stream API使用内部迭代） |

~~~java
        List<Integer> integers = Arrays.asList(12, 45, 14, 77, 10, -98);
        boolean b = integers.stream().allMatch(e -> e > 19);
        System.out.println(b);

        long count = integers.stream().count();

        System.out.println(count);
~~~



## 8.2 归约

- reduce(T iden,BinaryOperator b)：可以将流中元素反复结合起来，得到一个值。返回T
- reduce(BinaryOperator b)：可以将流中元素反复结合起来，得到一个值。返回Optional<T>



**注意**：map和reduce的连接通常被称为map-reduce模式

~~~java
       //计算1-10的自然数和
        List<Integer> integers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        Integer reduce = integers.stream().reduce(0, Integer::sum);

        System.out.println(reduce);
~~~



## 8.3 收集

- collect(Collection c)：将流转换为其他形式。接受一个Collector接口的实现，用于给Stream中元素做汇总的方法



Collector接口中方法的实现决定了如何对流执行收集的操作（如收集到List，Set，Map）

另外，Collectors实用类提供了很多静态方法，可以方便地创建常见收集器实例。



### Collectors

| 方法               | 返回类型             | 作用                                    |
| ------------------ | -------------------- | --------------------------------------- |
| **toList**         | List<T>              | 把流中元素收集到list                    |
| **toSet**          | Set<T>               | 把流中元素收集到set                     |
| **toCollection**   | Collection<T>        | 把流中元素收集到创建的集合              |
| **counting**       | Long                 | 计算流中元素的个数                      |
| **summingInt**     | Integer              | 对流中元素的整数属性求和                |
| **averagingInt**   | Double               | 计算流中元素Integer属性的平均值         |
| **summarizingInt** | IntSummaryStatistics | 收集流中Integer属性的统计值。如：平均值 |
|                    |                      |                                         |



~~~java
        List<Integer> integers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        Set<Integer> collect = integers.stream().
            							filter(e->e>3).
            							collect(Collectors.toSet());

        System.out.println(collect);
~~~





# 九、Optional类的操作

Optional<T> 是一个容器类，它可以保存类型T的值，代表这个值存在。或者仅仅保存null，表示这个值不存在，

原来null表示一个值不存在，现在Optional可以更好的表达这个概念，并且避免空指针异常。



- Optional类的javadoc描述：

  这是一个可以为null的容器对象，如果值存在则isPresend()方法返回为true，调用get()方法会返回该对象





## Optional类的方法

- 创建Optional类对象的方法：
  - Optional.of(T t)：创建一个Optional实例，**t必须非空**
  - Optional.empty()：创建一个空的Optional实例
  - Optional.ofNullable(T t)：t可以为null
- 判断Optional容器是否包含对象：
  - boolean isPresent()：判断是否包含对象
  - void isPresend(Consumer<? super T consumer>：如果有值，就执行Consumer接口的实现代码，并且该值会作为参数传给它
- 获取Optional容器的对象
  - T get()：如果调用对象包含值，返回该值，否则抛异常
  - T orElse(T other):如果有值则将其返回，否则返回指定的other对象
  - T orElseGet(Supplier<? extends T> other)：如果有值则将其返回，否则返回由Supplier接口实现提供的对象
  - T orElseThrow(Supplier<? extends X>):如果有值则将其返回，否则抛出有Suppiler接口实现提供的异常





# 十、注意事项

## lambda表达式使用局部变量

```java
String catCry = "猫: 喵喵叫";
Animal cat  = () -> System.out.println(catCry);
cat.cry();
打印输出:猫: 喵喵叫
```



**lambda表达式可以使用局部变量,但是必须是final类型的或事实上final类型的(不可改变)**



- 第一，实例变量和局部变量背后的实现有一个关键不同。**实例变量都存储在堆中，而局部变量则保存在栈上**。

- 如果Lambda可以直接访问局部变量，而且Lambda是在一个线程中使用的，则使用Lambda的线程，可能会在分配该变量的线程将这个变量收回之后，去访问该变量。

- 因此，Java在访问自由局部变量时，实际上是在访问它的**副本**，而不是访问原始变量。如果局部变量仅仅赋值一次那就没有什么区别了——因此就有了这个限制。

- 第二，这一限制**不鼓励你使用改变外部变量的典型命令式编程模式**（我们会在以后的各章中
  解释，这种模式会阻碍很容易做到的**并行**处理）。











































