---

		title:  深入探索 Gradle 自动化构建技术（九、Gradle 插件开发平台化框架 ByteX 解密）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


# 一、前置知识

## 1、函数式编程

###  1）、什么是函数式编程？

面向对象编程是对数据进行抽象，而函数式编程是对行为进行抽象。现实世界中，数据和行为并存，而程序也是如此。


### 2）为什么要学习函数式编程？

- 用函数（行为）对数据处理，是学习大数据的基石。
- 好的效率（并发执行），
- 完成一个功能使用更少的代码。
- **对象转向面向函数编程的思想有一定难度，需要大量的练习**


## 2、Java 1.8 Lambda 表达式 

###  1）、什么是 Lambda 表达式？

Lambda 是一个匿名函数，即没有函数名的函数，它简化了匿名委托的使用，让代码更加简洁。

###  2）、两个简单的 Lambda 表达式示例

```java
//匿名内部类
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.print("hello toroot");
    }
};

//lambda
Runnable r2 = ()->System.out.print("hello toroot");

//匿名内部类
TreeSet<String> ts = new TreeSet<>(new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return Long.compare(o1.length(),o2.length());
    }
});

//lambda
TreeSet<String> ts2 = new TreeSet<>((o1,o2)-> Long.compare(o1.length(),o2.length()));
```

### 3）、Lambda  表达式语法

Lambda 表达式在 Java 语言中引入了一个新的语法元素和操作符。这个操作符为 “->” ，该操作符被称为 Lambda 操作符或剪头操作符。

它将 Lambda 分为两个部分：

- **左侧：指定了 Lambda 表达式需要的所有参数**。
- **右侧：指定了 Lambda 体，即 Lambda 表达式要执行的功能**。


### 4）、Lambda  表达式语法格式

- 1、无参数，无返回值：
	 		```() -> System.out.println("Hello Lambda!");```
- 2、有一个参数，并且无返回值：
	 		```(x) -> System.out.println(x)```
- 3、若只有一个参数，小括号可以省略不写：
	 		```x -> System.out.println(x)```
- 4、有两个以上的参数，有返回值，并且 Lambda 体中有多条语句：
```java
		Comparator<Integer> com = (x, y) -> {
			System.out.println("函数式接口");
			return Integer.compare(x, y);
		};
```
- 5、若 Lambda 体中只有一条语句， return 和 大括号 都可以省略不写：
	 		```Comparator<Integer> com = (x, y) -> Integer.compare(x, y);```
- 6、Lambda 表达式的参数列表的数据类型可以省略不写，因为 JVM 编译器可以通过上下文推断出数据类型，即“类型推断”：
	 		```(Integer x, Integer y) -> Integer.compare(x, y);```


### 5）、函数式接口

Lambda 表达式需要“函数式接口”的支持。函数式接口即 **接口中只有一个抽象方法的接口，称为函数式接口**。 可以使用注解 ```@FunctionalInterface``` 修饰，它可以检查是否是函数式接口。函数式接口的使用示例如下所示：

```java
@FunctionalInterface
public interface MyFun {
    public double getValue();
}

@FunctionalInterface
public interface MyFun<T> {
    public T getValue(T t);
}

public static void main(String[] args) {
    String newStr = toUpperString((str)->str.toUpperCase(),"toroot");
    System.out.println(newStr);
}

public static String toUpperString(MyFun<String> mf,String str) {
    return mf.getValue(str);
}
```


### 6）、Java 内置函数式接口

|接口 | 参数 | 返回类型| 示例 |
| ---- | ---- | ---- | ---- |
| Predicate<T> |  T |boolean| 这道题对了吗？|
| Consumer<T>      |       T       |     void           |   输出一个值 |
| Function<T,R>      |      T       |      R         |       获得 Person对象的名字 |
| Supplier<T>            |  None    |     T             |   工厂方法 |
| UnaryOperator<T>      |   T          |  T        |        逻辑非 （!） |
| BinaryOperator<T>    |    (T, T)       |   T         |       求两个数的乘积 （*） |


###  7）、方法引用

当要传递给 Lambda 体的操作，已经有实现的方法了，可以使用方法引用。方法引用使用 **操作符 “ ::” 将方法名和对象或类的名字分隔开来**。它主要有如下 **三种** 使用情况 ：

- **对象 :: 实例方法**
- **类 :: 静态方法**
- **类 ::  实例方法**


### 8）、什么时候可以用 **::** 方法引用（重点）

在我们使用 Lambda 表达式的时候，”->” 右边部分是要执行的代码，即要完成的功能，可以把这部分称作 Lambda 体。有时候，**当我们想要实现一个函数式接口的那个抽象方法，但是已经有类实现了我们想要的功能，这个时候我们就可以用方法引用来直接使用现有类的功能去实现**。示例代码如下所示：

```java
Person p1 = new Person("Av",18,90);
        Person p2 = new Person("King",20,0);
        Person p3 = new Person("Lance",17,100);
        List<Person> list = new ArrayList<>();
        list.add(p1);
        list.add(p2);
        list.add(p3);

        // 这里我们需要比较 list 里面的 person,按照年龄排序
        // 那么我们最常见的做法是
        // sort(List<T> list, Comparator<? super T> c)
        // 1、因为我们的 sort 方法的第二个参数是一个接口，所以我们需要实现一个匿名内部类
        Collections.sort(list, new Comparator<Person>() {
            @Override
            public int compare(Person person1, Person person2) {
                return person1.getAge().compareTo(person2.getAge());
            }
        });
        // 2、因为第二个参数是一个 @FunctionalInterface 的函数式接口，所以我们可以用 lambda 写法
        Collections.sort(list, (person1,person2) -> p1.getScore().compareTo(p2.getAge()));
        // 3、因为第二个参数我们可以用lambda的方式去实现，
        // 但是刚好又有代码 Comparator.comparing 已经实现了这个功能
        // 这个时候我们就可以采用方法引用了
        /**
         * 重点：
         * 当我们想要实现一个函数式接口的那个抽象方法，但是已经有类实现了我们想要的功能，
         * 这个时候我们就可以用方法引用来直接使用现有类的功能去实现。
         */
        Collections.sort(list, Comparator.comparing(Person::getAge));

        System.out.println(list);
```

其它 Java 内置的函数式接口示例如下所示：

```java
public static void main(String[] args) {
    Consumer<String>  c = x->System.out.println(x);
    // 等同于
    Consumer<String> c2 = System.out::print;
}

public static void main(String[] args) {
    BinaryOperator<Double> bo = (n1,n2) ->Math.pow(n1,n2);
    BinaryOperator<Double> bo2 = Math::pow;
}

public static void main(String[] args) {
    BiPredicate<String,String> bp = (str1,str2) ->str1.equals(str2);
    BiPredicate<String,String> bp2 = String::equals;
}
```


