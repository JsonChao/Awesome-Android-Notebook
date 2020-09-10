---

		title:   Python入门篇
		date: 2018/07/26 22:46:00   
		tags: 
		- python
		categories: python
		thumbnail: https://gss3.bdstatic.com/-Po3dSag_xI4khGkpoWK1HF6hhy/baike/s%3D220/sign=cd18c513dab44aed5d4eb9e6831c876a/472309f790529822390eff9ad7ca7bcb0a46d4ae.jpg
---

---

### 简介

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

随着人工智能和AI行业的兴起，Python已经成为程序员不得不学的一门编程语言了，本系列文章主要是涉及使用Python入门方面的知识。
    
### 初识Python

    what：python是什么？
        它是一门跨平台的高级编程语言，相对其它高级语言（如：C、Java）
        来说，它封装的功能更完善，能用更少的代码实现同样的功能。
        python的作用？
        Python的用处很多，它主要使用的领域有：
        1.前后端开发
        2.工具脚本开发
        3.爬虫
        4.人工智能
        优点：简洁、易懂、用更少的代码实现功能模块。
        缺点：
        1.程序运行速度很慢，因为python是解释性语言，计算机每执行一行
        python代码就会把它翻译成自身能识别的机器码，而其它语言，如
        C语言，则会在运行前就会被编译成计算机能识别的机器码。
        2.python不能够加密，发布就是将源码发布出去，这正是解释性语言
        的缺陷，而编译型语言则不会，如C语言，它是将编译后得到的二进制
        码发布出去。
    why：为什么要使用python？
        因为它的优点——简洁、易懂、能用更少的代码实现功能模块，特别适合做一些脚本工具。
    how：如何使用它？
        请往下看。。。
    
#### 安装Python开发工具（Windows系统）

    python有两个版本（2.x，3.x），只演示新版本的安装。
    打开python网站，选择安装Install->windows->最新relese 64bit即可，可选框全选即可。
    
    官网安装的python环境用的是CPython解释器（包含python代码的文档称
    为.py,解释器就是用来执行python代码的）。解释器有很多种，CPython是主流。
    
#### 先用起来？

    启动方式？
    1.打开命令行模式，输入python进入python交互模式。输入exit()回到命令行模式。
    2.直接点击python终端，进入python交互模式。输入exit()退出命令行模式。
    
    注意：python交互模式的代码是输入一行，输出一行，它只适合初学者用来
    调试代码时使用，正常开发都是编写*.py文件，使用python *.py运行*.py文件，
    这样就会一次性执行python源代码。（文件名只能是英文字母、数字、下划线的组合）。
    
##### 输入和输出

    输出：使用print()，可以“”，‘’的形式输出单个字符串，也可输出多个字符串如print("hello","I","am","jsonchao"),输出时，号相当于一个空格。
    
    输入：使用input()，括号内可以写入输入的提示信息，如：
    name = print("please enter your name:")
    print("hello", name)
    其中name为字符串变量。
    
#### python基础

    python、C、JAVA都是高级语言，它们不同于自然语言，它们需要通过解释器
    或编译期将符合自身语法规则的语言转换为计算机能够执行的机器码。
    
    约定俗成的规则：
    1.#后面为注释；
    2.语句后面加：号结尾时，缩进的语句变为代码块；缩进一般为4个空格=tab键；
    3.不同于java，复制粘贴时，缩进的格式可能复制不过来，需要重起缩进格式；
    4.python程序是大小写敏感的。
    
##### 数据类型和变量

    1.r""表示字符串里面的内容默认不转义。
    2.'''hello
        I am
        jsonchao'''为避免加入多个\n的写法。
    3.布尔值：True、False首字母为大写，and、or、not为运算符。
    4.None为python中一个特殊的空值。
    5.变量a = 10
        a = "10"，说明python是一门变量可以动态赋值为不同类型的语言，
        称为动态语言，而java的变量，则一开始则指定了类型，如：
        int a = 10；所以java是一门静态语言。
    6.用大写字母规范地表示常量，虽然它的值还是可以动态改变。。。
    7.除法：/，//（地板除，值取整）得到的都是浮点数结果，%（取余，结果为整数）。
    8.python的整数没有大小限制，浮点数没有大小限制，但是超过一定值会表示为inf（无限大）。
    
