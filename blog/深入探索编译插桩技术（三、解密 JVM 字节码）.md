---

		title:  深入探索编译插桩技术（三、JVM字节码）
		date: 2020/4/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---
# 前言

### 成为一名优秀的Android开发，需要一份完备的 [知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


本篇是 **《深入探索编译插桩技术》系列文章** 的第三篇，相比前两篇文章来说，难度上升了不止一个档次，所以含金量比较高。并且，**拥有扎实的 JVM 字节码基础能让我们更好地掌握 ASM 这个强大的编译插桩工具，而灵活地运用 ASM 能让我们的个人以及项目团队的生产力有质的提升**，这一点，无论是在中小型公司，还是在一二线的大公司来说，都是能极大地提升自身的生产力以及个人与团队的价值。因此，掌握 JVM 字节码便是一件迫不及待的事情了。

下面👇，让我们扬帆起航开始我们的 **JVM 字节码探索之旅** 吧。

> 本文吸取了市面上绝大部分经典 JVM 著作与优秀博文的优势之处，将其中易于理解的部分重新编排并精心处理，相比于仅仅阅读一本 JVM 书籍来说，能让我们在更短的时间内去理解更多对我们重要的知识。注意：标 🔥 的章节为重点章节，建议多多复习，加深对其的理解。


# 思维导图大纲

