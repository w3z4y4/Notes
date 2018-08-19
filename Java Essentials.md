# Java设计模式
## 装饰者模式（Java设计模式）---20180626
父类和子类继承自同一接口，父对象通过构造函数传递给子类，子类动态扩展父对象的方法。遵循了开闭原则，即扩展而非修改。
示例：
```java
interface Component  
{  
    void Operation();  
}  
class Father : Component  
{  
    public void Operation()  
    {  
        Console.WriteLine("Base operation.");  
    }  
}  
class Child : Component  
{  
    private Component _father;  
  
    public Child(Component father)  
    {  
        _father = father;  
    }  
  
    public void Operation()  
    {  
        Console.WriteLine("Before operation.");  
        _father.Operation();  
        Console.WriteLine("After operation.");  
    }  
} 
```
实际应用比如（java.io.InputStream）
![](https://camo.githubusercontent.com/097b63fca6d99ada8a9b2f0aa9ad1304dd5a8ca0/68747470733a2f2f696d672d626c6f672e6373646e2e6e65742f3230313631303130313131373037393233)
```java
public static void main(String[] args) throws FileNotFoundException {
    InputStream a = new FileInputStream("") ;//要装饰FileInputStream
    a = new DataInputStream(a);//添加DataInputStream可以读取原始数据类型的数据。
    a = new BufferedInputStream(a);//添加缓冲
}
```
参考：

https://blog.csdn.net/alreadyfor/article/details/52776464

https://blog.csdn.net/gscienty/article/details/43002635

## 观察者模式（Java设计模式）---20180626

# JVM
推荐《深入理解JAVA虚拟机》
## Java类加载机制---20180819
JAVA源码编译由三个过程组成：
1、源码编译机制。
2、类加载机制
3、类执行机制
我们这里主要介绍编译和类加载这两种机制。

### 源码编译
代码编译由JAVA源码编译器来完成。主要是将源码编译成字节码文件（class文件）。字节码文件格式主要分为两部分：常量池和方法字节码。
### 类加载
类的生命周期是从被加载到虚拟机内存中开始，到卸载出内存结束。过程共有七个阶段，其中到初始化之前的都是属于类加载的部分。  
加载----验证----准备----解析-----初始化----使用-----卸载  
类加载又可以分为三个大类：Loading(加载)、Linking(连接)、Initialization(初始化)  
在这五个阶段中，加载、验证、准备和初始化这四个阶段发生的顺序是确定的，而解析阶段则不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定。另外这里的几个阶段是按顺序开始，而不是按顺序进行或完成，通常在一个阶段执行的过程中调用或激活另一个阶段。  
**系统可能在第一次使用某个类时加载该类，也可能采用预加载机制来加载某个类，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（LinkageError错误）,如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误.当运行某个java程序时，会启动一个java虚拟机进程，两次运行的java程序处于两个不同的JVM进程中，两个jvm之间并不会共享数据。**

1、加载阶段(Loading)  
这个流程中的加载是类加载机制中的一个阶段，这两个概念不要混淆，这个阶段需要完成的事情有：  
1）通过一个类的全限定名来获取定义此类的二进制字节流。(.class文件可能来自本地、远程、数据库、zip、jar)  
2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。  
3）在java堆中生成一个代表这个类的Class对象，作为访问方法区中这些数据的入口。  
由于第一点没有指明从哪里获取以及怎样获取类的二进制字节流，所以这一块区域留给我开发者很大的发挥空间。

2、验证阶段(Verification)  
验证是保证二进制字节码在结构上的正确性，具体来说，工作包括检测类型正确性，接入属性正确性（public、private），检查final class 没有被继承，检查静态变量的正确性等。验证阶段是非常重要的，但不是必须的，如果所引用的类经过反复验证，那么可以考虑采用-Xverify:none参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

3、准备阶段(Preparation)  
这个阶段正式为静态变量（被static修饰的变量）分配内存并设置类变量默认值，这个内存分配是发生在方法区中。  
1)、注意这里并没有对实例变量进行内存分配，实例变量将会在对象实例化时随着对象一起分配在JAVA堆中。  
2)、这里设置的初始值，通常是指数据类型的零值，引用类型全部为null。(问题：Java的数据类型和引用类型有哪些？初始值的设置？如何存储？)  
```java
private static int a = 3;
```
这个类变量a在准备阶段后的值是0，将3赋值给变量a是发生在初始化阶段。  
如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。 假设上面的类变量value被定义为： public static final int value = 3； 编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为3。被final和static修饰的变量必须在声明时被显示赋值。

