    注意：本笔记为C++核心语法学习笔记，为笔者快速复习C++时使用，更完整的教程请查看更专业的C++教程。

### C/C++区别：

- 1、C：字符常量 -> 整数 C++：字符常量 -> 字符
- 2、C：main函数可调用 C++：main函数不可调用
- 3、C：可取寄存器变量地址 C++：不可取寄存器变量地址
- 4、C++增加了bool基本类型和wchar_t扩展类型
- 5、C++可省略“struct 结构体名 变量名” 中的struct

### C++新特性：

1、名字空间：防止名称冲突而声明的领域 ，可省略iostream.h后面的h

    # include <iostream> 
    using namespacpe std；
    
2、新增string类型

3、控制台输入和输出 ：

    # 输出
    count << “...";
    # 输入
    cin >> j; 

### 一、C++名字空间

#### 1、名字空间的定义

##### 定义：

    namespace na { 
        void print(int n) 
        {
            count << "na::print: " << n <<endl;
            
        } 
    }
    
##### 使用：

    na::print(3)
    
注意：同一个名字空间可以定义多次。

#### 2、using的使用

using可使用多次，相当于java中的import，防止滥用。

#### 3、别名代替嵌套名字空间

    namespace na {
        namespace nb { 
            namespace nc { 
                int sum(int a, int b) 
                {
                .. 
                } 
            } 
        } 
    }
    
    namespace A = na::nb::nc;
    int main () 
    { 
        count << A::sum(6, 12) << endl;
    }
    
#### 4、无名名字空间

内部可以引用，外部不能引用。

    namespace na 
    { 
        namespace 
        {
        
        } 
    }

### 二、参数的缺省值

某个参数设置了缺省值，后面的参数也必须设置。

### 三、函数的重载

同Java，参数缺省值的设置，实际上在编译时等价于重载，被生成了多个函数。
注意自动类型转换时引发的错误。

### 四、C++引用

引用性变量可以看做引用型变量的“别名”，好处是可以“逆向引用”。

- 1、引用型变量定义时必须初始化，确保不为空或空指针。
- 2、被引用的对象不能为地址，即指针变量、数组变量等不能被引用。
- 3、逆向引用：
通过函数结果设置改变源头设置

     ## 1
     int &f(); 
     int x; 
     
     int main() 
     {
         ## 3
         f() = 100;
         ...
     } 
     
     ## 2
     int &f()
     {
         return x;
     }
     

### 五、类相关定义

    class CSample 
    {
        public: 
            int x1;
            CSample();
        protected:
            int a;
            float sum(float f1, float f2);
        private(一般放在类下面，且可以直接省略):
            int m;
            double sum(double f1, double f2);
        ...
    };
    
#### 1、构造函数和析构函数

- 构造函数：初始化时调用。
- 析构函数：类的实例被销毁时调用，例如释放动态分配的内存。

#### 2、成员函数的定义和声明分开

    // test.h
    class CA {
        void a;
        public:
            CA();
            ~CA();
            void setA(int x);
            void print();
    };
    
    // text.cpp
    #include  <iostream>
    #include "test.h"
    using namespace std;
    
    CA::CA() 
    {
        ...
    }
    
    CA::~CA() 
    {
        ...
    }
    
    void CA::setA(int x) 
    {
        ...
    }
    
    void CA::print()
    {
        ...
    }
    
#### 3、生成实例的三种方式

##### 1、声明为变量 

    string strA("...")
    
##### 2、从无名对象复制 

    string strB; 
    strB = string("...");
    
##### 3、声明为指针并动态生成 

    string *strC; 
    strC = new string("...");
    # 释放内存
    delete strC;

### 六、成员函数和运算符的重载

#### 1、构造函数的重载

    class stuff {
        string name;
        int age;
        public:
            stuff() 
            {
                ...
            }
            # 在成员变量分配空间时，将参数的值赋值给成员变量
            stuff(string n, int a):name(n),age(a)
            { 
                count << name << "---" << age << endl; 
            }
    };
            
