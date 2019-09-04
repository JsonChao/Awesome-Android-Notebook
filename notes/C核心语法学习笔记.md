    注意：本笔记为C核心语法学习笔记，为笔者快速复习C时使用，更完整的教程请查看更专业的C教程。

1、0xA3U：表示无符号的十六制数0xA3

2、字符常量：‘a' == 97

3、字符串常量："aa"，系统会在存储字符串的时候自动加上字符串结束标志：’\0'

4、符号常量/宏常量：#define 常量名 常量值 #define PI 3.1415926

5、int a = 1 &a：内存地址的值

6、sizeof：某个数据类型所占内存空间大小

7、C语言没有String类型，只能使用字符数组来存储字符串，最后要加'\0'

8、string.h中的字符串操作函数：

- strcpy(a,b)：将b的字符串复制到a中
- strcat(a,b)：将字符串b的内容放到a后面
- strcmp(a,b)：从左到右，按ASCll码比较，一样返回0，大于返回正整数，小于返回负整数
- strlen(a)：返回字符串长度，不包括‘\0'

9、指针、数组（一维数组、二维数组、字符数组、字符串数组）传递的都是引用

10、struct 结构体变量的存储空间一般是成员变量的空间和，使用sizeof(struct student)获得结构体所占空间大小

11、union 共用体

- 同一时间只有一个成员的值有意义
- 各个成员共享同一块存储空间
- 共用体变量的大小取决于成员数据类型最大的那个

12、typedef 

    typedef char NAME[20]；
    NAME a1,a2; ==> char a1[20],a2[20]
    
    typedef struct Student{...} STU 
    STU s1 = {...}; ==> struct Student s1 = {...};

13、指向函数的指针：函数的首地址一般称为函数的指针。

    int max(int a,int b){...} 
    int min(int a,int b){...}
    
    void point(int x,int y,int (*p)()){ 
        ... 
        res = (*p)(x,y);
        ...
    }
    
    int main() {
        ... 
        point(a,b,min);
        point(a,b,max);
    }

14、static变量

- 局部：生存期为整个源程序，只能在定义该变量的函数内使用。
- 全局：只在定义该变量的源文件有效，而非static变量则在整个源程序内有效。
- static函数：只能在本文件的函数中调用。

15、指针函数：返回值是指针或地址

    char *day_name(int n)
    {
        ...
    }
    
16、带参数的主函数：

    int main(int argc,char *argv[])
    {
        ...
    } 参数个数 = 程序名个数1个 + 本身参数1个 + argv[]个数 
    
17、二维数组和指针数组

    char name[4][15] = {"","","",""}
    每个字符串的内存分配都是头尾相连，连续分配，节省内存。
    
    char *p_name[4] = {"", "","",""}
    当字符串所需内存空间不同时，由于每个字符串的内存分配不是连续的，所以浪费内存。
    
18、二级指针：指向一个指针变量的指针，通常用于存储指针数组元素的地址以实现指针数组元素的间接访问。

19、内存动态分配：

- malloc()：分配若干字节的连续内存空间，分配成功返回该存储区首地址，否则返回空指针(0)。
 

    int *p;
    p = (int*) malloc(10 * sizeof(int));

- calloc()：同上，但使用方式不同。


    int *p;
    p = (int*) calloc(10, sizeof(int));
    
- realloc()：将p所指向的存储区域大小更改为 5 * sizeof(int)。


    int *q;
    q = - (int*) realloc(p, 5 * sizeof(int));
    
- free()：释放动态申请的空间给系统，由系统重新支配。


    free(p);

20、const修饰指针变量：具有不变性

    const int *const A;//指针a和a指向的内容都不可变
    
21、位运算：
按位异或：相同为0，否则为1

22、文件基础，基本格式：

    FILE *fp; 
    fp = fopen("..."); 
    if(fp == NULL) {
    ..
        
    } 
    fclose(fp);
    
- rewind(fp)：移动位置指针重新回到文件开头。
- fseek(fp，位移量，起始点)：文件定位。位移量为long类型。
- ftell(fp)：返回位置指针的相对位移量。

23、文件读写：

- 字符读写：fgetc和fputc。
- 字符串读写：fgets和fputs。
- 格式化读写fprint：控制台 -> 文件；fscanf：文件 -> 控制台；
- 数据块写入fwrite()：结构体 -> 文件 fread()：文件 -> 结构体; 参数为
数据起始地址，每次写入数据块的字节流，数据块的数目，文件指针
- 字读写（整数）：getw和putw。

24、变量的作用域和生命周期：

- auto（自动变量）：默认类型。
- static（静态变量）：编译时初始化，生命周期是整个程序的运行周期。
- register（寄存器变量）：用于声明频繁读写的变量。
- extern（外部变量）：用于扩充作用域。

25、编译预处理：

- 宏定义
- 文件包含
- 条件编译 三种形式：

    
    #ifdef
    #ifndef 
    #if

