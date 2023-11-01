---
layout: post
title: H5移动端适配
categories: [H5]
description: H5移动端适配
keywords: H5, 移动端
---

### 尺寸

一般用英寸描述屏幕的大小，如显示器的17、22，手机显示器的4.8、5.7等使用的单位都是英寸。英寸(inch，缩写in)在荷兰语中的本意是大拇指，一英寸就是普通人拇指的宽度。英寸和厘米的换算：1英寸=2.54厘米。 屏幕英寸的大小指的是屏幕对角线的长度。

### 分辨率

分辨率表示屏幕像素点的数量（宽高各有多少个），越高屏幕越清晰。

### 像素

像素分为物理像素和逻辑像素。像素指的是组成图片或屏幕的最小单位，图片和屏幕是由无数个具有特定颜色和位置的小方块组成，这些小方块就是物理像素。而在css布局中使用的px是一个相对单位，我们称之为逻辑像素。物理像素是我们在车站看到的那种电子屏幕，每个像素块都是一个晶体管。举个例子，我们买显示器的时候，参数信息中会有分辨率，表示就是物理像素，跟硬件相关的。但是我们在使用的时候，可以通过配置修改屏幕的分辨率，修改的就是逻辑像素。

### PPI

PPI(Pixel Per Inch)：每英寸包括的像素数。

PPI用于描述的图片的之类，PPI图片质量越高，PPI描述屏幕时，PPI越高，屏幕越清晰。


### DPI

DPI(Dot Per Inch)：即每英寸包括的点数。

这里的点是抽象单位，它可以是屏幕像素点、图片像素点也可以是打印机的墨点。


### 设备独立像素DIP

为了防止同样的像素在高分辨率的屏幕上展示越来越小，出现了设备独立像素（Device Independent Pixels），使得不同的图片在不同分辨率的屏幕展示的大小一致。

### 设备像素比DPR

设备像素比(Device Pixel Ratio)简称DPR，即物理像素和设备独立像素的比值。

在web中，浏览器为我们提供了`window.devicePixelRation`来帮助获取dpr。

### 视口ViewPort

ViewPort是视口的意思，也就是浏览器中用于显示网页的区域。在PC端，其大小就是浏览器可视区域的大小。而在移动端，绝大多数情况下viewport都大于浏览器可视区域。移动端有三种类型的viewport:layoutviewport、visualviewport、idealviewport。具体解释如下：

- layoutviewport布局视口：大于实际屏幕，元素的宽度继承于layoutviewport，用于保证网站的外观特性与桌面浏览器一样。layoutviewport有多宽，每个浏览器不同。iPhone的safari为980px，通过document.documentElement.clientWidth获取。
- visualviewport视觉视口：当前显示在屏幕上的页面，即浏览器的可视区域。通过window.innerWidth获取。
- idealviewport理想视口：为浏览器定义的可完美适配移动端的理想viewport，固定不变，可认为是设备视口宽度。比如iphone 7为375px，iphone 7p为414px。通过window.screen.width获取。

### 视口适配Meta ViewPort

我们可以借助\<meta>元素的viewport来帮助我们设置视口、缩放等，从而让移动端得到更好的效果。
```
<meta name="viweport"  content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no">
```

|Value|可能值|描述|
|:-|:-|:-|
|width|正整数或device-width|以pixels（像素）为单位，定义布局视口的宽度。|
|height|正整数或device-height|以pixels（像素）为单位，定义布局视口的高度|
|initial-scale|0.0 - 10.0|定义页面初始化缩放比率|
|minimum-scale|0.0 - 10.0|定义缩放的最小值；必须小于或等于maximum-scale的值|
|maximum-scale|0.0 - 10.0|定义缩放的最小值；必须大于或等于minimum-scale的值|
|user-scalable|一个布尔值（yes或no）|如果设置为no，用户不能缩放网页，默认为yes|

device-width就等于理想视口宽度，所以设置width=device-width就相当于让布局视口等于理想视口。

由于initail-scale=理想视口宽度/视觉视口宽度，所以设置initail-scale=1,相当于让视觉视口等于理想视口。

### 获取窗口大小API

- window.innerHeight: 获取浏览器视觉视口高度（包括垂直滚动条）；
- window.outerHeight: 获取浏览器外部高度。表示整个浏览器窗口的高度，包括侧边栏、窗口镶边、调正窗口大小的边框。
- window.screen.Height: 获取屏幕理想视口高度，这个数值是固定的。
- window.screen.availHeight: 浏览器窗口可以高度。
- document.documentElement.clientHeigth: 获取浏览器布局视口高度，包括内边距，但不包括垂直滚动条、边框和外边距。
- document.documentElement.offsetHeight: 包括内边距、滚动条、边框和外边距。
- document.documentElement.scrollHeight: 在不使用滚动条的情况下适合视口中的所有内容所需的最小高度。测量方式与clientHeight相同：它包含元素的内边距，但不包括垂直滚动条、边框和外边距。

### rem适配基本原理

在移动端中UI提供的设计图一般提供的是750px的物理像素，如何适配不同分辨率的屏幕，就引入了rem单位。rem是一个相对单位，相对于html根元素字体的大小。

对于不同屏幕大小的移动适配的方法是：将屏幕视口的宽度10等分，将html根元素字体的大小设置为视口宽度的1/10，即1rem=1/10视口的宽度。可以通过js动态跳转根元素的字体大小：
```
function setRemUnit(){
    var rem = document.documentElment.clientWidth / 10;
    document.documentElment.style.fontSize=rem + 'px';
}
setRemUnit();
```
那么如何设计图中给定元素的大小是50px，转换为rem计算方法为：
`50/750 * 10rem`。以设备像素比2的设备为例，即750px对于的逻辑像素为375px，这个计算过程在sass中通过定义函数的方式实现：
```
vw_fontsize=37.5
vw_design=750

rem(px)
  (px / vw_design) * 10rem

div {
    width: rem(50);
}
```

### viewport单位

上面介绍的rem适配方案已经过时，目前由于主流浏览器都已经支持vw和vh的viewport单位，所以采用vw和vh更方便。

vw是视口宽度100等分，即视口一共是100vw，vh同理，之前rem像素的转发通过vw实现：
```
vw_design=750
vw(px)
  px / vw_desing * 100vw

div {
    width: vw(50);
}
```

另外还有vmin和vmax，vmin：选取vw和vh的较小值，vmax: 选取vw和vh的较大值。主要使用场景就是移动端在横屏切换时，希望logo的大小保持不变。