#### 2、运算符重载（区别于普通函数和成员函数）

    class stuff {
        string name;
        int age;
        public:
            ...
            void operator +(int x);
            void operator +(string s);
    };
    
    void stuff::operator +(int x)
    { 
        age = age + x; 
    }
    
    void stuff::operator +(string s)
    { 
        name = name + s;
    }
    
    int main () 
    { 
        stuff st2("...", 3);
        st2 + 2;
        st2 + "..."; 
    }
    
#### 3、拷贝构造函数和赋值运算符

    // 赋值运算符重载
    stuff& operator = (stuff& x); 
    
    // 拷贝构造函数重载
    stuff(stuff& x):name(x.name),age(20)
    {
        ...
    } 
    
    stuff& stuff::operator =(stuff& x) 
    {
        ... 
        return *this; 
    }
    
    int main() 
    {
        stuff st("...", 3); 
        // 不产生新的实例，使用的是赋值运算符 
        st1 = st; 
        // 产生新的实例，使用的是拷贝构造函数
        stuff st2 = st; 
    }
    
#### 4、类型转换

    // stuff -> int
    operator int() 
    {
        return age; 
    } 
    
    // stuff -> string
    operator string() 
    {
        return name; 
    } 
    
    int main() 
    { 
        stuff st("...", 3); 
        string m_name = st; 
        int m_age = st; 
    }

### 七、C++继承

#### 1、公有继承

**将基类的公有成员变成自己的公有成员，就基类的私有成员变成自己的私有成员。**

- 基类的私有成员：在派生类和外部都不可以访问。
- 基类的公有成员：在派生类和外部都可以访问。
- 基类的保护成员：在派生类可以访问，在外部不可以访问。

#### 2、私有继承

**将基类的公有成员和私有成员变成自己的私有成员。**

- 基类的私有成员：在派生类和外部都不可以访问。
- 基类的公有成员：在派生类可以访问，在外部不可以访问。
- 基类的保护成员：在派生类可以访问，在外部不可以访问。

#### 3、保护继承

**将基类的公有成员和私有成员变成自己的保护成员。**

- 基类的私有成员：在派生类和外部都不可以访问。
- 基类的公有成员：在派生类可以访问，在外部不可以访问。
- 基类的保护成员：在派生类可以访问，在外部不可以访问。
- 私有继承和保护继承的区别：派生类再派生时，如果是私有继承则都不可以访问了，但是保护继承的公有和保护成员还是可以访问。

### 八、C++基类和派生类的构造函数

#### 1、缺省构造函数的调用关系

创建时先基类后派生类，销毁时先派生类后基类。

#### 2、有参数时的传递

当有参数时，参数必须传送给基类。

    class CBase {
        string name;
        public:
            CBase(string s) : name(s) 
            {
                ...
            }
    };
    
    class CDerive : public CBase {
        int age;
        public:
        CDerive(string s, int a) : CBase(s), age(a)
        {
            ...
        }
    };
        
#### 3、祖孙三代的参数传递

形式同2一样。

#### 4、多重继承的参数传递

    class CDerive : public CBase1, public CBase2 {
    
        string id;
        public:
            CDerive(string s1, int a, string s2) : CBase1(s1), CBase2(a), id(s2) 
            { 
                ...     
            }
    };
            
这种情况会先执行先继承的那个基类的构造函数，即CBase1的构造函数。

### 九、C++派生类中调用基类成员

#### 1、基类和派生类具有相同成员时（注意：参数不同也要加上基类名::）

    class CDerive : public CBase {
        string id;
        int age;
        public:
            CDerive(string s1, int a, string s2) : age(a), id(s2), CBase(s1, s1) 
            {    
                ... 
            }
    
        void show()
        {
            CBase::show(); 
        }
    };
    
    int main() 
    { 
        CDerive d("...", 3, "..."); 
        d.CBase::show(); 
    }

#### 2、多重继承的调用方法

    class CDerive : public CBase1, public CBase2 {
    
        public:
            ...
            void show() 
            {
                CBase1::show();
                CBase2::show();
                CBase1::id;
                CBase2::id;
            }
    };

