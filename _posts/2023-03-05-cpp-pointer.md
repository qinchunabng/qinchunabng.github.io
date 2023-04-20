---
layout: post
title: C++中的指针
categories: [C++]
description: C++中的指针
keywords: C++, 指针
---

## 指针的声明和使用
C++指针是用来保存地址的变量。
指针定义的语法：数据类型 * 指针变量名;
```
int a=10;
int *p;
//让指针存储变量a的地址
p=&a;
```

## 常量指针(const修饰指针)

特点：指针的指向可以修改，指针的值不可以修改
```
int a=10;
const int *cp=&a;
// *cp=20; 
cp=&b;
```

## 指针常量(const修饰常量)

特点：指针的指向不可修改，指针的指向的值可以修改
```
int a=10;
int b=20;
int * const cp1 = &a;
// cp1=&b;
*cp1=20;
```

## const即修饰指针又修饰常量

特点：指针的指向和指针指向的值都不可以修改
```
int a=10;
const int * const cp2 = &a;
```