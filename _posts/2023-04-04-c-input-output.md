---
layout: post
title: C语言中的输入输出
categories: [C]
description: C语言中的输入输出
keywords: C, I/O
---

## 输入输出专题

### 格式化输入输出函数：scanf、printf

```
int printf(const char *format,...);
```
- format: "%[修饰符] 格式字符串"，参考下表标准输出修饰符与输入输出格式字符。
  
  <table>
    <tr>
        <td>d,i</td>
        <td>十进制格式</td>
        <td>int a=567;printf("%d",a);</td>
        <td>567</td>
    </tr>
    <tr>
        <td>x,X</td>
        <td>十六进制格式无符号整数</td>
        <td>int a=255;printf("%x",a);</td>
        <td>ff</td>
    </tr>
    <tr>
        <td>o</td>
        <td>八进制无符号整数</td>
        <td>int a=65;printf("%o",a);</td>
        <td>101</td>
    </tr>
    <tr>
        <td>u</td>
        <td>不带符号十进制整数</td>
        <td>int a=567;printf("%u",a);</td>
        <td>567</td>
    </tr>
    <tr>
        <td>c</td>
        <td>单一字符</td>
        <td>char a=65;printf("%c",a);</td>
        <td>567</td>
    </tr>
    <tr>
        <td>s</td>
        <td>字符串</td>
        <td>printf("%s","ABC");</td>
        <td>ABC</td>
    </tr>
    <tr>
        <td>e,E</td>
        <td>指数形式浮点小数</td>
        <td>float a=567.789;printf("%e",a);</td>
        <td>5.677890e+02</td>
    </tr>
    <tr>
        <td>f</td>
        <td>小数形式浮点小数</td>
        <td>float a=567.789;printf("%f",a);</td>
        <td>567.789000</td>
    </tr>
    <tr>
        <td>g</td>
        <td>e和f较短的一种</td>
        <td>float a=567.789;printf("%g",a);</td>
        <td>567.789</td>
    </tr>
    <tr>
        <td>%%</td>
        <td>百分号本身</td>
        <td>printf("%%");</td>
        <td>%</td>
    </tr>
  </table>

  <table>
    <tr>
        <td>修饰符</td>
        <td>功能</td>
    </tr>
    <tr>
        <td>m</td>
        <td>输出数据域宽，数据长度<m，左补空格；否则按照实际输出 
    </tr>
    </tr>
    <tr>
        <td rowspan="2">.n</td>
        <td>对实数，指定小数点后位数（四舍五入）</td>
    </tr>
    <tr>
        <td>对字符串，指定实际输出位数</td>
    </tr>
    <tr>
        <td>-</td>
        <td>输出数据在域内左对齐（缺省右对齐）</td>
    </tr>
    <tr>
        <td>+</td>
        <td>指定在有符号数的正数前显示正号（+）</td>
    </tr>
    <tr>
        <td>0</td>
        <td>输出数值时指左面不使用的空位置自动填0</td>
    </tr>
    <tr>
        <td>#</td>
        <td>在八进制和十六进制数前显示前导0，0x</td>
    </tr>
    <tr>
        <td rowspan="2">l</td>
        <td>在d,o,x,u前，指定输出精度为long型</td>
    </tr>
    <tr>
        <td>在e,f,g前，指定输出精度为double型</td>
    </tr>
  </table>

  - 缓冲区
    
    看下面代码:
    
    ```
    printf("[%s:%d]before while().", __FUNCTION__, __LINE__);
    while(1);
    printf("[%s:%d]after while().", __FUNCTION__, __LINE__);
    ```
    这个代码不会输出任何内容，原因是printf输出的内容是放到输出缓冲区的，遇到`\n`或者显式调用`fflush()`才会刷新缓冲区，或者等缓冲区满了才会自动刷新缓冲区。

```
int scanf(const char *format,...);
```
示例1：
```
int i;
float f;
printf("Please enter:\n");
//错误示例：%d后面加\n，加了\n输入时输入数字后面终端需要原模原样输入\n才能输入进来
//scanf("%d\n",&i);
//输入时，类型要匹配，且%d和%f直接需要输入','
//如果%d和%f直接不加任何东西，两次输入直接可以是空格、tab、换行任何类型
scanf("%d,%f",&i,&f);
printf("i = %d\n", i);
printf("f = %d\n", f);
```
输出：
```
Please enter:
10,90
i = 10
f = 90.000000
```

示例2：
```
char str[32];
printf("Please enter:\n");

scanf("%s", str);
printf("%s\n", str);
```
输出：
```
Please enter:
hello
hello
```
注意：
- 通过scanf输入字符串时，如果字符串包含任何间隔符，间隔符后面的内容不会保存，因为间隔符会被当做输入结束符，需要使用字符串专用的输入输出函数
- scanf输入字符串，可能会出现越界现象（例如字符串str长度长度为3，而实际输入长度超过3就会出现越界），并且越界后不会出现段错误等任何现象

示例3：
```
int i;
printf("Please enter:\n");

while(1){
    scanf("%d",&i);
    printf("i = %d\n", i);
}
```
这段代码如果输入一个非整型，会一直循环输出变量i当前的值，如果在循环中获取输入值，一定要判断scanf的返回值：
```
int i,ret;
printf("Please enter:\n");

while(1){
    ret = scanf("%d",&i);
    if(ret != 1){
        printf("Enter error!\n");
        break;
    }
    printf("i = %d\n", i);
}
```

示例4：
```
int i;
char ch;
printf("Please enter:\n");

scanf("%d", &i);
scanf("%c", &ch);
printf("i = %d, ch = %c", i, ch);
```
如果先输入10然后回车再输入a，会出现如下结果：
```
Please enter:
10
i = 10, ch =
```
在输入回车之后就打印结果了，我们调整ch的输出格式，输出ch的ASCII值：
```
Please enter:
89
i = 89, ch = 10
```
ch的对应的ASCII码为10，即换行。在只有一个scanf函数时，回车换行会当作scanf结束符，但是如果是两个连续的输入函数，需要使用抑制符*来抑制回车换行，或者使用getchar()吃掉中间的换行符：
```
int i;
char ch;
printf("Please enter:\n");

scanf("%d", &i);
//或者使用getchar()吃掉换行符
scanf("%*c%c", &ch);
printf("i = %d, ch = %c", i, ch);
```
### 字符串输入输出函数：getchar、putchar

### 字符串输入输出函数：gets、puts

使用gets会出现`warning: the `gets' function is dangerous and should not be used.`的警告，因为gets不会验证数据组越界，应该使用fgets替代，或使用getline，但是getline属于方言，只存在GNU的glibc的库中。