#### 3、删除派生类中的同名函数

如果有多个基类，且基类中有多个同名方法，则编译出错，不知从哪个基类继承。

### 十、C++多重继承问题

#### 1、多重继承的先后问题（菱形问题）

    class CBase {
        public:
        string id;
    };
    
    class CDerive1 : public CBase {
        ...
    };
    
    class CDerive2 : public CBase {
        ...
    };
    
    class CSon : public CDerive2, public CDerive1 {
        ...
    };
    
    int main() {
        CSon s;
        s.CDerive1::id = "1";
        s.CDerive2::id = "2";
        s.CBase::id = "3";
        s.show1();
        s.show2();
        cout << s.CBase::id << endl;
        return 0;
    }
    
默认将第一个继承的“父亲”的“父亲”当做"爷爷"。"s.CBase::id" 等价于 "s.CDerive2:CBase::id"。

#### 2、实例地址的调查

代码同1，补充如下。

    int main() {
        Cson s;
        CDerive1 *pd1 = $s;
        CDerive2 *pd2 = $s;
        CBase *pb1 = pd1;
        CBase *pb2 = pd2;
        // CBase *pb = &s; // 编译出错
        return 0；
    }
    
可以看出，第一个继承的CDerive2是可以接受到同名地址的，而其后的地址CDerive1会不同。

#### 3、虚继承

基类不管继承多少个，"父亲们"都不产生"爷爷"的副本，"孙子"只有一个"爷爷"的副本。

虚继承形式如下:

    class CDerive1 : virtual public CBase {
        ... 
    }
    
### 十一、C++虚函数和纯虚函数

#### 1、虚函数（类似于Java中抽象类的抽象方法，但抽象方法有实现代码）

当基类和派生类有同名函数时，此时将派生类对象转换为基类对象，在调用该基类对象的方法，会执行该基类的同名函数。为了解决这个问题，可以使用虚函数，其定义形式如下：

    ...
    
    calss CBase {
    public：
        virtual void show() {
            cout << "BASE SHOW" << endl;
        }
        virtual void print() {
            cout << "BASE PRINT" << endl;
        }
    };
    
    ...

此时，再次调用该基类的对象方法，会执行派生类的同名函数。可以得出虚函数只影响派生类的实装，不影响基类自身的实例。

补充："虚继承"和"虚函数"的区别

- 1、"虚继承"是针对基类的成员变量的二义性，而"虚函数"是针对成员函数的二义性。
- 2、"虚继承"是将"virtual"加在派生类的定义处，而"虚函数"是将"virtual"加在基类成员函数的定义处。

#### 2、纯虚函数（类似于Java中抽象类的抽象方法）

为了控制基类的成员函数在基类什么也不做，让派生类去实现该成员函数。可以使用"纯虚函数"，含有它的类不能产生实例，称为"抽象类"，"纯虚函数"的定义如下：

    ...

    class CBase {
    public:
        virtual void show() = 0;
        ...
    };
    
    class CDerive : public CBase {
    public:
        # 派生类中必须实装"纯虚函数"
        void show() {
            ...
        }
    }

    ...
    
### 十二、C++ 实例指针 this

"this"表示类的实例指针。

#### 1、参数与成员变量同名

    class ca {
        string id;
    public:
        ca(string id) {
            # 类似于Java中的this.id
            this->id = id;
        } 
        
        void setId(string id) {
            this->id = id;
        }
        ...
    }
    
    ...

这样就将外部传入的参数id赋值给当前类的全局变量id。

#### 2、判断一个对象是否为自己

可使用if(&obj == this) { ... }的形式，示例如下：

    ...

    class base {
    public:
        void check(base *obj) {
            if (obj == this) {
                cout << "这是当前的对象 " << endl;
            } else {
                cout << "这不是同一个对象 " << endl; 
            }
        }
    };
    
    ...

#### 3、运算符重载

    ...
    
    class rect {
        int x1, y1, x2, y2;
    public:
        ...
        
        rect operator++() {
            x1++, y1++, x2++, y2++;
            
            return *this;
        }
    };
    
    ...
    
