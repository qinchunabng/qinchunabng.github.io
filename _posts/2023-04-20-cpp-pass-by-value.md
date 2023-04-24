---
layout: post
title: C++中值传递
categories: [C++]
description: C++中值传递
keywords: C++, 值传递
---

## 值传递

所谓值传递就是在函数调用时，实参将数值传给形参，如果形参发生改变，并不影响实参。

示例：
```
void swap(int n1,int n2);

int main(){
    int n1 = 10;
    int n2 = 20;

    swap(n1, n2);

    cout << "n1 = " << n1 << ", n2 = "<< n2 <<endl;

    exit(0);
}

void swap(int n1,int n2){
    cout << "交换前:" << endl;
    cout << "n1 = " << n1 << endl;
    cout << "n2 = " << n2 << endl;

    int temp = n1;
    n1 = n2;
    n2 = temp;

    cout << "交换后:" << endl;
    cout << "n1 = " << n1 << endl;
    cout << "n2 = " << n2 << endl;
}
```

输出结果：
```
交换前:
n1 = 10
n2 = 20
交换后:
n1 = 20
n2 = 10
n1 = 10, n2 = 20
``` 
可以看到swap并没有改变实参。