![](https://user-gold-cdn.xitu.io/2020/4/5/1714ab5f17f722af?w=3082&h=1278&f=png&s=456516)


# 目录

- [一、Class 文件结构初识](https://juejin.im/post/5e899721518825739f6b0351#heading-4)
    - 1、无符号数
    - 2、表
- [二、常量池](https://juejin.im/post/5e899721518825739f6b0351#heading-7)
    - 1、字面量（Literal）
    - 2、符号引用（Symbolic References）
- [三、信息描述规则](https://juejin.im/post/5e899721518825739f6b0351#heading-14)
    - 1、数据类型
    - 2、成员变量
    - 3、成员函数描述规则
- [四、filed_info 与 method_info](https://juejin.im/post/5e899721518825739f6b0351#heading-18)
- [五、access_flags](https://juejin.im/post/5e899721518825739f6b0351#heading-19)
    - 1、Class 的 access_flags 取值类型
    - 2、Filed 的 access_flag 取值类型
    - 3、Method 的 access_flag 取值
- [六、属性](https://juejin.im/post/5e899721518825739f6b0351#heading-23)
    - 1、attribute_name_index
    - 2、Code_attribute
- [七、JVM 指令码](https://juejin.im/post/5e899721518825739f6b0351#heading-29)
    - 1、运行时的栈帧
    - 2、操作数栈
    - 3、局部变量区
    - 4、字节码指令用途分类汇总
- [八、总结](https://juejin.im/post/5e899721518825739f6b0351#heading-51)


# 一、Class 文件结构初识

**“与平台无关”** 的理想最终实现在操作系统的应用层面上：众多虚拟机厂商发布了许多可以运行在各种不同平台上的虚拟机，而这些虚拟机都可以载入和执行同一种与平台无关的字节码，从而实现了程序的 **“一次编写，到处运行”**。

而 **字节码（ByteCode）正是构成其平台无关性的基石**。Java 虚拟机不和包括 Java 在内的任何语言绑定，它 **只与 “Class文件” 这种特定的二进制文件格式所关联，Class 文件中包含 了 Java 虚拟机指令集和符号表以及若干其他辅助信息**。

虚拟机并不关心 Class 的来源是何种语言，**有了字节码，也解除了 Java 虚拟机和 Java 语言之间的耦合**。

**Java 语言中的各种变量、关键字和运算符号的语义最终都是由多条字节码命令组合而成的**，因此，字节码命令所能提供的语义描述能力肯定会比 Java 语言本身更加强大。所以，有一些 Java 语言本身无法有效支持的语言特性不代表字节码本身就无法有效地支持，这也为其他语言实现一些有别于 Java 的语言特性提供了基础。

字节码文件是由 **十六进制值组成** 的，对于 JVM 来说，在读取数据的时候，它会 **以两个十六进制值为一组**，即 **一个字节** 进行读取。在 Java 中，我们通常会采用 javac 命令将源代码编译成字节码文件，下面这幅 Java 官方图展示了一个 **.java 文件从编译到运行的过程**，如下所示：

![](https://user-gold-cdn.xitu.io/2020/4/5/171497d540842211?w=1958&h=870&f=png&s=814083)


**Class 文件是一组以 8 位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在 Class 文件之中，中间没有添加任何分隔符，这使得整个 Class 文件中存储的内容几乎 全部是程序运行的必要数据，没有空隙存在**。

当遇到需要占用 8 位字节以上空间的数据项 时，则会按照高位在前的方式分割成若干个 8 位字节进行存储。（**高位在前指 ”Big-Endian"，即指最高位字节在地址最低位，最低位字节在地址最高位的顺序来存储数据，而 X86 等处理器则是使用了相反的 “Little-Endian” 顺序来存储数据**）

根据 JVM 规范的规定，**Class 文件格式采用了一种类似于 C 语言结构体的伪结构来存 储数据，而这种伪结构中有且只有两种数据类型：无符号数和表**。

## 1、无符号数

无符号数属于基本的数据类型，**以 u1、u2、u4、u8 来分别代表 1 个字节、2 个字节、4 个字节和 8 个字节的无符号数**，无符号数可以用来 **描述数字、索引引用、数量值或者按照UTF-8 码构成字符串值**。

## 2、表

表是 **由多个无符号数或者其他表作为数据项构成的复合数据类型**，所有表都习惯性地以 `“_info”` 结尾。表用于 **描述有层次关系的复合结构的数据，而整个 Class 文件其本质上就是一张表**。

**对比 Linux、Windows 上的可执行文件（例如 ELF）而言，Class 文件可以看做是 JVM 的可执行文件**。其 **表格式** 如下所示：


> u4：表示能够保存4个字节的无符号整数，u2同理。

```jvm
    ClassFile { 
        u4 magic;  // 魔法数字，表明当前文件是.class文件，固定0xCAFEBABE
        u2 minor_version; // 分别为Class文件的副版本和主版本
        u2 major_version; 
        u2 constant_pool_count; // 常量池计数
        cp_info constant_pool[constant_pool_count-1];  // 常量池内容
        u2 access_flags; // 类访问标识
        u2 this_class; // 当前类
        u2 super_class; // 父类
        u2 interfaces_count; // 实现的接口数
        u2 interfaces[interfaces_count]; // 实现接口信息
        u2 fields_count; // 字段数量
        field_info fields[fields_count]; // 包含的字段信息 
        u2 methods_count; // 方法数量
        method_info methods[methods_count]; // 包含的方法信息
        u2 attributes_count;  // 属性数量
        attribute_info attributes[attributes_count]; // 各种属性
    }
```

对于 Class 表结构而言，其 **前 8 个字节** 依次是如下 **三个元素**：

- 1）、`magic`：**每个 Class 文件的头 4 个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机所接受的 Class 文件。很多文件存储标准中都使用魔数来进行身份识别， 譬如图片格式，如 gif 或者 jpeg 等在文件头中都存有魔数。使用魔数而不是扩展名来进行识别主要是基于安全方面的考虑，因为文件扩展名可以随意地改动。并且，Class 文件的魔数获得很有 “浪漫气息”，值为：0xCAFEBABE（咖啡宝贝）**。
- 2）、`minor_version`：**2 个字节长，表示当前 Class 文件的次版号**。
- 3）、`major_version`：**2 个字节长，表示当前 Class 文件的主版本号。（Java 的版本号是从 45 开始 的，JDK 1.1 之后的每个 JDK 大版本发布会在主版本号向上加 1(JDK 1.0~1.1 使用了 45.0~45.3 的版本号)，例如 JDK 1.8  就是 52.0）。需要注意的是，虚拟机会拒绝执行超过其版本号的 Class 文件**。


然后，我们再来简单地了解下 **其它元素** 的含义：


- 4)、`constant_pool_count`：**常量池数组元素个数**。
- 5)、`constant_pool`：**常量池，是一个存储了 cp_info 信息的数组，每一个 Class 文件都有一个与之对应的常量池。（注意：cp_info 数组的索引从 1 开始）**
- 6)、`access_flags`：**表示当前类的访问权限，例如：public、private**。
- 7)、`this_class 和 super_class`：**存储了指向常量池数组元素的索引，this_class  中索引指向的内容为当前类名，而 super_class 中索引则指向其父类类名**。
- 8)、`interfaces_count 和 interfaces`：**同上，它们存储的也只是指向常量池数组元素的索引。其内容分别表示当前类实现了多少个接口和对应的接口类类名**。
- 9)、`fields_count 和 fields`：**表示成员变量的数量和其信息，信息由  field_info 结构体表示**。
- 10)、`methods_count 和 methods`：**表示成员函数的数量和它们的信息，信息由 method_info 结构体表示**。
- 11)、`attributes_count 和 attributes`：**表示当前类的属性信息，每一个属性都有一个与之对应的 attribute_info 结构。常见的属性信息如调试信息，它需要记录某句代码对应源代码的哪一行，此外，如函数对应的 JVM 字节码、注解信息也是属性信息**。


需要注意的是，**Class 表的结构不像 XML 等描述语言，由于它没有任何分隔符号，所以在上面中的这些数据项，无论是顺序还是数量，甚至于数据存储的字节序（Byte Ordering，Class 文件中字节序为 Big-Endian）这样的细节，都是被严格限定的**。


对于上面的各个属性来说，有不少属性是我们需要重点掌握的，而 **常量池可以被认为是 Class 表结构中的重中之重**。下面👇，我们就先来了解下常量池。


# 二、常量池

常量池可以理解为 **Class 文件之中的资源仓库，其它的几种结构或多或少都会最终指向到这个资源仓库之中**。

此外，常量池是 Class 文件结构中与其他项 **关联最多** 的数据类型，也是 **占用 Class 文件空间最大** 的数据项之一，同时它还是 **在 Class 文件中第一个出现的表类型数据项**。因此，如果没有充分了地解常量池，后面其它的 Class 表类型数据项的学习会变得举步维艰。

假设一个常量池的容量（偏移地址:0x00000008）为十六进制数 0x0016，即十进制的 22，这就代表常量池中有 21 项常量，索引值范围为 1~21。**在 Class 文件格式规范制定之时，设计者将第 0 项常量空出来是有特殊考虑的，这样做的目的在 于满足后面某些指向常量池的索引值的数据在特定情况下需要表达 “不引用任何一个常量池项”的含义**。

而常量池中主要存放两大类常量：`字面量（Literal）和符号引用（Symbolic References）`。 

## 1、字面量（Literal）

字面量比较接近于 Java 语言层面的常量概念，如文本字符串、声明为 final 的常量值等。

## 2、符号引用（Symbolic References）（🔥）

而 **符号引用** 则属于编译原理方面的概念，包括了 **三类常量**，如下所示：

- 1）、**类和接口的全限定名（Fully Qualified Name）**
- 2）、**字段的名称和描述符（Descriptor)）**
- 3）、**方法的名称和描述符**


此外，**在虚拟机加载 Class 文件的时候会进行动态链接，因为其字段、方法的符号引用不经过运行期转换的话就无法得到真正的内存入口地址，也就无法直接被虚拟机使用。当虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建或运行时进行解析，并翻译到具体的内存地址之中**。

**connstant_pool 中存储了一个一个的 cp_info 信息，并且每一个 cp_info 的第一个字节（即一个 u1 类型的标志位）标识了当前常量项的类型，其后才是具体的常量项内容**。

下面👇，我们看看有哪些具体的 **常量项的类型**，如下表所示：


类型 |	标志 |	描述
---|---|---
CONSTANT_Utf8_info |	1 |	用于存储UTF-8编码的字符串，它真正包含了字符串的内容。
CONSTANT_Integer_info |	3 |	表示int型数据的信息
CONSTANT_Float_info	| 4	| 表示float型数据的信息
CONSTANT_Long_info |	5 |	表示long型数据的信息
CONSTANT_Double_info |	6	| 表示double型数据的信息
CONSTANT_Class_info	| 7	| 表示类或接口的信息
CONSTANT_String_info	| 8	| 表示字符串，但该常量项本身不存储字符串的内容，它仅仅只存储了一个索引值
CONSTANT_Fieldref_info	| 9	| 字段的符号引用
CONSTANT_Methodref_info	| 10 |	类中方法的符号引用
CONSTANT_InterfaceMethodref_info	| 11 |	接口中方法的符号引用
CONSTANT_NameAndType_info	| 12 |	描述类的成员域或成员方法相关的信息
CONSTANT_MethodHandle_info |	15 |	表示方法句柄信息，其和反射相关
CONSTANT_MethodType_info |	16 |	标识方法类型，仅包含方法的参数类型和返回值类型
CONSTANT_InvokeDynamic_info	| 18 |	表示一个动态方法调用点，用于 invokeDynamic 指令，Java 7引入


然后，我们需要了解其中涉及到的重点常量项类型。这里我们需要先明白 CONSTANT_String 和 CONSTANT_Utf8 的区别。

### CONSTANT_String 和 CONSTANT_Utf8 的区别

- `CONSTANT_Utf8`：**真正存储了字符串的内容，其对应的数据结构中有一个字节数组，字符串便酝酿其中**。
- `CONSTANT_String`：**本身不包含字符串的内容，但其具有一个指向 CONSTANT_Utf8 常量项的索引**。


我们必须要了解的是，**在所有常见的常量项之中，只要是需要表示字符串的地方其实际都会包含有一个指向 CONSTANT_Utf8_info 元素的索引。而一个字符串最大长度即 u2 所能代表的最大值为 65536，但是需要使用 2 个字节来保存 null 值，所以一个字符串的最大长度为 65534**。


对于常见的常量项来说一般可以细分为如下 **三个维度**。

### 常量项 Utf8 

常量项 Utf8 的数据结构如下所示：

```jvm
    CONSTANT_Utf8_info {
        u1 tag; 
        u2 length; 
        u1 bytes[length]; 
    }
```

其元素含义如下所示：

- 1）、`tag`：**值为 1，表示是 CONSTANT_Utf8_info 类型表**。
- 2）、`length`：**length 表示 bytes 的长度，比如 length = 10，则表示接下来的数据是 10 个连续的 u1 类型数据**。
- 3）、`bytes`：**u1 类型数组，保存有真正的常量数据**。


### 常量项 Class、Filed、Method、Interface、String

常量项 Class、Filed、Method、Interface、String 的数据结构分别如下所示：

```jvm
    CONSATNT_Class_info {
        u1 tag;
        u2 name_index; 
    }
```

```jvm
    CONSTANT_Fieldref_info {
        u1 tag;
        u2 class_index;
        u2 name_and_type_index;
    }
```

```jvm
    CONSTANT_MethodType_info {
        u1 tag;
        u2 descriptor_index;
    }
```

```jvm
    CONSTANT_InterfaceMethodref_info {
        u1 tag;
        u2 class_index;
        u2 name_and_type_index;
    }
```

```jvm
    CONSTANT_String_info {
        u1 tag;
        u2 string_index;
    }
```
  
```jvm
    CONSATNT_NameAndType_info {
        u1 tag;
        u2 name_index;
        u2 descriptor_index
    }
``` 
    
其元素含义如下所示：
    
- `name_index`：**指向常量池中索引为 name_index 的常量表。比如 name_index = 6，表明它指向常量池中第 6 个常量**。
- `class_index`：**指向当前方法、字段等的所属类的引用**。
- `name_and_type_index`：**指向当前方法、字段等的名字和类型的引用**。
- `name_index`：**指向某字段或方法等的名称字符串的引用**。
- `descriptor_index`：**指向某字段或方法等的类型字符串的引用**。

    
### 常量项 Integer、Long、Float、Double

常量项 Integer、Long、Float、Double 对应的数据结构如下所示：

```jvm
    CONSATNT_Integer_info {
        u1 tag;
        u4 bytes;
    }
```

```jvm
    CONSTANT_Long_info {
        u1 tag;
        u4 high_bytes;
        u4 low_bytes;
    }
```

```jvm
    CONSTANT_Float_info {
        u1 tag;
        u4 bytes;
    }
```

```jvm
    CONSTANT_Double_info {
        u1 tag;
        u4 high_bytes;
        u4 low_bytes;
    }
```
    
可以看到，**在每一个非基本类型的常量项之中，除了其 tag 之外，最终包含的内容都是字符串。正是因为这种互相引用的模式，才能有效地节省 Class 文件的空间**。（ps：**利用索引来减少空间占用是一种行之有效的方式**）


# 三、信息描述规则

对于 JVM 来说，其 **采用了字符串的形式来描述数据类型、成员变量及成员函数 这三类**。因此，在讨论接下来各个的 Class 表项之前，我们需要了解下 JVM 中的信息描述规则。下面，我们来一一对此进行探讨。

## 1、数据类型

数据类型通常包含有 **原始数据类型、引用类型（数组）**，它们的描述规则分别如下所示：

- 1)、**原始数据类型**：
    - Java 类型的 `byte、char、double、float、int、long、short、boolean` => `"B"、"C"、"D"、"F"、"I"、"J"、"S"、"Z"`。