> 注意：当需要引用方法的第一个参数是调用对象，并且第二个参数是需要引用方法的第二个参数（或无参数）时，使用 `ClassName::methodName`。

### 9）、构造器引用

格式： `ClassName :: new ` 与函数式接口相结合，自动与函数式接口中方法兼容。
可以把构造器引用赋值给定义的方法，但是构造器参数列表要与接口中抽象方法的参数列表一致。示例如下所示：

```java
public static void main(String[] args) {
    Supplier<Person> x = ()->new Person();
    Supplier<Person> x2 = Person::new;
}

public static void main(String[] args) {
    Function<String,Person> f  = x->new Person(x);
    Function<String,Person> f2 = Person::new;
}
```


### 10）、数组引用 

格式： type[] :: new，示例如下所示：

```java
public static void main(String[] args) {
   Function<Integer,Person[]> f  = x->new Person[x];
   Function<Integer,Person[]>  f2 = Person[]::new;
}
```


## 3、Stream API 

### 1）、Stream 是什么?

**Stream 是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列**。记住：“集合讲的是数据，流讲的是计算！”

### 2）、特点

-  1）、**Stream 自己不会存储元素**。
-  2）、**Stream 不会改变源对象。相反，他们会返回一个持有结果的新 Stream**。
-  3）、**Stream 操作是延迟执行的。这意味着他们会等到需要结果的时候才执行**。


###  3）、一个 Stream 操作实例

取出所有大于18岁人的姓名，按字典排序，并输出到控制台，代码如下所示：

```java
   private static  List<Person> persons = Arrays.asList(
            new Person("CJK",19,"女"),
            new Person("BODUO",20,"女"),
            new Person("JZ",21,"女"),
            new Person("anglebabby",18,"女"),
            new Person("huangxiaoming",5,"男"),
            new Person("ROY",18,"男")
    );
public static void main(String[] args) throws IOException {
 persons.stream().filter(x-    	>x.getAge()>=18).map(Person::getName).sorted().forEach(System.out::println);
}
```


### 4）、Stream 的操作三个步骤 

- 1、**创建 Stream：一个数据源（如：集合、数组），获取一个流**。
- 2、**中间操作：一个中间操作链，对数据源的数据进行处理**。
- 3、**终止操作(终端操作)：一个终止操作，执行中间操作链，并产生结果**。


#### 1、创建 Steam 

创建流主要有四种方式，其示例代码如下所示：

```java
@Test
public void test1(){
    //1. Collection 提供了两个方法  stream() 与 parallelStream()
    List<String> list = new ArrayList<>();
    Stream<String> stream = list.stream(); //获取一个顺序流
    Stream<String> parallelStream = list.parallelStream(); //获取一个并行流

    //2. 通过 Arrays 中的 stream() 获取一个数组流
    Integer[] nums = new Integer[10];
    Stream<Integer> stream1 = Arrays.stream(nums);

    //3. 通过 Stream 类中静态方法 of()
    Stream<Integer> stream2 = Stream.of(1,2,3,4,5,6);
    
    //4. 创建无限流
    //迭代
    Stream<Integer> stream3 = Stream.iterate(0, (x) -> x + 2).limit(10);
    stream3.forEach(System.out::println);

    //生成
    Stream<Double> stream4 = Stream.generate(Math::random).limit(2);
    stream4.forEach(System.out::println);
}
```


#### 2、中间操作

- 1）、**筛选与切片**
	- **filter：接收 Lambda ，从流中排除某些元素**。
	- **limit：截断流，使其元素不超过给定数量**。
	- **skip(n)：跳过元素，返回一个扔掉了前 n 个元素的流。若流中元素不足 n 个，则返回一个空流。与 limit(n) 互补**。
	- **distinct：筛选，通过流所生成元素的 hashCode() 和 equals() 去除重复元素**。
- 2）、**映射**
    - **map：接收 Lambda ，将元素转换成其他形式或提取信息。接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素，类似于 python、go 的 map 语法**。
    - **flatMap：接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流**。
- 3）、**排序**
    - **sorted()：自然排序**。
    - **sorted(Comparator com)：定制排序**。


这里，我们给出一些常见的使用示例，如下所示：

##### 1、有个数组 Integer[] ary = {1,2,3,4,5,6,7,8,9,10}，取出中间的第三到第五个元素。

```java
List<Integer> collect = Arrays.stream(ary).skip(2).limit(3).collect(Collectors.toList());
```


##### 2、有个数组 Integer[] ary = {1,2,2,3,4,5,6,6,7,8,8,9,10}，取出里面的偶数，并去除重复。

```java
List<Integer> list = Arrays.stream(ary).filter(x -> x % 2 == 0).distinct().collect(Collectors.toList());
Set<Integer> integerSet = Arrays.stream(ary).filter(x -> x % 2 == 0).collect(Collectors.toSet());
```


##### 3、有个二维数组，要求把数组组合成一个一维数组，并排序（1，2，3，4，5……12）

Integer[][] ary = {{3,8,4,7,5}, {9,1,6,2}, {0,10,12,11} };

```java
Arrays.stream(ary).flatMap(item->Arrays.stream(item)).sorted().forEach(System.out::println);
```


#### 3）、终止操作 

终端操作会从流的流水线生成结果。其结果可以是任何不是流的值，例如：List、Integer，甚至是 void 。

##### 1、查找与匹配 

|接口 | 说明 |
| ---- | ---- |
| allMatch(Predicate p)   | 检查是否匹配所有元素 |
| anyMatch(Predicate p)  |  检查是否至少匹配一个元素 |
| noneMatch(Predicate p) |  检查是否没有匹配所有元素 |
| findFirst()  | 返回第一个元素 |
| findAny()  | 返回当前流中的任意元素 |
| count()  | 返回流中元素总数 |
| max(Comparator c) | 返回流中最大值 |
| min(Comparator c) |  返回流中最小值 |
| forEach(Consumer c)  | 迭代 |


##### 2、归约 

reduce(T iden, BinaryOperator b) 可以将流中元素反复结合起来，得到一个值。返回 Optional<T>。例如使用 reduce 来求所有人员学生的总分的示例代码如下所示：

```
   Integer all = persons.stream().map(Person::getScore).reduce((integer, integer2) -> integer + integer2).get()
```

##### 3、收集

