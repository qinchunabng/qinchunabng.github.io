---
layout: post
title: C语言中的常量和变量
categories: [C]
description: C语言中的常量和变量
keywords: C, 常量, 变量
---

### 变量和常量

#### 常量：在程序过程中值不会发生变化的量。
```
float f=0.1;
//1为常量，常量是不能赋值的
1=f;
```
分类：整型常量、实型常量、字符常量、字符串常量、标识常量。
- 整型常量： `1`，`790`，`76`，`52`
- 实型常量：`3.14`，`5.26`，`1.9999`
- 字符常量：有单引号引起来的单个字符或者转移字符，如：`'a'`,`'X'`,`'\n'`,`'\t'`,`'\015'`,`'\x7f'`
- 字符串常量：有双引号引起来的一个或者多个字符串组的序列，比如："","a","aXYZ","abc\n\021\018"
- 标识常量：#define定义的常量
  
  看如下代码：
  ```
  #include <stdlib.h>
  #include <stdio.h>

  #define PI 3.14

  int main(){
    int a,b,c;

    a*PI;

    b+PI;

    c/PI;

    exit(0);
  }
  ```
  通过命令`gcc -E main.c`预编译结果：
  ```
  int main(){
    int a,b,c;

    a*3.14;

    b+3.14;

    c/3.14;

    exit(0);
  }
  ```
  使用宏定义的好处，宏定义占用的是编译时间，能够提高程序的运行速度。例如下面例子，MAX和max实现效果是一样的，MAX占用的是编译时间，max占用的是运行时时间：
  ```
  #include <stdlib.h>
  #include <stdio.h>

  #define PI 3.14

  #define MAX(a,b) ((a)>(b)?(a):(b))

  int max(int a,int b){
      return a > b ? a : b;
  }

  int main(){
      int a=10,b=20;
      printf("a=%d,b=%d\n",a,b);
      printf("MAX(a,b)=%d\n", MAX(a,b));
      printf("max(a,b)=%d\n", max(a,b));
      printf("a=%d,b=%d\n",a,b);

      exit(0);
  }
  ```
  预编译后的结果：
  ```
  int max(int a,int b){
      return a > b ? a : b;
  }

  int main(){
      int a=10,b=20;
      printf("a=%d,b=%d\n",a,b);
      printf("MAX(a,b)=%d\n", ((a)>(b)?(a):(b)));
      printf("max(a,b)=%d\n", max(a,b));
      printf("a=%d,b=%d\n",a,b);

      exit(0);
  }
  ```

#### 变量：用来保存特定内容，在程序执行过程中随时会发生变化的量。

定义：[存储类型] 数据类型 标识符 = 值
                TYPE    NAME  = VALUE;
- 标识符：由字母，数字，下划线组成且不能以数字开头的标识序列。标识符尽量做到望文生义。
- 数据类型：基本数据类型，构造类型
- 存储类型：auto,static,register,extern(说明型关键字)
  - auto: 默认，自动分配释放空间。
  - register: （建议型）寄存器类型，只能定义局部变量，不能定义全局变量；大小有限制，只能定义32位大小的数据类型，如double就不可以，寄存器没有地址，所以一个寄存器类型的变量无法打印地址查看或使用。
  - static: 静态型，自动初始化为0值或空值，并且变量的值有继承性。同时，static修饰的变量只在当前文件范围有效，其他程序中定义同样的的static变量互不影响。同样static修饰的函数，在其他的.c文件无法使用，类似Java中的private，表示防止当前函数对外扩展。
  - extern: 说明型，说明变量的定义不在当前位置，意味着不能改变被说明的变量的值和类型。例如，两个.c文件中都定义了变量i，b.c文件中extern修饰的变量i的值和类型。
  
变量类型说明表：
<table>
  <tr>
    <th></th>
    <th colspan="3">局部变量</th>
    <th colspan="2">外部变量</th>
  </tr>
  <tr>
    <td>存储类别</td>
    <td>auto</td>
    <td>register</td>
    <td>局部static</td>
    <td>外部static</td>
    <td>外部</td>
  </tr>
  <tr>
    <td>存储方式</td>
    <td colspan="2">动态</td>
    <td colspan="3">静态</td>
  </tr>
  <tr>
    <td>存储区</td>
    <td>动态区</td>
    <td>寄存器</td>
    <td colspan="3">静态存储区</td>
  </tr>
  <tr>
    <td>生存期</td>
    <td colspan="2">函数调用开始至结束</td>
    <td colspan="3">程序整个运行期间</td>
  </tr>
  <tr>
    <td>作用域</td>
    <td colspan="3">定义变量的函数或复合语句内</td>
    <td>本文件</td>
    <td>其他文件</td>
  </tr>
  <tr>
    <td>赋初值</td>
    <td colspan="2">每次函数调用时</td>
    <td colspan="3">编译时赋初值，只赋一次</td>
  </tr>
  <tr>
    <td>未赋初值</td>
    <td colspan="2">不确定</td>
    <td colspan="3">自动赋初值0或空字符串</td>
  </tr>
</table>
  
- 局部变量默认未auto型
- register型变量个数受限，且不能未long,double,float型
- 局部static变量具有全局寿命和局部可见性
- 局部变量具有可继承性
- extern不是变量定义，可扩展外部变量作用域



