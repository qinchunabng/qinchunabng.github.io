---
layout: post
title: C语言中的数据类型
categories: [C]
description: C语言中的数据类型
keywords: C
---

### 基本数据类型


<table>
    <tr>
        <th>类型</th>
        <th>符号</th>
        <th>关键字</th>
        <th>所占位数</th>
        <th>数的表示范围</th>
    </tr>
    <tr>
        <td rowspan="9">整型</td>
        <td rowspan="4">有</td>
        <td>(signed)int</td>
        <td>32</td>
        <td>-2147483648~2147483647</td>
    </tr>
    <tr>
        <td>(signed)short</td>
        <td>16</td>
        <td>-32768~32767</td>
    <tr>
    <tr>
        <td>(signed)long</td>
        <td>32</td>
        <td>-2147483648~2147483647</td>
    <tr>
    <tr>
        <td rowspan="4">无</td>
        <td>unsigned</td>
        <td>32</td>
        <td>0~4294967295</td>
    <tr>
    <tr>
        <td>unsigned short</td>
        <td>16</td>
        <td>0~65535</td>
    </tr>
    <tr>
        <td>unsigned long</td>
        <td>32</td>
        <td>0~4294967295</td>
    </tr>
    <tr>
        <td rowspan="2">实型</td>
        <td>有</td>
        <td>float</td>
        <td>32</td>
        <td>3.4e-38~3.4e38</td>
    </tr>
    <tr>
        <td>有</td>
        <td>double</td>
        <td>64</td>
        <td>1.7e-308~1.7e308</td>
    </tr>
    <tr>
        <td rowspan="2">字符型</td>
        <td>有</td>
        <td>char</td>
        <td>8</td>
        <td>-128~127</td>
    </tr>
    <tr>
        <td>无</td>
        <td>unsigned char</td>
        <td>8</td>
        <td>0~255</td>
    </tr>
</table>

整型数在计算机中存储是以补码的形式存储的，正数的补码是正数二进制码本身，负数是负数的绝对值的二进制码取反加1。
```
-254 -> 254 -> 1111 1110取反 + 1 -> 0000 0010
```

进制转换示例：
```
(254)10 = (B1111 1110)2 = (0376)8 = (0xFE)16
```

### 变量和常量

#### 常量：在程序过程中值不会发生变化的量。
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
```
float f=0.1;
//1为常量，常量是不能负值的
1=f;
```