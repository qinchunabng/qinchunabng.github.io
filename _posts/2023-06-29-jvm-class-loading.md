---
layout: post
title: JVM的类加载器及类加载过程
categories: [Java]
description: JVM的类加载器及类加载过程
keywords: Java, JVM
---

## JVM的类加载器及类加载过程

### 类加载器子系统的作用

![类加载器子系统](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/JVM%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%AD%90%E7%B3%BB%E7%BB%9F.jpg?raw=true)

- 类加载器子系统负责从文件子系统或者网络中加载class文件，class文件开头有特定的文件标识。
- ClassLoader只负责class文件的加载，至于它是否可以运行，则由Execution Engine决定。
- 加载的类信息存放于一块号称方法区的内存空间。除了类的信息外，方法区中还会存放运行时常量池信息，可能还包括字符串字面量和数字常量（这部分常量信息是Class文件中常量池部分的内存映射）。

![Java类加载工作原理示意图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/Java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E7%A4%BA%E6%84%8F%E5%9B%BE.png?raw=true)

以上图为例，演示一个类的加载过程：
1. class file存在于本地硬盘上，可以理解为设计师在纸上的模板，而最终这个模板在执行的时候是要加载到JVM当中，然后根据这个文件实例化n个一模一样的实例。
2. class file加载到JVM中，被称为DNA元数据模板，放在方法区。
3. 在.class文件->JVM->最终成为元数据模板，此过程就要一个运输工具（类装载器Class Loader），扮演一个快递员的角色。

加入现在有一个Java类HelloLoader：
```
public class HelloLoader{

    public static void main(String[] args){
        System.out.println("谢谢ClassLoader加载我...");
        System.out.println("你的大恩大德，我下辈子再报！");
    }
}
```
要执行HelloLoader的过程如下：

![类的加载过程](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E7%B1%BB%E7%9A%84%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B.png?raw=true)

### 类的加载过程

![类的加载过程示意图]()

#### 类的加载过程：Loading

加载：
1. 通过一个类的全限定名获取定义类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

补充：加载class文件的方式

- 从本地系统中直接加载
- 通过网络获取，典型场景：Web Applet
- 从zip压缩包中读取，成为日后jar、war格式的基础
- 运行时计算生成，使用最多的是：动态代理技术
- 由其他文件生成，典型场景：JSP应用
- 从专有数据库中提取.class文件，比较少见
- 从加密文件中获取，典型的防Class文件被反编译的保护措施

#### 类的加载过程：Linking

**验证（Verify）**

- 目的在于确保Class文件的字节流中包含信息符合当前虚拟机要求，保证被加载类的正确性，不会危害虚拟机自身安全。
- 主要包括四种验证：文件格式验证、元数据验证、字节码验证、符号引用验证。

**准备（Prepare）**

- 为类变量分配内存并且设置该类变量的默认初始值，即零值。
- 这里不包含用final修饰的static，因为final在编译的时候就会分配了，准备阶段会显式初始化。
- 这里不会为实例变量分配初始化，类变量会分配在方法区中，而实例变量是会随着对象一起分配到Java堆中。

**解析（Resulotion）**

- 将常量池内的符号引用转换为直接引用的过程。
- 事实上解析操作往往伴随着JVM在执行完初始化之后再执行。
- 符号引用就是一组符号来描述所引用的目标。符号引用的字面形式明确定义在《Java虚拟机规范》的Class文件格式中。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位目标的句柄。
- 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等。对应常量池中的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等。

#### 类的加载过程：Initailization

初始化：

- 初始化阶段就是执行类构造方法\<clinit>()的过程。
- 此方法不需要定义，是Javac编译器自动收集类中的所有类变量的赋值动作和静态代码块中的语句合并而来。如果类中没有初始化赋值语句，JVM不会生成\<clinit>()方法。
- 构造器方法中指令按语句在源文件中出现的顺序执行。
- \<clinit>()不同于类的构造器。（关联：构造器是虚拟机视角下的\<init>()）。
- 若该类具有父类，JVM会保证子类的\<clinit>()执行前，父类的\<clinit>()已经执行完毕。
- 虚拟机必须保证一个类的\<clinit>()方法在多线程下被同步加锁。

看下面一段代码：
```
public class ClassInitTest {


    static {
        number = 20;
    }

    private static int number = 10;

    public static void main(String[] args) {
        System.out.println(ClassInitTest.number);
    }
}
```
输出结果为：10。number是在初始化接口，安装从上到下的方式给number设置初始值。

再看下面一段代码：
```
public class DeadThreadTest {

    public static void main(String[] args) {
        Runnable r = () -> {
            System.out.println(Thread.currentThread().getName() + "开始");
            DeadThread deadThread = new DeadThread();
            System.out.println(Thread.currentThread().getName() + "结束");
        };

        Thread t1 = new Thread(r, "线程1");
        Thread t2 = new Thread(r, "线程2");

        t1.start();
        t2.start();
    }
}

class DeadThread{

    static {
        if(true){
            System.out.println(Thread.currentThread().getName() + "初始化当前类");
            while(true){}
        }
    }

}
```
这段代码在加载DeadThread会被阻塞，导致另外一个线程也被阻塞，因为\<clinit>()方法在多线程下会加同步锁，并且只会执行一次。

### 类加载器的分类