### 十三、C++友元函数(friend)

作用：在类的外部，友元函数能通过实例来访问类的私有或保护成员。

注意点：

- 友元函数必须在类里面声明，而且友元函数一定不是该类的成员函数。
- 友元函数的声明在派生类无效。

#### 1、全局函数作友元

实例如下：

    ...
    
    class ca {
        string id;
        ...
    protected:
        string name;
        void setName(string s) {
            name = s;
        }
    public:
        ...
        // 在类里面声明全局函数作友元
        friend void fun(ca& a);
    }
    
    void fun(ca& a) {
        # 外部可通过友元函数访问私有或保护成员的值
        a.id = "987";
        a.setName("xyz");
    }
    
    int main() {
        ca a;
        fun(a);
        a.print(); // 输出987 xyz
    }
    
#### 2、其它类的成员函数作友元

注意将类的声明、定义和实装分开，示例如下：

    ...
    
    // 实现声明ca类
    calss ca;
    
    class cb {
    public:
        void test(ca& a);
    };
    
    class ca {
        ...
    public:
        ...
        // 在ca类中可以声明cb类的test()函数作友元
        friend void cb::test(ca& a);
    };
    
    void cb::test(ca& a) {
        ad.id = "123";
        a.setName("abc");
    }
    }
    
    
#### 3、运算符重载中使用友元

以上实例纯粹为区分类的运算符重载和全局运算符重载。一般使用前者即可。

    ...
    
    class rect {
        ...
    public:
        ...
        // rect operator++(); 类的运算符重载
        // 全局运算符重载，不是所有操作符都可以定义成"友元"，如"="
        friend rect operator++(rect &ob);
    };
    
    rect operator++(rect &ob) {
        ...
    }
    
### 十四、C++构造函数、explicit、类合成

#### 1、但成员变量类实例赋值会发生缺省转换

这种转换是编译期造成的，示例如下：

    ...
    
    class classA {
        int x;
    public:
        classA(int x) { this->x = x; }
        classA(char *x) { this->x = atoi(x); }
        ...
    };
    
    int main() {
        classA ca(5);
        
        # 构造函数参数重新赋值
        // 缺省调用classA(100)
        ca = 100; 
        // 缺省调用classA("255")
        ca = "255";
    }
    
#### 2、用explicit禁止默认转换

示例如下：

    ...
    
    class classA {
        int x;
    public:
        # 使用explicit声明，使构造函数参数不能够重新赋值
        explicit classA(int x) { this->x = x; }
        explicit classA(char *x) { this->x = atoi(x); }
        ...
    };
    
    int main() {
        classA ca (5);
        
        // 编译均出错
        ca = 100;
        ca = "255";
    }
    
#### 3、类的合成

"合成"即指一个类的成员变量是另一个类的情况。这种情况在构造函数赋值时只能适用创建时赋值的方式。

    ...
    
    class classB {
        ...
    }
    
    class classA {
        classB xb;
    public:
        // classA(classB b) { xb = b; } // 编译出错
        # 类的合成只能使用创建时合成的方式
        classA(classB b) : xb(b) { }
        ...
    }
    
### 十五、C++ new 和 delete 的使用

#### 1、基本数据类型的动态分配

new和delete完全包含malloc和free的功能，记得释放内存和出错处理。

    // 调用构造函数
    int *j = new int; 
    int *iArr = new int[3];
    
    // 调用析构函数
    delete i;
    delete []iArr;
    
#### 2、内存分配时的出错处理

有2种方案：

- 1、根据指针是否为空来判断。(一般使用这种)
- 2、根据根据出错处理try/catch来处理。

    ...

    // 1
    double *arr = new double[10];
    # 使用if(!arr)的形式判断指针是否为空
    if (!arr) {
        count << "内存分配出错！" << endl;
        return 1;
    }
    delete []arr;
    
    // 2
    try {
        double *p = new double[10];
        delete []p;
    } catch (bad_alloc xa) {
        count << "内存分配出错！" << endl;
        return 1;
    }
    
    // 2、使用new(nothrow)进行强制例外，不抛出错误
    dobule *ptr = new(nothrow) double[10];
    if (!ptr) {
        count << "内存分配出错！" << endl;
        return 1;
    }
    delete []ptr;
    
    ...
    
