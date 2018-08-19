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
*系统可能在第一次使用某个类时加载该类，也可能采用预加载机制来加载某个类，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（LinkageError错误）,如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误.当运行某个java程序时，会启动一个java虚拟机进程，两次运行的java程序处于两个不同的JVM进程中，两个jvm之间并不会共享数据。*

1、加载阶段  
这个流程中的加载是类加载机制中的一个阶段，这两个概念不要混淆，这个阶段需要完成的事情有：  
1）通过一个类的全限定名来获取定义此类的二进制字节流。(.class文件可能来自本地、远程、数据库、zip、jar)  
2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。  
3）在java堆中生成一个代表这个类的Class对象，作为访问方法区中这些数据的入口。  
由于第一点没有指明从哪里获取以及怎样获取类的二进制字节流，所以这一块区域留给我开发者很大的发挥空间。

2、验证阶段  