- **collect(Collector c) 将流转换为其他形式。它接收一个 Collector 接口的实现，用于给 Stream 中元素做汇总的方法**。
- **Collector 接口中方法的实现决定了如何对流执行收集操作(如收集到 List、Set、Map)**。
- **Collectors 实用类提供了很多静态方法，可以方便地创建常见收集器实例**。

收集相关的 Stream API 与其实例代码如下所示：

- 1）、toList List<T> 把流中元素收集到 List：
```
List<Person> emps= list.stream().collect(Collectors.toList());
```
- 2）、toSet Set<T> 把流中元素收集 到Set：
```
Set<Person> emps= list.stream().collect(Collectors.toSet());
```
- 3）、toCollection Collection<T> 把流中元素收集到创建的集合：
```
Collection<Person> emps=list.stream().collect(Collectors.toCollection(ArrayList::new));
```
- 4）、counting Long 计算流中元素的个数：
```
long count = list.stream().collect(Collectors.counting());
```
- 5）、summing Int Integer 对流中元素的整数属性求和：
```
int total=list.stream().collect(Collectors.summingInt(Person::getAge));
```
- 6）、averaging Int Double 计算流中元素 Integer 属性的平均值：
```
double avg= list.stream().collect(Collectors.averagingInt(Person::getAge));
```
- 7）、summarizingInt IntSummaryStatistics 收集流中 Integer 属性的统计值。如平均值：
```
Int SummaryStatisticsiss= list.stream().collect(Collectors.summarizingInt(Person::getAge));
```
- 8）、joining String 连接流中每个字符串：
```
String str= list.stream().map(Person::getName).collect(Collectors.joining());
```
- 9）、maxBy Optional<T> 根据比较器选择最大值：
```
Optional<Person> max= list.stream().collect(Collectors.maxBy(comparingInt(Person::getSalary)));
```
- 10）、minBy Optional<T> 根据比较器选择最小值：
```
Optional<Person> min = list.stream().collect(Collectors.minBy(comparingInt(Person::getSalary)));
```
- 11）、reducing 归约产生的类型，从一个作为累加器的初始值开始，利用 BinaryOperator 与流中元素逐个结合，从而归约成单个值：
```
int total=list.stream().collect(Collectors.reducing(0, Person::getSalary, Integer::sum));
```
- 12）、collectingAndThen 转换函数返回的类型，包裹另一个收集器，对其结果转换函数
```
int how= list.stream().collect(Collectors.collectingAndThen(Collectors.toList(), List::size));
```
- 13）、groupingBy Map<K, List<T>> 根据某属性值对流分组，属性为 K，结果为 V：
```
Map<Person.Status, List<Person>> map= list.stream().collect(Collectors.groupingBy(Person::getStatus));
```
- 14）、partitioningBy Map<Boolean, List<T>> 根 据true 或 false 进行分区：
```
Map<Boolean,List<Person>>vd= list.stream().collect(Collectors.partitioningBy(Person::getManage));
```

##### 4、终止操作练习案例

- 1）、取出Person对象的所有名字，放到 List 集合中：
```
List<String> collect2 = persons.stream().map(Person::getName).collect(Collectors.toList());
```
- 2、求 Person 对象集合的分数的平均分、总分、最高分，最低分，分数的个数：
```
IntSummaryStatistics collect = persons.stream().collect(Collectors.summarizingInt(Person::getScore));
System.out.println(collect);
```
- 3、根据成绩分组，及格的放一组，不及格的放另外一组：
```
Map<Boolean, List<Person>> collect1 = persons.stream().collect(Collectors.partitioningBy(person -> person.getScore() >= 60));
System.out.println(collect1);
```
- 4、统计 aa.txt 里面的单词数：
```java
public static void main(String[] args) throws IOException {
    InputStream resourceAsStream = Person.class.getClassLoader().getResourceAsStream("aa.txt");
    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(resourceAsStream));
    bufferedReader.lines().flatMap(x->Stream.of(x.split(" "))).sorted().collect(Collectors.groupingBy(String::toString)).forEach((a,b)-> System.out.println(a+":"+b.size()));
    bufferedReader.close();
}
```


## 4、复杂泛型

### 1）、泛型是什么？

泛型，即 **“参数化类型”**。就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）。

### 2）、泛型的好处

- **适用于多种数据类型执行相同的代码**。
- **泛型中的类型在使用时指定，不需要强制类型转换**。


### 3）、泛型类和泛型接口

泛型的本质是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。而这种参数类型可以用在类、接口和方法中，分别被称为 **泛型类、泛型接口、泛型方法**。

#### 泛型类

引入一个类型变量T（其他大写字母都可以，不过常用的就是T，E，K，V等等），并且用<>括起来，并放在类名的后面。**泛型类是允许有多个类型变量的**。常见的示例代码如下所示：

```java
public class NormalGeneric<K> {
    private K data;

    public NormalGeneric() {
    }

    public NormalGeneric(K data) {
        this.data = data;
    }

    public K getData() {
        return data;
    }

    public void setData(K data) {
        this.data = data;
    }
}
```

```java
public class NormalGeneric2<T,K> {
    private T data;
    private K result;

    public NormalGeneric2() {
    }

    public NormalGeneric2(T data) {
        this();
        this.data = data;
    }
    
    public NormalGeneric2(T data, K result) {
        this.data = data;
        this.result = result;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public K getResult() {
        return result;
    }

    public void setResult(K result) {
        this.result = result;
    }
}
```


#### 泛型接口

泛型接口与泛型类的定义基本相同。示例代码如下所示：

```java
public interface Genertor<T> {
    public T next();
}
```

但是，**实现泛型接口的类，有两种实现方法**：

##### 1、未传入泛型实参

在 new 出类的实例时，需要指定具体类型：

```java
public class ImplGenertor<T> implements Genertor<T> {
    @Override
    public T next() {
        return null;
    }
}
```

##### 2、传入泛型实参

在 new 出类的实例时，和普通的类没区别。

```java
public class ImplGenertor2 implements Genertor<String> {
    @Override
    public String next() {
        return null;
    }
}
```

#### 泛型方法

泛型方法的 <T> 定义在 **修饰符与返回值** 的中间。示例代码如下所示：

```java
public <T> T genericMethod(T...a){
    return a[a.length/2];
}
```


泛型方法，是在调用方法的时候指明泛型的具体类型，泛型方法可以在任何地方和任何场景中使用，包括普通类和泛型类。

##### 泛型类中定义的普通方法和泛型方法的区别

在普通方法中：

