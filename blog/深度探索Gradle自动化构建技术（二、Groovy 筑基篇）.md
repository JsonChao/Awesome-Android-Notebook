---

		title:  深入探索Gradle自动化构建技术（二、Groovy 筑基篇）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的 [知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

Groovy 作为 Gradle 这一强大构建工具的核心语言，其重要性不言而喻，但是 Groovy 本身是十分复杂的，要想全面地掌握它，我想几十篇万字长文也无法将其彻底描述。所幸的是，在 Gradle 领域中涉及的 Groovy 知识都是非常基础的，因此，本篇文章的目的是为了在后续深入探索 Gradle 时做好一定的基础储备。

# 一、DSL 初识

**DSL（domain specific language）**，即领域特定语言，例如：**Matliba、UML、HTML、XML** 等等 DSL 语言。可以这样理解，**Groovy** 就是 **DSL** 的一个分支。

## 特点

- 1）、<span style="color:#0e88eb;font-weight:bold;">解决特定领域的专有问题。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">它与系统编程语言走的是两个极端，系统编程语言是希望解决所有的问题，比如 Java 语言希望能做 Android 开发，又希望能做服务器开发，它具有横向扩展的特性。而 DSL 具有纵向深入解决特定领域专有问题的特性。</span>


总的来说，DSL 的 **核心思想** 就是：<span style="color:rgb(248,57,41);font-weight:bold;">“求专不求全，解决特定领域的问题”</span>。

# 二、Groovy 初识

## 1、Groovy 的特点

Groovy 的特点具有如下 <span style="color:#773098;font-weight:bold;">三点：</span>


- 1）、<span style="color:#0e88eb;font-weight:bold;">Groovy 是一种基于 JVM 的敏捷开发语言。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">Groovy 结合了 Python、Ruby 和 Smalltalk 众多脚本语言的许多强大的特性。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">Groovy 可以与 Java 完美结合，而且可以使用 Java 所有的库。</span>


### 那么，在已经有了其它脚本语言的前提下，为什么还要制造出 Grvooy 语言呢？

因为 Groovy 语言相较其它编程语言而言，其 **入门的学习成本是非常低的**，因为它的语法就是对 Java 的扩展，所以，我们可以用学习 Java 的方式去学习 Groovy。


## 2、Groovy 语言本身的特性

其特性主要有如下 <span style="color:#773098;font-weight:bold;">三种：</span>


- 1）、<span style="color:#0e88eb;font-weight:bold;">语法上支持动态类型，闭包等新一代语言特性。并且，Groovy 语言的闭包比其它所有语言类型的闭包都要强大。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">它可以无缝集成所有已经存在的 Java 类库，因为它是基于 JVM 的。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">它即可以支持面向对象编程（基于 Java 的扩展），也可以支持面向过程编程（基于众多脚本语言的结合）。</span>


需要注意的是，**在我们使用 Groovy 进行 Gradle 脚本编写的时候，都是使用的面向过程进行编程的**。


## 3、Groovy 的优势

Groovy 的优势有如下 <span style="color:#773098;font-weight:bold;">四种：</span>

- 1）、<span style="color:#0e88eb;font-weight:bold;">它是一种更加敏捷的编程语言：在语法上构建除了非常多的语法糖，许多在 Java 层需要写的代码，在 Groovy 中是可以省略的。因此，我们可以用更少的代码实现更多的功能。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">入门简单，但功能非常强大。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">既可以作为编程语言也可以作为脚本语言</span>
- 4）、<span style="color:#0e88eb;font-weight:bold;">熟悉掌握 Java 的同学会非常容易掌握 Groovy。</span>


## 4、Groovy 包的结构

> [Groovy 官方网址](http://www.groovy-lang.org/download.html)

从官网下载好 **Groovy** 文件之后,我们就可以看到 **Groovy 的目录结构**，其中我们需要 **重点关注 bin 和 doc 这个两个文件夹**。

### bin 文件夹

**bin** 文件夹的内容如下所示：


![](https://imgkr.cn-bj.ufileos.com/d9d64617-362c-4f6d-bcee-64537a518085.png)


这里我们了解下三个重要的可执行命令文件，如下所示：

- 1）、<span style="color:#0e88eb;font-weight:bold;">groovy 命令类似于 Java 中的 java 命令，用于执行 groovy Class 字节码文件。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">groovyc 命令类似于 Java 中的 javac 命令，用于将 groovy 源文件编译成 groovy 字节码文件。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">groovysh 命令是用来解释执行 groovy 脚本文件的。</span>


### doc 文件夹

在 **doc** 文件夹的下面有一个 **html** 文件，其中的内容如下所示：


![](https://imgkr.cn-bj.ufileos.com/fa0579b0-6a47-4f26-867f-bc5438502fea.png)


这里的 api 和 documentation 是我们需要重点关注的，其作用分别如下所示：

- <span style="color:rgb(248,57,41);font-weight:bold;">api：</span><span style="color:#0e88eb;font-weight:bold;">groovy 中为我们提供的一系列 API 及其 说明文档。</span>
- <span style="color:rgb(248,57,41);font-weight:bold;">documentation：</span><span style="color:#0e88eb;font-weight:bold;">groovy 官方为我们提供的一些教程。</span>


## 5、Groovy 中的关键字

下面是 Groovy 中所有的关键字，命名时尤其需要注意，如下所示：

```groovy
as、assert、break、case、catch、class、const、continue、def、default、
do、else、enum、extends、false、finally、for、goto、if、implements、
import、in、instanceof、interface、new、null、package、return、super、
switch、this、throw、throws、trait、true、try、while
```

## 6、Groovy && Java 差异学习

### 1）、getter / setter

对于每一个 field，Groovy 都会⾃动创建其与之对应的 getter 与 setter 方法，从外部可以直接调用它，并且 **在使⽤ object.fieldA 来获取值或者使用 object.fieldA = value 来赋值的时候，实际上会自动转而调⽤ object.getFieldA() 和 object.setFieldA(value) 方法**。

**如果我们不想调用这个特殊的 getter 方法时则可以使用 .@ 直接域访问操作符**。


### 2）、除了每行代码不用加分号外，Groovy 中函数调用的时候还可以不加括号。

**需要注意的是，我们在使用的时候，如果当前这个函数是 Groovy API 或者 Gradle
API 中比较常用的，比如 println，就可以不带括号。否则还是带括号。不然，Groovy 可能会把属性和函数调用混淆**。


### 3）、Groovy 语句可以不用分号结尾。

### 4）、函数定义时，参数的类型也可以不指定。

### 5）、Groovy 中函数的返回值也可以是无类型的，并且无返回类型的函数，其内部都是按返回 Object 类型来处理的。

### 6）、当前函数如果没有使用 return 关键字返回值，则会默认返回 null，但此时必须使用 def 关键字。

### 7）、在 Groovy 中，所有的 Class 类型，都可以省略 .class。

### 8）、在 Groovy 中，== 相当于 Java 的 equals，，如果需要比较两个对象是否是同一个，需要使用 .is()。

### 9）、Groovy 非运算符如下：

```groovy
assert (!"android") == false                      
```

### 10）、Groovy 支持 ** 次方运算符，代码如下所示：

```groovy
assert  2 ** 3 == 8
```


### 11）、判断是否为真可以更简洁：

```groovy
    if (android) {}
```


### 12）、三元表达式可以更加简洁：

```groovy
// 省略了name
def result = name ?: "Unknown"
```


### 13）、简洁的非空判断

```groovy
println order?.customer?.address
```


### 14）、使用 assert 来设置断言，当断言的条件为 false 时，程序将会抛出异常。

### 15）、可以使用 Number 类去替代 float、double 等类型，省去考虑精度的麻烦。

### 16）、switch 方法可以同时支持更多的参数类型。

注意，swctch 可以匹配列表当中任一元素，示例代码如下所示：

 ```groovy   
// 输出 ok
def num = 5.21
switch (num) {
    case [5.21, 4, "list"]:
        return "ok"
        break
    default:
        break
}
```


# 三、Groovy 基础语法

Groovy 的基础语法主要可以分为以下 <span style="color:#773098;font-weight:bold;">四个部分：</span>

- 1）、<span style="color:#0e88eb;font-weight:bold;">Groovy 核心基础语法。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">Groovy 闭包。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">Groovy 数据结构。</span>
- 4）、<span style="color:#0e88eb;font-weight:bold;">Groovy 面向对象</span>


## 1、Groovy 核心基础语法

### Groovy 中的变量

#### 变量类型

Groovy 中的类型同 Java 一样，也是分为如下 <span style="color:#773098;font-weight:bold;">两种：</span>

- 1）、<span style="color:#0e88eb;font-weight:bold;">基本类型。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">对象类型。</span>


但是，**其实 Groovy 中并没有基本类型，Groovy 作为动态语言， 在它的世界中，所有事物都是对象，就如 Python、Kotlin 一样：所有的基本类型都是属于对象类型**。为了验证这个 Case，我们可以新建一个 **groovy** 文件，创建一个 int 类型的变量并输出它，结果如下图所示：


![](https://imgkr.cn-bj.ufileos.com/d070b09b-7e4e-472d-8db0-eec7f3231ae0.png)


可以看到，上面的输出结果为 'class java.lang.Integer'，因此可以验证我们的想法是正确的。实际上，**Groovy 的编译器会将所有的基本类型都包装成对象类型**。


#### 变量定义

groovy 变量的定义与 Java 中的方式有比较大的差异，对于 groovy 来说，它有 **两种定义方式**，如下所示：

- 1）、<span style="color:rgb(248,57,41);font-weight:bold;">强类型定义方式：</span><span style="color:#0e88eb;font-weight:bold;">groovy 像 Java 一样，可以进行强类型的定义，比如上面直接定义的 int 类型的 x，这种方式就称为强类型定义方式，即在声明变量的时候定义它的类型。</span>
- 2）、<span style="color:rgb(248,57,41);font-weight:bold;">弱类型定义方式：</span><span style="color:#0e88eb;font-weight:bold;">不需要像强类型定义方式一样需要提前指定类型，而是通过 def 关键字来定义我们任何的变量，因为编译器会根据值的类型来为它进行自动的赋值。</span>


下面，我们就使用 **def** 关键字来定义一系列的变量，并输出它们的类型，来看看是否编译器会识别出对应的类型，其结果如下图所示：


![](https://imgkr.cn-bj.ufileos.com/1f2311a0-f40f-46db-be39-ca65d8dcbc09.png)


可以看到，编译器的确会自动自动推断对应的类型。

#### 那么，这两种方式应该分别在什么样的场景中使用呢？

**如果这个变量就是用于当前类或文件，而不会用于其它类或应用模块，那么，建议使用 def 类型，因为在这种场景下弱类型就足够了**。

但是，**如果你这个类或变量要用于其它模块的，建议不要使用 def，还是应该使用 Java 中的那种强类型定义方式，因为使用强类型的定义方式，它不能动态转换为其它类型，它能够保证外界传递进来的值一定是正确的**。如果你这个变量要被外界使用，而你却使用了 def 类型来定义它，那外界需要传递给你什么才是正确的呢？这样会使调用方很疑惑。

如果此时我们在后面的代码中改变上图中 x1 的值为 String 类型，那么 x1 又会被编译器推断为 String 类型，如下图所示：


![](https://imgkr.cn-bj.ufileos.com/189c343a-975d-4d3e-9097-b565d6684ea5.png)


于是我们可以猜测到，其实使用 **def** 关键字定义出来的变量就是 **Obejct** 类型。


### Groovy 中的字符串

**Groovy 中的字符串与 Java 中的字符串有比较大的不同**，所以这里我们需要着重了解一下。

**Groovy 中的字符串除了继承了 Java 中传统 String 的使用方式之前，还 新增 了一个 GString 类型，它的使用方式至少有七、八种，但是常用的有三种定义方式**。此外，**在 GString 中新增了一系列的操作符**，这能够让我们对 String 类型的变量有 **更便捷的操作**。最后，在 GString 中还 **新增 了一系列好用的 API**，我们也需要着重学习一下。


#### Groovy 中常用的三种字符串定义方式

在 Groovy 中有 <span style="color:#773098;font-weight:bold;">三种常用</span> 的字符串定义方式，如下所示：

- 1）、<span style="color:#0e88eb;font-weight:bold;">单引号 '' 定义的字符串</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">双引号 "" 定义的字符串</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">三引号 '""' 定义的字符串</span>


首先，需要说明的是，'不管是单引号、双引号还是三引号，它们的类型都是 java.lang.String'。

##### 那么，单引号与三引号的区别是什么呢？

既生瑜何生亮，其实不然。**当我们编写的单引号字符串中有转义字符的时候，需要添加 '\'，并且，当字符串需要具备多行格式的时候，强行将单引号字符串分成多行格式会变成由 '+' 号组成的字符串拼接格式**。


##### 那么，双引号定义的变量又与单引号、三引号有什么区别呢？

<span style="color:rgb(248,57,41);font-weight:bold;">双引号不同与单、三引号，它定义的是一个可扩展的变量。</span>这里我们先看看两种双引号的使用方式，如下图所示：


![](https://imgkr.cn-bj.ufileos.com/cdfd87fd-3d40-42ce-806d-427b4b3332e1.png)


在上图中，第一个定义的 **author** 字符串就是常规的 **String** 类型的字符串，而下面定义的 **study** 字符串就是可扩展的字符串，因为它里面使用了 '${author}' 的方式引用了 **author** 变量的内容。而且，从其最后的类型输出可以看到，可扩展的类型就是 'org.codehaus.groovy.runtime.GStringImpl' 类型的。

需要注意的是，可扩展的字符串是可以扩展成为任意的表达式，例如数学运算，如下图所示：


![](https://imgkr.cn-bj.ufileos.com/c6f1f45b-afcd-46d4-a688-76935ac75976.png)


有了 Groovy 的这种可扩展的字符串，我们就可以 **避免 Java 中字符串的拼接操作,提升 Java 程序运行时的性能**。

##### 那么，既然有 String 和 GString 两种类型的字符串，它们在相互赋值的场景下需要不需要先强转再赋值呢？

这里，我们可以写一个 小栗子🌰 来看看实际的情况，如下图所示：


![](https://imgkr.cn-bj.ufileos.com/b27fbf04-6e48-406a-98f6-8137325526bb.png)


可以看到，我们将 success 字符串传入了 come 方法，但是最终得到的类型为 result，所以，可以说明 **编译器可以帮我们自动在 String 和 GString 之间相互转换，我们在编写的时候并不需要太过关注它们的区别**。


## 2、Groovy 闭包（Closure）

**闭包的本质其实就是一个代码块**，闭包的核心内容可以归结为如下三点：

- 1)、闭包概念
    - 定义
    - 闭包的调用
- 2)、闭包参数
    - 普通参数
    - 隐式参数
- 3)、闭包返回值
    - 总是有返回值


### 闭包的调用

```groovy
clouser.call()
clouser() 
def xxx = { paramters -> code } 
def xxx = { 纯 code }
```
    
从 C/C++ 语言的角度看，闭包和函数指针很像，闭包可以通过 .call 方法来调用，也可以直接调用其构造函数，代码如下所示：

```groovy
闭包对象.call(参数)
闭包对象(参数)
```   
    
**如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫 it，和 this 的作用类 似**。it 代表闭包的参数。表示闭包中没有参数的示例代码：

```groovy
def noParamClosure = { -> true }
```

#### 注意点：省略圆括号

函数最后一个参数都是一个闭包，类似于回调函数的用法，代码如下所示：

```groovy
task JsonChao {
    doLast ({
        println "love is peace~"
    }
})

// 似乎好像doLast会立即执行一样
task JsonChao {
    doLast {
        println "love is peace~"
    }
}
```

### 闭包的用法

闭包的常见用法有如下 <span style="color:#773098;font-weight:bold;">四种：</span>

- 1）、<span style="color:#0e88eb;font-weight:bold;">与基本类型的结合使用。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">与 String 类的结合使用。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">与数据结构的结合使用。</span>
- 4）、<span style="color:#0e88eb;font-weight:bold;">与文件等结合使用。</span>


### 闭包进阶

- 1）、闭包的关键变量
    - this
    - owner
    - delegate
- 2）、闭包委托策略


#### 闭包的关键变量

##### this 与 owner、delegate 

其差异代码如下代码所示：


```groovy
def scrpitClouser = {
    // 代表闭包定义处的类
    printlin "scriptClouser this:" + this 
    // 代表闭包定义处的类或者对象
    printlin "scriptClouser this:" + owner
    // 代表任意对象，默认与 ownner 一直
    printlin "scriptClouser this:" + delegate 
}
    
// 输出都是 scrpitClouse 对象
scrpitClouser.call()

def nestClouser = {
    def innnerClouser = {
        // 代表闭包定义处的类
        printlin "scriptClouser this:" + this 
        // 代表闭包定义处的类或者对象
        printlin "scriptClouser this:" + owner
        // 代表任意对象，默认与 ownner 一直
        printlin "scriptClouser this:" + delegate 
    }
    innnerClouser.call()
}
    
// this 输出的是 nestClouser 对象，而 owner 与 delegate 输出的都是 innnerClouser 对象
nestClouser.call()
```
    

可以看到，如果我们直接在类、方法、变量中定义一个闭包，那么这三种关键变量的值都是一样的，但是，如果我们在闭包中又嵌套了一个闭包，那么，this 与 owner、delegate 的值就不再一样了。换言之，**this 还会指向我们闭包定义处的类或者实例本身，而 owner、delegate 则会指向离它最近的那个闭包对象**。 
    
##### delegate 与 this、owner 的差异

其差异代码如下代码所示：


```groovy
def nestClouser = {
    def innnerClouser = {
        // 代表闭包定义处的类
        printlin "scriptClouser this:" + this 
        // 代表闭包定义处的类或者对象
        printlin "scriptClouser this:" + owner
        // 代表任意对象，默认与 ownner 一致
        printlin "scriptClouser this:" + delegate 
    }
    
    // 修改默认的 delegate
    innnerClouser.delegate = p 
    innnerClouser.call()
}

nestClouser.call()
```


可以看到，**delegate 的值是可以修改的，并且仅仅当我们修改 delegate 的值时，delegate 的值才会与 ownner 的值不一样**。


#### 闭包的委托策略

其示例代码如下所示：


```groovy
def stu = new Student()
def tea = new Teacher()
stu.pretty.delegate = tea
// 要想使 pretty 闭包的 delegate 修改生效，必须选择其委托策略为 Closure.DELEGATE_ONLY，默认是 Closure.OWNER_FIRST。
stu.pretty.resolveStrategy = Closure.DELEGATE_ONLY
println stu.toString()
```


需要注意的是，要想使上述 pretty 闭包的 delegate 修改生效，必须选择其委托策略为 Closure.DELEGATE_ONLY，默认是 Closure.OWNER_FIRST 的。

## 3、Groovy 数据结构

Groovy 常用的数据结构有如下 <span style="color:#773098;font-weight:bold;">四种：</span>

- 1）、<span style="color:#0e88eb;font-weight:bold;">数组</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">List</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">Map</span>
- 4）、<span style="color:#0e88eb;font-weight:bold;">Range</span>


数组的使用和 Java 语言类似，最大的区别可能就是定义方式的扩展，如下代码所示：

```groovy
// 数组定义
def array = [1, 2, 3, 4, 5] as int[]
int[] array2 = [1, 2, 3, 4, 5]
```

下面，我们看看其它三种数据结构。

### 1、List

即链表，**其底层对应 Java 中的 List 接口，一般用 ArrayList 作为真正的实现类，List 变量由[]定义，其元素可以是任何对象**。

链表中的元素可以通过索引存取，而且 **不用担心索引越界。如果索引超过当前链表长度，List 会自动往该索引添加元素**。下面，我们看看 Map 最常使用的几个操作。

#### 1）、排序

```groovy
def test = [100, "hello", true]
// 左移位表示向List中添加新元素
test << 200
// list 定义
def list = [1, 2, 3, 4, 5]
// 排序
list.sort()
// 使用自己的排序规则
sortList.sort { a, b -> 
    a == b ？0 : 
            Math.abs(a) < Math.abs(b) ? 1 : -1
} 
```

#### 2）、添加

```groovy
// 添加
list.add(6)
list.leftShift(7)
list << 8
```

#### 3）、删除

```groovy
// 删除
list.remove(7)
list.removeAt(7)
list.removeElement(6)
list.removeAll { return it % 2 == 0 }
```

#### 4）、查找

```groovy
// 查找
int result = findList.find { return it % 2 == 0 }
def result2 = findList.findAll { return it % 2 != 0 }
def result3 = findList.any { return it % 2 != 0 }
def result4 = findList.every { return it % 2 == 0 }
```

#### 5）、获取最小值、最大值

```groovy
// 最小值、最大值
list.min()
list.max(return Math.abs(it))
```

#### 6）、统计满足条件的数量

```groovy
// 统计满足条件的数量
def num = findList.count { return it >= 2 }
```


### Map

表示键-值表，其 **底层对应 Java 中的 LinkedHashMap**。

**Map 变量由[:]定义，冒号左边是 key，右边是 Value。key 必须是字符串，value 可以是任何对象。另外，key 可以用 '' 或 "" 包起来，也可以不用引号包起来**。下面，我们看看 Map 最常使用的几个操作。


#### 1）、存取

其示例代码如下所示：

```groovy
aMap.keyName
aMap['keyName']
aMap.anotherkey = "i am map"
aMap.anotherkey = [a: 1, b: 2]
```
    
    
#### 2）、each 方法

如果我们传递的闭包是一个参数，那么它就把 entry 作为参数。如果我们传递的闭包是 2 个参数，那么它就把 key 和 value 作为参数。


```groovy
def result = ""
[a:1, b:2].each { key, value -> 
    result += "$key$value" 
}
    
assert result == "a1b2"
    
def socre = ""
[a:1, b:2].each { entry -> 
    result += entry
}
    
assert result == "a=1b=2"
``` 
    
    
#### 3）、eachWithIndex 方法

如果闭包采用两个参数，则将传递 Map.Entry 和项目的索引（从零开始的计数器）；否则，如果闭包采用三个参数，则将传递键，值和索引。


```groovy
def result = ""
[a:1, b:3].eachWithIndex { key, value, index -> result += "$index($key$value)" }
assert result == "0(a1)1(b3)"

def result = ""
[a:1, b:3].eachWithIndex { entry, index -> result += "$index($entry)" }
assert result == "0(a=1)1(b=3)"
```


#### 4）、groupBy 方法

按照闭包的条件进行分组，代码如下所示：


```groovy
def group = students.groupBy { def student ->
    return student.value.score >= 60 ? '及格' : '不及格'
}
```


#### 5）、findAll 方法

它有两个参数，findAll 会将 Key 和 Value 分别传进 去。并且，如果 Closure 返回 true，表示该元素是自己想要的，如果返回 false 则表示该元素不是自己要找的。
    

### Range

表示范围，它其实是 **List 的一种拓展。其由 begin 值 + 两个点 + end 值表示。如果不想包含最后一个元素，则 begin 值 + 两个点 + < + end 表示。我们可以通过 aRange.from 与 aRange.to 来获对应的边界元素**。


如果需要了解更多的数据结构操作方法，我们可以直接查 [Groovy API 详细文档](http://docs.groovy-lang.org/latest/html/groovy-jdk/index-all.html) 即可。


## 4、Groovy 面向对象

如果不声明 public/private 等访问权限的话，**Groovy 中类及其变量默认都是 public 的**。

### 1）、元编程（Groovy 运行时)

Groovy 运行时的逻辑处理流程图如下所示:


![](https://imgkr.cn-bj.ufileos.com/8fb54e7b-30ad-4797-9b17-c5f62eae2a07.png)


为了更好的讲解元编程的用法，我们先创建一个 Person 类并调用它的 cry 方法，代码如下所示：


```groovy
// 第一个 groovy 文件中
def person = new Person(name: 'Qndroid', age: 26)
println person.cry()

// 第二个 groovy 文件中
class Person implements Serializable {

    String name

    Integer age

    def increaseAge(Integer years) {
        this.age += years
    }

     /**
      * 一个方法找不到时，调用它代替
      * @param name
      * @param args
      * @return
      */
     def invokeMethod(String name, Object args) {

        return "the method is ${name}, the params is ${args}"
    }


    def methodMissing(String name, Object args) {

        return "the method ${name} is missing"
    }
}
```


为了实现元编程，我们需要使用 metaClass，具体的使用示例如下所示：


```groovy
ExpandoMetaClass.enableGlobally()
//为类动态的添加一个属性
Person.metaClass.sex = 'male'
def person = new Person(name: 'Qndroid', age: 26)
println person.sex
person.sex = 'female'
println "the new sex is:" + person.sex
//为类动态的添加方法
Person.metaClass.sexUpperCase = { -> sex.toUpperCase() }
def person2 = new Person(name: 'Qndroid', age: 26)
println person2.sexUpperCase()
//为类动态的添加静态方法
Person.metaClass.static.createPerson = {
    String name, int age -> new Person(name: name, age: age)
}
def person3 = Person.createPerson('renzhiqiang', 26)
println person3.name + " and " + person3.age
```


需要注意的是**通过类的 metaClass 来添加元素的这种方式每次使用时都需要重新添加，幸运的是，我们可以在注入前调用全局生效的处理**，代码如下所示：


```groovy
ExpandoMetaClass.enableGlobally()
// 在应用程序初始化的时候我们可以为第三方类添加方法
Person.metaClass.static.createPerson = { String name,
                                              int age ->
    new Person(name: name, age: age)
}
```


### 2)、脚本中的变量和作用域

**对于每一个 Groovy 脚本来说，它都会生成一个 static void main 函数，main 函数中会调用一个 run 函数，脚本中的所有代码则包含在 run 函数之中**。我们可以通过如下的 groovyc 命令用于将编译得到的 class 文件拷贝到 classes 文件夹下：


```groovy
// groovyc 是 groovy 的编译命令，-d classes 用于将编译得到的 class 文件拷贝到 classes 文件夹 下
groovyc -d classes test.groovy
```


**当我们在 Groovy 脚本中定义一个变量时，由于它实际上是在 run 函数中创建的，所以脚本中的其它方法或其他脚本是无法访问它的。这个时候，我们需要使用 @Field 将当前变量标记为成员变量**，其示例代码如下所示：


```groovy
import groovy.transform.Field; 
    
@Field author = JsonChao
```


# 四、文件处理

## 1、常规文件处理
    
### 1）、读文件

#### eachLine 方法

我们可以使用 eachLine 方法读该文件中的每一行，它唯一的参数是一个 Closure，Closure 的参数是文件每一行的内容。示例代码如下所示：
    
 
```groovy
def file = new File(文件名)
file.eachLine{ String oneLine ->
    println oneLine
} 
    
def text = file.getText()
def text2 = file.readLines()

file.eachLine { oneLine, lineNo ->
    println "${lineNo} ${oneLine}"
}
```


然后，我们可以使用 'targetFile.bytes' 直接得到文件的内容。


#### 使用 InputStream

此外，我们也可以通过流的方式进行文件操作，如下代码所示：


```groovy
//操作 ism，最后记得关掉
def ism = targetFile.newInputStream() 
// do sth
ism.close
```


#### 使用闭包操作 inputStream

利用闭包来操作 inputStream，其功能更加强大，推荐使用这种写法，如下所示：
    
  
```groovy
targetFile.withInputStream{ ism ->
    // 操作 ism，不用 close。Groovy 会自动替你 close 
}
``` 
    

### 2）、写文件

关于写文件有两种常用的操作形式，即通过 withOutputStream/withInputStream 或 withReader/withWriter 的写法。示例代码如下所示：

#### 通过 withOutputStream/、withInputStream copy 文件
   
   
```groovy
def srcFile = new File(源文件名)
def targetFile = new File(目标文件名) targetFile.withOutputStream{ os->
    srcFile.withInputStream{ ins->
        os << ins //利用 OutputStream 的<<操作符重载，完成从 inputstream 到 OutputStream //的输出
    } 
}
```

#### 通过 withReader、withWriter copy 文件


```groovy
def copy(String sourcePath, String destationPath) {
    try {
        //首先创建目标文件
        def desFile = new File(destationPath)
        if (!desFile.exists()) {
            desFile.createNewFile()
        }
    
        //开始copy
        new File(sourcePath).withReader { reader ->
            def lines = reader.readLines()
            desFile.withWriter { writer ->
                lines.each { line ->
                    writer.append(line + "\r\n")
            }
            }
        }
        return true
    } catch (Exception e) {
        e.printStackTrace()
    }
    return false
}
```
  
  
此外，我们也可以通过 withObjectOutputStream/withObjectInputStream 来保存与读取 Object 对象。示例代码如下所示：

#### 保存对应的 Object 对象到文件中

```groovy
def saveObject(Object object, String path) {
    try {
        //首先创建目标文件
        def desFile = new File(path)
        if (!desFile.exists()) {
            desFile.createNewFile()
        }
        desFile.withObjectOutputStream { out ->
            out.writeObject(object)
        }
    return true
    } catch (Exception e) {
    }
    return false
}
```
    
    
#### 从文件中读取 Object 对象


```groovy
def readObject(String path) {
    def obj = null
    try {
        def file = new File(path)
        if (file == null || !file.exists()) return null
        //从文件中读取对象
        file.withObjectInputStream { input ->
            obj = input.readObject()
        }
    } catch (Exception e) {

    }
    return obj
}
```
    
    
## 2、XML 文件操作

### 1）、获取 XML 数据

首先，我们定义一个包含 XML 数据的字符串，如下所示：


```groovy
final String xml = '''
    <response version-api="2.0">
        <value>
            <books id="1" classification="android">
                <book available="20" id="1">
                    <title>疯狂Android讲义</title>
                    <author id="1">李刚</author>
                </book>
                <book available="14" id="2">
                   <title>第一行代码</title>
                   <author id="2">郭林</author>
               </book>
               <book available="13" id="3">
                   <title>Android开发艺术探索</title>
                   <author id="3">任玉刚</author>
               </book>
               <book available="5" id="4">
                   <title>Android源码设计模式</title>
                   <author id="4">何红辉</author>
               </book>
           </books>
           <books id="2" classification="web">
               <book available="10" id="1">
                   <title>Vue从入门到精通</title>
                   <author id="4">李刚</author>
               </book>
           </books>
       </value>
    </response>
'''
```


然后，我们可以 **使用 XmlSlurper 来解析此 xml 数据**，代码如下所示：


```groovy
def xmlSluper = new XmlSlurper()
def response = xmlSluper.parseText(xml)

// 通过指定标签获取特定的属性值
println response.value.books[0].book[0].title.text()
println response.value.books[0].book[0].author.text()
println response.value.books[1].book[0].@available

def list = []
response.value.books.each { books ->
    //下面开始对书结点进行遍历
    books.book.each { book ->
        def author = book.author.text()
        if (author.equals('李刚')) {
            list.add(book.title.text())
        }
    }
}
println list.toListString()
```


### 2)、获取 XML 数据的两种遍历方式

获取 XML 数据有两种遍历方式：深度遍历 XML 数据 与 广度遍历 XML 数据，下面我们看看它们各自的用法，如下所示：

    
#### 深度遍历 XML 数据


```groovy
def titles = response.depthFirst().findAll { book ->
    return book.author.text() == '李刚' ? true : false
}
println titles.toListString()
```


#### 广度遍历 XML 数据


```groovy
def name = response.value.books.children().findAll { node ->
    node.name() == 'book' && node.@id == '2'
}.collect { node ->
    return node.title.text()
}
```


在实际使用中，我们可以 **利用 XmlSlurper 求获取 AndroidManifest.xml 的版本号(versionName)**，代码如下所示：


```groovy
def androidManifest = new XmlSlurper().parse("AndroidManifest.xml") println androidManifest['@android:versionName']
或者
println androidManifest.@'android:versionName'
```


### 3)、生成 XML 数据

除了使用 XmlSlurper 解析 XML 数据之外，我们也可以 **使用 xmlBuilder 来创建 XML 文件**，如下代码所示：


```groovy
/**
 * 生成 xml 格式数据
 * <langs type='current' count='3' mainstream='true'>
 <language flavor='static' version='1.5'>Java</language>
 <language flavor='dynamic' version='1.6.0'>Groovy</language>
 <language flavor='dynamic' version='1.9'>JavaScript</language>
 </langs>
 */
def sw = new StringWriter()
// 用来生成 xml 数据的核心类
def xmlBuilder = new MarkupBuilder(sw) 
// 根结点 langs 创建成功
xmlBuilder.langs(type: 'current', count: '3',
        mainstream: 'true') {
    //第一个 language 结点
    language(flavor: 'static', version: '1.5') {
        age('16')
    }
    language(flavor: 'dynamic', version: '1.6') {
        age('10')
    }
    language(flavor: 'dynamic', version: '1.9', 'JavaScript')
}
    
// println sw

def langs = new Langs()
xmlBuilder.langs(type: langs.type, count: langs.count,
        mainstream: langs.mainstream) {
    //遍历所有的子结点
    langs.languages.each { lang ->
        language(flavor: lang.flavor,
                version: lang.version, lang.value)
    }
}

println sw
    
// 对应 xml 中的 langs 结点
class Langs {
    String type = 'current'
    int count = 3
    boolean mainstream = true
    def languages = [
            new Language(flavor: 'static',
                    version: '1.5', value: 'Java'),
            new Language(flavor: 'dynamic',
                    version: '1.3', value: 'Groovy'),
            new Language(flavor: 'dynamic',
                    version: '1.6', value: 'JavaScript')
    ]
}
//对应xml中的languang结点
class Language {
    String flavor
    String version
    String value
}
```


### 4)、Groovy 中的 json 

我们可以 **使用 Groovy 中提供的 JsonSlurper 类去替代 Gson 解析网络响应，这样我们在写插件的时候可以避免引入 Gson 库**，其示例代码如下所示：


```groovy
def reponse =
        getNetworkData(
                'http://yuexibo.top/yxbApp/course_detail.json')

println reponse.data.head.name
    
def getNetworkData(String url) {
    //发送http请求
    def connection = new URL(url).openConnection()
    connection.setRequestMethod('GET')
    connection.connect()
    def response = connection.content.text
    //将 json 转化为实体对象
    def jsonSluper = new JsonSlurper()
    return jsonSluper.parseText(response)
}
```

    
# 五、总结

在这篇文章中，我们从以下 <span style="color:#773098;font-weight:bold;">四个方面</span> 学习了 Groovy 中的必备核心语法：

- 1）、<span style="color:#0e88eb;font-weight:bold;">groovy 中的变量、字符串、循环等基本语法。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">groovy 中的数据结构：列表、映射、范围。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">groovy 中的方法、类等面向对象、强大的运行时机制。</span>
- 4）、<span style="color:#0e88eb;font-weight:bold;">groovy 中对普通文件、XML、json 文件的处理。</span>


在后面我们自定义 Gradle 插件的时候需要使用到这些技巧，因此，<span style="color:rgb(248,57,41);font-weight:bold;">掌握好 Groovy 的重要性不言而喻，只有扎实基础才能让我们走的更远。</span>


## 参考链接：

- 1、[Groovy API 详细文档](http://docs.groovy-lang.org/latest/html/groovy-jdk/index-all.html)

- 2、《慕课网之Gradle3.0自动化项目构建技术精讲+实战》1 - 5章

- 3、《深入理解 Android 之 Gradle》

- 4、[Gradle从入门到实战 - Groovy基础](https://blog.csdn.net/singwhatiwanna/article/details/76084580)

- 5、[Groovy脚本基础全攻略](https://blog.csdn.net/yanbober/article/details/49047515)


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