- 2)、**引用数据类型**：
    - **ClassName => L + 全路径类名（其中的 "." 替换为 "/"，最后加分号）**，例如 `String => Ljava/lang/String;`。
- 3)、**数组（引用类型）**：
    - **不同类型的数组 => "[该类型对应的描述名"**，例如 `int 数组 => "[I"，String 数组 => "[Ljava/lang/Sting;"，二维 int 数组 => "[[I"`。


## 2、成员变量

在 JVM 规范之中，成员变量即 Field Descriptor 的描述规则如下所示：

```jvm
    FiledDescriptor：
    # 1、仅包含 FieldType 一种信息
    FieldType
    FiledType：
    # 2、FiledType 的可选类型
    BaseType | ObjectType | ArrayType
    BaseType：
    B | C | D | F | I | J | S | Z
    ObjectType：
    L + 全路径ClassName；
    ArrayType：
    [ComponentType：
    # 3、与 FiledType 的可选类型一样
    ComponentType：
    FiledType
```  
    
在注释1处，FiledDescriptor 仅仅包含了 FieldType 一种信息；注释2处，可以看到，**FiledType 的可选类型为3中：BaseType、ObjectType、ArrayType**，对于每一个类型的规则描述，我们在 **数据类型** 这一小节已详细分析过了。而在注释3处，这里 `ComponentType` 是一种 JVM 规范中新定义的类型，不过它是 **由 FiledType 构成，其可选类型也包含 BaseType、ObjectType、ArrayType 这三种**。此外，**对于字节码来讲，如果两个字段的描述符不一致， 那字段重名就是合法的**。


## 3、成员函数描述规则

在 JVM 规范之中，成员函数即 Method Descriptor 的描述规则如下所示：

```jvm
    MethodDescriptor:
    # 1、括号内的是参数的数据类型描述，* 表示有 0 至多个 ParameterDescriptor，最后是返回值类型描述
    ( ParameterDescriptor* ) ReturnDescriptor
    ParameterDescriptor:
    FieldType
    ReturnDescriptor:
    FieldType | VoidDescriptor
    VoidDescriptor:
    // 2、void 的描述规则为 "V"
    V
```