```java
// 虽然在方法中使用了泛型，但是这并不是一个泛型方法。
// 这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
// 所以在这个方法中才可以继续使用 T 这个泛型。
public T getKey(){
    return key;
}
```

在泛型方法中：

```java
/**
 * 这才是一个真正的泛型方法。
 * 首先在 public 与返回值之间的 <T> 必不可少，这表明这是一个泛型方法，并且声明了一个泛型 T
 * 这个 T 可以出现在这个泛型方法的任意位置，泛型的数量也可以为任意多个。
 */
public <T,K> K showKeyName(Generic<T> container){
    // ...
}
```


### 4）、限定类型变量

```java
public class ClassBorder<T extends Comparable> {
...
}

public class GenericRaw<T extends ArrayList&Comparable> {
...
}
```

-` <T extends Comparable>`：**T 表示应该绑定类型的子类型，Comparable 表示绑定类型，子类型和绑定类型可以是类也可以是接口**。
- **extends 左右都允许有多个，如 T,V extends Comparable&Serializable**。
- **注意限定类型中，只允许有一个类，而且如果有类，这个类必须是限定列表的第一个**。
- **限定类型变量既可以用在泛型方法上也可以用在泛型类上**。


### 5）、泛型中的约束和局限性

- 1、不能用基本类型实例化类型参数。
- 2、运行时类型查询只适用于原始类型。
- 3、泛型类的静态上下文中类型变量失效：不能在静态域或方法中引用类型变量。因为泛型是要在对象创建的时候才知道是什么类型的，而对象创建的代码执行先后顺序是 static 的部分，然后才是构造函数等等。所以在对象初始化之前 static 的部分已经执行了，如果你在静态部分引用泛型，那么毫无疑问虚拟机根本不知道是什么东西，因为这个时候类还没有初始化。
- 4、不能创建参数化类型的数组，但是可以定义参数化类型的数组。
- 5、不能实例化类型变量。
- 6、不能使用 try-catch 捕获泛型类的实例。


### 6）、泛型类型的继承规则

泛型类可以继承或者扩展其他泛型类，比如 List 和 ArrayList：

```java
private static class ExtendPair<T> extends Pair<T>{
...
}
```


### 7）、通配符类型

- `？extends X`：**表示类型的上界，类型参数是 X 的子类**。
- `？super X`：**表示类型的下界，类型参数是 X 的超类**。


#### ？extends X

如果其中提供了 get 和 set 类型参数变量的方法的话，set 方法是不允许被调用的，会出现编译错误，而 get 方法则没问题。

？extends X 表示类型的上界，类型参数是 X 的子类，那么可以肯定的说，get 方法返回的一定是个 X（不管是 X 或者 X 的子类）编译器是可以确定知道的。但是 set 方法只知道传入的是个 X，至于具体是 X 的哪个子类，是不知道的。

因此，**？extends X 主要用于安全地访问数据，可以访问 X 及其子类型，并且不能写入非 null 的数据**。


#### ？super X

如果其中提供了 get 和 set 类型参数变量的方法的话，set 方法可以被调用，且能传入的参数只能是 X 或者 X 的子类。而 get 方法只会返回一个 Object 类型的值。

？ super  X  表示类型的下界，类型参数是 X 的超类（包括 X 本身），那么可以肯定的说，get 方法返回的一定是个 X 的超类，那么到底是哪个超类？不知道，但是可以肯定的说，Object 一定是它的超类，所以 get 方法返回 Object。编译器是可以确定知道的。对于 set 方法来说，编译器不知道它需要的确切类型，但是 X 和 X 的子类可以安全的转型为 X。

因此，**？super X 主要用于安全地写入数据，可以写入 X 及其子类型**。


#### 无限定的通配符 ?

**表示对类型没有什么限制，可以把 ？看成所有类型的父类，如 ArrayList<?>**。


### 8）、虚拟机是如何实现泛型的？

泛型思想早在 C++ 语言的模板（Template）中就开始生根发芽，**在 Java 语言处于还没有出现泛型的版本时，只能通过 Object 是所有类型的父类和类型强制转换两个特点的配合来实现类型泛化**。

由于 Java 语言里面所有的类型都继承于 java.lang.Object，所以 Object 转型成任何对象都是有可能的。但是也因为有无限的可能性，就只有程序员和运行期的虚拟机才知道这个 Object 到底是个什么类型的对象。在编译期间，编译器无法检查这个 Object 的强制转型是否成功，如果仅仅依赖程序员去保障这项操作的正确性，许多 ClassCastException 的风险就会转嫁到程序运行期之中。

此外，泛型技术在 C#/C++ 和 Java 之中的使用方式看似相同，但实现上却有着根本性的分歧，C# 里面的泛型无论在程序源码中、编译后的 IL 中（Intermediate Language，中间语言，这时候泛型是一个占位符），或是运行期的 CLR 中，都是切实存在的，**List＜int＞ 与 List＜String＞ 就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为类型膨胀，基于这种方法实现的泛型称为真实泛型**。

而 **Java 语言中的泛型则不一样，它只在程序源码中存在，在编译后的字节码文件中，就已经替换为原来的原生类型（Raw Type，也称为裸类型）了，并且在相应的地方插入了强制转型代码**，因此，对于运行期的 Java 语言来说，ArrayList＜int＞ 与 ArrayList＜String＞ 就是同一个类，所以 **泛型技术实际上是 Java 语言的一颗语法糖，Java 语言中的泛型实现方法称为类型擦除，基于这种方法实现的泛型称为伪泛型**。
将一段 Java 代码编译成 Class 文件，然后再用字节码反编译工具进行反编译后，将会发现泛型都不见了，程序又变回了 Java 泛型出现之前的写法，泛型类型都变回了原生类型。

由于 Java 泛型的引入，各种场景（虚拟机解析、反射等）下的方法调用都可能对原有的基础产生影响和新的需求，如在泛型类中如何获取传入的参数化类型等。因此，JCP 组织对虚拟机规范做出了相应的修改，引入了诸如 Signature、LocalVariableTypeTable 等新的属性用于解决伴随泛型而来的参数类型的识别问题，**Signature 是其中最重要的一项属性，它的作用就是存储一个方法在字节码层面的特征签名，这个属性中保存的参数类型并不是原生类型，而是包括了参数化类型的信息**。修改后的虚拟机规范要求所有能识别 49.0 以上版本的 Class 文件的虚拟机都要能正确地识别 Signature 参数。

最后，**从 Signature 属性的出现我们还可以得出结论，擦除法所谓的擦除，仅仅是对方法的 Code 属性中的字节码进行擦除，实际上元数据中还是保留了泛型信息，这也是我们能通过反射手段取得参数化类型的根本依据**。



