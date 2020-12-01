---
layout: post
title:  "Caffeine"
categories: tools
tags: tools
author: steve
---

* content
{:toc}










## 环境安装

### centos

```bash
yum -y install gcc
yum -y install gcc-c++
```

```c++
#include <iostream>
 
int count ;
//声明一个函数，这个函数在另外一个cpp文件中
extern void write_extern();
 
int main()
{
   count = 5;
   write_extern();
}
```

```c++
#include <iostream>

extern int count;
 
void write_extern(void)
{
   std::cout << "Count is " << count << std::endl;
}
```

```bash
g++ main.cpp support.cpp -o myapp
./myapp
```

### windows

## 语法

### 指针

```c++
#include <iostream>
 
using namespace std;
 
int main ()
{
   int  var = 20;   // 实际变量的声明
   int  *ip;        // 指针变量的声明
 
   ip = &var;       // 在指针变量中存储 var 的地址
 
   cout << "Value of var variable: ";
   cout << var << endl;
 
   // 输出在指针变量中存储的地址
   cout << "Address stored in ip variable: ";
   cout << ip << endl;
 
   // 访问指针中地址的值
   cout << "Value of *ip variable: ";
   cout << *ip << endl;
 
   return 0;
}

//带星号*代表访问的是指针对应的值
//不带星号*代表访问的是指针本身，也就是变量的内存地址，例如0xbfc601ac十六进制地址
```

#### 空指针

```c++
#include <iostream>

using namespace std;

int main ()
{
   //NULL指的就是为0的内存地址，是系统保留的地址
   int  *ptr = NULL;

   cout << "ptr 的值是 " << ptr ;
 
   return 0;
}

//if(ptr)     /* 如果 ptr 非空，则完成 */
//if(!ptr)    /* 如果 ptr 为空，则完成 */
```

#### 指针运算

```c++
//不同的类型移动的字节数不一样，int 移动4个字节32位；char 移动1个字节8位
ptr++
ptr--
//可以通过指针移动来操作数组元素，跟数组下标访问是一样的
#include <iostream>
 
using namespace std;
const int MAX = 3;
 
int main ()
{
   int  var[MAX] = {10, 100, 200};
   int  *ptr;
 
   // 指针中的数组地址
   ptr = var;
   for (int i = 0; i < MAX; i++)
   {
      cout << "var[" << i << "]的内存地址为 ";
      cout << ptr << endl;
 
      cout << "var[" << i << "] 的值为 ";
      cout << *ptr << endl;
 
      // 移动到下一个位置
      ptr++;
   }
   return 0;
}
```

#### 指针的指针（指针链）

