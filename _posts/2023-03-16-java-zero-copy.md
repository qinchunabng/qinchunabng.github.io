---
layout: post
title: Java中的零拷贝
categories: [Java]
description: Java中的零拷贝
keywords: Java, 零拷贝
---

很多Web应用都有静态资源，访问静态资源需要服务端从磁盘读取数据然后再写入响应套接字中。这些操作只需要少量的CPU资源，但是却是很低效的：内核从磁盘读取数据，然后将数据从内核空间拷贝到用户空间的应用中，然后应用程序再从用户空间拷贝到socket中。通过这种中转方式从磁盘读取数据到socket很明显是低效的。

每次数据从内核空间和用户空间之间复制，都要消耗CPU时间和占用内存带宽。幸运的是，我们可以通过零拷贝技术消除这些数据拷贝。应用可以使用零拷贝请求内核直接将数据从磁盘文件拷贝到socket中，不需要经过应用程序。零拷贝可以极大的提高应用程序的性能，减少内核态和用户态之间上下文切换的次数。

在UNIX和Linux操作系统上，Java中使用java.nio.channels.FileChannel的transferTo()方法实现零拷贝。你可以使用transferTo()方法将数据从一个channel传输到另外一个可写的channel，而数据不需要经过应用程序。下面首先通过传统的文件传输方式演示它的开销，然后在演示怎么通过transferTo()方法使用零拷贝获得更好的性能。

### 传统的数据传输方式

考虑这样一个场景，从文件中读取数据并通过网络将数据传给另外一个程序。（很多服务程序都是这样的场景，包括提供静态访问资源的Web服务、FTP服务、邮件服务等等。）参考下面代码：

服务端:
```
package sendfile;

import java.net.*;
import java.io.*;

public class TraditionalServer {
    
    public static void main(String args[]) {
	
	int port = 2000;
	ServerSocket server_socket;
	DataInputStream input;
	
	try {
	    
	    server_socket = new ServerSocket(port);
	    System.out.println("Server waiting for client on port " + 
			       server_socket.getLocalPort());
	    
	    // server infinite loop
	    while(true) {
		Socket socket = server_socket.accept();
		System.out.println("New connection accepted " +
				   socket.getInetAddress() +
				   ":" + socket.getPort());
		input = new DataInputStream(socket.getInputStream()); 
		// print received data 
		try {
			byte[] byteArray = new byte[4096];
		    while(true) {
		    	int nread = input.read(byteArray , 0, 4096);
		    	if (0==nread) 
		    		break;
		    }
		}
		catch (IOException e) {
		    System.out.println(e);
		}
		
		// connection closed by client
		try {
		    socket.close();
		    System.out.println("Connection closed by client");
		}
		catch (IOException e) {
		    System.out.println(e);
		}
		
	    }
	    
	    
	}
	
	catch (IOException e) {
	    System.out.println(e);
	}
    }
}
```

客户端:
```
package sendfile;

import java.io.DataOutputStream;
import java.io.FileInputStream;
import java.io.IOException;
import java.net.Socket;
import java.net.UnknownHostException;

public class TraditionalClient {
    
    
    
    public static void main(String[] args) {

	int port = 2000;
	String server = "localhost";
	Socket socket = null;
	String lineToBeSent;
	
	DataOutputStream output = null;
	FileInputStream inputStream = null;
	int ERROR = 1;
	
	
	// connect to server
	try {
	    socket = new Socket(server, port);
	    System.out.println("Connected with server " +
				   socket.getInetAddress() +
				   ":" + socket.getPort());
	}
	catch (UnknownHostException e) {
	    System.out.println(e);
	    System.exit(ERROR);
	}
	catch (IOException e) {
	    System.out.println(e);
	    System.exit(ERROR);
	}
	
	try {
		String fname = "sendfile/NetworkInterfaces.c";
		inputStream = new FileInputStream(fname);
		
	    output = new DataOutputStream(socket.getOutputStream());
	    long start = System.currentTimeMillis();	    
	    byte[] b = new byte[4096];
	    long read = 0, total = 0;
	    while((read = inputStream.read(b))>=0) {
		total = total + read;
	    	output.write(b);
	    }
	    System.out.println("bytes send--"+total+" and totaltime--"+(System.currentTimeMillis() - start));
	}
	catch (IOException e) {
	    System.out.println(e);
	}

	try {
		output.close();
	    socket.close();
	    inputStream.close();
	}
	catch (IOException e) {
	    System.out.println(e);
	}
    }    
}
```