# 二、初识 ByteX 


ByteX 使用了纯 Java 来编写源码，它是一个基于 Gradle transform api 和 ASM 的字节码插桩平台。


> 调试:gradle clean :example:assembleRelease -Dorg.gradle.debug=true --no-daemon


## 1、优势

- 1）、**自动集成到其它宿主和插件一起整合为一个单独的 MainTransformFlow，结合 class 文件多线程并发处理，避免了打包的额外时间呈线性增长**。
- 2）、**插件、宿主之间完全解耦，便于协同开发**。
- 3）、**common module 提供通用的代码复用，每个插件只需专注自身的字节码插桩逻辑**。


## 2、MainTransformFlow 基本流程

在 MainTransformFlow implements MainProcessHandler
常规处理过程，会遍历两次工程构建中的所有 class。

- 1）、第一次，遍历 traverse 与 traverseAndroidJar 过程，以形成完整的类图。
- 2）、第二次，执行 transform：再遍历一次工程中所有的构建产物，并对 class 文件做处理后输出。


## 3、如何自定义独立的 TransformFlow？

重写 IPlugin 的 provideTransformFlow 即可。


## 4、类图对象

context.getClassGraph() 获取类图对象，两个 TransformFlow 的类图是隔离的。


## 5、MainProcessHandler

- 通过复写 process 方法，注册自己的 FlieProcessor 来处理。
- FileProcessor 采用了责任链模式，每个 class 文件都会流经一系列的 FileProcessor 来处理。


## 6、IPlugin.hookTransformName()

**使用 反射 Hook 方式 将 Transform 注册到 proguard 之后**。


# 三、ByteX 插件平台构建流程探秘

添加 apply plugin: 'bytex' 之后，bytex 可以在 Gradle 的构建流程中起作用了。这里的插件 id 为 bytex，我们找到 bytex.properties 文件，查看里面映射的实现类，如下所示：

```java
implementation-class=com.ss.android.ugc.bytex.base.ByteXPlugin
```


可以看到，bytex 的实现类为 ByteXPlugin，其源码如下所示：

```java
public class ByteXPlugin implements Plugin<Project> {
    @Override
    public void apply(@NotNull Project project) {
        // 1
        AppExtension android = project.getExtensions().getByType(AppExtension.class);
        // 2
        ByteXExtension extension = project.getExtensions().create("ByteX", ByteXExtension.class);
        // 3
        android.registerTransform(new ByteXTransform(new Context(project, android, extension)));
    }
}
```


首先，注释1处，获取 Android 为 App 提供的扩展属性 AppExtension 实例。然后，在注释2处，获取 ByteX 自身创建的扩展属性 ByteXExtension 实例。最后，在注释3处，注册 ByteXTransform 实例。ByteXTransform 继承了抽象类 CommonTransform，其实现了关键的 transform 方法，其实现源码如下所示：

```java
@Override
public final void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
    super.transform(transformInvocation);
    // 1、如果不是增量模式，则清楚输出目录的文件。
    if (!transformInvocation.isIncremental()) {
        transformInvocation.getOutputProvider().deleteAll();
    }
    // 2、获取 transformContext 实例。
    TransformContext transformContext = getTransformContext(transformInvocation);
    // 3、初始化 HtmlReporter（生成 ByteX 构建产生日志的 HTML 文件）
    init(transformContext);
    // 4、过滤掉没有打开插件开关的 plugin。
    List<IPlugin> plugins = getPlugins().stream().filter(p -> p.enable(transformContext)).collect(Collectors.toList());
    Timer timer = new Timer();
    // 5、创建一个 transformEngine 实例。
    TransformEngine transformEngine = new TransformEngine(transformContext);
    try {
        if (!plugins.isEmpty()) {
            // 6、使用 PriorityQueue 对每一个 TransformFlow 进行优先级排序（在这里添加的是与之对应的实现类 MainTransformFlow）。
            Queue<TransformFlow> flowSet = new PriorityQueue<>((o1, o2) -> o2.getPriority() - o1.getPriority());
            MainTransformFlow commonFlow = new MainTransformFlow(transformEngine);
              flowSet.add(commonFlow);
            for (int i = 0; i < plugins.size(); i++) {
                // 7、给每一个 Plugin 注册 MainTransformFlow，其实质是将每一个 Plugin 的 MainProcessHandler 添加到 MainTransformFlow 中的 handlers 列表中。
                IPlugin plugin = plugins.get(i);
                TransformFlow flow = plugin.registerTransformFlow(commonFlow, transformContext);
                if (!flowSet.contains(flow)) {
                    flowSet.add(flow);
                }
            }
            while (!flowSet.isEmpty()) {
                TransformFlow flow = flowSet.poll();
                if (flow != null) {
                    if (flowSet.size() == 0) {
                        flow.asTail();
                    }
                    // 8、按指定优先级执行每一个 TransformFlow 的 run 方法，默认只有一个 MainTransformFlow 实例。
                    flow.run();
                    // 9、获取流中的 graph 类图对象并清除。
                    Graph graph = flow.getClassGraph();
                    if (graph != null) {
                        //clear the class diagram.we won’t use it anymore
                        graph.clear();
                    }
                }
            }
        } else {
            transformEngine.skip();
        }
        // 10
        afterTransform(transformInvocation);
    } catch (Throwable throwable) {
        LevelLog.sDefaultLogger.e(throwable.getClass().getName(), throwable);
        throw throwable;
    } finally {
        for (IPlugin plugin : plugins) {
            try {
                plugin.afterExecute();
            } catch (Throwable throwable) {
                LevelLog.sDefaultLogger.e("do afterExecute", throwable);
            }
        }
        transformContext.release();
        release();
        timer.record("Total cost time = [%s ms]");
        if (BooleanProperty.ENABLE_HTML_LOG.value()) {
            HtmlReporter.getInstance().createHtmlReporter(getName());
            HtmlReporter.getInstance().reset();
        }
    }
}
```


在注释7处，调用了 plugin.registerTransformFlow 方法，其源码如下所示：

```java
@Nonnull
@Override
public final TransformFlow registerTransformFlow(@Nonnull MainTransformFlow mainFlow, @Nonnull TransformContext transformContext) {
    if (transformFlow == null) {
        transformFlow = provideTransformFlow(mainFlow, transformContext);
        if (transformFlow == null) {
            throw new RuntimeException("TransformFlow can not be null.");
        }
    }
    return transformFlow;
}
```


这里继续调用了 provideTransformFlow 方法，其源码如下所示：

