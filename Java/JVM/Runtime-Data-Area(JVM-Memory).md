## JVM 运行时数据结构---20180910
虚拟机在执行java 程序的过程中会把它所管理的内存划分为若干不同的数据区域，这些区域都有各自的用途，以及创建和销毁的时间。有的区域随着虚拟机进程（方法区、堆）的启动而存在，有些区域则依赖线程的启动和结束而建立和销毁（程序计数器、虚拟机栈、本地方法栈）。

![jvmspec7](https://user-images.githubusercontent.com/6982311/45546348-9c5bf700-b84f-11e8-91c4-c6ef544601c4.png)

![20180115160639802](https://user-images.githubusercontent.com/6982311/45585841-1fe41980-b91e-11e8-8dfd-d9fdb1940e35.png)

### 程序计数器(Program Counter Register)
程序计数器是一块较小的内存空间，它的作用可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。- - 摘自《深入理解Java虚拟机》

这里的“pc寄存器”是在抽象的JVM层面上的概念——当执行Java方法时，这个抽象的“pc寄存器”存的是Java字节码的地址。实现上可能有两种形式，一种是相对该方法字节码开始处的偏移量，叫做bytecode index，简称bci；另一种是该Java字节码指令在内存里的地址，叫做bytecode pointer，简称bcp。

特 点

1.如果线程正在执行的是Java 方法，则这个计数器记录的是正在执行的虚拟机字节码指令地址

2.如果正在执行的是Native 方法，则这个技术器值为空（Undefined）

3.此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域

一、程序计数器会随着线程的启动而创建，先来直观的看下计数器中会存哪些内容

有代码如下
```java
package com.xnccs.cn.share;

public class ShareCal {

    public int calc(){
        int a = 100;
        int b = 200;
        int c = 300;
        return ( a + b ) * c;
    }
}
```
这是一段非常简单的计算代码，我们先编译成Class 文件再使用 javap 反汇编工具看下class 文件中数据格式，如下图
![20180102165306848](https://user-images.githubusercontent.com/6982311/45544726-b6dfa180-b84a-11e8-957d-3718601207d6.png)

图中使用红框框起来的就是字节码指令的偏移地址，偏移地址对应的bipush 等等是jvm 中的操作指令,这是入栈指令。这里不作详细分析，有机会再分享。 当执行到方法calc()时在当前的线程中会创建相应的程序计数器，在计数器中为存放执行地址 （红框中的）0 2 3…等等

这也说明在我们程序运行过程中计数器中改变的只是值，而不会随着程序的运行需要更大的空间，也就不会发生溢出情况。–特点3

二、举例理解程序计数器

说线程恢复等基础功能都需要依赖这个程序计数器来完成，首先我们得知道：

线程是CPU 最小的调度单元
Java 虚拟机的多线程是通过切换线程并分配处理器执行时间的方式来实现的，在任何一个确定的时间，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令。

如当A 线程先向处理器发出指令，但当执行到中途一半时，B线程过来执行，且优先级高，此时处理器将A 挂起，B 执行，当B 执行结束需要唤醒A 同时得知道A 的执行位置，就可以查看线程A 中的计数器指令。

三、为什么执行的是native 方法时，为undefined

由上我们知道计数器记录的字节码指令地址，但是native 本地（如：System.currentTimeMillis()/ public static native long currentTimeMillis();）方法是大多是通过C实现并未编译成需要执行的字节码指令所以在计数器中当然是空（undefined）.

问： 那native 方法的多线程是如何实现的呢？

答： native 方法是通过调用系统指令来实现的，那系统是如何实现多线程的则 native 就是如何实现的

摘博客一段话：

Java线程总是需要以某种形式映射到OS线程上。映射模型可以是1:1（原生线程模型）、n:1（绿色线程 / 用户态线程模型）、m:n（混合模型）。以HotSpot VM的实现为例，它目前在大多数平台上都使用1:1模型，也就是每个Java线程都直接映射到一个OS线程上执行。此时，native方法就由原生平台直接执行，并不需要理会抽象的JVM层面上的“pc寄存器”概念——原生的CPU上真正的PC寄存器是怎样就是怎样。就像一个用C或C++写的多线程程序，它在线程切换的时候是怎样的，Java的native方法也就是怎样的。

### Java 虚拟机栈(Java Virtual Machine Stack)
每个线程在创建时都会创建一个虚拟机栈，由栈帧（Stack Frame）组成，对应着一次次的 Java 方法调用。每个方法在执行的同时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每个方法从调用直至执行完成的过程，就对应这一个栈帧在虚拟机栈中入栈到出栈的过程。在一个时间点，对应的只会有一个活动的栈帧，通常叫作当前帧，方法所在的类叫作当前类。如果在该方法中调用了其他方法，对应的新的栈帧会被创建出来，成为新的当前帧，一直到它返回结果或者执行结束。JVM 直接对 Java 栈的操作只有两个，就是对栈帧的压栈和出栈。（-Xss是虚拟机给每个线程分配的内存(栈空间)）

看如下简图：

![20180115153657688](https://user-images.githubusercontent.com/6982311/45546793-f5785a80-b850-11e8-94a5-a58c8ae3fd9b.png)

栈帧的结构图：

![20180115154642718](https://user-images.githubusercontent.com/6982311/45546924-80f1eb80-b851-11e8-979b-a4dec2e01382.png)

* 问题：两种栈溢出的区别：StackOverFlowError和OutOfMemoryError？

前者一般是因为方法递归没终止条件，后者一般是方法中线程启动过多。

在《java虚拟机规范中文版》第二章第五节中有这么几句话：    

1.如果线程请求分配的栈容量超过java虚拟机栈允许的最大容量的时候，java虚拟机将抛出一个StackOverFlowError异常。    

2.如果java虚拟机栈可以动态拓展(当前大部分Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈)，并且扩展的动作已经尝试过，但是目前无法申请到足够的内存去完成拓展，或者在建立新线程的时候没有足够的内存去创建对应的虚拟机栈，那java虚拟机将会抛出一个OutOfMemoryError异常。

StackOverFlowError示例：
```java
/**
 * 栈溢出测试
 * 
 * VM : -Xss128k  栈内存容量
 * @author j_nan
 *
 */
public class JavaVMStackSOF {

    private int stackLength = 1;


    public void stackLeak(){
        stackLength ++;
        stackLeak();
    }


    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try{
            oom.stackLeak();
        }catch(Throwable e){
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

输出：

![20180115151728460](https://user-images.githubusercontent.com/6982311/45551241-d46b3600-b85f-11e8-83d5-f56f9dbe65e8.jpg)

并且方法的参数越多，栈帧的局部变量表所需的内存空间越大，栈深越小。

* 问题：JVM为什么要分堆和栈？

在Java中，栈stack内存是用来存储函数的主体和变量名的。Java中的代码是在函数体中执行的，每个函数主体都会被放在栈内存中，比如main函数。加入main函数里调用了其他的函数，比如add（），那么在栈里面的的存储就是最底层是main，mian上面是add。栈的运行时后入先出的，所以会执行时会先销毁add，再销毁main。

在Java中，堆内存是用来存储实例的。比如main函数里面声明了一个people的类per，people per；这个per是存储在栈stack内存中的，实例化后（per = new people（））；实例后的对象实体是存在堆heap内存中的。栈stack内存中存储的per存储着指向堆heap内存的地址指向。堆heap内存的存在是为了更好的管理内存，实现garbage collection。当per不再指向堆heap内存中的实例的时候，garbage collection机制就会把这个堆heap内存中的new people（）实例删除，释放内存。

### 本地方法栈（Native Method Stack）
本地方法栈与虚拟机栈所发挥的作用是非常相似的，它们之前的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的native 方法服务。

如 System.currentTimeMillis(); public static native long currentTimeMillis(); 此获取当前毫秒就是native方法。

与虚拟机栈一样，本地方法栈区域也回会抛出StackOverflowError 和 OutOfMemoryError异常。

### 堆（Heap）
对于大多数应用来说，Java 堆 是Java 虚拟机所管理的的内存中最大的一块，此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。在Java 虚拟机规范中的描述是：所有的对象以及数组都要在堆上分配，但是随着JIT编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生，所有的对象都分配在堆上也就不那么绝对了。

简单理解就是，在堆上的大部分对象都会在发生GC 时被回收释放内存，但GC 同样是需要耗费资源，那如果确定一个对象只在一个方法区内出现，不存在逃逸现象，则可以考虑将此对象创建在栈上，随着方法的执行完成而回收对象的空间也就优化了GC的负担。但就我们常见的HotSpot 虚拟机来说目前没有这种机制，不存在在栈上创建对象的情况。

Java 堆是垃圾收集器管理的主要区域，从内存回收的角度来看，由于现在收集器基本都采用分代收集算法，所以Java堆中还可以细分为：新生代和老年代；新生代中还可以分为Eden空间、From Survivor空间、To Survivor空间。它们比例默认是 8：1：1 。

![20180115165503456](https://user-images.githubusercontent.com/6982311/45586121-d1d21480-b923-11e8-82f6-9783e8a6f8fb.png)

如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

异常示例:
```java
package com.xnccs.cn.share;

import java.util.ArrayList;
import java.util.List;

/**
 * 堆溢出测试
 * 
 * VM: -Xms20m -Xmx20m
 * @author j_nan
 *
 */
public class HeapOOM {

    static class OOMObject{
    }

    public static void main(String[] args) throws InterruptedException {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while(true){
            try{
                list.add(new OOMObject());
            }catch(Throwable e){
                System.out.println("集合中对象数量是:"+list.size());
                throw e;
            }
        }

    }
}
```

啰嗦：
* JVM规范中的描述是Java堆包含数组（Arrays）和对象实例（Class Instances）。不过随着技术的发展这也不一定了。
* 可处于物理上不连续的内存空间，保证逻辑上连续即可。
* 主流VM中都按可扩展容量实现（通过-Xmx和-Xms控制）。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，会抛出OutOfMemoryError。



### 方法区（Method Area）
这也是所有线程共享的一块内存区域，用于存储所谓的元（Meta）数据，例如类结构信息，以及对应的运行时常量池、字段、方法代码、JIT编译后的代码等数据。

由于早期的 Hotspot JVM 实现，很多人习惯于将方法区称为永久代（Permanent Generation）。Oracle JDK 8 中将永久代移除，同时增加了元数据区（Metaspace）。

方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。

元空间（Metaspace）的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。理论上取决于32位/64位系统可虚拟的内存大小。可见也不是无限制的，需要配置参数。‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。

### 运行时常量池（Run-Time Constant Pool）
A run-time constant pool is a per-class or per-interface run-time representation of the constant_pool table in a class file (§4.4). It contains several kinds of constants, ranging from numeric literals known at compile-time to method and field references that must be resolved at run-time. The run-time constant pool serves a function similar to that of a symbol table for a conventional programming language, although it contains a wider range of data than a typical symbol table.

Each run-time constant pool is allocated from the Java Virtual Machine's method area (§2.5.4). The run-time constant pool for a class or interface is constructed when the class or interface is created (§5.3) by the Java Virtual Machine.

The following exceptional condition is associated with the construction of the run-time constant pool for a class or interface:

* When creating a class or interface, if the construction of the run-time constant pool requires more memory than can be made available in the method area of the Java Virtual Machine, the Java Virtual Machine throws an OutOfMemoryError.

啰嗦：
具备动态性，非预置入Class文件常量池的新常量也可能在运行期间放入池中（利用的比较多的是String的intern()）

### 问题
#### 什么是直接内存（Directed Memory）？
不是运行时数据区的一部分，也不是规范中定义的内存区域。不过也会导致OutOfMemoryError。

JDK1.4中加入的NIO（New Input/Output）类引入一种基于Channel和Buffer的I/O方式，可以使用Native函数库直接分配堆外内存，然后通过一个存在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，在一些场景可以提升性能，因为避免了在Java堆和Native堆中来回复制数据。

不受Java堆大小的限制，受本机总内存和处理器寻址空间的限制。

可能出现OOM的情况：设置的-Xmx等各个内存区域总和大于物理内存限制，导致动态扩展中出现OOM Error。

DirectMemory导致的内存溢出在Heap Dump文件中不会看见明显的异常。若OOM后Dump文件很小，同时程序中直接或间接地使用了NIO，即可考虑堆外内存溢出。

啰嗦：
JVM 本身是个本地程序，还需要其他的内存去完成各种基本任务，比如，JIT Compiler 在运行时对热点方法进行编译，就会将编译后的方法储存在 Code Cache 里面；GC 等功能需要运行在本地线程之中，类似部分都需要占用内存空间。这些是实现 JVM JIT 等功能的需要，但规范中并不涉及。

#### 如果-Xmx 8G，那么这个8G包括方法区吗？

如果-Xmx 8G，那么方法区不在其中。但是永久代/方法区也属于GC Heap的一部分。另外，方法区（method area）只是JVM规范中定义的一个概念，用于存储类信息、常量池、静态变量、JIT编译后的代码等数据，具体放在哪里，不同的实现可以放在不同的地方。而永久代是Hotspot虚拟机特有的概念，是方法区的一种实现，别的JVM都没有这个东西。在Java 6中，方法区中包含的数据，除了JIT编译生成的代码存放在native memory的CodeCache区域，其他都存放在永久代；在Java 7中，Symbol的存储从PermGen移动到了native memory，并且把静态变量从instanceKlass末尾（位于PermGen内）移动到了java.lang.Class对象的末尾（位于普通Java heap内）；在Java 8中，永久代被彻底移除，取而代之的是另一块与堆不相连的本地内存——元空间（Metaspace）,‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。


### 参考资料
官方jvm文档非常好：

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4