在注释1处，**MethodDescriptor 由两个部分组成，括号内的是参数的数据类型描述，表示有 0 至多个 ParameterDescriptor，最后是返回值类型描述**。注释2处，要注意 **void 的描述规则为 "V"**。例如，一个 `void hello(String str)` 的函数 => `（Ljava/lang/String;)V`。


了解了信息的描述规则之后，我们就可以来看看 Class 表中的其它重要的表项：filed_info 与 method_info。

# 四、filed_info 与 method_info

字段表（field_info）用于描述接口或者类中声明的变量。字段（field）包括类级变量以及实例级变量，但 **不包括在方法内部声明的局部变量**。

filed_info 与 method_info 数据结构的伪代码分别如下所示：

```jvm
    field_info {
        u2              access_flags;
        u2              name
        u2              descriptor_index
        u2              attributes_count
        attribute_info  attributes[attributes_count]
    }
```
    
```jvm
    method_info {
        u2              access_flags;
        u2              name
        u2              descriptor_index
        u2              attributes_count
        attribute_info  attributes[attributes_count]
    }
```

可以看到，filed_info 与 method_info 都包含有 **访问标志、名字引用、描述信息、属性数量与存储属性** 的数据结构。对于 method_info 所描述的成员函数来说，它的内容经过编译之后得到的 Java 字节码会保存在属性之中。

注意：**类构造器为 “< clinit >” 方法，而实例构造器为 “< init >” 方法**。

下面，我们就来了解下 access_flags 的相关知识。


# 五、access_flags

access_flag 的取值类型在 Class、Filed、Method 之中都是不同的，我们分别来看看。

## 1、Class 的 access_flags 取值类型

access_flags 中一共有 16 个标志位可以使用，当前只定义了其中 8 个（JDK 1.5 增加了后面 3 种），没有使用到的标志位要求一律为 0。Class 的 access_flags 取值类型如下表示：


标志名 | 标志值 | 标志含义 
---|---|---
ACC_PUBLIC |	0x0001 |	public类型 
ACC_FINAL |	0x0010 |	final类型 
ACC_SUPER |	0x0020 |	使用新的invokespecial语义 
ACC_INTERFACE |	0x0200 |	接口类型 
ACC_ABSTRACT |	0x0400 |	抽象类型 
ACC_SYNTHETIC |	0x1000 |	该类不由用户代码生成 
ACC_ANNOTATION | 0x2000	 | 注解类型 
ACC_ENUM  |	0x4000 |	枚举类型 


例如一个 “public Class JsonChao” 的类所对应的 access_flags 为 0021（0X0001 和 0X0020 相结合）。下面的 Filed 与 Method 的计算也是同理。


## 2、Filed 的 access_flag 取值类型

接口之中的字段必须有 ACC_PUBLIC、ACC_STATIC、ACC_FINAL 标志，这些都是由 Java 本身的语言规则所决定的。Filed 的 access_flag 取值类型如下表所示：


名称 |	值 |	描述
---|---|---
ACC_PUBLIC |	0x0001 |	public
ACC_PRIVATE	 | 0x0002 |	private
ACC_PROTECTED |	0x0004 |	protected
ACC_STATIC |	0x0008 |	static
ACC_FINAL |	0x0010 |	final
ACC_VOLATILE |	0x0040 |	volatile
ACC_TRANSIENT |	0x0080 |	transient，不能被序列化
ACC_SYNTHETIC |	0x1000	| 由编译器自动生成
ACC_ENUM |	0x4000 |	enum，字段为枚举类型


## 3、Method 的 access_flag 取值

Method 的 access_flag 取值如下表所示：


名称 |	值 | 描述
---|---|---
ACC_PUBLIC |	0x0001 |	public
ACC_PRIVATE | 	0x0002 |	private
ACC_PROTECTED |	0x0004 |	protected
ACC_STATIC |	0x0008 |	static
ACC_FINAL |	0x0010 |	final
ACC_SYNCHRONIZED |	0x0020 |	synchronized
ACC_BRIDGE |	0x0040 |	bridge，方法由编译器产生
ACC_VARARGS |	0x0080 |	该方法带有变长参数
ACC_NATIVE |	0x0100 |	native
ACC_ABSTRACT |	0x0400 |	abstract
ACC_STRICT |	0x0800 |	strictfp
ACC_SYNTHETIC |	0x1000 |	方法由编译器生成


需要注意的是，当 Method 的 access_flags 的取值为 `ACC_SYNTHETIC` 时，该 Method 通常被称之为 **合成函数**。此外，**当内部类访问外部类的私有成员时，在 Class 文件中也会生成一个 ACC_SYNTHETIC 修饰的函数**。


# 六、属性

只要不与已有属性名重复，任何人 实现的编译器都可以向属性表中写入自己定义的属性信息，Java 虚拟机运行时会忽略掉它所不认识的属性。

attribute_info 的数据结构伪代码如下所示：

```jvm
    attribute_info {  
        u2 attribute_name_index;
        u4 attribute_length;
        u1 info[attribute_length];
    }
```

attribute_info 中的各个元素的含义如下所示：

- `attribute_name_index`：**为 CONSTANT_Utf8 类型常量项的索引，表示属性的名称**。
- `attribute_length`：**属性的长度**。
- `info`：**属性具体的内容**。


## 1、attribute_name_index

attribute_name_index 所指向的 Utf8 字符串即为属性的名称，而 **属性的名称是被用来区分属性的**。所有的属性名称如下所示（其中下面👇 **标红的为重要属性**）：

