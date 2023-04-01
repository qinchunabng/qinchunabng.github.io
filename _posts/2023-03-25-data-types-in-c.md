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

