## JVM 运行时数据结构---20180910
虚拟机在执行java 程序的过程中会把它所管理的内存划分为若干不同的数据区域，这些区域都有各自的用途，以及创建和销毁的时间。有的区域随着虚拟机进程（方法区、堆）的启动而存在，有些区域则依赖线程的启动和结束而建立和销毁（程序计数器、虚拟机栈、本地方法栈）。

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
这也是所有线程共享的一块内存区域，用于存储所谓的元（Meta）数据，例如类结构信息，以及对应的运行时常量池、字段、方法代码、JIT编译后的代码等数据。如，当程序中通过getName、isInterface等方法来获取信息时，这些数据来源于方法区。

由于早期的 Hotspot JVM 实现，很多人习惯于将方法区称为永久代（Permanent Generation）。Oracle JDK 8 中将永久代移除，同时增加了元数据区（Metaspace）。

方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。

由于使用反射机制的原因，虚拟机很难推测哪个类信息不再使用，因此这块区域的回收很难！另外，对这块区域主要是针对常量池回收，值得注意的是JDK1.7已经把常量池转移到堆里面了。

元空间（Metaspace）的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。理论上取决于32位/64位系统可虚拟的内存大小。可见也不是无限制的，需要配置参数。‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。

### 运行时常量池（Run-Time Constant Pool）
A run-time constant pool is a per-class or per-interface run-time representation of the constant_pool table in a class file (§4.4). It contains several kinds of constants, ranging from numeric literals known at compile-time to method and field references that must be resolved at run-time. The run-time constant pool serves a function similar to that of a symbol table for a conventional programming language, although it contains a wider range of data than a typical symbol table.

Each run-time constant pool is allocated from the Java Virtual Machine's method area (§2.5.4). The run-time constant pool for a class or interface is constructed when the class or interface is created (§5.3) by the Java Virtual Machine.

The following exceptional condition is associated with the construction of the run-time constant pool for a class or interface:

* When creating a class or interface, if the construction of the run-time constant pool requires more memory than can be made available in the method area of the Java Virtual Machine, the Java Virtual Machine throws an OutOfMemoryError.

啰嗦：
* 其空间从方法区域（JDK1.7后为堆空间）中分配。
* 具备动态性，非预置入Class文件常量池的新常量也可能在运行期间放入池中（利用的比较多的是String的intern()）

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

#### Intern方法的奥秘？
在<深入理解Java虚拟机>一书中解释道: ‘String.intern()是一个Native方法,它的作用是:如果字符常量池中已经包含一个等于此String对象的字符串,则返回常量池中字符串的引用,否则,将新的字符串放入常量池,并返回新字符串的引用’，不同版本的JAVA虚拟机对此方法的实现可能不同。

但是呢，JDK 7 以后，HotSpot 已将常量池从永久代转移到了堆中。看例子：
```java
/**String.intern()返回引用的测试**/
public class RuntimeConstantPoolOOM {

    public static void main(String[] args) {
        String str1 = new StringBuilder("computer").append("software").toString();
        System.out.println(str1.intern() == str1);

        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2); // str2.intern()并不会修改str2的值
    }
}
```

在JDK1.6中会得到两个false，在JDK1.7中会得到一个true和一个false。因为JDK1.6中会把首次遇到的String实例复制到常量池中，返回的也是常量池中该实例的引用，与StringBuilder创建的在堆上的对象不是同一个引用；而JDK1.7开始的intern()不再复制实例，而是在常量池中记录首次出现的实例的引用并返回该引用(猜测可能使用类似equals的方法比对)，故指向的是同一个实例。对str2返回false是因为"java"这个字符串在执行StringBuilder.toString()之前已经出现过并存在常量池中了[3]，所以返回的是之前存在的实例的引用。

Intern的好处是可以节约内存（String对象数量少），但每次都要去常量池查一把，程序运行时间较慢，但是网上说回收对象的GC所占用的时间是要大于这个时间的。

### 参考资料
官方jvm文档非常好：

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4

其他文章：

https://qintongbaba.github.io/2017/08/03/JVM%E4%B9%8B%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA-Runtime-Data-Area/

https://www.jianshu.com/p/fcb3c28b19fe#fn2

1、程序计数器

​ 程序计数器(Program Counter Register)是一块较小的内存空间，它的作用可以看做为当前线程所执行字节码的行号指示器。在虚拟机的概念模型中（仅是概念模型，各种虚拟机可能会根据一些高效的方式实现），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成。

​ 由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说为一个内核）只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

​ 如果线程正在执行的为一个Java方法，这个计数器纪录的是正在执行的虚拟机字节码指令地址；如果正在执行的是Native方法，这个计数器则为空（Undefined）。此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

2、Java虚拟机栈

​ 与程序计数器一样，Java虚拟机栈（Java Virtual Machie Stacks）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