- 1）、`ConstantValue`：**仅出现在 filed_info 中，描述常量成员域的值，通知虚拟机自动为静态变量赋值。对于非 static 类型的变量（也就是实例变量）的赋值是在实例构造器<init>方法中进行的;而对 于类变量，则有两种方式可以选择：在类构造器<clinit>方法中或者使用 ConstantValue 属性。如果变量没有被 final 修饰，或者并非基本类型及字 符串，则将会选择在<clinit>方法中进行初始化**。
- 2）、`Code`：**仅出现 method_info 中，描述函数内容，即该函数内容编译后得到的虚拟机指令，try/catch 语句对应的异常处理表等等**。
- 3）、`StackMapTable`：**在 JDK 1.6 发布后增加到了 Class 文件规范中，它是一个复杂的变长属性。这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（Type Checker）使用，目的在于代替以前比较消耗性能的基于数据流 分析的类型推导验证器。它省略了在运行期通过数据流分析去确认字节码的行为逻辑合法性的步骤，而是在编译阶 段将一系列的验证类型（Verification Types）直接记录在 Class 文件之中，通过检查这些验证类型代替了类型推导过程，从而大幅提升了字节码验证的性能。这个验证器在 JDK 1.6 中首次提供，并在 JDK 1.7 中强制代替原本基于类型推断的字节码验证器。StackMapTable 属性中包含零至多个栈映射帧（Stack Map Frames），其中的类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束**。
- 4）、`Exceptions`：**当函数抛出异常或错误时，method_info 将会保存此属性**。
- 5）、InnerClasses：用于记录内部类与宿主类之间的关联。
- 6）、EnclosingMethod
- 7）、Synthetic：标识方法或字段为编译器自动生成的。
- 8）、`Signature`：**JDK 1.5 中新增的属性，用于支持泛型情况下的方法签名，由于 Java 的泛型采用擦除法实现，在为了避免类型信息被擦除后导致签名混乱，需要这个属性记录泛型中的相关信息**。
- 9）、`SourceFile`：**包含一个指向 Utf8 常量项的索引，即 Class 对应的源码文件名**。
- 10）、SourceDebugExtension：用于存储额外的调试信息。
- 11）、`LineNumberTable`：**Java 源码的行号与字节码指令的对应关系**。
- 12）、`LocalVariableTable`：**局部变量数组/本地变量表，用于保存变量名，变量定义所在行**。
- 13）、`LocalVariableTypeTable`：**JDK 1.5 中新增的属性，它使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加**。
- 14）、Deprecated
- 15）、RuntimeVisibleAnnotations
- 16）、RuntimeInvisibleAnnotations
- 17）、RuntimeVisibleParameterAnnotations
- 18）、RuntimeInvisibleParameterAnnotations
- 19）、AnnotationDefault
- 20）、BootstrapMethods：JDK 1.7中新增的属性，用于保存 invokedynamic 指令引用的引导方法限定符。切记，类文件的属性表中最多也只能有一个 BootstrapMethods 属性。


在上述表格中，我们可以发现，不同类型的属性可能会出现在 ClassFile 中不同的成员里，**当 JVM 在解析 Class 文件时会校验 Class 成员应该禁止携带有哪些类型的属性**。此外，**属性也可以包含子属性，例如："Code" 属性中包含有 "LocalVariableTable"**。


## 2、Code_attribute（🔥）

首先，要注意 **并非所有的方法表都必须存在这个属性，例如接口或者抽象类中的方法就不存在 Code 属性**。

Code_attribute 的数据结构伪代码如下所示：

```jvm
    Code_attribute {  
        u2 attribute_name_index; 
        u4 attribute_length;
        u2 max_stack;
        u2 max_locals;
        u4 code_length;
        u1 code[code_length];
        u2 exception_table_length; 
        { 
            u2 start_pc;
            u2 end_pc;
            u2 handler_pc;
            u2 catch_type;
        } exception_table[exception_table_length];
        u2 attributes_count;
        attribute_info attributes[attributes_count];
    }
```

Code_attribute 中的各个元素的含义如下所示：

- `attribute_name_index、attribute_length`：**attribute_length 的值为整个 Code 属性减去 attribute_name_index 和 attribute_length 的长度**。
- `max_stack`：**为当前方法执行时的最大栈深度，所以 JVM 在执行方法时，线程栈的栈帧（操作数栈，operand satck）大小是可以提前知道的。每一个函数执行的时候都会分配一个操作数栈和局部变量数组，而 Code_attribure 需要包含它们，以便 JVM 在执行函数前就可以分配相应的空间**。
- `max_locals`：**为当前方法分配的局部变量个数，包括调用方式时传递的参数。long 和 double 类型计数为 2，其他为 1。max_locals 的单位是 Slot,Slot 是
虚拟机为局部变量分配内存所使用的最小单位。局部变量表中的 Slot 可以重用，当代码执行超出一个局部变量的作用域时，这个局部变量 所占的 Slot 可以被其他局部变量所使用，Javac 编译器会根据变量的作用域来分配 Slot 给各个 变量使用，然后计算出 max_locals 的大小**。
- `code_length`：**为方法编译后的字节码的长度**。
- **code**：**用于存储字节码指令的一系列字节流。既然叫字节码指令，那么每个指令就是一个 u1 类型的单字节。一个 u1 数据类型的取值范围为 0x00~0xFF，对应十进制的 0~255，也就是一共可以表达 256 条指令**。
- `exception_table_length`：**表示 exception_table 的长度**。
- `exception_table`：**每个成员为一个 ExceptionHandler，并且一个函数可以包含多个 try/catch 语句，一个 try/catch 语句对应 exception_table 数组中的一项**。
- `start_pc、end_pc`：**为异常处理字节码在 code[] 的索引值。当程序计数器在 [start_pc, end_pc) 内时，表示异常会被该 ExceptionHandler 捕获**。
- `handler_pc`：**表示 ExceptionHandler 的起点，为 code[] 的索引值**。
- `catch_type`：**为 CONSTANT_Class 类型常量项的索引，表示处理的异常类型。如果该值为 0，则该 ExceptionHandler 会在所有异常抛出时会被执行，可以用来实现 finally 代码。当 catch_type 的值为 0 时，代表任意异常情况都需要转向到 handler_pc 处进行处理。此外，编译器使用异常表而不是简单的跳转命令来实现 Java 异常及 finally 处理机制**。
- `attributes_count 和 attributes`：**表示该 exception_table 拥有的 attribute 数量与数据**。


在 Code_attribute 携带的属性中，`"LineNumberTable"` 与 `"LocalVariableTable"` 对我们 Android 开发者来说比较重要，所以，这里我们将再单独来讲解一下它们。


### 1）、LineNumberTable 属性

LineNumberTable 属性 **用于 Java 的调试，可指明某条指令对应于源码哪一行**。

LineNumberTable 属性的结构如下所示：


```java
LineNumberTable_attribute {  
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc;
        u2 line_number;    
    } line_number_table[line_number_table_length];
}
```


其中最重要的是 `line_number_table` 数组，该数组元素包含如下 **两个成员变量**：


- 1、`start_pc`：**为 code[] 数组元素的索引，用于指向 Code_attribute 中 code 数组某处指令**。
- 2、`line_number`：**为 start_pc 对应源文件代码的行号。需要注意的是，多个 line_number_table 元素可以指向同一行代码，因为一行 Java 代码很可能被编译成多条指令**。


### 2、LocalVariableTable 属性