##### 字符编码和字符串

    字符编码：
    最初的ASCLL编码为1个字节表示一个字符，由于不同国家有不同的编码，为
    了解决文本显示不同语言乱码的问题，国际统一了Unicode编码，一般为2个
    字节表示一个字符（生僻的中文为4个字节表示一个字符），为了节省
    Unicode在保存数据和传输数据时字符占用过多字节的问题，后面在储存和
    传输时会将Unicode转换为UTF-8编码，文本显示时又会转变回Unicode编码。
    最常用的编码为utf-8。
    
    Python的字符串：
    在最新的Python3中，字符是Unicode编码的，因此，它能适配多语言。
    1.ord()获取字符的整数表示，chr()获取整数对应的字符。
    2.如果需要将字符串在进行网络传输或者存储到磁盘，就必须将其转换成bytes（字节）。
    3.b''或b""表示里面的为字节，使用encode()将字符串编码为字节，
    decode()将字节编码为字符串，括号内为指定的编译码格式，\x后面指定
    为不能被ASCLL识别的字符。b'23\x3d3j'.decode('utf-8',errors='ignore'),以此格式指定忽略错误。
    4.len()表示计算出字符串的长度。
    5.使用%d、%s、%f、%x来格式化字符串，形式为：
    "emm..., is %s." % (jsonchao)
    "emm...，$%d，is %s." % (10000000， jsonchao)
    还可以使用%0d表示0x，%.2f表示3.14这样的形式。
    用%%表示%字符串。
    还有另一种格式化字符串的方式，使用.format()，如：
    'haha,i'm {0} year\’s old, {1:.1f}%percent power'.format(24, 30.555)
    注意，30.5会四舍五入为30.6。
    6.'haha'.replace('h', 'd')替换指定字符。

##### List和Tuple

    List：
    MyList = [10,'haha'，[20, 'lala']]，可存储不同类型数据，元素还可以是List。
    MyList[-1], MyList[-2]表示取出倒数第一，二个值。
    MyList.append(20)，结尾添加值。
    MyList.insert(1, ‘haha’)，下标为1处添加值。
    MyList.pop()，弹出最后一个值。
    MyList.pop(0)，弹出第一个值。
    MyList[2][1],二维取值。
    MyList.sort()从小到大进行排序。
    
    Tuple:
    是不可变的，定义为：
    MyTuple = (20, 'haha')
    当Tuple中只表示一个元素时，必须使用MyTuple = (20,)来消除来Python以为是括号()+值的歧义。
    记住不变，是指Tuple的每一个元素的指向不变，并不是指向的元素内容不变。
    （相当于Java中指向元素的地址，C语言的指针）
    
    为什么需要元组？
    
    1.旧式字符串格式化中参数要用元组；
    2.在字典中当作键值；
    3.数据库的返回值……
    
    区别：即为可变与不可变。

    
##### 条件判断

    if:后面会执行接下来缩进的两行代码。
    1.elif为else if的缩写。
    2.if还有如下写法：
    if 20:
        print('nice')
    只要if后的内容是非0，非空List，一切非空内容即为Ture。
    elif 20 <= bmi < 25不同于java，java为bmi >= 20 &&bmi < 25。
    **为java平方符合^。
    
##### 循环

    names = ['tianshen', 'jsonchao', 'zhanshen']
    for x in names：
        print("Hello, " + x + "!")
    while
    break
    continue
    break和continue尽量少用，易造成程序逻辑混乱。
    
##### dict和set

    dict称为字典，编写形式如下：
    d = {'hello' : 1, 'haha' : 2, 'emem' : 3}
    判断是否有对应的key：
    'hello' in d,有则True，无则False。
    d.get('hello')获取key对应的值。
    没有则返回none，python交互命令环境下不显示结果。
    d.pop(key)删除指定key和对应的值。
    dict和List的区别：
    dict的查找和插入速度极快，不会随着元素的增多而变慢。
    dict占用的内存较多。
    空间换时间。
    记住,dict中的key必须是不可变元素。
    key + Hash算法计算出值的内存地址。  
    
    s = set([1,2,3,3,4])
    s
    {1,2,3,4}
    set：创建一个set，需要传入一个List进入。
    add(key)
    remove(key)
    set相当于一个无序和不重复元素的集合。可以使用&和|来进行交并集计算。
    
    注意：创建空集合的时候只能用set来创建，因为在Python中{}创建的是一个空的字典：

    s = {}
    type(s)
    dict
    
    区别：唯一区别是set没有存储对应的value
    
    整数的内存地址是不可变的    
    对一些简单的数值，为了提高效率，python会重用对象内存：
    x = 2
    y = 2
    x is y
    True
    
    python的一些数据值被视为False的有：
    False
    0
    None
    空字符串、空列表、空集合、空字典
    