​ 经常有人把Java内存区分为堆内存（Heap）和栈内存（Stack），这种分法比较粗糙，Java内存区域的划分实际上远比这复杂。其中所讲的“堆”在后面会专门的讲述，而所指的“栈”就是现在讲的虚拟机栈，或者虚拟机栈中的局部变量表部分。

​ 局部变量表存放了编译期可知的各种基本数据类型（byte、short、int、long、double、float、char、boolean）、对象引用（reference类型，它不等同与对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的指针，也可能表示指向一个对象的句柄或者其它与此对象相关的位置）和returnAddress类型（指向一条字节码指令地址）。

​ 其中64位长度的long和double类型的数据会占用两个局部变量的空间（slot），其余的数据类型只占用一个。局部表所需的内存在编译期间完成分配，当进入一个方法的时候，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

​ 在Java虚拟机规范中，对这个区域规定了两种异常情况；如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分Java虚拟机都可以动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。

3、本地方法栈

​ 本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的， 其区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到Native方法服务。虚拟机规范中本地方法栈中的方法使用的语言、使用方式与数据结构并没有强制规定，因此具体的虚拟机可以自由的实现它。甚至有些虚拟机（譬如Sun HotSpot 虚拟机）直接就把本地方法栈和虚拟机栈合二为一。与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。

4、Java堆

​ 对于大多数引用来讲，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。Java堆是被所有的线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。这点在Java虚拟机中的描述是：所有的对象实例以及数组都要在堆上分配，但是随着JIT编译器的发展与逃逸分析技术的逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生，所有的对象都分配在堆上渐渐变得不是那么“绝对”了。

​ Java堆是垃圾收集器管理的主要区域，因此很多时候也被称为“GC堆”（Garbage Collected Heap）。如果从内存回收的角度看，由于现在收集器基本都是采用的分代收集算法，所以Java堆中还可以细分为：新生代和老年代；再细致一点的有Eden空间，From Survivor空间、To Survivor空间等。如果从内存分配的角度来看，线程共享的Java堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。不过，无论如何划分，都与存放的内容无关，无论哪个区域，存放的都仍然是对象的实例，进一步的划分的目的是为了更好的回收内存，或者更快地分配内存。

​ 根据Java虚拟机规范规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

5、方法区

​ 方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被Java虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范将方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做NonHeap（非堆），目的是与Java堆分开。

​ 对于习惯在HotSpot虚拟机上开发和部署程序的开发者来说，很多人愿意将方法区称为“永久代”（Permanent Generation），本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已。对于其它虚拟机（如BEA JRockit、IBM J9等）来说是不存在永久代的概念的。即时是HotSpot虚拟机本身，根据官方路线图信息，现在也有放弃永久代并“搬家”至Native Memory来实现方法区的规划了。

​ Java虚拟机规范对这个区域的限制非常的宽松，除了和Java堆一样不需要连续的内存和可以固定大小或者可扩展外，还可以选择不实现垃圾收集。相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入了方法区就如永久代的名字一样“永久”存在了。这个区域的内存回收目标主要针对常量池的回收和对类型的卸载，一般来说这个区域的回收“成绩”比较难以令人满意，尤其是类型的卸载，条件相当的苛刻，但是这区域的回收确实比较有必要的。在Sun公司的BUG列表中，曾出现过若干个严重的BUG就是由于低版本的HotSpot虚拟机对此区域未完全回收导致内存泄漏。

​ 根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

6、运行时常量池

​ 运行时常量池（Runtime Constant Pool）是方法的一部分。Class文件中除了有类的版本、字段、方法、接口等描述等信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。

​ Java虚拟机对Class文件的每一部分（自然也包括常量池）的格式都有严格的规定，每一个字节用于存储那种数据都必须符合规范上的要求，这样才会被虚拟机认可、装载和执行。但对于运行时常量池，Java虚拟机规范没有做任何细节的要求，不同的提供商实现的虚拟机可以按照自己的需要来实现这个内存区域。不过，一般来说，除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

​ 运行时常量池相对Class文件常量池的另一个重要的特征是具备动态性，Java语言并不要求常量一定只能在编译期产生，也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特征被开发人员利用得比较多的便是String类的intern（）方法。

​ 既然运行时常量池是方法区的一部分，自然会受到方法区的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

7、直接内存

​ 直接内存（Direct Memory）并不是虚拟机运行时数据区一部分，也不是Java虚拟机规范中定义的内存区域，但是这个内存也被频繁使用，而且也可能导致OutOfMemoryError异常出现，所以我们放到这里一起讲解。

​ 在JDK1.4中新加入NIO（New Input／Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I／O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

​ 显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，则肯定还是会受到本机总内存（包括RAM及SWAP区或者分页文件）的大小及处理器寻址空间的限制。服务器管理员配置虚拟机参数时，一般会根据实际内存设置-Xmx等参数信息，但经常会忽略掉直接内存，使得各个内存区域的总和大于物理内存限制（包括物理上的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常。