LocalVariableTable 属性用于 **描述栈帧中局部变量表中的变量与 Java 源码中定义的变量之间的关系**，它也不是运行时必需的属性，但默认会生成到 Class 文件之中。

LocalVariableTable 的数据结构如下所示：


```java
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {
        u2 start_pc;
        u2 length;
        u2 name_index;
        u2 descriptor_index;
        u2 index;
    } local_variable_table[local_variable_table_length];
}
```


其中最重要的元素是 `local_variable_table` 数组，其中的 `start_pc` 与 `length` 这两个参数
**决定了一个局部变量在 code 数组中的有效范围**。

需要注意的是，**每个非 static 函数都会自动创建一个叫做 this 的本地变量，代表当前是在哪个对象上调用此函数。并且，this 对象是位于局部变量数组第1个位置（即 Slot = 0），它的作用范围是贯穿整个函数的**。

此外，**在 JDK 1.5 引入泛型之后，LocalVariableTable 属性增加了一个 “姐妹属性”: LocalVariableTypeTable，这个新增的属性结构与 LocalVariableTable 非常相似，仅仅是把记录 的字段描述符的 descriptor_index 替换成了字段的特征签名（Signature），对于非泛型类型来 说，描述符和特征签名能描述的信息是基本一致的，但是泛型引入之后，由于描述符中泛型的参数化类型被擦除掉，描述符就不能准确地描述泛型类型了，因此出现了 LocalVariableTypeTable**。


#### Slot 是什么？

**JVM 在调用一个函数的时候，会创建一个局部变量数组（即 LocalVariableTable），而 Slot 则表示当前变量在数组中的位置**。


# 七、JVM 指令码（🔥）

在上面，我们了解了 **常量池、属性、field_info、method_info** 等等一系列的源码文件组成结构，**它们是仅仅是一种静态的内容，这些信息并不能驱使 JVM 执行我们在源码中编写的函数**。

从前可知，Code_attribute 中的 code 数组存储了一个函数源码经过编译后得到的 JVM 字节码，其中仅包含如下 **两种** 类型的信息：

- 1)、`JVM 指令码`：**用于指示 JVM 执行的动作，例如加操作/减操作/new 对象。其长度为 1 个字节，所以 JVM 指令码的个数不会超过 255 个（0xFF）**。
- 2)、`JVM 指令码后的零至多个操作数`：**操作数可以存储在 code 数组中，也可以存储在操作数栈（Operand stack）中**。


**一个 Code 数组里指令和参数的组织格式** 如下所示：

> 1字节指令码 0或多个参数（N字节，N>=0）
    

可以看到，**Java 虚拟机的指令由一个字节长度的、代表着某种特定操作含义的数字（称为操作 码，Opcode）以及跟随其后的零至多个代表此操作所需参数（称为操作数，Operands）而构成。此外，大多数的指令都不包含操作数，只有一个操作码**。

字节码指令集是一种具有鲜明特点、优劣势都很突出的指令集架构，**由于限制了 Java 虚拟机操作码的长度为一个字节（即 0~255），这意味着指令集的操作码总数不可能超过 256 条**。

如果不考虑异常处理的话，那么 Java 虚拟机的解释器可以使用下面这个伪代码当做 **最基本的执行模型** 来理解，如下所示：

```java
do {
    自动计算PC寄存器的值加1; 
    根据PC寄存器的指示位置，从字节码流中取出操作码; 
    if(字节码存在操作数)从字节码流中取出操作数; 
    执行操作码所定义的操作;
} while (字节码流长度>0);
```


由于 Java 虚拟机的操作码长度只有一个字节，所以，Java 虚拟机的指令集 **对于特定的操作只提供了有限的类型相关指令去支持它**。例如 **在 JVM 中，大部分的指令都没有支持整数类型 byte、char 和 short，甚至没有任何指令支持 boolean 类型。因此，我们在处理 boolean、byte、short 和 char 类型的数组时，需要转换为与之对应的 int 类型的字节码指令来处理**。

众所周知，JVM 是基于栈而非寄存器的计算模型，并且，基于栈的实现能够带来很好的跨平台特性，因为寄存器指令往往和硬件挂钩。但是，**由于栈只是一个 FILO 的结构，需要频繁地压栈与出栈，因此，对于同样的操作，基于栈的实现需要更多指令才能完成。此外，由于 JVM 需要实现跨平台的特性，因此栈是在内存实现的，而寄存器则位于 CPU 的高速缓存区，因此，基于栈的实现其速度速度相比寄存器的实现要慢很多。要深入了解 JVM 的指令集，我们就必须先从 JVM 运行时的栈帧讲起**。

## 1、运行时的栈帧

**栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟 机运行时数据区中的虚拟机栈（Virtual Machine Stack）的栈元素**。

栈帧中存储了方法的 **局部变量表、操作数栈、动态连接和方法返回地址、帧数据区** 等信息。**每一个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程**。

一个线程中的方法调用链可能会很长，很多方法都同时处于执行状态。**对于 JVM 的执行引擎来 说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧（Current Stack Frame），与这个栈帧相关联的方法称为当前方法（Current Method）。执行引擎运行的所有 字节码指令都只针对当前栈帧进行操作**，而 **栈帧的结构** 如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/5/1714a6968ba0c6ea?w=826&h=703&f=png&s=37869)


Java 中当一个方法被调用时会产生一个栈帧（Stack Frame）,而此方法便位于栈帧之内。而Java方法栈帧 **主要包括三个部分**，如下所示：

- 1）、**局部变量区**
- 2）、**操作数栈区**
- 3）、**帧数据区（常量池引用）** 


帧数据区，即常量池引用在前面我们已经深入地了解过了，但是还有两个重要部分我们需要了解，一个是操作数栈，另一个则是局部变量区。通常来说，**程序需要将局部变量区的元素加载到操作数栈中，计算完成之后，然后再存储回局部变量区**。


### 查看字节码的工具

我们可以使用 [jclasslib](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer) 这个字节码工具去查看字节码，使用效果如下图所示，**代码编译后在菜单栏 ”View” 中选择 ”Show Bytecode With jclasslib”，可以很直观地看到当前字节码文件的类信息、常量池、方法区等信息**。


