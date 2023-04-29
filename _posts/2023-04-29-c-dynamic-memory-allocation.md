---
layout: post
title: C语言中的动态内存管理
categories: [C++]
description: C语言中的动态内存管理
keywords: C, 动态内存管理
---

## 动态内存分配

### malloc函数

malloc函数原型：
```
void *malloc(size_t size);
```
malloc函数用来动态分配内存，size为要申请的内存大小，返回一个指针指向分配内存的起始位置。

### realloc函数

realloc函数原型：
```
void *realloc(void *ptr, size_t size);
```
realloc函数用来重新分配内存，ptr为指向一块内存的指针，size为重新分配的内存大小。


### free函数

free函数原型：
```
void free(void *ptr);
```
free函数用来释放指向ptr指针指向的空间，如果ptr是NULL，不会执行任何操作。


看下面例子：
```
char *s=NULL;
s=malloc(sizeof(char) * 6); 
if(s == NULL){
    printf("malloc() error!\n");
    exit(1);
}   
strcpy(s, "hello");

printf("%ld\n", strlen(s));
printf("%s\n", (char *)s);
free(s);
```
用s接受malloc返回值，虽然malloc返回值为void *，但是不需要强转，因为void *是和任何类型都匹配的（除了函数指针）。如果在编译时出现类型不匹配的警告，肯定是由于没有引入头文件，因为在C语言中，如果编译器没有见到函数原型（引入头文件），会默认把函数的返回当作int处理。

再看一个例子：
```
void func(int *p,int n){
    p=malloc(n);
    if(p == NULL){
        exit(1);
    }
}

int main(){
    int num = 10;
    int *p = NULL;

    func(p,num);
    free(p);
    exit(0);
}
```
这个例子中用func函数动态申请10个字节的内存，并再程序结束的时候释放申请的内存。这个函数看起来没有什么问题，但是其实已经出现了内存泄露。在func函数调用的时候，传入的是指针变量p的形参，而形参的p的生命周期旨在func函数中有效，所以在func函数执行结束后就无法再引用到申请的内存。那么这些申请的内存会只整个程序结束之后才会释放掉，那么就出现了内存泄漏。

解决办法有两种，解决方法一：
```
void func(int **p,int n){
    *p=malloc(n);
    if(*p == NULL){
        exit(1);
    }
}

int main(){
    int num = 10;
    int *p = NULL;

    func(&p,num);
    free(p);
    exit(0);
}
```
将函数参数改为二级指针，传参的时候传入指针变量p的地址。

解决方法二：
```
void *func(int *p,int n){
    p=malloc(n);
    if(p == NULL){
        exit(1);
    }
    return p;
}

int main(){
    int num = 10;
    int *p = NULL;

    p = func(&p,num);
    free(p);
    exit(0);
}
```
修改func函数的返回值为指针，将指向申请内存的指针返回，并且重新给p赋值。