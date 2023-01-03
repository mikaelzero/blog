### noexcept

指定某个函数不抛出异常, 属于优化操作.
比如, vector 通常保证强异常安全性, 如果元素类型没有提供一个保证不抛异常的移动构造函数, vector 通常会使用拷贝构造函数. 因此, 对于拷贝代价较高的自定义元素类型, 我们应当定义移动构造函数, 并标其为 noexcept, 或只在容器中放置对象的智能指针.

### explicit

C++提供了关键字 explicit, 可以阻止不应该允许的经过转换构造函数进行的隐式转换的发生,声明为 explicit 的构造函数不能在隐式转换中使用.
C++中, 一个参数的构造函数(或者除了第一个参数外其余参数都有默认值的多参构造函数), 承担了两个角色.
1 是个构造；2 是个默认且隐含的类型转换操作符.
比如 Point(int x) , 如果编写为 Point point = 1; 是可以编译通过的并创建一个 Point 对象, explicit 关键字就是为了避免这个情况, 指定这个构造器只能被明确的调用/使用

### extern

在 C 语言中, 修饰符 extern 用在变量或者函数的声明前, 用来说明“此变量/函数是在别处定义的, 要在此处引用”. extern 声明不是定义, 即不分配存储空间.
也就是说, 在一个文件中定义了变量和函数, 在其他文件中要使用它们, 可以有两种方式：

1. 使用头文件, 然后声明它们, 然后其他文件去包含头文件
2. 在其他文件中直接 extern
   extern"C" 作用：C++语言在编译的时候为了解决函数的多态问题, 会将函数名和参数联合起来生成一个中间的函数名称, 而 C 语言则不会, 因此会造成链接时无法找到对应函数的情况, 此时 C 函数就需要用 extern “C”进行链接指定, 这告诉编译器, 请保持我的名称, 不要给我生成用于链接的中间函数名,比如 JNI.

### const

● 当 const 在*的前面时, 允许修改指针 p 的位置, 而不允许修改*p 的值
● 当 const 在*的后面时, 指针 p 不能变, *p 可以变
● 当 const 在*的前后都有时, 指针 p 不能变, *p 也不可以变
● 顶层 const 表示 const 定义的变量本身是个常量, 底层 const 表示定义变量所指对象是一个常量
● const 放在返回值类型前面修饰返回值为常量, 放在函数后面修饰该函数为常量函数.

### static

● C 语言的 static 关键字的两种用法
○ 用于函数内部修饰变量, 即函数内的静态变量. 这种变量的生存期长于该函数, 使得函数具有一定的“状态”. 使用静态变量的函数一般是不可重入的, 也不是 线程安全的, 比如 strtok(3).
○ 用在文件级别(函数体之外), 修饰变量或函数, 表示该变量或函数只在本文件 可见, 其他文件看不到也访问不到该变量或函数. 专业的说法叫“具有 internal linkage”(简言之:不暴露给别的 translation unit).
● C++ 语言的 static 关键字的四种用法 : 由于 C++ 引入了 class, 在保持与 C 语言兼容的同时, static 关键字又有了两种 新用法:
○ 用于修饰 class 的数据成员, 即所谓“静态成员”. 这种数据成员的生存期大 于 class 的对象(实例/instance). 静态数据成员是每个 class 有一份, 普通 数据成员是每个 instance 有一份, 因此也分别叫做 class variable 和 instance variable.
○ 用于修饰 class 的成员函数, 即所谓“静态成员函数”. 这种成员函数只能访 问 class variable 和其他静态程序函数, 不能访问 instance variable 或 instance method.

### #与##运算符

● # 字符串化的意思, 出现在宏定义中的#是把跟在后面的参数转换成一个字符串.
○ #define MKSTR( x ) #x
○ cout << MKSTR(HELLO C++) << endl;
● ## 连接符号, 把参数连在一起.

### 共用体

共用体（union）, 也叫联合体是一种数据格式, 它能够存储不同的数据类型, 但只能同时存储其中的一种类型. 也就是说, 结构体可以同时存储 int、 long 和 double, 共用体只能存储 int、long 或 double. 共用体的句法与结构相似, 但含义不同. 如：
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/6TQDGx.jpg)
共用体每次只能存储一个值, 因此它必须有足够的空间来存储最大的成员, 所以, 共用体的长度为其最大成员的. 共用体的用途之一是, 当数据项使用两种或更多种格式（但不会同时使用）时, 比如一个 ID 属性有时候为 int 类型, 有时候为 char 类型, 可节省内存空间, 其实对当前内存充裕的计算机来说, 并非很有必要使用共用体, 对于嵌入式编程来说, 内存可能非常宝贵.

### 预定义宏

● **LINE**：这会在程序编译时包含当前行号.
● **FILE**：这会在程序编译时包含当前文件名.
● **DATE**：这会包含一个形式为 month/day/year 的字符串, 它表示把源文件转换为目标代码的日期.
● **TIME**：这会包含一个形式为 hour:minute:second 的字符串, 它表示程序被编译的时间.

### 内存四区

1. 代码区：你所写的所有代码都会放入到代码区中, 代码区的特点是共享和只读.
2. 全局区：存放的数据有：全局变量、静态变量、常量（如字符串常量）
   a. data 区：存放的是已经初始化的全局变量、静态变量和常量
   b. bss 区：存放的是未初始化的全局变量、静态变量, 这些未初始化的数据在程序执行前会自动被系统初始化为 0 或者 NULL
   c. 常量区：常量区是全局区中划分的一个小区域, 里面存放的是常量, 如 const 修饰的全局变量、字符串常量等
3. 栈区：由编译器自动分配释放, 存放函数的参数值、返回值、局部变量等
4. 堆区：用于动态内存分配, 位于 BSS 区和栈区之间

### override

复写某个方法时, 让编译器发现错误.
比如基类定义了一个 print 的函数, 子类如果写成了 print1, 那么无法发现错误, 如果子类加上了 override, 那么编译器马上会提示错误.

### （纯）虚函数

虚函数作用：允许用基类的指针来调用子类的这个函数
比如 Animal \*animal = new Cat; , Animal 中的 run 如果不加 virtual, 则该语句调用的时候调用的是 Animal 中的 run
纯虚函数：表示没有实现, 需要被实现, 类似接口功能, 起到一个规范的作用, 规范继承这个类的程序员必须实现这个函数.

### 纯虚函数和抽象类

● 在 C++中, 可以将虚函数声明为纯虚函数, 语法格式为：virtual 返回值类型 函数名 (函数参数) = 0;
● 纯虚函数没有函数体, 只有函数声明, 在虚函数声明的结尾加上=0, 表明此函数为纯虚函数.
● 最后的=0 并不表示函数返回值为 0, 它只起形式上的作用, 告诉编译系统“这是纯虚函数”.
● 包含纯虚函数的类称为抽象类（Abstract Class）. 之所以说它抽象, 是因为它无法实例化, 也就是无法创建对象. 原因很明显, 纯虚函数没有函数体, 不是完整的函数, 无法调用, 也无法为其分配内存空间.
● 抽象类通常是作为基类, 让派生类去实现纯虚函数. 派生类必须实现纯虚函数才能被实例化.