```java
/**
 * create a new transformFlow or just return mainFlow and append a handler.
 * It will be called by {@link IPlugin#registerTransformFlow(MainTransformFlow, TransformContext)} when
 * handle start.
 *
 * @param mainFlow         main TransformFlow
 * @param transformContext handle context
 * @return return a new TransformFlow object if you want make a new flow for current plugin
 */
protected TransformFlow provideTransformFlow(@Nonnull MainTransformFlow mainFlow, @Nonnull TransformContext transformContext) {
    return mainFlow.appendHandler(this);
}
```


可以看到，通过调用 mainFlow.appendHandler(this) 方法将每一个 Plugin 的 MainProcessHandler 添加到 MainTransformFlow 中的 handlers 列表之中。

在注释8处，按指定优先级执行了每一个 TransformFlow 的 run 方法，默认只有一个 MainTransformFlow 实例。我们看到了 MianTransformFlow 的 run 方法：

```java
@Override
public void run() throws IOException, InterruptedException {
    try {
        // 1
        beginRun();
        // 2
        runTransform();
    } finally {
        // 3
        endRun();
    }
}
```


首先，在注释1出，调用了 beginRun 方法，其实现如下：

```java
// AbsTransformFlow
protected void beginRun() {
    transformEngine.beginRun();
}

// TransformEngine
public void beginRun(){
    context.markRunningState(false);
}

// TransformContext
private final AtomicBoolean running = new AtomicBoolean(false);

void markRunningState(boolean running) {
    this.running.set(running);
}
```


最后，在 TransformContext 实例中使用了一个 AtomicBoolean 实例标记 MainTransformFlow 是否正在运行中。

然后，在注释2处执行了 runTransform 方法，这里就是真正执行 transform 的地方，其源码如下所示：

```java
private void runTransform() throws IOException, InterruptedException {
    if (handlers.isEmpty()) return;
    Timer timer = new Timer();
    timer.startRecord("PRE_PROCESS");
    timer.startRecord("INIT");
    // 1、初始化 handlers 列表中的每一个 handler。
    for (MainProcessHandler handler : handlers) {
        handler.init(transformEngine);
    }
    timer.stopRecord("INIT", "Process init cost time = [%s ms]");
    // 如果不是 跳过 traverse 仅仅只执行 Transform 方法时，才执行 traverse 过程。
    if (!isOnePassEnough()) {
        if (!handlers.isEmpty() && context.isIncremental()) {
            timer.startRecord("TRAVERSE_INCREMENTAL");
            // 2、如果是 增量模式，则执行 traverseArtifactOnly（仅仅增量遍历产物）调用每一个 plugin 的对应的 MainProcessHandler 的 traverseIncremental 方法。这里最终会调用 ClassFileAnalyzer.handle 方法进行遍历分发操作。
            traverseArtifactOnly(getProcessors(Process.TRAVERSE_INCREMENTAL, new ClassFileAnalyzer(context, Process.TRAVERSE_INCREMENTAL, null, handlers)));
            timer.stopRecord("TRAVERSE_INCREMENTAL", "Process project all .class files cost time = [%s ms]");
        }
        handlers.forEach(plugin -> plugin.beforeTraverse(transformEngine));
        timer.startRecord("LOADCACHE");
        // 3、创建一个 CachedGraphBuilder 对象：能够缓存 类图 的 类图构建者对象。
        GraphBuilder graphBuilder = new CachedGraphBuilder(context.getGraphCache(), context.isIncremental(), context.shouldSaveCache());
        if (context.isIncremental() && !graphBuilder.isCacheValid()) {
            // 4、如果是增量更新 && graphBuilder 的缓存失效则直接请求非增量运行。
            context.requestNotIncremental();
        }
        timer.stopRecord("LOADCACHE", "Process loading cache cost time = [%s ms]");
        // 5、内部会调用 running.set(true) 来标记正在运行的状态。
        running();
        if (!handlers.isEmpty()) {
            timer.startRecord("PROJECT_CLASS");
            // 6、执行 traverseArtifactOnly（遍历产物）调用每一个 plugin 的对应的 MainProcessHandler 的 traverse 方法，这里最终会调用 ClassFileAnalyzer.handle 方法进行遍历分发操作。
            traverseArtifactOnly(getProcessors(Process.TRAVERSE, new ClassFileAnalyzer(context, Process.TRAVERSE, graphBuilder, handlers)));
            timer.stopRecord("PROJECT_CLASS", "Process project all .class files cost time = [%s ms]");
        }
        if (!handlers.isEmpty()) {
            timer.startRecord("ANDROID");
            // 7、仅仅遍历 Android.jar
            traverseAndroidJarOnly(getProcessors(Process.TRAVERSE_ANDROID, new ClassFileAnalyzer(context, Process.TRAVERSE_ANDROID, graphBuilder, handlers)));
            timer.stopRecord("ANDROID", "Process android jar cost time = [%s ms]");
        }
        timer.startRecord("SAVECACHE");
        // 8、构建 mClassGraph 类图实例。
        mClassGraph = graphBuilder.build();
        timer.stopRecord("SAVECACHE", "Process saving cache cost time = [%s ms]");
    }
    timer.stopRecord("PRE_PROCESS", "Collect info cost time = [%s ms]");
    if (!handlers.isEmpty()) {
        timer.startRecord("PROCESS");
        // 9、遍历执行每一个 plugin 的 transform 方法。
        transform(getProcessors(Process.TRANSFORM, new ClassFileTransformer(handlers, needPreVerify(), needVerify())));
        timer.stopRecord("PROCESS", "Transform cost time = [%s ms]");
    }
}
```


首先，在注释1处，遍历调用了每一个 MainProcessHandler 的 init 方法，它是用于 transform 开始前的初始化实现方法。

MainProcessHandler 接口的 init 方法是一个 default 方法，里面直接调用了每一个 pluign 实现的 init 方法（如果 plugin 没有实现，则仅仅调用 CommonPlugin 的实现的 init 方法：这里通常是用于把不需要处理的文件添加到 mWhiteList 列表），这里可以做一些plugin 的准备工作。


## 1、仅仅遍历产物

```java
traverseArtifactOnly(getProcessors(Process.TRAVERSE, new ClassFileAnalyzer(context, Process.TRAVERSE, graphBuilder, handlers)));
```

getProcessors 方法的源码如下所示：