- Java支持两种类型的类加载器，分别为引导类加载器（Bootstrap ClassLoader）和自定义类加载器（User-Defined ClassLoader）。
- 从概念上来讲，自定义类加载器一般指的是程序中由开发人员自定义的一类加载器，但是虚拟机规范却没有这么定义，而是将所有派生于抽象类ClassLoader的类加载器都规划为自定义类加载器。

  
- 无论类加载器的类型如何划分，在程序中我们常见的类加载器始终只有3个，如下图所示：
  
  ![类加载器分类](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E5%88%86%E7%B1%BB.png?raw=true)

  这里四者之间的关系是包含关系，不是上层下层，也不是继承关系。

  所以BootstrapClassLoder为一类加载器，其他为为一类加载器。

  ![ClassLoader类图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/ClassLoader%E7%B1%BB%E5%9B%BE.png?raw=true)

  从上图也可以看出，ExtensionClassLoader（扩展类加载器）和AppClassLoader（系统加载器）继承自ClassLoader，所以都是自定义类加载器。

下面的代码演示如何获取各个类加载器：
```
public class ClassLoaderTest {

    public static void main(String[] args) {
        //获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);

        //获取上层：扩展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader);

        //获取上层
        ClassLoader bootstrapClassLoader = extClassLoader.getParent();
        System.out.println(bootstrapClassLoader);

        //用户自定义类加载器
        ClassLoader classLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(classLoader);

        ClassLoader classLoader1 = String.class.getClassLoader();
        System.out.println(classLoader1);
    }
}
```
输出结果：
```
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@2e5c649
null
sun.misc.Launcher$AppClassLoader@18b4aac2
null
```
因为BootstrapClassLoader不是Java代码实现的，所以获取的是null。用户自定义的类默认用系统类加载器加载。String类获取的加载是null，间接的说明String是用启动类加载器加载的，并且Java中所有的核心类都是由BootstrapClassLoader加载的。

#### 虚拟机自带的类加载器

- **启动类加载器（Bootstap ClassLoader）**
  
  - 这个类加载使用C/C++语言实现，嵌套在JVM内部。
  - 它用来加载Java的核心库（JAVA_HOME/jre/lib/rt.jar、resource.jar或sun.boot.class.path路径下的内容），用于提供JVM自身需要的类。
  - 并不继承自java.lang.ClassLoader，没有父加载器。
  - 加载扩展类和应用程序类加载器，并指定为他们的父类加载器。
  - 出于安全考虑，Bootstrap启动类加载器只加载包名为java、javax、sun等开头的类。

- **扩展类加载器（Extension ClassLoader）**
  - Java语言编写，由sun.misc.Launcher$ExtClassLoader实现。
  - 派生于ClassLoader类。
  - 父类加载器为启动类加载器。
  - 从java.ext.dirs系统属性所指定的目录中加载类库，或从JDK的安装目录的jre/lib/ext子目录（扩展目录）下加载类库。如果用户创建的JAR放在此目录，也会自动由扩展类加载器加载。

- **应用程序类加载器（系统类加载器，AppClassLoader）**
  - java语法编写，由sun.misc.Launcher$AppClassLoader实现。
  - 派生于ClassLoader类。
  - 父类加载器为扩展类加载器。
  - 它负责加载环境变量classpath或系统属性java.class.path指定路径下的类库。
  - 该类加载是程序中默认的类加载器，一般来说，Java应用的类都是由它来加载。
  - 通过ClassLoader#getSystemLoader()方法可以获取该类加载器。

- **用户自定义类加载器**
  - 在Java的日常应用程序中，类的加载几乎是由上述3种类加载器相互配合执行的，在必要时，我们还可以自定义类加载器，来定制类的加载方式。
  - 为什么要自定义类加载器？
    - 隔离加载类
    - 修改类加载的方式
    - 扩展加载源
    - 防止源码泄漏
  - 用户自定义类加载器的实现步骤：
    
    1. 开发人员可以通过继承抽象类java.lang.ClassLoader类的方式，实现自己的类加载器，以满足一些特殊的要求。
    2. 在JDK1.2之前，在自定义类加载器时，总会去继承ClassLoader类并重写loadClass()方法，从而实现自定义的类加载器，但是在JDK1.2之后，已不在建议用户去覆盖loadClass()方法，而是建议把自定义的类加载逻辑写在fingClass()方法种。
    3. 在白那些类加载器时，如果没有太过于复杂的需求，可以直接继承URLClassLoader类，这样就可以避免自己去编写findClass()方法及其获取字节码流的方式，使自定义类加载器编写更加简洁。

  自定义ClassLoader实例：
  ```
  public class CustomClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] result = getClassFromCustomPath(name);
            if(result == null){
                throw new FileNotFoundException();
            }else{
                return defineClass(name, result, 0, result.length);
            }
        }catch (FileNotFoundException e){
            e.printStackTrace();
        }

        throw new ClassNotFoundException(name);
    }

    private byte[] getClassFromCustomPath(String name){
        //从自定义路径加载class文件，细节略
        //如果字节码文件有加密，则需要在这里执行解密操作
        return null;
    }

    public static void main(String[] args) throws Exception {
        CustomClassLoader customClassLoader = new CustomClassLoader();
        Class cls = Class.forName("One", true, customClassLoader);
        Object obj = cls.newInstance();
        System.out.println(obj.getClass().getClassLoader());
    }
  }

  ```