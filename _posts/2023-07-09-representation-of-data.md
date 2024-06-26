---
layout: post
title: 数据的表示
categories: [计算机]
description: 数据的表示
keywords: 计算机
---

## 数据的表示

### 原码、反码、补码和移码

- 原码：一个符号位加上其绝对值的二进制。
  
  例子，如果机器字长为8，9、-9源码的表示：
  ```
  9 = 0 000 1001
  -9 = 1 000 1001
  ```

- 反码：正数的是其源码，负数的反码符号位不变，原码的其余位取反。
  
  例子，如果机器字长为8，-9反码的表示：
  ```
  -9 = 1 111 0110
  ```

- 补码：正数的补码为其原码，负数的源码为其反码+1。
  
  例子，如果机器字长为8，-9补码的表示：
  ```
  -9 = 1 111 0111
  ```

- 移码：不管正负数，补码的符号位取反。
  
  例子，如果机器字长为8，9、-9补码的表示：
  ```
  9 = 1 000 1001
  -9 = 0 111 0111
  ```

### 小数的二进制转换

首先我们看一下整数的二进制是如何转换为十进制的：
```
B(101) = 1 * 2^2 + 0 * 2^1 + 1 * 2^0 = D(5)
``` 
那如果二进制是小数如何转换为十进制：
```
B(101.11) = 1 * 2^2 + 0 * 2^1 + 1 * 2^0 + 1 * 2^-1 + 1 * 2^-2 = D(5.75)
```
整数为整数各个位的值乘以2^(n-1)相加，小数部分为二进制位的值乘以2^-n相加。如果理解呢，以十进制3.25为例，0.2为2/10=2\*10^-1，0.05为5/100=5\*10^-2。以此类推，二进制也是如此。

那反过来，十进制的小数如何转换为二进制的小数？下面以0.125为例：
```
0.125
x   2 
------
0.25    = 0
x  2   
------
0.5     = 0
x 2
------
1.0     = 1
```
计算过程就是将小数部分*2，看得到整数部分是0还是1，只到小数部分为都为0，最后得到结果就是`0.`加上整数部分的结果组合起来。0.125转换二进制的结果为0.001。再看一个例子：
```
0.75
x  2
-----
1.5   = 1
x 2
-----
1.0   = 1
```
0.75\*2等于1.5，1.5\*2等于3，但是二进制只有0和1，所以不对，正确的应该是相乘的时候只取计算结果的小数部分。即应该用0.5\*2=1.0，最后转换的结果为0.11，即十进制的0.75转换为二进制之后为0.11。

### 定点数和浮点数

- 定点数。
  
  所谓定点数，就是小数点的位置固定不变的数。小数点的位置通常有两种约定方式：定点整数（纯整数，小数点在最低有效数值位之后）和定点小数（纯小数，小数点在最高有效数值位之前）。
  
- 浮点数。
  
  什么是浮点数，浮点数就是小数点浮动，就是小数点可以左右移动。移动了小数点，需要保证小数的大小不变，请看下面的例子：
  ```
  3.1425 = 0.31425 * 10 = 31.425 * 10^-1
  ```
  小数点移动的公式：
  ```
  R进制数 x R^±n
  ```
  n为小数点移动的位数，向左移动符号为+，向右移动符号为负。

  二进制科学计数法公式：
  ```
  N = S x 2^P
  ```
  S:尾数，P:阶码，阶码和尾数都使用二进制。数值位都放在小数点后，最左侧的1，紧邻小数点。格式：

  |Pf|阶码位|Sf|尾数位|
  |:-:|:-:|:-:|:-:|
  |阶码符号（0：正数，1：负数）|阶码|尾数符号（0：正数，1：负数）|尾数|

  下图展示如何将二进制小数-11.001规格化的过程：

  ![浮点数规格化示意图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/computer/%E6%B5%AE%E7%82%B9%E6%95%B0%E8%A7%84%E5%88%99%E5%8C%96%E7%A4%BA%E6%84%8F%E5%9B%BE.png?raw=true)

  
  - 浮点数相加、相减

    两个相加可能产生"溢出"，导致出错。所以采取变形补码：即采用两个符号位，即正数符号位为00，负数的符号位为11。采用变形补码后，两个数想加后，结果的符号位是"01"或"10"，发生溢出。例如：
    ```
    x = +1100   y = +1000
    [x]补=001100    [y]补=001000

      [x]补  001100
     +[y]补  001000
    ---------------
    产生溢出  010100
    ```

    两个浮点数相加、减的步骤：

    1. 0操作数检查。两个浮点数中有一个为0，那么两个数相加的结果就是另外一个浮点数。
    2. 对阶，小阶向大阶看齐。下面以10进制数为例，演示什么是对阶：
       ```
       10^2 x 2 + 10^3 x 3
       = 10^2 x 2 + 10^2 x 30
       = 10^2 x (2 + 30)
       = 10^2 x 32
       ```

    3. 尾数加减运算。
    4. 对原码规格化（小数点后第一位必为1）。
    
    例子：设阶码3位，尾数6位，按浮点运算方法，完成下列取值的[x+y]，[x-y]运算：x=2^-011x0.100101,y=2^-010x(-0.011110)。
    ```
    解：(1) 0操作数检查
        (2) [x]浮=11011 ; 0.100101  
            [y]浮=11110 ; -0.011110
            阶码用变形补码表示。阶码不同，对阶。x的阶码是5，y的阶码是6。x向y看齐，[x]浮=11110 ; 0.010010(1)
        (3) 尾数运算 [x]尾补=00.010010(1) [y]尾补=11.100010
               [x]尾补 00.010010(1)
            +  [x]尾补 11.100010
          -------------------------
             没有溢出  11.110100(1) 结果的补码
        (4) 规格化  则结果的原码是 1.001011(1) (从右往左第1个1不变，往左走除符号位，其他取反)
            移动小数点，小数点左移两位，尾数不足6位补0 1.101110
            小数点移动之后，为了保证数值不变，阶码-2得到  11100
        最终x+y=-0.101110 x 2^-4 
    ```