![](https://user-gold-cdn.xitu.io/2020/4/5/1714a6ce05818b99?w=1166&h=926&f=png&s=127456)

下面👇，我们就先来看看操作数栈是怎么运转的。

## 2、操作数栈

操作数栈是为了 **存放计算的操作数和返回结果。在执行每一条指令前，JVM 要求该指令的操作数已经被压入到操作数栈中，并且，在执行指令时，JVM 会将指令所需的操作数弹出，并将计算结果压入操作数栈中**。

对于操作数栈相关的操作指令有如下 **三类**：

### 1）、直接作用于操作数据栈的指令：

- `dup`：**复制栈顶元素，常用于复制 new 指令所生成的未初始化的引用**。
- `pop`：**舍弃栈顶元素，常用于舍弃调用指令的返回结果**。
- `wap`：**交换栈顶的两个元素的值**。


需要注意的是，**当值为 long 或 double 类型时，需要占用两个栈单元**，此时需要使用 dup2/pop2 指令替代 dup/pop 指令。


### 2）、直接将常量加载到操作数栈的指令：

对于 `int（boolean、byte、char、short）` 类型来说，有如下三类常用指令：

- `iconst`：**用于加载 [-1 ,5] 的 int 值**。
- `biconst`：**用于加载一个字节（byte）所能代表的 int 值即 [-128-127]**。
- `sipush`：**用于加载两个字节（short）所能代表的 int 值即 [-32768-32767]**。


而对于 `long、float、double、reference` 类型来说，各个类型都仅有一类，其实就是类似于 iconst 指令，即 `lconst、fconst、dconst、aconst`。


### 3）、加载常量池中的常量值的指令：

- `ldc`：**用于加载常量池中的常量值，如 int、long、float、double、String、Class 类型的常量。例如 ldc #35 将加载常量池中的第 35 项常量值**。


**正常情况下，操作数栈的压入弹出都是一条条指令完成。唯一的例外是在抛异常时，JVM 会清除操作数栈的所有内容，然后将异常实例压入操作数栈中**。


## 3、局部变量区

局部变量区一般用来 **缓存计算的结果**。实际上，JVM 会把局部变量区当成一个 **数组，里面会依次缓存 this 指针（非静态方法）、参数、局部变量**。

需要注意的是，**同操作数栈一样，long 和 double 类型的值将占据两个单元，而其它的类型仅仅占据一个单元**。

而对于局部变量区来说，它常用的操作指令有 **三种**，如下所示：

### 1）、将局部变量区的值加载到操作数栈中

- `int（boolean、byte、char、short）`：**iload**
- `long`：**lload**
- `float`：**fload**
- `double`：**dload**
- `reference`：**aload**


### 2）、将操作数栈中的计算结果存储在局部变量区中

- `int（boolean、byte、char、short）`：**istore**
- `long`：**lstore**
- `float`：**fstore**
- `double`：**dstore**
- `reference`：**astore**


这里需要注意的是，**局部变量的加载与存储指令都需要指明所加载单元的下标**，例如：**iload_0 就是加载普通方法局部变量区中的 this 指针**。

### 3）、增值指令之 iinc

可以看到，上面两种类型的指令操作都需要操作局部变量区和操作数栈，那么，有没有 **仅仅只作用在局部变量区的指令呢**？

它就是 **iinc M N（M为负整数，N为整数），它会将局部变量数组中的第 M 个单元中的 int 值增加 N，常用于 for 循环中自增量的更新，如 i++/i--**。

了解了以上 JVM 的基础指令之后，我们来看一个具体的栗子🌰，代码和其对应的 JVM 指令如下所示：

```jvm
    public static int bar(int i) {
        return ((i + 1) - 2) * 3 / 4;
    }
    
    // 对应的字节码如下：
    Code:
    stack=2, locals=1, args_size=1
        0: iload_0
        1: iconst_1
        2: iadd
        3: iconst_2
        4: isub
        5: iconst_3
        6: imul
        7: iconst_4
        8: idiv
        9: ireturn
```

这里我们解释下上面的几处字节码的含义，如下所示：

- `Code`：**JVM 字节码**。
- `stack`：**表示该方法需要的操作数栈空间为 2**。
- `locals`：**表示该方法需要的局部变量区空间为 1**。
- `args_size`：**表示方法的参数大小为 1**。


最后，我们来看看 **每条指令执行前后局部变量区和操作数栈的变化情况**，如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/5/1714a7a702123f83?w=668&h=1420&f=png&s=135324)


了解了指令在操作数栈与局部变量区之间的转换规律，我们下面再回过头来系统地了解下以下 **九类按用途分类的字节码指令**。


## 4、字节码指令用途分类汇总

### 1）、加载和存储指令

加载和存储指令用于 **将数据在栈帧中的局部变量表和操作数栈之间来回传输**，其指令如下所示：

- 1）、**将一个局部变量加载到操作栈**：`iload、iload_<n>、lload、lload_<n>、fload、fload_
<n>、dload、dload_<n>、aload、aload_<n>`。 
- 2）、**将一个数值从操作数栈存储到局部变量表**：`istore、istore_<n>、lstore、lstore_<n>、
fstore、fstore_<n>、dstore、dstore_<n>、astore、astore_<n>`。 
- 3）、**将一个常量加载到操作数栈**：`bipush、sipush、ldc、ldc_w、ldc2_w、aconst_null、
iconst_m1、iconst_<i>、lconst_<l>、fconst_<f>、dconst_<d>`。
- 4）、**扩充局部变量表的访问索引的指令**：`wide`。


**类似于 iload_<n>，它代表了 iload_0、iload_1、iload_2 和 iload_3 这几条指令。这几组指令都是某个带有一个操作数的通用指令（例如iload，iload_0 的语义与操作数为 0 时的 iload 指令语义完全一致）**。


### 2）、运算指令

运算或算术指令用于 **对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操 作栈顶**。大体上算术指令可以分为 **两种：对整型数据进行运算的指令与对浮点型数据进行运算的指令**。其指令如下所示：

- 1）、**加法指令**：`iadd、ladd、fadd、dadd`。 
- 2）、**减法指令**：`isub、lsub、fsub、dsub`。 
- 3）、**乘法指令**：`imul、lmul、fmul、dmul`。 
- 4）、**除法指令**：`idiv、ldiv、fdiv、ddiv`。 
- 5）、**求余指令**：`irem、lrem、frem、drem`。 
- 6）、**取反指令**：`ineg、lneg、fneg、dneg`。 
- 7）、**位移指令**：`ishl、ishr、iushr、lshl、lshr、lushr`。 
- 8）、**按位或指令**：`ior、lor`。 
- 9）、**按位与指令**：`iand、land`。 
- 10）、**按位异或指令**：`ixor、lxor`。 
- 11）、**局部变量自增指令**：`iinc`。 
- 12）、**比较指令**：`dcmpg、dcmpl、fcmpg、fcmpl、lcmp`。


