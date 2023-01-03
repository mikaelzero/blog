### 声明指针

星号两边的空白符无关紧要, 下面的声明都是等价的：

```cpp
int* pi;
int * pi;
int *pi;
int*pi;
```

### 如何阅读声明

> const int \*pci;
> ![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/dazTGQ.jpg)

### 指针相关的预定义类型

使用指针时经常用到以下四种预定义类型.
● size_t 用于安全地表示长度
○ size_t 类型表示 C 中任何对象所能达到的最大长度
○ 它是无符号整数, 因为负数在这里没有意义
○ 目的是提供一种可移植的方法来声明与系统中可寻址的内存区域一致的长度
○ size_t 用做 sizeof 操作符的返回值类型, 同时也是很多函数的参数类型, 包括 malloc 和 strlen
○ 当需要用指针长度时, 一定要用 sizeof 操作符
● ptrdiff_t 用于处理指针算术运算
● intptr_t 和 uintptr_t 用于存储指针地址
○ intptr_t 和 uintptr_t 类型用来存放指针地址
○ 它们提供了一种可移植且安全的方法声明指针, 而且和系统中使用的指针长度相同, 对于把指针转化成整数形式来说很有用
○ uintptr_t 是 intptr_t 的无符号版本

### 指针操作符

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/yjgO5R.jpg)
指针的常见用法
● 多层间接引用
○ 指针可以用不同的间接引用层级. 把变量声明为指针的指针并不少见, 有时候称它们为双重指针
● 常量指针
○ 可以将指针定义为指向常量, 这意味着不能通过指针修改它所引用的值
○ 可以改变指针
○ 数据类型和 const 关键字的顺序不重要, const int *a 和 int const *a 是等价的
○ ![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/h4cl9V.jpg)
