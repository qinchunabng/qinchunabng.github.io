---
layout: post
title: NIO三大组件
categories: [Java]
description: NIO三大组件
keywords: Java, NIO
---

## NIO三大组件

NIO全称non-blocking IO，即非阻塞式IO。

### Channel & Buffer

Channel有一点类似于stream，但是它是读写数据的双向通道，可以从channel将数据读入buffer，也可以将buffer的数据写入channel，而之前的stream要么是输入，要么是输出，channel比stream更为底层。

常见的Channel有：
- FileChannel。用于文件数据传输。
- DatagramChannel。用于UDP的数据传输。
- SocketChannel。用于TCP的数据传输，可用于TCP服务端和客户端。
- ServerSocketChannel。用于TCP的数据传输，专用于TCP服务端。

buffer则用来缓存读写数据，常见的buffer有：
- ByteBuffer
  - MappedByteBuffer
  - DirectByteBuffer
  - HeapByteBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer
- CharBuffer
  
### Selector

Selector但从字面意思不好理解，需要结合服务器的设计演化来理解它的用途。

#### 多线程版设计

最早服务器的网络模型是为每个socket连接创建一个线程来处理读写操作：

![多线程版IO模型](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%89%88IO%E6%A8%A1%E5%9E%8B.png?raw=true)

缺点：
- 内存占用高
- 线程切换上下文切换成本高
- 只适合连接数少的场景

#### 线程池版设计

为了防止多线程版的缺点，在多线程版的基础上使用线程池进行优化：

![线程池版IO模型](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%89%88IO%E6%A8%A1%E5%9E%8B.png?raw=true)

但是线程池版的设计也有自己的缺点：

- 阻塞模式下，线程仅能处理一个socket连接，线程没有得到多个充分利用。
- 仅适合短连接场景。正是由于上一个原因，为了使线程能够复用，所以在socket使用之后需要断开连接，释放线程，使得线程池能够为其他socket服务。

#### selector版设计

selector的作用就是配合一个线程来管理多个channel，获取这些channel上发生的事件，这些channel工作在非阻塞模式下，不会让线程吊死在一个channel上。适合连接数特别多，但流量低的场景（low traffic）。

![selector版IO模型](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/selector%E7%89%88IO%E6%A8%A1%E5%9E%8B.png?raw=true)

调用selector的select()会阻塞直到channel发生了读写就绪事件，这些事件发生，select方法就会返回这些事件交给thread来处理。