4、解析阶段(Resolution)  
解析的过程就是对类中的接口、类、方法、变量的符号引用进行解析并定位，解析成直接引用（符号引用就是编码是用字符串表示某个变量、接口的位置，直接引用就是根据符号引用翻译出来的地址），并保证这些类被正确的找到。解析的过程可能导致其它的类被加载。需要注意的是，根据不同的解析策略，这一步不一定是必须的，有些解析策略在解析时递归的把所有引用解析，这是early resolution，要求所有引用都必须存在；还有一种策略是late resolution，这也是Oracle 的JDK所采取的策略，即在类只是被引用了，还没有被真正用到时，并不进行解析，只有当真正用到了，才去加载和解析这个类。

5、初始化(Initialization)  
初始化是类加载机制的最后一步，这个时候才正真开始执行类中定义的JAVA程序代码。在前面准备阶段，类变量已经赋过一次系统要求的初始值，在初始化阶段最重要的事情就是对类变量进行初始化，关注的重点是父子类之间各类资源初始化的顺序。  
java类中对类变量指定初始值有两种方式：1、声明类变量时指定初始值；2、使用静态初始化块为类变量指定初始值。  
初始化的时机(主动)：  
1）创建类实例的时候，分别有：1、使用new关键字创建实例；2、通过反序列化方式创建实例。
```java
new Test();
```
2) 反射，没有创建实例
```java
Class.forName(“com.mengdd.Test”);
```
3)调用某个类的静态方法
```java
Test.doSomething();
```
4)访问某个类或者接口的静态变量，或者对该静态变量赋值（如果访问静态编译时常量(即编译时可以确定值的常量)不会导致类的初始化）
```java
int b=Test.a;
Test.a=b;
```
5）初始化某个类的子类。当初始化子类的时候，该子类的所有父类都会被初始化。不过直接通过子类引用父类元素，不会引起子类的初始化。  
6）直接使用java.exe命令来运行某个主类。  
除了上面几种方式会自动初始化一个类，其他访问类的方式都称不会触发类的初始化，称为被动引用。  
类与接口的初始化不同，如果一个类被初始化，则其父类或父接口也会被初始化，但如果一个接口初始化，则不会引起其父接口的初始化。  

初始化的步骤  
1、如果该类还没有加载和连接，则程序先加载该类并连接。  
2、如果该类的直接父类没有加载，则先初始化其直接父类。  
3、如果类中有初始化语句，则系统依次执行这些初始化语句。  
在第二个步骤中，如果直接父类又有直接父类，则系统会再次重复这三个步骤来初始化这个父类，依次类推，JVM最先初始化的总是java.lang.Object类。当程序主动使用任何一个类时，系统会保证该类以及所有的父类都会被初始化。

static块的本质。注意下面的代码：
```java
class StaticBlock {
        static final int c = 3;

        static final int d;

        static int e = 5;
        static {
                d = 5;
                e = 10;
                System.out.println("Initializing");
        }

        StaticBlock() {
                System.out.println("Building");
        }
}

public class StaticBlockTest {
        public static void main(String[] args) {
                System.out.println(StaticBlock.c);
                System.out.println(StaticBlock.d);
                System.out.println(StaticBlock.e);
        }
}
```
这段代码的输出是什么呢？Initialing在c、d、e之前输出，还是在之后？e输出的是5还是10？  
执行一下，结果为：  
3  
Initializing  
5  
10  
答案是3最先输出，Intializing随后输出，e输出的是10，为什么呢？  
原因是这样的：输出c时，由于c是编译时常量，不会引起类初始化，因此直接输出，输出d时，d不是编译时常量，所以会引起初始化操作，即static块的执行，于是d被赋值为5，e被赋值为10，然后输出Initializing，之后输出d为5，e为10。  
但e为什么是10呢？原来，JDK会自动为e的初始化创建一个static块（参考：http://www.java3z.com/cwbwebhome/article/article8/81101.html?id=2497），所以上面的代码等价于：
```java
class StaticBlock {
        static final int d;

        static int e;
        
        static {
           e=5; 
        }

        static {
            d = 5;
            e = 10;
            System.out.println("Initializing");
        }

        StaticBlock() {
            System.out.println("Building");
        }
}    
```
可见，按顺序执行，e先被初始化为5，再被初始化为10，于是输出了10。  
类似的，容易想到下面的代码：  
```java
class StaticBlock {
        static {
                d = 5;
                e = 10;
                System.out.println("Initializing");
        }

        static final int d;

        static int e = 5;

        StaticBlock() {
                System.out.println("Building");
        }
}
```
在这段代码中，对e的声明被放到static块后面，于是，e会先被初始化为10，再被初始化为5，所以这段代码中e会输出为5。  

双亲委派模型
https://blog.csdn.net/mooneal/article/details/78397751
http://www.importnew.com/23742.html
https://gitbook.cn/gitchat/activity/5a751b1391d6b7067048a213
http://ifeve.com/jvm-classloader/