### 3）、类型转换指令

类型转换指令可以 **将两种不同的数值类型进行相互转换**，例如我们可以将小范围类型向大范围类型的安全转换，其指令如下所示：

-1）、`i2b、i2c、i2s`
-2）、`l2i`
-3）、`f2i、f2l`
-4）、`d2i、d2l、d2f`


### 4）、对象创建与访问指令

其指令如下所示：

- 1）、**创建类实例的指令**：`new`。
- 2）、**创建数组的指令**：`newarray、anewarray、multianewarray`。
- 3）、**访问类字段（static字段，或者称为类变量）和实例字段（非 static 字段，或者称为实例变量）的指令**：`getfield、putfield、getstatic、putstatic`。
- 4）、**把一个数组元素加载到操作数栈的指令**：`baload、caload、saload、iaload、laload、 faload、daload、aaload`。
- 5）、**将一个操作数栈的值存储到数组元素中的指令**：`bastore、castore、sastore、iastore、 fastore、dastore、aastore`。
- 6）、**取数组长度的指令**：`arraylength`。 
- 7）、**检查类实例类型的指令**：`instanceof、checkcast`。


### 5）、操作数栈管理指令

用于 **直接操作操作数栈** 的指令，如下所示：

- 1）、**将操作数栈的栈顶一个或两个元素出栈**：`pop、pop2（用于操作 Long、Double）`。
- 2）、**复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶**：`dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2`。
- 3）、**将栈最顶端的两个数值互换**：`swap`。


### 6）、控制转移指令

控制转移指令就是 **在有条件或无条件地修改 PC 寄存器的值**。其指令如下所示：

- 1）、**条件分支**：`ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、 if_icmplt、if_icmpgt、if_icmple、if_icmpge、if_acmpeq 和 if_acmpne`。
- 2）、**复合条件分支**：`tableswitch、lookupswitch`。
- 3）、**无条件分支**：`goto、goto_w、jsr、jsr_w、ret`。


其中的 tableswitch 与 lookupswitch 含义如下：

- `tableswitch`：**条件跳转指令，针对密集的 case**。
- `lookupswitch`：**条件跳转指令，针对稀疏的 case**。


可以看到，**Java 虚拟机提供的 int 类型的条件分支指令是最为丰富和强大的**。


### 7）、方法调用指令


常用的有 **5条** 用于方法调用的指令。 如下所示：

- 1）、`invokevirtual`：**用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是 Java 语言中最常见的方法分派方式**。 
- 2）、`invokeinterface`：**用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用**。
- 3）、`invokespecial`：**用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法**。
- 4）、`invokestatic`：**用于调用类方法（static方法）**。
- 5）、`invokedynamic`：**用于在运行时动态解析出调用点限定符所引用的方法，并执行该方法，前面 4 条调用指令的分派逻辑都固化在 Java 虚拟机内部，而 invokedynamic 指令的分派逻辑是由用户所设定的引导方法决定的**。


这里我们需要着重注意 `invokespecial` 指令，它用于 **调用构造器与方法，当调用方法时，会将返回值仍然压入操作数栈中，如果当前方法没有返回值则需要使用 pop 指令弹出**。

除了 invokespecial 之外，其它方法调用指令所消耗的操作数栈元素是根据调用类型以及目标方法描述符来确定的。

### 8）、方法返回指令

返回指令是区分类型的，如下所示，为不同返回类型对应的返回指令：

- `void`：**return**
- `int（boolean、byte、char、short）`：**ireturn**
- `long`：**lreturn**
- `float`：**freturn**
- `double`：**dreturn**
- `reference`：**areturn**


方法调用指令与数据类型无关，而 **方法返回指令是根据返回值的类型区分的，包括 ireturn（当返回值是 boolean、byte、char、short 和 int 类型时使用）、lreturn、freturn、dreturn 和 areturn，另外还有一条 return 指令供声明为 void 的方法、实例初始化方法以及类和接口的类初始化方法使用**。


### 9）、异常处理指令

**在 Java 程序中显式抛出异常的操作（throw语句）都由 athrow 指令来实现，在 Java 虚拟机中，处理异常是采用异常表来完成的**。


### 10）、同步指令

Java 虚拟机可以 **支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor）来支持的**。

**方法级的同步是隐式的，即无须通过字节码指令来控制，它实现在方法调用和返回操作 之中。虚拟机可以从方法常量池的方法表结构中的 ACC_SYNCHRONIZED 访问标志得知一个方法是否声明为同步方法**。

**当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时会释放管程**。

**同步一段指令集序列** 通常是由 Java 语言中的 **synchronized 语句块** 来表示的，**Java 虚拟机的指令集中有 monitorenter 和 monitorexit 两条指令来支持 synchronized 关键字的语义，而正确实现 synchronized 关键字需要 Javac 编译器与 Java 虚拟机两者共同协作支持**。

编译器必须确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都必须执行其对应的 monitorexit 指令，而无论这个方法是正常结束还是异常结束。并且，它会**自动产生一个异常处理器，这个异常处理器被声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令**。


# 八、总结

深入学习 JVM 字节码无疑会对我们的整体实力有 **质的提升**，如果对 JVM 字节码了解较深，那么，我们在学习 Groovy、Kotlin 等这些基于 JVM 的语言时就能够 **在较短的学习时间内进阶到语言的高级层面**。此外，**深入了解 JVM 字节码，能够赋予我们通过表象透析本质的能力，而这，也正是极客们真正所追求的一通百通的灵魂之力**。


## 参考链接：

- 1、《深入理解Java虚拟机 JVM高级特性与最佳实践》第6章 类文件结构、第8章 虚拟机字节码执行引擎

- 2、《深入理解Android Java虚拟机ART》第2章 深入理解Class文件格式

- 3、[深入拆解Java虚拟机 - 19 | Java字节码（基础篇）](https://time.geekbang.org/column/article/14794)

- 4、[Android 工程师进阶 34 讲 - 第03讲：字节码层面分析 class 类文件结构](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=67#/detail/pc?id=1857)

- 5、[极客时间之Android开发高手课 编译插桩的三种方法：AspectJ、ASM、ReDex](https://time.geekbang.org/column/article/71982)

- 6、[Java 字节码 (Bytecode) 与 ASM 简单说明](http://blog.hakugyokurou.net/?p=409)

- 7、[字节码操纵技术探秘](https://www.infoq.cn/article/Living-Matrix-Bytecode-Manipulation)

- 8、[字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)

- 9、[The Java® Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se11/html/index.html)


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