### 十六、C++静态类成员

C++相比C，static除了修饰变量外还可以修饰类的成员。以下是静态类成员的一些特性：

- 修饰全局变量，仅在当前文件有效。
- 修饰的变量或函数具有共有性。
- static修饰的变量的初始值为0。
- 在类的静态成员函数中不能使用this指针，也不能作成静态虚函数。
- 类的静态成员函数不能用const、volatile申明。

#### 1、静态成员变量

静态成员变量必须在类外初始化。

    ...
    
    class classA {
        static int sx;
        static string sstr;
    public:
        static int sy;
        ...
    }
    
    // 在类外申明并初始化静态成员变量，为0或""
    int classA::sx;
    string classA::sstr;
    int classA::sy;
    
    int main() {
        
        classA ca1, ca2;
        
        // static修饰的值是公有的，所有ca2中的3个值也会改变
        ca1.set(25, "C++");
        ca1.sy = 100;
        // 公有静态成员也可以通过类名来获取
        classA::sy = 125;
        
        ...
    }
    
#### 2、静态成员函数

可以通过类名来调用公共静态成员函数，示例如下：

    class Integer {
    public:
        static int atoi(const char *s) {
            return ::atoi(s);
        }
        static float atof(const char *s) {
            return ::atof(s);
        }
    };
    
    int main() {
        // 可以通过类名来调用静态成员函数
        int x = Integer::atoi("322");
        float y = Integer::atof("3.14");
        ...
    }

### 十七、C++ const 和 mutable 的使用

#### 1、const修饰变量

常量定义必须有初始值，示例如下：

    ...
    
    struct mytype {
        const int x;
        ...
    };
    
    int main() {
        // 常量定义必须有初始值
        mytype s1 = {15, 30};
    }

#### 2、const修饰类的成员函数

作用：可以使函数内不能对任何成员变量修改，示例如下：

    ...

    class classA {
        int x;
        string y;
    public:
        void get(int& i, string& s) const {
            int k;
            // 非成员变量可以修改
            k = 10;
            // 成员变量不可以修改
            // x = k;
            // y = "C++";
            i = x, s= y;
        }
        ...
    };
    
    ...
    
#### 3、mutable与const为敌

在上述示例中，将y在申明时定义为mutable（易变的）之后，可以在const修饰的成员函数中修改y的值，如下所示：

    ...

    class classA {
        int x;
        mutable string y;
    public:
        void get(int& i, string& s) const {
            int k;
            // 非成员变量可以修改
            k = 10;
            // 成员变量不可以修改
            // x = k;
            // mutable变量可以被修改
            y = "C++";
            i = x, s= y;
        }
        ...
    };
    
    ...
    
### 十八、C++ const修饰指针

#### 1、const修饰某类型的指针

特点：变量的地址可以改变，值和类型不能改变，示例如下：

    ...
    
    int main() {
        ...
        
        const int * p1;
        
        // 不能改变类型
        // p1 = &c;
        
        // 不能修改p1的值
        // *p1 = 156;
        
        ...
    }
    
#### 2、const修饰指针后再限定类型

"int const * p2",将指针变量p2作为常量再限制为int类型，同const修饰某类型的指针变量特点一样。

#### 3、const修饰指针地址

"int * const p3",将地址作为常量并把指针限制为int类型。变量的地址不能改变，内容可以改变。
定义时必须同时赋初始值（这一点同Java的final一样必须赋初始值）。

    ...
    
    int main() {
        ...
        
        // 初始化时必须赋值
        int * const p3 = &x;
        
        ...
    }
    
#### 4、const修饰某类型的指针并修饰地址

"const in * const p4", 地址和内容都不能修改。