核心操作其实就两个：读取文件，将文件内容发送到网络。
```
File.read(fileDesc, buf, len);
Socket.send(socket, buf, len);
```
虽然操作很简单，但是操作服务器内部却需要用户态和内核态之间的四次上下文切换，并且数据需要拷贝四次。图1展示了服务器内部数据是怎么从文件流向网络套接字的：

图1 传统数据的拷贝方式

![传统模式数据服务器内部数据流向](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/java_zero_cp_figure1.gif?raw=true)

图2 传统方式的上下文切换

![传统方式的上下文切换](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/java_zero_cp_figure2.gif?raw=true)

数据拷贝整个过程如下：

1. read()操作引起了一次用户态到内核态的切换。内部发起一个sys_read()操作（或者等价的操作）从文件中读取数据。第一次拷贝（见图1）是通过DMA引擎执行的，从硬盘读取数据并且存储在内核地址空间的缓存中。
2. 读取的数据从读缓冲区拷贝到用户缓冲区后，read()方法返回。read()方法的返回引起另外一次从内核态切换回用户态。现在数据存储在用户地址空间的缓存中。
3. socket的send()操作引起用户态到内核态的切换。第三次数据拷贝又将数据拷贝到内核地址空间的缓存中。这次数据拷贝到内核中不同的缓冲区，与发送的socket是关联的。
4. send()方法返回，导致第四次上下文切换。DMA引擎将数据从内核缓存区传递给协议引擎是，会异步进行第四次拷贝。 

使用中间内核缓冲区看起来很低效（相比直接将数据传输到用户缓冲区）。但是进程中引入中间内核缓冲是为了提升性能。当应用读取的数据没有内核缓冲区大，中间内核缓冲区可以充当预读缓存，这会显著的提升性能。执行写操作时，中间缓冲区允许写操作可以异步的完成。

但是这种方式在请求的数据远大于内核缓冲区大小的时候，就会变为性能瓶颈。在数据最终发送前，数据会从磁盘、内核缓冲区、用户缓冲区之间拷贝多次。

零拷贝可以消除这些多余的数据拷贝来提升性能。

### 零拷贝的数据传输方式

如果你自己检查传统的数据传输方式，你会发现第二次和第三次数据拷贝是没有必要的。数据拷贝到用户缓存后应用程序没有做任何操作，又传回内核空间的socket缓冲区。相反，数据是可以直接从内核的读取数据的缓冲区直接传输到socket缓冲区。通过transferTo()方法可以完成这个操作。
transferTo()方法的签名：
```
public void transferTo(long position, long count, WritableByteChannel target);
```
transferTo()方法将数据从文件channel传输到一个指定的可写的channel。它的内部实现是依赖操作系统实现的零拷贝，在UNIX和Linux中，是调用的sendfile()系统调用，这个系统调用可以从一个文件描述符传输数据到另外一个文件描述符：
```
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```
file.read()读取文件和socket.send()发送数据两个操作可以使用transferTo()来完成，图3展示了transferTo()调用过程中数据的传输过程：

图3 transferTo()的数据拷贝
![transferTo()的数据拷贝](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/java_zero_cp_figure3.gif?raw=true)

图4 transferTo()的上下文切换
![transferTo()的上下文切换](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/java_zero_cp_figure4.gif?raw=true)

 transferTo()的执行流程如下：
 
 1. transferTo()方法调用引起DMA引擎将文件内容从硬盘拷贝的到内核读缓冲，然后数据又被拷贝到关联socket的内核缓冲区。
 2. 第三次数据拷贝发生在DMA引擎从内核的socket缓冲区传输到协议引擎。

这里将上下文切换的次数从4次减少到2次，数据拷贝的次数从4次减少到3次（只有一次需要CPU参与）。但是这还是没有达到零拷贝的目标，如果底层的网卡支持gather操作，我们可以进一步减少数据的重复复制。在Linux2.4及以后的版本中，socket缓冲区描述符支持这个操作。这种方式不光减少上下文的切换，同时也消除的需要CPU参与的那个数据拷贝。transferTo()使用方式没有变，只是内部的实现变了：

1.  transferTo()方法调用DMA引擎将文件内容从硬盘拷贝的到内核读缓冲。
2.  数据不会被拷贝到socket缓冲区，只有带有数据地址和数据长度的描述符附加到socket缓冲区。DMA引擎直接将数据从内核缓冲区拷贝到协议引擎中，这样就消除了最后一次CPU的数据拷贝。

图5展示了使用gather操作的transferTo()的数据拷贝过程：

图5

![使用gather操作的transferTo()](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/java_zero_cp_figure5.gif?raw=true)