![C++ 中指向指针的指针](https://www.runoob.com/wp-content/uploads/2014/09/pointer_to_pointer.jpg)

```c++
#include <iostream>
 
using namespace std;
 
int main ()
{
    int  var;
    int  *ptr;
    int  **pptr;
 
    var = 3000;
 
    // 获取 var 的地址
    ptr = &var;
 
    // 使用运算符 & 获取 ptr 的地址
    pptr = &ptr;
 
    // 使用 pptr 获取值
    cout << "var 值为 :" << var << endl;
    cout << "*ptr 值为:" << *ptr << endl;
    cout << "**pptr 值为:" << **pptr << endl;
 
    return 0;
}

var 值为 :3000
*ptr 值为:3000
**pptr 值为:3000
```

#### 指针和数组

```c++
//能接受指针作为参数的函数，也能接受数组作为参数
#include <iostream>
using namespace std;
 
// 函数声明
double getAverage(int *arr, int size);
 
int main ()
{
   // 带有 5 个元素的整型数组
   int balance[5] = {1000, 2, 3, 17, 50};
   double avg;
 
   // 传递一个指向数组的指针作为参数
   avg = getAverage( balance, 5 ) ;
 
   // 输出返回值
   cout << "Average value is: " << avg << endl; 
    
   return 0;
}
 
double getAverage(int *arr, int size)
{
  int    i, sum = 0;       
  double avg;          
 
  for (i = 0; i < size; ++i)
  {
    sum += arr[i];
   }
 
  avg = double(sum) / size;
 
  return avg;
}
```

#### 引用 vs 指针

引用很容易与指针混淆，它们之间有三个主要的不同：

- 不存在空引用。引用必须连接到一块合法的内存。
- 一旦引用被初始化为一个对象，就不能被指向到另一个对象。指针可以在任何时候指向到另一个对象。
- 引用必须在创建时被初始化。指针可以在任何时间被初始化。

### 参数调用

#### 传值调用

```c++
// 传值调用：参数的实际值复制给函数的形式参数，不会改变传入的变量的原始值
void swap(int x, int y)
{
   int temp;
 
   temp = x; /* 保存 x 的值 */
   x = y;    /* 把 y 赋值给 x */
   y = temp; /* 把 x 赋值给 y */
  
   return;
}
```

```c++
#include <iostream>
using namespace std;
 
// 函数声明
void swap(int x, int y);
 
int main ()
{
   // 局部变量声明
   int a = 100;
   int b = 200;
 
   cout << "交换前，a 的值：" << a << endl;
   cout << "交换前，b 的值：" << b << endl;
 
   // 调用函数来交换值
   swap(a, b);
 
   cout << "交换后，a 的值：" << a << endl;
   cout << "交换后，b 的值：" << b << endl;
 
   return 0;
}
```

#### 指针调用

```c++
//指针调用：把参数的地址复制给形式参数，会改变传入变量的原始值
void swap(int *x, int *y)
{
   int temp;
   temp = *x;    /* 保存地址 x 的值 */
   *x = *y;        /* 把 y 赋值给 x */
   *y = temp;    /* 把 x 赋值给 y */
  
   return;
}
```

```c++
#include <iostream>
using namespace std;

// 函数声明
void swap(int *x, int *y);

int main ()
{
   // 局部变量声明
   int a = 100;
   int b = 200;
 
   cout << "交换前，a 的值：" << a << endl;
   cout << "交换前，b 的值：" << b << endl;

   /* 调用函数来交换值
    * &a 表示指向 a 的指针，即变量 a 的地址 
    * &b 表示指向 b 的指针，即变量 b 的地址 
    */
   swap(&a, &b);

   cout << "交换后，a 的值：" << a << endl;
   cout << "交换后，b 的值：" << b << endl;
 
   return 0;
}
```

#### 引用调用

```c++
//引用调用：把引用的地址复制给形式参数
void swap(int &x, int &y)
{
   int temp;
   temp = x; /* 保存地址 x 的值 */
   x = y;    /* 把 y 赋值给 x */
   y = temp; /* 把 x 赋值给 y  */
  
   return;
}
```

```c++
#include <iostream>
using namespace std;
 
// 函数声明
void swap(int &x, int &y);
 
int main ()
{
   // 局部变量声明
   int a = 100;
   int b = 200;
 
   cout << "交换前，a 的值：" << a << endl;
   cout << "交换前，b 的值：" << b << endl;
 
   /* 调用函数来交换值 */
   swap(a, b);
 
   cout << "交换后，a 的值：" << a << endl;
   cout << "交换后，b 的值：" << b << endl;
 
   return 0;
}
```

#### 匿名函数，lambda表达式

```c++
[capture](parameters)->return-type{body}

//capture代表lambda表达式里面访问外部变量的方式，指定值传递还是引用传递
[]      // 沒有定义任何变量。使用未定义变量会引发错误。
[x, &y] // x以传值方式传入（默认），y以引用方式传入。
[&]     // 任何被使用到的外部变量都隐式地以引用方式加以引用。
[=]     // 任何被使用到的外部变量都隐式地以传值方式加以引用。
[&, x]  // x显式地以传值方式加以引用。其余变量以引用方式加以引用。
[=, &z] // z显式地以引用方式加以引用。其余变量以传值方式加以引用。


[](int x, int y) -> int { int z = x + y; return z + x; }
//没有返回值，没有入参，则可以忽略
[]{ ++global_x; } 
```

## cmake

[cmake-demo](https://github.com/wzpan/cmake-demo)

## Java调用C++



## SeetaFace6

编译

下载解压[cmake](https://cmake.org/download/)，配置PATH环境变量

安装Visual Studio2019社区版

添加路径`C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.27.29110\bin\Hostx64\x64`到PATH环境变量

添加路径`C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.27.29110\lib\x64`到PATH环境变量

下载解压[qt-jom](https://wiki.qt.io/Jom)，配置PATH环境变量

## 参考