```java
private FileProcessor[] getProcessors(Process process, FileHandler fileHandler) {
    List<FileProcessor> processors = handlers.stream()
            .flatMap((Function<MainProcessHandler, Stream<FileProcessor>>) handler -> handler.process(process).stream())
            .collect(Collectors.toList());
    switch (process) {
        case TRAVERSE_INCREMENTAL:
            processors.add(0, new FilterFileProcessor(fileData -> fileData.getStatus() != Status.NOTCHANGED));
            processors.add(new IncrementalFileProcessor(handlers, ClassFileProcessor.newInstance(fileHandler)));
            break;
        case TRAVERSE:
        case TRAVERSE_ANDROID:
        case TRANSFORM:
            processors.add(ClassFileProcessor.newInstance(fileHandler));
            processors.add(0, new FilterFileProcessor(fileData -> fileData.getStatus() != Status.NOTCHANGED && fileData.getStatus() != Status.REMOVED))
            break;
        default:
            throw new RuntimeException("Unknow Process:" + process);
    }
    return processors.toArray(new FileProcessor[0]);
}
```

这里的 processor 的添加由 增量 进行界定，具体的处理标准如下：

- `TRAVERSE_INCREMENTAL`：
    - `FilterFileProcessor`：**按照不同的过程过滤掉不需要的 FileData**。
    - `IncrementalFileProcessor`：**用于进行 增量文件的处理**。
- `TRAVERSE/TRAVERSE_ANDROID/TRANSFORM`:
    - `FilterFileProcessor`：**按照不同的过程过滤掉不需要的 FileData**。
    - `ClassFileProcessor`：**用于处理 .class 文件**。
    

## 2、仅仅遍历 Android Jar 包

```java
traverseAndroidJarOnly(getProcessors(Process.TRAVERSE_ANDROID, new ClassFileAnalyzer(context, Process.TRAVERSE_ANDROID, graphBuilder, handlers)));
```


## 3、构建 mClassGraph 类图对象

```java
mClassGraph = graphBuilder.build();
```


## 4、执行 Transform

```java
transform(getProcessors(Process.TRANSFORM, new ClassFileTransformer(handlers, needPreVerify(), needVerify())));
```


transform 的源码如下所示：

```
// AbsTransformFlow 类中
protected AbsTransformFlow transform(FileProcessor... processors) throws IOException, InterruptedException {
        beforeTransform(transformEngine);
        transformEngine.transform(isLast, processors);
        afterTransform(transformEngine);
        return this;
    }
```


### 1）、beforeTransform(transformEngine)

```java
// MainTransformFlow
@Override
protected AbsTransformFlow beforeTransform(TransformEngine transformEngine) {
    // 1
    handlers.forEach(plugin -> plugin.beforeTransform(transformEngine));
    return this;
}
```


注释1处，遍历执行 每一个 plugin 的 beforeTransform 方法做一些自身 transform 前的准备工作。


### 2）、transformEngine.transform(isLast, processors)

```
// TranformEngine
public void transform(boolean isLast, FileProcessor... processors) {
    Schedulers.FORKJOINPOOL().invoke(new PerformTransformTask(context.allFiles(), getProcessorList(processors), isLast, context));
}
```


Shedulers.FORKJOINPOOL() 方法的源码如下所示：

```java
public class Schedulers {
    private static final int cpuCount = Runtime.getRuntime().availableProcessors();
    private final static ExecutorService IO = new ThreadPoolExecutor(0, cpuCount * 3,
            30L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>());

    // 1
    private static final ExecutorService COMPUTATION = Executors.newWorkStealingPool(cpuCount);

    public static Worker IO() {
        return new Worker(IO);
    }

    public static Worker COMPUTATION() {
        return new Worker(COMPUTATION);
    }

    public static ForkJoinPool FORKJOINPOOL() {
        return (ForkJoinPool) COMPUTATION;
    }
}
```


可以看到，最终是执行 Executors.newWorkStealingPool(cpuCount) 方法生成了一个 ForkJoinPool 实例。

ForkJoinPool 与 ThreadPoolExecutor 是属于平级关系，ForkJoinPool 线程池是为了实现“分治法”这一思想而创建的，通过把大任务拆分成小任务，然后再把小任务的结果汇总起来就是最终的结果，和 MapReduce 的思想很类似。除了“分治法”之外，ForkJoinPool 还使用了工作窃取算法，即所有线程均尝试找到并执行已提交的任务，或是通过其他任务创建的子任务。有了它我们就可以尽量避免一个线程执行完自己的任务后“无所事事”的情况。

然后这里会回调 PerformTransformTask 实例的 compute 方法，源码如下所示：

```java
@Override
protected void compute() {
    if (outputFile) {
        // 1、如果是最后一个 TransformFlow，则递归调用所有的 FileTransformTask。
        List<FileTransformTask> tasks = source.map(cache -> new FileTransformTask(context, cache, processors)).collect(Collectors.toList());
        // 2、对于Fork/Join模式，假如Pool里面线程数量是固定的，那么调用子任务的fork方法相当于A先分工给B，然后A当监工不干活，B去完成A交代的任务。所以上面的模式相当于浪费了一个线程。那么如果使用invokeAll相当于A分工给B后，A和B都去完成工作。这样可以更好的利用线程池，缩短执行的时间。
        invokeAll(tasks);
    } else {
        // 3、、递归调用 FileTransformTask
        PerformTraverseTask traverseTask = new PerformTraverseTask(source, processors);
        invokeAll(traverseTask);
    }
}
```


在注释1处，如果是最后一个 TransformFlow，则调用所有的 FileTransformTask。注释2处，对于 Fork/Join 模式，假如 Pool 里面线程数量是固定的，那么调用子任务的 fork 方法相当于 A 先分工给 B，然后 A 当监工不干活，B 去完成 A 交代的任务。所以上面的模式相当于浪费了一个线程。那么如果使用 invokeAll 相当于 A 分工给 B 后，A 和 B 都去完成工作。这样可以更好的利用线程池，缩短执行的时间。注释3处，执行了 ForkJoinTask 的 invokeAll 方法，这里便会回调 compute 方法，源码如下所示：

```java
@Override
protected void compute() {
    List<FileTraverseTask> tasks = source.map(cache -> new FileTraverseTask(cache, processors)).collect(Collectors.toList());
    // 1
    invokeAll(tasks);
}
```


注释1处，继续回调所有的 FileTraverseTask 实例的 compute 方法，源码如下所示：

```java
@Override
protected void compute() {
    List<TraverseTask> tasks = fileCache.stream().map(file -> new TraverseTask(fileCache, file, processors))
            .toList().blockingGet();
    // 1
    invokeAll(tasks);
}
```


注释1处，继续回调所有的 TraverseTask 实例的 compute 方法，源码如下所示：