#### 不可变集合

    对应于元组（tuple）与列表（list）的关系，对于集合（set），Python提供了一种叫做不可变集合（frozen set）的数据结构。
    
    使用 frozenset 来进行创建：


    s = frozenset([1, 2, 3, 'a', 1])
    s
    frozenset({1, 2, 3, 'a'})
    
    与集合不同的是，不可变集合一旦创建就不可以改变。
    不可变集合的一个主要应用是用来作为字典的键。
    
#### 函数

    Python中的函数类似于数学中的函数。
    
##### 调用函数

[Python中内置的函数](http://docs.python.org/3/library/functions.html#abs)
    
    例如：
    
    计算类函数：abs(x)，max(...)。
    
    数据类型转换函数：int()，str(), bool(), float()。
    
    函数名复制给变量，该变量指向了该函数的地址。因而，具有函数的功能。
    a = abs
    a(-10)
    输出10。
    
##### 定义函数

    def myAbs(x)：
        if x >= 0：
            return x
        else： 
            return -x
    以def为前缀 + 函数名 + (参数...), return返回函数返回值，没有return则返回None， return = return None。
    
    空函数：使用pass构造空函数
    def test:
        pass
    也可以：
    if a > 0:
        pass
    
##### 参数类型检查

    使用isinstance检测参数类型：
    def myAbs(x):
        if not isinstance (x, (int, float)):
            raise TypeError("bad opread error")
        if x >= 0:
            return x
        else:
            return -x
        
##### 返回多个值

    当一个函数返回值有多个时，返回的是一个tuple，如(20, 30)。
    
##### 函数的参数

    位置参数：
    
    test(x)、test(x, y)x、y的参数定义即为位置参数。
    
    默认参数：
    
    def test(x , age = 3, city = 'shenzhen')，其中age和city为默认参数。
    1.传入test(0)即为传入test(0, 3, 'shenzhen')。
    2.传入test(0, 25)即为传入test(0, 25, 'shenzhen')。
    3.传入test(0, city = 'guangzhou')即为传入'guangzhou'，注意，当参数位置不对应时，需要指明参数类型，即city。
    4.默认参数必须指向不变对象，使用test(city = none)替代test(city = [])，
    写入
    if(city = none):
        city = []
    即可。

    额外的：为什么要设计str、none这样的不可变对象?
    可以避免在多线程中对象改变而造成的的错误，因此，尽量用不可变对象替代可变对象。
    
    可变参数：
    
    def test(*nums)
    1.可变参数在函数调用时自动组装成一个Tuple。
    2.nums可以是0个或多个数据。
    3.nums可以是一个List或者Tuple，此时*nums表示将List或者Tuple中的元素转化成可变参数传递进去。（内容拷贝）
    
    
    关键字参数：
    
    def test(**nums)
    1.关键字参数在函数调用时自动组装成一个dict。
    2.nums可以是0个或多个数据。
    3.nums可以是一个dict，此时**nums表示将dict中的元素转化为关键字参数传递进去。（内容拷贝）
    
    命名关键字参数：
    
    def test(a, *, b, c)，*，后面的为命名关键字参数。
    1.当函数中存在可变参数*x时，*x的作用等效于*，即此时，b、c也为命名关键字参数。
    2.调用含有关键字参数的函数时，应该使用key = value的形式，如本例：
    test(a, b = 1, c = 'haha')。
    3.当函数中指定了缺省值时，如def test(a, *x, b = 1, c)，此时，使用函数时可不填b参数。
    
    参数组合：
    
    5种参数的组合顺序为：
    位置参数、默认参数、可变参数、命名关键字参数、关键字参数。
    任意参数组合的函数都能给函数传入function(*x, **y)的组合传值形式。
    注意：参数组合过多会影响语义，尽量避免使用多参数组合。
    使用*args和**kw是习惯写法，建议遵循。
    
##### 递归函数

    1.优点：逻辑简单清晰，缺点：调用过深会导致栈溢出。
    2.可使用尾递归(返回自身本身)优化的方式避免栈溢出。
    3.大多数编程语言(包括Python)的编译器或解释器都没有针对尾递归进行优化。
    
#### 高级特性

    代码越少，效率越高。
    
##### 切片

    nums = list(range(100))
    切片：nums[0:2] == nums[:2]表示取下标为0到2(不包括2)的数据。
    倒数切片：nums[-2:0] == nums[-2:]表示取下标为-2到0(不包括0)的数据。
    nums[:10:2]前10个数，每2个取一个。
    nums[::5]所有数，每5个取一个。
    nums[::-1]取倒数。
    nums[:]输出该list。
    注意：nums指向的数据类型是什么，nums[...]取出来的数据类型就是什么。
    
##### 迭代器

    for i in nums
    不管是否有下标，只要能迭代，就能使用迭代器。
    对于dict，迭代的是key，
    迭代value：for i in nums.values()
    迭代key、value for i in nums.items()
    1.通过collections的Iterable来判读是否能迭代：
    from collections import Iterable
    isinstance('abcd', Iterable)
    2.使用内置的enumerate将list变成索引-元素对：
    for i, j in enumerate(['a', 'b', 'c']):
        print(i, j)
    
##### 列表生成式

    [i * i for i in range(1, 10)]
    [i * i for i in range(1, 10) if i % 2 == 0]
    [i * j for i in range(1, 10) for j in range(1, 10)]
    [i * j for i in range(1, 10) if i % 2 == 0 for j in range(1, 10) if j % 2 == 0]
    注意：'a' + 1，不同于java，python计算会出错。
    
    此外，列表生成式还可以生成集合和字典：
    {i ** 2 for i in range(1, 10)}
    {i : i ** 2 for i in range(1, 10)}
    
    可以使用sum()得到生成式的和：
    total = sum([i ** 2 for i in range(1, 10)])
    但是这样python为它生成了一个列表，并且由于没有变量指向它，它会被放在垃圾回收器中，因此，此时使用产生式列表替代它：
    total = sum(i ** 2 for i in range(1, 10))

    
###### 生成器

    g = (x * x for x in range(10))
    for i in g
    一边循环，一边计算的机制称为生成器。
    获取返回值，必须捕获StopIterable异常。返回值就在包含在StopIterable的value中。
    except StopIterable as e：
        print(e.value)
        break
    普通函数和generate函数的区别
    普通函数调用直接返回结果，generate函数调用返回generate对象。
    
    def fib(max):
    n, a, b = 0, 0, 1
    while n < max:
        yield b
        a, b = b, a + b
        n = n + 1
    return 'done'
    
    函数中使用yield关键字，每次调用next()时，遇到yield语句就返回，再次执行next()时，又继续从上次的位置往下执行。
    
##### 迭代器对象

    直接作用于for循环的数据类型有以下几种：
    1.集合类型：list、tuple、dict、set、str等等。
    2.generate类型：generate对象和generate函数。
    这些可直接作用于for循环的对象称为Iterable对象。
    
    1.可直接作用于next()函数的数据类型称为Iterator对象。
    所有的生成器都是Iterator，而list、dict、str则不是。
    why：Iterator至少需满足2个条件：
        1.长度不能够被提前知道。
        2.可以表示无限大的数据。
    
    2.可通过iter()函数来获得一个Iterator对象。
    3.python的for循环的本质就是不断调用next()函数来实现的。
    
### 函数式编程语言

    抽象程度很高的编程范式。只要输入确定，输出就确定，就是纯函数式编程语言。否则，则不是(当有变量存在函数中)。
    特点：允许将函数作为参数传入另一个函数，也可以返回一个函数。
    
#### 高阶函数

##### 变量可以指向函数

    x = abs
    x(-10)
    10
    说明变量指向函数
    
##### 函数名指向变量

    abs是函数名，同时它也是一个变量，它指向一个计算绝对值的函数。
    
##### 传入函数

    既然变量可以作为参数传入到函数中，那么因为函数名指向变量得缘故，所以函数中可以传入函数，传入函数作为变量的函数称为高阶函数。
    
##### map/reduce

    def f(x):
        return x * x
    y = map(f, [1, 3, 5])
    list(y)
    [1, 9, 25]
    map函数传入2个参数，第一个参数是函数，第二个是Iterable对象，map将传入的结果依次作用到Iterable对象的每一个元素，最后返回一个Iterator(map)对象。
    此外，map函数还可以作用于多个参数：
    def f(x, y):
        return x * y
    y = map(f, [2, 3], (2, 4))
    
    from functools import reduce
    def f(x, y):
        return x + y
    reduce(f, [1, 2, 3, 4]) == f(f(f(1, 2), 3), 4)
    reduce函数传入2个参数，第一个参数是函数，第二个是Iterable对象，reduce将每次取2个元素使用函数计算，得出的结果继续和下个元素传入函数做计算，依此类推。
    
##### filter

    def is_odd(x):
        return x % 2 == 1
    list(filter(is_odd, [1, 2, 3, 4]))
    [1, 3]
    过滤序列，同map/reduce一样，传入2个参数，第一个参数是函数，第二个是Iterable对象，对Iterable对象的每一个元素作用函数，返回True则保留，否则不保留。
    
##### sorted

    sorted(['Hello', 'JsonChao', 'quchao'], key = str.lower, reverse = True)
    对字符串排序是按首字母的ASCLL码对应的值大小来进行。
    key给要排序的元素作用同一个函数，reverse = True为反转排序结果。
    用sorted排序的关键在于实现一个映射函数。
    
#### 返回函数

##### 函数作为返回值

    def sum(*nums):
        def add():
            ax = 0
            for n in nums:
                ax = ax + i
            return ax
        return add
    其中，add是返回函数，输出add为函数本身，add()为函数返回值。
    注意：x1 = sum(1, 3, 5)
        x2 = sum(1, 3, 5)
        x1 == x2
        输出False。
        
#### 闭包

    注意：返回闭包时应牢记不要引用循环变量和后面会发生改变的变量。
    如要使用，在内部再创建一个函数，用该函数的参数绑定循环变量的值。
    
#### 匿名内部类

    使用lambda实现匿名内部类，形如：
    : 前面的为参数，lambda x, y : x * y
    无参则为lambda: x * x
    
#### 模块和包

##### __name__属性

    当我们想让当前.py文件既可以当成一个模块，又可以当成作为一个脚本使用
    时，可以写成如下：test()是该脚本中的方法。只有.py被当做脚本执行时
    ，__name__的值才会是__main__。

    if __name__ == ‘__main__’:
        test()

##### 导入模块或模块中的变量

    import os 
    导入模块
    from ex2 import PI
    导入ex2模块中的变量PI
    from ex2 import *
    导入所有变量
    
##### 包

    __init__文件存在文件夹中则表明这是一个包。
    
#### 异常

    try:
        ...
    catch ValueError as exc：
        exc.message
        
    捕捉单个异常
    
    try:
        ...
    catch (ValueError, ZeroDivisionError):
        ...
        
    try:
        ...
    catch ValueError:
        ...
    catch ZeroDivisionError：
        ...
        
    捕捉多个异常
    
    try:
        ...
    catch Expection：
        ...
        
    捕捉所有异常
    
    raise ValueError("Value error")
    
    使用raise来抛出异常
    
    try:
        ... ##有异常的代码块
    finally:
        ...
        
    finally会在try后执行，抛出异常前执行。
    
    try:
        ...
    catch Expection:
        ...
    finally:
        ...
    
    如果异常被捕获了，finally则会最后执行。
    
####  警告

    使用 warnings.warn(msg, RuntimeWarning)抛出警告，在调用抛出警告的方法前加上
    warnings.filterwarnings(action = 'ignore', category = RuntimeWarning)
    可过滤指定的警告。
    
#### 文件读写

    %%writefile test.txt
    ...
    
    写入文件
    
    f = open("test.txt")
    f = file("test.txt")

    打开文件
    
    f.read()
    
    读入文件夹中的所有内容。
    
    f.readLines()
    
    行读入文件，返回一个列表，格式为:
    [..../n..../n...]
    
    f.close()
    
    文件读写完毕，关闭文件。
    
    f = open('test.txt', 'w')
    f.write('hello, python, I come in~')
    f.close()
    print open('test.txt').read()
    
    使用open函数的写入模式来写文件,如果文件不存在，则创建该文件写入，
    如果之前已经存在该文件，则会把之前写入的内容覆盖。将'w'改为'a'，
    即使用追加模式加入新的内容到文件中。将'w'改为'w+'即可使用读写
    模式，使用f.read()即可读出文件内容。
    
    注意：二进制文件的写入，读取格式为'wb','rb'。
    
    
## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_play.jpg" width=20%><img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/Apaliy.jpg" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_qrcode.jpg" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。


    
    