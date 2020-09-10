---

		title:  深入探索编译插桩技术（四、ASM）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的 [知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


在 [《深入探索编译插桩技术（二、AspectJ）》](https://juejin.im/post/5e84384af265da47e1593102) 一文中我们深入学习了 AspectJ 在 Android 下的使用。可以看到 AspectJ 非常强大，但是它也只能实现 50% 的字节码操作场景，如果想要实现 100% 的字节码操作场景，那么就不得不使用 ASM。

此外，AspectJ 有着一系列弊端： **由于其基于规则，所以其切入点相对固定，对于字节码文件的操作自由度以及开发的掌控度就大打折扣。并且，他会额外生成一些包装代码，对性能以及包大小有一定影响**。

而 ASM 基本上可以实现任何对字节码的操作，也就是自由度和开发的掌控度很高。它提供了 **访问者模式来访问字节码文件，并且只注入我们想要注入的代码**。

ASM 最初起源于一个博士的研究项目，**在 2002 年开源，并从 5.x 版本便开始支持 Java 8。并且，ASM 是诸多 JVM 语言钦定的字节码生成库，它在效率与性能方面的优势要远超其它的字节码操作库如 javassist、AspectJ**。


# 思维导图大纲


![](https://user-gold-cdn.xitu.io/2020/4/8/1715969f8a4917e2?w=2368&h=1422&f=png&s=388421)


# 目录

- 一、[ASM 的优势和逆势](https://juejin.im/post/5e8d87c4f265da47ad218e6b#heading-4)
    - 1、ASM 的优势
    - 2、ASM 的逆势
- 二、[ASM 的对象模型（ASM Tree API）](https://juejin.im/post/5e8d87c4f265da47ad218e6b#heading-7)
    - 1、优点
    - 2、缺点
    - 3、获取节点
    - 4、操控操作码
- 三、[ASM 的事件模型（ASM Core API）](https://juejin.im/post/5e8d87c4f265da47ad218e6b#heading-23)
    - 2、类读取（解析）者 ClassVisitor
    - 3、小结
- 四、[综合实战训练](https://juejin.im/post/5e8d87c4f265da47ad218e6b#heading-30)
    - 1、使用 ASM Bytecode Outline
    - 2、使用 ASM 编译插桩统计方法耗时
    - 3、全局替换项目中所有的 new Thread
- 五、[总结](https://juejin.im/post/5e8d87c4f265da47ad218e6b#heading-34)


# 一、ASM 的优势和逆势

使用 ASM 操作字节码的优势与逆势都 **比较明显**，其分别如下所示。

## 1、ASM 的优势

- 1）、**内存占用很小**。
- 2）、**运行速度非常快**。
- 3）、**操作灵活：对于字节码的操作非常地灵活，可以进行插入、删除、修改等操作**。
- 4）、**想象空间大，能够借用它提升生产力**。
- 5）、**丰富的文档与众多社区的支持**。


## 2、ASM 的逆势

**上手难度较大，需要对 Java 字节码有比较充分的了解**。

对于 ASM 而言，它提供了 **两种模型：对象模型和事件模型**。

下面，我们就先来讲讲 ASM 的对象模型。


# 二、ASM 的对象模型（ASM Tree API）

对象模型的 **本质** 是一个 **被封装过后的事件模型**，它 **使用了树状图的形式来描述一个类，其中包含多个节点，例如方法节点、字段节点等等，而每个节点又有子节点，例如方法节中有操作码子节点** 等等。下面我们先来了解下由这种树状图模式实现的对象模型的利弊。


## 1、优点

- 1）、**适宜处理简单类的修改**。
- 2）、**学习成本较低**。
- 3）、**代码量较少**。


## 2、缺点

- 1）、**处理大量信息会使代码变得复杂**。
- 2）、**代码难以复用**。


在对象模型下的 ASM 有 **两类操作纬度**，分别如下所示：

- 1）、`获取节点`：**获取指定类、字段、方法节点**。
- 2）、`操控操作码（针对方法节点）`：**获取操作码位置、替换、删除、插入操作码、输出字节码**。


下面我们就来分别来了解下 ASM 的这两类操作。


## 3、获取节点

### 1）、获取指定类的节点

获取一个类节点的代码如下所示：

 ```java   
    ClassNode classNode = new ClassNode();
    // 1
    ClassReader classReader = new ClassReader(bytes);
    // 2
    classReader.accept(classNode, 0);
```

在注释1处，**将字节数组传入一个新创建的 ClassReader，这时 ASM 会使用 ClassReader 来解析字节码**。接着，在注释2处，**ClassReader 在解析完字节码之后便可以通过 accept 方法来将结果写入到一个 ClassNode 对象之中**。

那么，一个 ClassNode 具体又包含哪些信息呢？

如下所示：

#### 类节点信息


类型 | 名称 | 说明 
---|---|---
int | version |	class文件的major版本（编译的java版本） |
int	| access | 访问级
String | name | 类名，采用全地址，如java/lang/String
String | signature | 签名，通常是null 
String | superName | 父类类名，采用全地址
List<String> | interfaces |	实现的接口,采用全地址
String | sourceFile	| 源文件，可能为null
String | sourceDebug | debug源，可能为null
String | outerClass | 外部类
String | outerMethod | 外部方法
String | outerMethodDesc | 外部方法描述（包括方法参数和返回值）
List<AnnotationNode> | visibleAnnotations |	可见的注解
List<AnnotationNode> | invisibleAnnotations | 不可见的注解
List<Attribute> | attrs	| 类的Attribute
List<InnerClassNode> | innerClasses |	类的内部类列表
List<FieldNode> | fields | 类的字段列表
List<MethodNode> | methods | 类的方法列表


### 2）、获取指定字段的节点

获取一个字段节点的代码如下所示：

```java
    for(FieldNode fieldNode : (List)classNode.fields) {
        // 1
        if(fieldNode.name.equals("password"))  {
            // 2
            fieldNode.access = Opcodes.ACC_PUBLIC;
        }
    }
```  
    
**字段节点列表 fields 是一个 ArrayList，它储存着类节点的所有字段**。在注释1处，我们通过遍历 fields 集合的方式来找到目标字段节点。接着，在注释2处，我们将目标字段节点的访问权限置为 public。

除此之外，我们还可以为类添加需要的字段，代码如下所示：

```java
    FieldNode fieldNode = new FieldNode(Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC, "JsonChao", "I", null, null);
    classNode.fields.add(fieldNode);
```
    
在上述代码中，我们直接给目标类节点添加了一个 "public static int JsonChao" 的字段，需要注意的是，第三个参数的 "I" 表示的是 int 的类型描述符。

那么，对于一个字段节点，又包含有哪些字段信息呢？

如下所示：

#### 字段信息


类型 | 名称 | 说明
---|---|---
int | access | 访问级
String | name | 字段名
String | signature | 签名，通常是 null
String | desc | 类型描述，例如 Ljava/lang/String、D（double）、F（float）
Object | value | 初始值，通常为 null
List<AnnotationNode> | visibleAnnotations |	可见的注解
List<AnnotationNode> | invisibleAnnotations | 不可见的注解
List<Attribute>	| attrs	| 字段的 Attribute


接下来，我们看看如何获取一个方法节点。


### 3）、获取指定的方法节点

获取指定的方法节点的代码如下所示：

```java
    for(MethodNode methodNode : (List)classNode.methods) {
        // 1、判断方法名是否匹配目标方法
        if(methodNode.name.equals("getName")) {
            // 2、进行操作
        }
    }
```
    
methods 同 fields 一样，也是一个 ArrayList，通过遍历并判断方法名的方式即可匹配到目标方法。


对于一个方法节点来说，它包含有如下信息：


#### 方法节点包含的信息

类型 | 名称 | 说明
---|---|---
int	| access | 访问级
String | name | 方法名
String | desc | 方法描述，其包含方法的返回值和参数
String | signature | 签名，通常是null
List<String> | exceptions | 可能返回的异常列表
List<AnnotationNode> | visibleAnnotations |	可见的注解列表
List<AnnotationNode> | invisibleAnnotations |  不可见的注解列表
List<Attribute> | attrs | 方法的Attribute列表
Object | annotationDefault | 默认的注解
List[]<AnnotationNode> |  visibleParameterAnnotations	| 可见的参数注解列表
List[]<AnnotationNode> |	invisibleParameterAnnotations |	不可见的参数注解列表
InsnList | instructions	| 操作码列表
List<TryCatchBlockNode>	| tryCatchBlocks  |	try-catch块列表
int	| maxStack | 最大操作栈的深度
int	| maxLocals | 最大局部变量区的大小
List<LocalVariableNode>	| localVariables |	本地（局部）变量节点列表


## 4、操控操作码

在操控字节码之前，我们必须先了解下 `instructions`，即 **操作码列表**，它是 **方法节点中用于存储操作码的地方**，其中 **每一个元素都代表一行操作码**。

**ASM 将一行字节码封装为一个 xxxInsnNode（Insn 表示的是 Instruction 的缩写，即指令/操作码），例如 ALOAD/ARestore 指令被封装入变量操作码节点  VarInsnNode，INVOKEVIRTUAL 指令则会被封入方法操作码节点 MethodInsnNode 之中**。

对于所有的指令节点 xxxInsnNode 来说，它们都继承自抽象操作码节点  `AbstractInsnNode`。其所有的派生类使用详情如下所示。


### 所有的指令码节点说明


名称 | 说明	| 参数
---|---|---
FieldInsnNode | 用于 GETFIELD 和 PUTFIELD 之类的字段操作的字节码 |	String owner 字段所在的类<br>String name 字段的名称<br>String desc 字段的类型
FrameNode | 栈映射帧的对应的帧节点 | 待补充
IincInsnNode | 用于 IINC 变量自加操作的字节码 |	int var：目标局部变量的位置<br>int incr： 要增加的数<br>
InsnNode | 一切无参数值操作的字节码，例如 ALOAD_0，DUP（注意不包含 POP） | 无
IntInsnNode | 用于 BIPUSH、SIPUSH 和 NEWARRAY 这三个直接操作整数的操作 | int operand：操作的整数值
InvokeDynamicInsnNode | 用于 Java7 新增的 INVOKEDYNAMIC 操作的字节码 | String name：方法名称<br>String desc：方法描述<br>Handle bsm：句柄<br>Object[] bsmArgs：参数常量
JumpInsnNode | 用于 IFEQ 或 GOTO 等跳转操作字节码 | LabelNode lable：目标 lable
LabelNode | 一个用于表示跳转点的 Label 节点 | 无
LdcInsnNode	| 使用 LDC 加载常量池中的引用值并进行插入的字节码 | Object cst：引用值
LineNumberNode | 表示行号的节点	| int line：行号<br>LabelNode start：对应的第一个 Label
LookupSwitchInsnNode | 用于实现 LOOKUPSWITCH 操作的字节码 | LabelNode dflt：default 块对应的 Lable<br>List<Integer> keys 键列表<br>List<LabelNode> labels：对应的 Label 节点列表
MethodInsnNode |  用于 INVOKEVIRTUAL 等传统方法调用操作的字节码，不适用于 Java7 新增的 INVOKEDYNAMIC | String owner ：方法所在的类<br>String name ：方法名称<br>String desc：方法描述
MultiANewArrayInsnNode |	 用于 MULTIANEWARRAY 操作的字节码 | String desc：类型描述<br>int dims：维数
TableSwitchInsnNode	| 用于实现 TABLESWITCH 操作的字节码	| int min：键的最小值<br>int max：键的最大值<br>LabelNode dflt：default 块对应的 Lable<br>List<LabelNode> labels：对应的 Label 节点列表
TypeInsnNode | 用于实现 NEW、ANEWARRAY 和 CHECKCAST 等类型相关操作的字节码 | String desc：类型<br>
VarInsnNode	| 用于实现 ALOAD、ASTORE 等局部变量操作的字节码 |	int var：局部变量


下面，我们就开始来讲解下字节码操控有哪几种常见的方式。


### 1、获取操作码的位置

获取指定操作码位置的代码如下所示：

```java
    for(AbstractInsnNode ainNode : methodNode.instructions.toArray()) {
        if(ainNode.getOpcode() == Opcodes.SIPUSH && ((IntInsnNode)ainNode).operand == 16) {
            ....//进行操作
        }
    }
```    

由于一般情况下我们都无法确定操作码在列表中的具体位置，因此 **通常会通过遍历的方式去判断其关键特征**，以此来定位指定的操作码，上述代码就能定位到一个 SIPUSH 16 的字节码，需要注意的是，**有时一个方法中会有多个相同的指令，这是我们需要靠判断前后字节码识别其特征来定位，也可以记下其命中次数然后设定在某一次进行操作，一般情况下我们都是使用的第二种**。


### 2、替换指定的操作码

替换指定的操作码的代码如下所示：

```java
    for(AbstractInsnNode ainNode : methodNode.instructions.toArray()) {
        if(ainNode.getOpcode() == Opcodes.BIPUSH && ((IntInsnNode)ainNode).operand == 16) {
            methodNode.instructions.set(ainNode, new VarInsnNode(Opcodes.ALOAD, 1));
        }
    }
```

这里我们 **直接调用了 InsnList 的 set 方法就能替换指定的操作码对象**，我们在获取了 "BIPUSH 64" 字节码的位置后，便将封装它的操作码替换为一个新的 VarInsnNode 操作码，这个新操作码封装了 "ALOAD 1" 字节码， 将原程序中 `将值设为16` 替换为 `将值设为局部变量1`。


### 3、删除指定的操作码

```java
    methodNode.instructions.remove(xxx);
```
    
xxx 表示的是要删除的操作码实例，我们直接调用用 InsnList 的 remove 方法将它移除掉即可。


### 4、插入指定的操作码

InsnList 主要提供了 **四类** 方法用于插入字节码，如下所示：

- 1）、`add(AbstractInsnNode insn)`： **将一个操作码添加到 InsnList 的末尾**。
- 2）、`insert(AbstractInsnNode insn)`： **将一个操作码插入到这个 InsnList 的开头**。
- 3）、`insert(AbstractInsnNode insnNode，AbstractInsnNode insn)`： **将一个操作码插入到另一个操作码的下面**。
- 4）、`insertBefore(AbstractInsnNode insnNode，AbstractInsnNode insn)` 将一个操作码插入到另一个操作码的上面


接下来看看如何使用这些方法插入指定的操作码，代码如下所示：

```java
    for(AbstractInsnNode ainNode : methodNode.instructions.toArray()) {
        if(ainNode.getOpcode() == Opcodes.BIPUSH && ((IntInsnNode)ainNode).operand == 16) {
            methodNode.instructions.insert(ainNode, new MethodInsnNode(Opcodes.INVOKEVIRTUAL, "java/awt/image/BufferedImage", "getWidth", "(Ljava/awt/image/ImageObserver;)I"));
            methodNode.instructions.insert(ainNode, new InsnNode(Opcodes.ACONSTNULL));
            methodNode.instructions.set(ainNode, new VarInsnNode(Opcodes.ALOAD, 1));
        }
    }
```  
    
这样，我们就能将

```java
    BIPUSH 16
```java

替换为

```java
    ALOAD 1
    ACONSTNULL
    INVOKEVIRTUAL java/awt/image/BufferedImage.getWidth(Ljava/awt/image/ImageObserver;)I
```java

**当我们操控完指定的类节点之后，就可以使用 ASM 的 ClassWriter 类来输出字节码**，代码如下所示：

```java
    // 1、让 ClassWriter 自行计算最大栈深度和栈映射帧等信息
    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTEFRAMES);
    classNode.accept(classWriter);
    return classWriter.toByteArray();
```

关于 ClassWriter 的具体用法，我们会在 ASM Core API 这部分来进行逐步讲解。下面👇，我们就先来看看 ASM 的事件模型。


# 三、ASM 的事件模型（ASM Core API）

对象模型是由事件模型封装而成，因此事件模型的上手难度会更大一些。

对于事件模型来说，它 **采用了设计模式中的访问者模式**。它的出现是为了更好地解决这样一种需求：**有 A 个元素和 N 种算法，每个算法都能作用于任意一个元素，并且在不同的元素上有不同的运行方式**。

在访问者模式出现之前，我们通常会在每一个元素对应的类中添加 N 个方法，然后在每一个方法中去实现一个算法，但是，这样的做法容易导致代码耦合性过高，并且可维护性差。

因此，访问者模式应运而生，我们可以 **建立 N 个访问者，并且每一个访问者拥有一个算法及其内部的 A 种运行方式。当我们需要调用一个算法时，就让相应的访问者去访问元素，然后让访问者根据被访问对象选择相应的算法**。

需要注意的是，访问者并没有直接去操作元素，而是先让元素类调用 accept 方法接收访问者，然后，访问者在元素类的内部方法中开始调用 visit 方法访问当前的元素类。这样，访问者便能直接访问元素类中的内部私有成员，其优势在于 **避免了暴露不必要的内部细节**。

要理解 ASM 的事件模型，我们就需要对其中的 **两个重要成员的工作原理** 有较深的了解。它们便是 **类访问者 ClassVisitor 与 类读取（解析）者 ClassReader**。

**从字节码的视角中，一个 Java 类由很多组件凝聚而成，而这之中便包括超类、接口、属性、域和方法等等。当我们在使用 ASM 进行操控时，可以将它们视为一个个与之对应的事件。因此 ASM 提供了一个 类访问者 ClassVisitor，以通过它来访问当前类的各个组件，当解析器 ClassReader 依次遇到上述的各个组件时，ClassVisitor 上对应的 visitor 事件处理器方法均会被一一调用**。

**与类相似，方法也是由多个组件凝聚而成的，其对应着方法属性、注解及编译后的代码（Class 字节码）。ASM 的 MethodVisitor 提供了一种 hook（钩子）机制，以便能够访问方法中的每一个操作码，这样我们便能够对字节码文件进行细粒度地修改**。

下面，我们便来一一分析下它们。


## 1、类访问者 ClassVisitor

通常我们在使用 ASM 的访问者模式有一个模板代码，如下所示：

```java
    InputStream is = new FileInputStream(classFile);
    // 1
    ClassReader classReader = new ClassReader(is);
    // 2
    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
    // 3
    ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
    // 4
    classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
```

首先，在注释1处，我们 **将目标文件转换为流的形式，并将它融入类读取器 ClassReader 之中**。然后，在注释2处，我们 **构建了一个类写入器 ClassWriter，其参数 COMPUTE_MAXS 的作用是将自动计算本地变量表最大值和操作数栈最大值的任务托付给了ASM**。接着，在注释3处，新建了一个自定义的类访问器，这个自定义的 `ClassVisitor` 的作用是为了**在每一个方法的开始和结尾处插入相应的记时代码，以便统计出每一个方法的耗时**。最后，在注释4处，**类读取器 ClassReader 实例这个被访问者调用了自身的 accept 方法接收了一个 classVisitor 实例**，需要注意的是，第二个参数指定了 `EXPAND_FRAMES`，旨在说明在**读取 class 的时候需要同时展开栈映射帧（StackMap Frame），如果我们需要使用自定义的 MethodVisitor 去修改方法中的指令时必须要指定这个参数**，。

上面，我们说到了栈映射帧（StackMap Frame），它到底是什么呢？


### 栈映射帧 StackMap Frame

它是 Java 6 以后引入的一种验证机制，用于 **检验 Java 字节码的正确性**。它的工作方式是 **记录每一个关键步骤完成后其方法中操作数栈的理论状态**，然后，**在实际运行的时候，ASM 会将其实际状态和理论状态对比，如果状态不一致则表明出现了错误**。

但栈映射帧的实现并不简单，因此通过调用 classReader 实例的 accept 方法我们便可以让 **ASM 自动去计算栈映射**帧，尽管这 **会增加 50% 的额外运算**。此外，可能会有小概率的情况遇到 **栈映射帧验证失败** 的情况，例如`：VerifyError: Inconsistent stackmap frames at branch target` 这个错误。

最常见的原因可能就是由于 **字节码写错造成的**，此时，我们应该去检查对应的字节码实现代码。此外，也可能是 JDK 版本的支持问题或是 ASM 自身的缺陷，但是，这种情况几乎不会发生。


## 2、类读取（解析）者 ClassVisitor

现在，让我们再回到上述注释4处的代码，在这里，我们调用了 classReader 的 accept 方法接收了一个访问者 classVisitor，下面，我们来看看其内部的实现，代码如下所示（源码实现较长，这里我们只需关注注释处的代码即可：

```java
    /**
     * Makes the given visitor visit the Java class of this {@link ClassReader}
     * . This class is the one specified in the constructor (see
     * {@link #ClassReader(byte[]) ClassReader}).
     * 
     * @param classVisitor
     *            the visitor that must visit this class.
     * @param flags
     *            option flags that can be used to modify the default behavior
     *            of this class. See {@link #SKIP_DEBUG}, {@link #EXPAND_FRAMES}
     *            , {@link #SKIP_FRAMES}, {@link #SKIP_CODE}.
     */
    public void accept(final ClassVisitor classVisitor, final int flags) {
        accept(classVisitor, new Attribute[0], flags);
    }
``` 

在 accept 方法中又继续调用了 classReader 的另一个 accept 重载方法，如下所示：

```java
    public void accept(final ClassVisitor classVisitor,
            final Attribute[] attrs, final int flags) {
        int u = header; // current offset in the class file
        char[] c = new char[maxStringLength]; // buffer used to read strings

        Context context = new Context();
        context.attrs = attrs;
        context.flags = flags;
        context.buffer = c;

        // 1、读取类的描述信息，例如 access、name 等等
        int access = readUnsignedShort(u);
        String name = readClass(u + 2, c);
        String superClass = readClass(u + 4, c);
        String[] interfaces = new String[readUnsignedShort(u + 6)];
        u += 8;
        for (int i = 0; i < interfaces.length; ++i) {
            interfaces[i] = readClass(u, c);
            u += 2;
        }

        // 2、读取类的属性信息，例如签名 signature、sourceFile 等等。
        String signature = null;
        String sourceFile = null;
        String sourceDebug = null;
        String enclosingOwner = null;
        String enclosingName = null;
        String enclosingDesc = null;
        int anns = 0;
        int ianns = 0;
        int tanns = 0;
        int itanns = 0;
        int innerClasses = 0;
        Attribute attributes = null;

        u = getAttributes();
        for (int i = readUnsignedShort(u); i > 0; --i) {
            String attrName = readUTF8(u + 2, c);
            // tests are sorted in decreasing frequency order
            // (based on frequencies observed on typical classes)
            if ("SourceFile".equals(attrName)) {
                sourceFile = readUTF8(u + 8, c);
            } else if ("InnerClasses".equals(attrName)) {
                innerClasses = u + 8;
            } else if ("EnclosingMethod".equals(attrName)) {
                enclosingOwner = readClass(u + 8, c);
                int item = readUnsignedShort(u + 10);
                if (item != 0) {
                    enclosingName = readUTF8(items[item], c);
                    enclosingDesc = readUTF8(items[item] + 2, c);
                }
            } else if (SIGNATURES && "Signature".equals(attrName)) {
                signature = readUTF8(u + 8, c);
            } else if (ANNOTATIONS
                    && "RuntimeVisibleAnnotations".equals(attrName)) {
                anns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
                tanns = u + 8;
            } else if ("Deprecated".equals(attrName)) {
                access |= Opcodes.ACC_DEPRECATED;
            } else if ("Synthetic".equals(attrName)) {
                access |= Opcodes.ACC_SYNTHETIC
                        | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
            } else if ("SourceDebugExtension".equals(attrName)) {
                int len = readInt(u + 4);
                sourceDebug = readUTF(u + 8, len, new char[len]);
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleAnnotations".equals(attrName)) {
                ianns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
                itanns = u + 8;
            } else if ("BootstrapMethods".equals(attrName)) {
                int[] bootstrapMethods = new int[readUnsignedShort(u + 8)];
                for (int j = 0, v = u + 10; j < bootstrapMethods.length; j++) {
                    bootstrapMethods[j] = v;
                    v += 2 + readUnsignedShort(v + 2) << 1;
                }
                context.bootstrapMethods = bootstrapMethods;
            } else {
                Attribute attr = readAttribute(attrs, attrName, u + 8,
                        readInt(u + 4), c, -1, null);
                if (attr != null) {
                    attr.next = attributes;
                    attributes = attr;
                }
            }
            u += 6 + readInt(u + 4);
        }

        // 3、访问类的描述信息
        classVisitor.visit(readInt(items[1] - 7), access, name, signature,
                superClass, interfaces);

        // 4、访问源码和 debug 信息
        if ((flags & SKIP_DEBUG) == 0
                && (sourceFile != null || sourceDebug != null)) {
            classVisitor.visitSource(sourceFile, sourceDebug);
        }

        // 5、访问外部类
        if (enclosingOwner != null) {
            classVisitor.visitOuterClass(enclosingOwner, enclosingName,
                    enclosingDesc);
        }

        // 6、访问类注解和类型注解
        if (ANNOTATIONS && anns != 0) {
            for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        classVisitor.visitAnnotation(readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && ianns != 0) {
            for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        classVisitor.visitAnnotation(readUTF8(v, c), false));
            }
        }
        if (ANNOTATIONS && tanns != 0) {
            for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        classVisitor.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && itanns != 0) {
            for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        classVisitor.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), false));
            }
        }

        // 7、访问类的属性
        while (attributes != null) {
            Attribute attr = attributes.next;
            attributes.next = null;
            classVisitor.visitAttribute(attributes);
            attributes = attr;
        }

        // 8、访问内部类
        if (innerClasses != 0) {
            int v = innerClasses + 2;
            for (int i = readUnsignedShort(innerClasses); i > 0; --i) {
                classVisitor.visitInnerClass(readClass(v, c),
                        readClass(v + 2, c), readUTF8(v + 4, c),
                        readUnsignedShort(v + 6));
                v += 8;
            }
        }

        // 9、访问字段和方法
        u = header + 10 + 2 * interfaces.length;
        for (int i = readUnsignedShort(u - 2); i > 0; --i) {
            u = readField(classVisitor, context, u);
        }
        u += 2;
        for (int i = readUnsignedShort(u - 2); i > 0; --i) {
            u = readMethod(classVisitor, context, u);
        }

        // 访问当前类结束时调用
        classVisitor.visitEnd();
    }
``` 
    
首先，在 classReader 实例的 accept 方法中的注释1和注释2处，我们会 **先开始进行类相关的字节码解析的工作：读取了类的描述和属性信息**。接着，在注释3 ~ 注释8处，我们**调用了 classVisitor 一系列的 visitxxx 方法访问 classReader 解析完字节码后保存在内存的信息**。然后，在注释9处，**分别调用了 readField 方法和 readMethod 方法去访问类中的方法和字段**。最后，**调用 classVisitor 的 visitEnd 标识已访问结束**。


### 1）、类内字段的解析

这里，我们先来看看 `readField` 的源码实现，如下所示：

```java
    /**
     * Reads a field and makes the given visitor visit it.
     * 
     * @param classVisitor
     *            the visitor that must visit the field.
     * @param context
     *            information about the class being parsed.
     * @param u
     *            the start offset of the field in the class file.
     * @return the offset of the first byte following the field in the class.
     */
    private int readField(final ClassVisitor classVisitor,
            final Context context, int u) {
        // 1、读取字段的描述信息
        char[] c = context.buffer;
        int access = readUnsignedShort(u);
        String name = readUTF8(u + 2, c);
        String desc = readUTF8(u + 4, c);
        u += 6;

        // 2、读取字段的属性
        String signature = null;
        int anns = 0;
        int ianns = 0;
        int tanns = 0;
        int itanns = 0;
        Object value = null;
        Attribute attributes = null;

        for (int i = readUnsignedShort(u); i > 0; --i) {
            String attrName = readUTF8(u + 2, c);
            // tests are sorted in decreasing frequency order
            // (based on frequencies observed on typical classes)
            if ("ConstantValue".equals(attrName)) {
                int item = readUnsignedShort(u + 8);
                value = item == 0 ? null : readConst(item, c);
            } else if (SIGNATURES && "Signature".equals(attrName)) {
                signature = readUTF8(u + 8, c);
            } else if ("Deprecated".equals(attrName)) {
                access |= Opcodes.ACC_DEPRECATED;
            } else if ("Synthetic".equals(attrName)) {
                access |= Opcodes.ACC_SYNTHETIC
                        | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleAnnotations".equals(attrName)) {
                anns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
                tanns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleAnnotations".equals(attrName)) {
                ianns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
                itanns = u + 8;
            } else {
                Attribute attr = readAttribute(context.attrs, attrName, u + 8,
                        readInt(u + 4), c, -1, null);
                if (attr != null) {
                    attr.next = attributes;
                    attributes = attr;
                }
            }
            u += 6 + readInt(u + 4);
        }
        u += 2;

        // 3、访问字段的声明
        FieldVisitor fv = classVisitor.visitField(access, name, desc,
                signature, value);
        if (fv == null) {
            return u;
        }

        // 4、访问字段的注解和类型注解
        if (ANNOTATIONS && anns != 0) {
            for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        fv.visitAnnotation(readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && ianns != 0) {
            for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        fv.visitAnnotation(readUTF8(v, c), false));
            }
        }
        if (ANNOTATIONS && tanns != 0) {
            for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        fv.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && itanns != 0) {
            for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        fv.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), false));
            }
        }

        // 5、访问字段的属性
        while (attributes != null) {
            Attribute attr = attributes.next;
            attributes.next = null;
            fv.visitAttribute(attributes);
            attributes = attr;
        }

        // 访问字段结束时调用
        fv.visitEnd();

        return u;
    }
```

**同读取类信息的时候类似**，首先，在注释1和注释2处，会 **先开始进行字段相关的字节码解析的工作：读取了字段的描述和属性信息**。然后，在注释3 ~ 注释5处 **按顺序访问了字段的描述、注解、类型注解及其属性信息**。最后，**调用了 FieldVisitor 实例的 visitEnd 方法结束了字段信息的访问**。


### 2）、类内方法的解析

下面，我们在看看 `readMethod` 的实现代码，如下所示：

```java 
    /**
     * Reads a method and makes the given visitor visit it.
     * 
     * @param classVisitor
     *            the visitor that must visit the method.
     * @param context
     *            information about the class being parsed.
     * @param u
     *            the start offset of the method in the class file.
     * @return the offset of the first byte following the method in the class.
     */
    private int readMethod(final ClassVisitor classVisitor,
            final Context context, int u) {
        // 1、读取方法描述信息
        char[] c = context.buffer;
        context.access = readUnsignedShort(u);
        context.name = readUTF8(u + 2, c);
        context.desc = readUTF8(u + 4, c);
        u += 6;

        // 2、读取方法属性信息
        int code = 0;
        int exception = 0;
        String[] exceptions = null;
        String signature = null;
        int methodParameters = 0;
        int anns = 0;
        int ianns = 0;
        int tanns = 0;
        int itanns = 0;
        int dann = 0;
        int mpanns = 0;
        int impanns = 0;
        int firstAttribute = u;
        Attribute attributes = null;

        for (int i = readUnsignedShort(u); i > 0; --i) {
            String attrName = readUTF8(u + 2, c);
            // tests are sorted in decreasing frequency order
            // (based on frequencies observed on typical classes)
            if ("Code".equals(attrName)) {
                if ((context.flags & SKIP_CODE) == 0) {
                    code = u + 8;
                }
            } else if ("Exceptions".equals(attrName)) {
                exceptions = new String[readUnsignedShort(u + 8)];
                exception = u + 10;
                for (int j = 0; j < exceptions.length; ++j) {
                    exceptions[j] = readClass(exception, c);
                    exception += 2;
                }
            } else if (SIGNATURES && "Signature".equals(attrName)) {
                signature = readUTF8(u + 8, c);
            } else if ("Deprecated".equals(attrName)) {
                context.access |= Opcodes.ACC_DEPRECATED;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleAnnotations".equals(attrName)) {
                anns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleTypeAnnotations".equals(attrName)) {
                tanns = u + 8;
            } else if (ANNOTATIONS && "AnnotationDefault".equals(attrName)) {
                dann = u + 8;
            } else if ("Synthetic".equals(attrName)) {
                context.access |= Opcodes.ACC_SYNTHETIC
                        | ClassWriter.ACC_SYNTHETIC_ATTRIBUTE;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleAnnotations".equals(attrName)) {
                ianns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleTypeAnnotations".equals(attrName)) {
                itanns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeVisibleParameterAnnotations".equals(attrName)) {
                mpanns = u + 8;
            } else if (ANNOTATIONS
                    && "RuntimeInvisibleParameterAnnotations".equals(attrName)) {
                impanns = u + 8;
            } else if ("MethodParameters".equals(attrName)) {
                methodParameters = u + 8;
            } else {
                Attribute attr = readAttribute(context.attrs, attrName, u + 8,
                        readInt(u + 4), c, -1, null);
                if (attr != null) {
                    attr.next = attributes;
                    attributes = attr;
                }
            }
            u += 6 + readInt(u + 4);
        }
        u += 2;

        // 3、访问方法描述信息
        MethodVisitor mv = classVisitor.visitMethod(context.access,
                context.name, context.desc, signature, exceptions);
        if (mv == null) {
            return u;
        }

        /*
         * if the returned MethodVisitor is in fact a MethodWriter, it means
         * there is no method adapter between the reader and the writer. If, in
         * addition, the writers constant pool was copied from this reader
         * (mw.cw.cr == this), and the signature and exceptions of the method
         * have not been changed, then it is possible to skip all visit events
         * and just copy the original code of the method to the writer (the
         * access, name and descriptor can have been changed, this is not
         * important since they are not copied as is from the reader).
         */
        if (WRITER && mv instanceof MethodWriter) {
            MethodWriter mw = (MethodWriter) mv;
            if (mw.cw.cr == this && signature == mw.signature) {
                boolean sameExceptions = false;
                if (exceptions == null) {
                    sameExceptions = mw.exceptionCount == 0;
                } else if (exceptions.length == mw.exceptionCount) {
                    sameExceptions = true;
                    for (int j = exceptions.length - 1; j >= 0; --j) {
                        exception -= 2;
                        if (mw.exceptions[j] != readUnsignedShort(exception)) {
                            sameExceptions = false;
                            break;
                        }
                    }
                }
                if (sameExceptions) {
                    /*
                     * we do not copy directly the code into MethodWriter to
                     * save a byte array copy operation. The real copy will be
                     * done in ClassWriter.toByteArray().
                     */
                    mw.classReaderOffset = firstAttribute;
                    mw.classReaderLength = u - firstAttribute;
                    return u;
                }
            }
        }

        // 4、访问方法参数信息
        if (methodParameters != 0) {
            for (int i = b[methodParameters] & 0xFF, v = methodParameters + 1; i > 0; --i, v = v + 4) {
                mv.visitParameter(readUTF8(v, c), readUnsignedShort(v + 2));
            }
        }

        // 5、访问方法的注解信息
        if (ANNOTATIONS && dann != 0) {
            AnnotationVisitor dv = mv.visitAnnotationDefault();
            readAnnotationValue(dann, c, null, dv);
            if (dv != null) {
                dv.visitEnd();
            }
        }
        if (ANNOTATIONS && anns != 0) {
            for (int i = readUnsignedShort(anns), v = anns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        mv.visitAnnotation(readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && ianns != 0) {
            for (int i = readUnsignedShort(ianns), v = ianns + 2; i > 0; --i) {
                v = readAnnotationValues(v + 2, c, true,
                        mv.visitAnnotation(readUTF8(v, c), false));
            }
        }
        if (ANNOTATIONS && tanns != 0) {
            for (int i = readUnsignedShort(tanns), v = tanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        mv.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), true));
            }
        }
        if (ANNOTATIONS && itanns != 0) {
            for (int i = readUnsignedShort(itanns), v = itanns + 2; i > 0; --i) {
                v = readAnnotationTarget(context, v);
                v = readAnnotationValues(v + 2, c, true,
                        mv.visitTypeAnnotation(context.typeRef,
                                context.typePath, readUTF8(v, c), false));
            }
        }
        if (ANNOTATIONS && mpanns != 0) {
            readParameterAnnotations(mv, context, mpanns, true);
        }
        if (ANNOTATIONS && impanns != 0) {
            readParameterAnnotations(mv, context, impanns, false);
        }

        // 6、访问方法的属性信息
        while (attributes != null) {
            Attribute attr = attributes.next;
            attributes.next = null;
            mv.visitAttribute(attributes);
            attributes = attr;
        }

        // 7、访问方法代码对应的字节码信息
        if (code != 0) {
            mv.visitCode();
            readCode(mv, context, code);
        }

        // 8、visits the end of the method
        mv.visitEnd();

        return u;
    }
```

**同类和字段的读取、访问套路一样**，首先，在注释1和注释2处，会 **先开始进行方法相关的字节码解析的工作：读取了方法的描述和属性信息**。然后，在注释3 ~ 注释7处 **按顺序访问了方法的描述、参数、注解、属性、方法代码对应的字节码信息**。需要注意的是，**在 readCode 方法中，也是先读取了方法内部代码的字节码信息，例如头部、属性等等，然后，便会访问对应的指令集**。最后，在注释8处 **调用了 MethodVisitor 实例的 visitEnd 方法结束了方法信息的访问**。

从以上对 ClassVisitor 与 ClassReader 的分析看来，ClassVisitor 被定义为了一个能接收并解析 ClassReader 传入信息的类。**当在 accpet 方法中 ClassVisitor 访问 ClassReader 时，ClassReader 便会先开始字节码的解析工作，并将保存在内存中的结果源源不断地通过调用各种 visitxxx 方法传入到 ClassVisitor 之中**。

需要注意的是，其中 **只有 visit 这个方法一定会被调用一次**，因为它 **获取了类头部的描述信息**，显然易见，它必不可少，而对于其它的 visitxxx 方法来说都不能确定。例如其中的 visitMethod 方法，只有当 ClassReader 解析出一个方法的字节码时，才会调用一次 visitMethod 方法，并由此生成一个方法访问者 MethodVisitor 的实例。

然后，这个 MethodVisitor 的实例便会同 ClassVisitor 一样开始访问当前方法的属性信息，对于 ClassVisitor 来说，它只处理和类相关的事，而方法的事情被外包给了 MethodVisitor 进行处理。这正是访问者的一大优势：**将访问一个复杂事物的职责通过各个不同类型但又相互关联的访问者分割开来**。

由前可知，**对象模型是事件模型的一个封装。其中的 ClassNode 其实就是 ClassVisitor 的一个子类，它负责将 ClassReader 传进来的信息进行分类储存。同样，MethodNode 也是 MethodVisitor 的一个子类，它负责将 ClassReader 传进来的操作码指令信息连接成一个列表并保存其中**。

**而 ClassWriter 也是 ClassVisitor 的一个子类，但是，它并不会储存信息，而是马上会将传入的信息转译成字节码，并在之后随时输出它们。对于 ClassReader 这个被访问者来说，它负责读取我们传入的类文件中的字节流数据，并提供解析流中包含的一切类属性信息的操作**。

最后，为了更进一步地将我们上面所讲解的 ClassReader 与 ClassVisitor 的工作机制更加形象化，这里借用 hakugyokurou 的一张流程图用于回顾梳理，如下所示：

> 注意：第二个"实例化，通过构造函数..."需要去掉


![](https://user-gold-cdn.xitu.io/2020/4/8/171596c7c02f6122?w=700&h=4359&f=gif&s=262253)


## 3、小结

**ASM Core API 类似于解析 XML 文件中的 SAX 方式，直接用流式的方法来处理字节码文件，而不需要把这个类的整个结构读进内存之中**。其好处是能够尽可能地节约内存，难度在于编程时需要有一定的 JVM 字节码基础。由于它的性能较好，所以通常情况下我们都会直接使用 Core API。下面，我们再来回顾下 **事件模型中 Core API 的关键组件**，如下所示：

- 1）、`ClassReader`：**用于读取已经编译好的 .class 文件**。
- 2）、`ClassWriter`：**用于重新构建编译后的类，如修改类名、属性以及方法，也可以生成新的类的字节码文件**。
- 3）、`各种 Visitor 类`：**如上所述，Core API 根据字节码从上到下依次处理，对于字节码文件中不同的区域有不同的 Visitor，比如用于访问方法的 MethodVisitor、用于访问类变量的 FieldVisitor、用于访问注解的 AnnotationVisitor 等等。为了实现 AOP，其重点是要灵活运用 MethodVisitor**。


# 四、综合实战训练

在开始使用 ASM Core API 之前，我们需要先了解一下 `ASM Bytecode Outline` 工具的使用。


## 1、使用 ASM Bytecode Outline

当我们使用 ASM 手写字节码的时候，通常会写一系列 `visitXXXXInsn()` 方法来写对应的助记符，所以 **需要先将每一行源代码转化对应的助记符，然后再通过 ASM 的语法转换为与之对应的 visitXXXXInsn()**。为了解决这个繁琐耗时的流程，因此，ASM Bytecode Outline 便应运而生。

首先，我们需要安装 ASM Bytecode Outline gradle 插件，安装完成后，我们就可 **以直接在目标类中右键选择下拉框底部区域的 Show Bytecode outline**，**然后，AS 的右侧就会出现目标类对应的字节码与 ASM 信息查看区域**。我们直接 **在新标签页中选择 ASMified 这个 tab 即可看到其与之对应的 ASM 代码**，如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/8/171596d48a652b28?w=2870&h=1438&f=png&s=797495)


为了更好地在实践中理解上面所学到的知识，我们可以 **使用 ASM 插桩实现方法耗时的统计** 和 **替换项目中所有的 new Thread**。这里直接给出 Android 开发高手课的 [ASM实战项目地址](https://github.com/AndroidAdvanceWithGeektime/Chapter-ASM)。


> 注意：如果当前需使用 ASM Outline 的类引用了工程中其它的 .java 文件，需要先使用 ASM Outline 将其编译成 .class 文件。


## 2、使用 ASM 编译插桩统计方法耗时

使用 ASM 编译插桩统计方法耗时主要可以细分为如下三个步骤：

- 1）、**首先，我们需要通过自定义 gradle plugin 的形式来干预编译过程**。
- 2）、**然后，在编译过程中获取到所有的 class 文件和 jar 包，然后遍历他们**。
- 3）、**最后，利用 ASM 来修改字节码，达到插桩的目的**。


刚开始的时候，我们可以在 Application 的 onCreate 方法 **先写下要插桩之后的代码**，如下所示：
  
```java  
    @Override
    public void onCreate() {
        long startTime = System.currentTimeMillis();
        super.onCreate();
        long endTime = System.currentTimeMillis() - startTime;
        StringBuilder sb = new StringBuilder();
        sb.append("com/sample/asm/SampleApplication.onCreate time: ");
        sb.append(endTime);
        Log.d("MethodCostTime", sb.toString());
    }
``` 
    
这样便于 **之后能使用 ASM Bytecode Outline 的 ASMified 下的 Show differences 去展示相邻两次修改的代码差异**，其修改之后 ASM 代码对比图如下所示：


![](https://user-gold-cdn.xitu.io/2020/4/8/171596dc8398b060?w=2876&h=1654&f=png&s=822705)


在右图中所示的差异代码就是我们需要添加的 ASM 代码。这里我们直接使用 ASM 的事件模式，即 ASM 的 Core API 来进行字节码的读取与修改，代码如下所示：

```java   
    ClassReader classReader = new ClassReader(is);
    ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
    // 1
    ClassVisitor classVisitor = new TraceClassAdapter(Opcodes.ASM5, classWriter);
    classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
```   
    
上面的实现代码我们在上面已经详细分析过了，当 classReader 调用 accept 方法时就会对类文件进行读取和被 classVisitor 访问。那么，我们是如何对方法中的字节码进行操作的呢？

在注释1处，我们 **自定义了一个 ClassVisitor**，其中的奥秘之处就在其中，其实现代码如下所示：

```java
    public static class TraceClassAdapter extends ClassVisitor {

        private String className;

        TraceClassAdapter(int i, ClassVisitor classVisitor) {
            super(i, classVisitor);
        }


        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, interfaces);
            this.className = name;

        }

        @Override
        public void visitInnerClass(final String s, final String s1, final String s2, final int i) {
            super.visitInnerClass(s, s1, s2, i);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc,
                                         String signature, String[] exceptions) {

            MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
            return new TraceMethodAdapter(api, methodVisitor, access, name, desc, this.className);
        }


        @Override
        public void visitEnd() {
            super.visitEnd();
        }
    }
```    
    
由于我们只需要对方法的字节码进行操作，直接处理  `visitMethod` 这个方法即可。**在这里我们直接将类观察者 ClassVisitor 通过访问得到的 MethodVisitor 进行了封装，使用了自定义的 AdviceAdapter 的方式来实现，而 AdviceAdapter 也是 MethodVisitor 的子类，不同于 MethodVisitor的是，它自身提供了 onMethodEnter 与 onMethodExit 方法，非常便于我们去实现方法的前后插桩**。其实现代码如下所示：

```java
    private int timeLocalIndex = 0;

    @Override
    protected void onMethodEnter() {
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        // 1
        timeLocalIndex = newLocal(Type.LONG_TYPE);
        mv.visitVarInsn(LSTORE, timeLocalIndex);
    }

    @Override
    protected void onMethodExit(int opcode) {
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);
        mv.visitVarInsn(LLOAD, timeLocalIndex);
        // 此处的值在栈顶
        mv.visitInsn(LSUB);
        // 因为后面要用到这个值所以先将其保存到本地变量表中
        mv.visitVarInsn(LSTORE, timeLocalIndex);
            
        int stringBuilderIndex = newLocal(Type.getType("java/lang/StringBuilder"));
        mv.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
        mv.visitInsn(Opcodes.DUP);
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
        // 需要将栈顶的 stringbuilder 指针保存起来否则后面找不到了
        mv.visitVarInsn(Opcodes.ASTORE, stringBuilderIndex);
        mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
        mv.visitLdcInsn(className + "." + methodName + " time:");
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitInsn(Opcodes.POP);
        mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
        mv.visitVarInsn(Opcodes.LLOAD, timeLocalIndex);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);
        mv.visitInsn(Opcodes.POP);
        mv.visitLdcInsn("Geek");
        mv.visitVarInsn(Opcodes.ALOAD, stringBuilderIndex);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
        // 注意： Log.d 方法是有返回值的，需要 pop 出去
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "android/util/Log", "d", "(Ljava/lang/String;Ljava/lang/String;)I", false);
        // 2
        mv.visitInsn(Opcodes.POP);
    }
```

首先，在 onMethodEnter 方法中的注释1处，**我们调用了 AdviceAdapter 的其中一个父类 LocalVariablesSorter 的 newLocal 方法，它会根据指定的类型创建一个新的本地变量，并直接分配一个本地变量的引用 index，其优势在于可以尽量复用以前的局部变量，而不需要我们考虑本地变量的分配和覆盖问题**。然后，**在 onMethodExit 方法中我们便可以将之前的差异代码拿过来适当修改调试即可**，需要注意的是，在注释2处，即 **onMethodExit 方法的最后需要保证栈的清洁，避免在栈顶遗留下不使用的数据，如果在栈顶还留有数据的话，不仅会导致后续代码的异常，也会对其他框架处理字节码造成影响，因此如果操作数栈还有数据的话需要消耗掉或者 POP 出去**。


## 3、全局替换项目中所有的 new Thread

首先，我们先将 MainActivity 的 startThread  方法里面的 Thread 对象改变成 CustomThread，然后通过 ASM Bytecode Outline  的 Show differences 查看在字节码上面的差异，如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/8/171596e0d63c4d8e?w=2872&h=1650&f=png&s=579968)


我们注意到，这里首先调用了 NEW 操作码创建了 thread 实例，然后才调用了 InvokeVirtual 操作码去执行 thread 实例的构造方法。通常情况下这两条指令是成对出现的，但是，偶尔会遇到从其他某个位置传递过来一个已经存在的实例，并直接强制调用构造方法的情况。因此，我们 **需要在代码里面判断  new 和 InvokeSpecial 是否是成对出现的**。其实现代码如下所示：

```java
    private final String methodName;
    private final String className;
    // 标识是否遇到了 new 指令
    private boolean find = false;

    protected TraceMethodAdapter(int api, MethodVisitor mv, int access, String name, String desc, String className) {
        super(api, mv, access, name, desc);
        this.className = className;
        this.methodName = name;
    }

    @Override
    public void visitTypeInsn(int opcode, String s) {
        // 方法可以像类一样就行转换，例如，通过使用一个方法适配器来转发 、、 
        // 那些带有修改的方法调用:改变参数可以被用来变更指令，不转发
        // 某个方法调用可以删除一 个指令，插入新的调用可以添加新的指令
        if (opcode == Opcodes.NEW && "java/lang/Thread".equals(s)) {
            // 遇到 new 指令
            find = true;
            mv.visitTypeInsn(Opcodes.NEW, "com/sample/asm/CustomThread");
                return;
        }
        super.visitTypeInsn(opcode, s);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
        //需要排查 CustomThread 自己
        if ("java/lang/Thread".equals(owner) && !className.equals("com/sample/asm/CustomThread") && opcode == Opcodes.INVOKESPECIAL && find) {
            find = false;
            mv.visitMethodInsn(opcode, "com/sample/asm/CustomThread", name, desc, itf);
            Log.e("asmcode", "className:%s, method:%s, name:%s", className, methodName, name);
                return;
        }
        super.visitMethodInsn(opcode, owner, name, desc, itf);
    }
```  

在使用 ASM 进行插桩的时候，我们尤其需要注意以下 **两点**：

- 1）、当我们使用 ASM 处理字节码时，需要 **逐步小量的修改、验证，切记不要编写大量的字节码并希望它们能够立即通过验证并且可以马上执行**。比较稳妥的做法是，**每编写一行字节码时就考虑一下操作数栈与局部变量表之间的变化情况，确定无误之后再写下一行**。此外，**除了 JVM 的验证器之外，ASM 还维护了一个单独的字节码验证器，它也会检查你的字节码实现是否符合 JVM 规范**。
- 2）、**注意本地变量表和操作数栈的数据交换以及 `try catch blcok` 的处理，关于异常处理可以使用 ASM 提供的 [CheckClassAdapter](https://asm.ow2.io/javadoc/org/objectweb/asm/util/CheckClassAdapter.html)，可以在修改完成后验证一下字节码是否正常**。


除了直接使用 ASM 进行插桩之外，**如果需求比较简单，我们可以使用基于 ASM 的字节码处理工具**，例如：[lancet](https://github.com/eleme/lancet)、[Hunter](https://github.com/Leaking/Hunter/blob/master/README_ch.md) 和 [Hibeaver](https://github.com/BryanSharp/hibeaver)，此时使用它们的投入产出比会更高。


# 五、总结

在 ASM Bytecode Outline 工具的帮助下，我们能够完成很多场景下的 ASM 插桩的需求，但是，当我们使用其处理字节码的时候还是需要考虑很多种可能出现的情况。如果想要具备这方面的深度思考能力，我们就 **必须对每一个操作码的特征都有较深的了解**，如果还不了解的同学可以去看看 [《深入探索编译插桩技术（三、JVM字节码）](https://juejin.im/post/5e899721518825739f6b0351)。因此，**要具备实现一个复杂 ASM 插桩的能力，我们需要对 JVM 字节码、ASM 字节码以及 ASM 源码中的核心工具类的实现 做到了然于心，并且在不断地实践与试错之后，我们才能够成为一个真正的 ASM 插桩高手**。


## 参考链接：
---
1、[ASM官方文档](https://asm.ow2.io/)

2、[《ASM 3.0 指南翻译》PDF](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/ASM3.0%E6%8C%87%E5%8D%97%E7%BF%BB%E8%AF%91.pdf)

2、[极客时间之Android开发高手课 编译插桩的三种方法：AspectJ、ASM、ReDex](https://time.geekbang.org/column/article/82761)

3、[极客时间之Android开发高手课 练习Sample跑起来 | ASM插桩强化练习](https://time.geekbang.org/column/article/83148)

4、[AndroidAdvanceWithGeektime /
Chapter07](https://github.com/AndroidAdvanceWithGeektime/Chapter07)

5、[Java字节码(Bytecode)与ASM简单说明](http://blog.hakugyokurou.net/?p=409)

6、[字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)

7、[字节码操纵技术探秘](https://www.infoq.cn/article/Living-Matrix-Bytecode-Manipulation)

8、[一起玩转Android项目中的字节码](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650244795&idx=1&sn=cdfc4acec8b0d2b5c82fd9d884f32f09&chksm=886377d4bf14fec2fc822cd2b3b6069c36cb49ea2814d9e0e2f4a6713f4e86dfc0b1bebf4d39&mpshare=1&scene=1&srcid=1217NjDpKNvdgalsqBQLJXjX%23rd)

9、[AndroidAdvanceWithGeektime / Chapter-ASM](https://github.com/AndroidAdvanceWithGeektime/Chapter-ASM)

10、[IntelliJ 插件 - ASM Bytecode Outline](https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline)

11、ASM封装库：[lancet](https://github.com/eleme/lancet)、[Hunter](https://github.com/Leaking/Hunter/blob/master/README_ch.md) 和 [Hibeaver](https://github.com/BryanSharp/hibeaver)

12、基于 Javassist 的字节码处理工具：[DroidAssist](https://github.com/didi/DroidAssist/)

13、除了编译期间修改 class 的方式，其实在运行期间我们也可以生成代码，例如现在比较流行的运行时代码生成库 [byte-buddy](https://github.com/raphw/byte-buddy)、[byte-buddy 中文文档](https://notes.diguage.com/byte-buddy-tutorial/#_%E5%89%8D%E8%A8%80)


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