```java
@Override
protected void compute() {
    try {
        Input input = new Input(fileCache.getContent(), file);
        ProcessorChain chain = new ProcessorChain(processors, input, 0);
        // 1、调用 ProcessorChain 的 proceed 方法。
        chain.proceed(input);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```


注释1处，调用了 ProcessorChain 的 proceed 方法。源码如下所示：

```java
@Override
public Output proceed(Input input) throws IOException {
    if (index >= processors.size()) throw new AssertionError();
    // 1
    FileProcessor next = processors.get(index);
    
    return next.process(new ProcessorChain(processors, input, index + 1));
}
```


注释1处，会从 processors 处理器列表中获取第一个处理器—FilterFileProcessor，并调用它的 process 方法，源码如下所示：

```java
@Override
public Output process(Chain chain) throws IOException {
    Input input = chain.input();
    if (predicate.test(input.getFileData())) {
        // 1
        return chain.proceed(input);
    } else {
        return new Output(input.getFileData());
    }
}
```


注释1处，如果有 FileData 的话，则继续调用 chain 的 proceed 方法，内部会继续调用 ClassFileProcessor 的 process 方法，源码如下：

```java
@Override
public Output process(Chain chain) throws IOException {
    Input input = chain.input();
    FileData fileData = input.getFileData();
    if (fileData.getRelativePath().endsWith(".class")) {
        // 1、如果 fileData 是 .class 文件,则调用 ClassFileTransformer 的 handle 方法进行处理。
        handler.handle(fileData);
    }
    // 2、
    return chain.proceed(input);
}
```


注释1处，如果 fileData 是 .class 文件,则调用 ClassFileTransformer 的 handle 方法进行处理。其源码如下所示：


```java
@Override
public void handle(FileData fileData) {
    try {
        byte[] raw = fileData.getBytes();
        String relativePath = fileData.getRelativePath();
        int cwFlags = 0;  //compute nothing
        int crFlags = 0;
        for (MainProcessHandler handler : handlers) {
            // 1、设置 ClassWrite 的 flag 的默认值为 ClassWriter.COMPUTE_MAXS。
            cwFlags |= handler.flagForClassWriter();
            if ((handler.flagForClassReader(Process.TRANSFORM) & ClassReader.EXPAND_FRAMES) == ClassReader.EXPAND_FRAMES) {
                crFlags |= ClassReader.EXPAND_FRAMES;
            }
        }
        ClassReader cr = new ClassReader(raw);
        ClassWriter cw = new ClassWriter(cwFlags);
        ClassVisitorChain chain = getClassVisitorChain(relativePath);
        if (needPreVerify) {
            // 2、如果需要预校验，则将责任链表头尾部设置为 AsmVerifyClassVisitor 实例。
            chain.connect(new AsmVerifyClassVisitor());
        }
        if (handlers != null && !handlers.isEmpty()) {
            for (MainProcessHandler handler : handlers) {
                // 3、遍历执行所有 plugin 的 transform。其内部会使用 chain.connect(new ReferCheckClassVisitor(context)) 的方式 将
                if (!handler.transform(relativePath, chain)) {
                    fileData.delete();
                    return;
                }
            }
        }
        // 4、兼容 ClassNode 处理的模式
        ClassNode cn = new SafeClassNode();
        chain.append(cn);
        chain.accept(cr, crFlags);
        for (MainProcessHandler handler : handlers) {
            if (!handler.transform(relativePath, cn)) {
                fileData.delete();
                return;
            }
        }
        cn.accept(cw);
        if (!GlobalWhiteListManager.INSTANCE.shouldIgnore(fileData.getRelativePath())) {
            raw = cw.toByteArray();
            if (needVerify) {
                ClassNode verifyNode = new ClassNode();
                new ClassReader(raw).accept(verifyNode, crFlags);
                AsmVerifier.verify(verifyNode);
            }
            // 5、如果不是白名单里的文件，则将 ClassWriter 中的数据放入 fileData 之中。
            fileData.setBytes(raw);
        }
    } catch (ByteXException e) {
        throw e;
    } catch (Exception e) {
        LevelLog.sDefaultLogger.e(String.format("Failed to handle class %s", fileData.getRelativePath()), e);
        if (!GlobalWhiteListManager.INSTANCE.shouldIgnore(fileData.getRelativePath())) {
            throw e;
        }
    }
}
```


在 ClassFileProcessor 的 process 方法的注释2处，又会继续调用 ProcessorChain 的 proceed 方法，这里会回调 BackupFileProcessor 实例的 process 方法，源码如下所示：

```java
@Override
public Output process(Chain chain) throws IOException {
    Input input = chain.input();
    // 仅仅是返回处理过的输出文件
    return new Output(input.getFileData());
}
```


按照这样的模式，PerformTraverseTask 实例的所有 task 都被遍历执行了。

最后，便会调用 MainTransformFlow 实例的 afterTransform 方法，源码如下：

```java
@Override
protected AbsTransformFlow afterTransform(TransformEngine transformEngine) {
    handlers.forEach(plugin -> plugin.afterTransform(transformEngine));
    return this;
}
```


这里遍历执行了所有 plugin 的 afterTransform 方法。

然后，我们再回到 CommonTransform 的 transform 方法，在执行完 MainTransformFlow 的 run 方法后，便会调用注释9处的代码来获取流中的 graph 类图对象并清除。最后，执行注释10处的 afterTransform 方法用来做 transform 之后的收尾工作。


# 四、总结

在本文中，我们一起对 ByteX 插件平台的构建流程进行了探秘。从 ByteX 的源码实现中，我们可以看出作者对 函数式编程、Java 1.8 Lambda 表达式、Java 1.8 Stream API、复杂泛型 等技术的灵活运用，所以，**几乎所有看似很 🐂 的轮子，其实质都是依赖于对基础技术的深度掌握**。那么，**如何才能达到深度掌握基础技术的程度呢？— 唯有不断地练习与有规律的复习**。


# 公众号

我的公众号 `JsonChao` 开通啦，欢迎关注~

![](https://user-gold-cdn.xitu.io/2020/6/11/172a29b8b626ef93?w=258&h=258&f=jpeg&s=28705)


# 参考链接：
---
- 1、[ByteX](https://github.com/bytedance/ByteX)

- 2、[高并发之 Fork/Join 框架使用及注意事项](https://zhuanlan.zhihu.com/p/101418412)

- 3、《深入理解 JVM》

- 4、《Java 编程思想》


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **由于微信群已超过 200 人，麻烦大家想进微信群的朋友们，加我微信拉你进群。**


##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。

















