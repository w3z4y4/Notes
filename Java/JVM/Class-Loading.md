## Java类加载机制---20180819
JAVA源码编译由三个过程组成：  
1、源码编译机制    
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
但e为什么是10呢？原来，JDK会自动为e的初始化创建一个static块，所以上面的代码等价于：
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

### 类加载器
在java.lang包中有一个抽象类，名为 ClassLoader ,这个类是所有类加载器的超类。这个类中有这几个重要方法：  
getParent() 返回该类加载器的父类加载器。  
loadClass(String name) 加载名称为name的类，返回的结果是 java.lang.Class类的实例。  
findClass(String name) 查找名称为name的类，返回的结果是 java.lang.Class类的实例。  
findLoadedClass(String name) 查找名称为name的已经被加载过的类，返回的结果是java.lang.Class类的实例。  
defineClass(String name, byte[] b, int off, int len) 把字节数组 b中的内容转换成Java 类，返回的结果是 java.lang.Class类的实例。这个方法被声明为 final的。  
resolveClass(Class c) 链接指定的 Java 类。  
从上面的说明可以看出来ClassLoader通过loadClass()方法来加载字节码文件，然后返回一个字节码对象。来看一下源码：
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
        /*首先通过findLoadedClass方法检查这个类是否被加载，过程是去元空间中查看是否有当前类的信息，是否被加载需要检查 包名+类名+类加载器的id*/
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                /*双亲委派模型，后面细讲*/
                    if (parent != null) {
                    /*如果没有被加载，就调用它的父加载器去加载*/
                        c = parent.loadClass(name, false);
                    } else {
                    /*一直向上委派到bootStrap类加载器*/
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
/*查找这个类是否能被这个类加载器加载，不能的话就返回null，回到它的子加载器，递归回来，查看子加载器是否能被加载*/
                    long t1 = System.nanoTime();
                    c = findClass(name);
。。。。
                }
            }
            if (resolve) {
            /*如果这个类加载了，那么就链接它*/
                resolveClass(c);
            }
            return c;
        }
    }
```

Java语言系统自带有三个类加载器:
* Bootstrap ClassLoader 
最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变BootstrapClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。  
* ExtentionClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。  
* AppclassLoader也称为SystemAppClass 加载当前应用的classpath的所有类。  
这三个加载器都是jvm通过一个Launcher类来初始化创造的，来看一下jvm是怎么加载它们的：   
<img width="300" alt="20171030201134651" src="https://user-images.githubusercontent.com/6982311/44326028-fd7cee80-a48c-11e8-9bb0-1e7054034c71.png">
有兴趣的可以查看Launcher的源码，三个类加载器的创建分别在这相应的三个内部类中。这里简单说一下，并介绍类加载器的继承体系和父子体系。创建AppclassLoader: 
```java
/*静态内部类继承了URLClassLoader*/
 static class AppClassLoader extends URLClassLoader {
        final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

        public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
        /*得到classpath下的所有类的路径*/
            final String var1 = System.getProperty("java.class.path");
            final File[] var2 = var1 == null?new File[0]:Launcher.getClassPath(var1);
            return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
                public Launcher.AppClassLoader run() {
                    URL[] var1x = var1 == null?new URL[0]:Launcher.pathToURLs(var2);
    /*这里创建AppClassLoader，注意这里并没有去加载路径下的类*/
                    return new Launcher.AppClassLoader(var1x, var0);
                }
            });
        }
```

ExtentionClassLoader 类似于这种方式去创建，那BootStrapClassLoader是怎么创建的呢？ 其实java中并没有这个类，只是一个概念，它嵌入到jvm中了，由c++编写的。也就是jvm一启动就会有这个类，但是在Launcher类中规定了它加载类所在的路径：
```java
 private static String bootClassPath = System.getProperty("sun.boot.class.path");
```
从上面的代码也看到了 AppclassLoader 继承的是URLClassLoader,但它的父加载器却是ExtentionClassLoader ，采用这种方式是有两个方面：

* 继承URLClassLoader,是为了继承这个类中的findClass()方法为了找到规定路径下的类。继续看URLClassLoader的源码：
```java
protected Class<?> findClass(final String name)
        throws ClassNotFoundException
    {
        。。。。
                    public Class<?> run() throws ClassNotFoundException {
                    /*解析类的路径*/
                        String path = name.replace('.', '/').concat(".class");
                        Resource res = ucp.getResource(path, false);
                        if (res != null) {
                            try {
           /*通过字节码文件的路径获得相应的Class类*/
                                return defineClass(name, res);
                            } 
        。。。。。
        return result;
    }
```
前面有个问题是只是找到字节码文件，并没有加载它，答案就在这里，通过defineClass(name, res)来加载它。这里又引出一个问题，为什么不在loadClass()中加载它，而是通过findClass()来加载？继续留问题，后面讲。  
* 所有类加载器中的基类ClassLoader中有一个属性。  
```java
private final ClassLoader parent;
```
是通过类与类组合的方式来定义一个父加载器的，为了构成双亲委派模型。

### 双亲委派模型
![20170211135054825](https://user-images.githubusercontent.com/6982311/44326722-24d4bb00-a48f-11e8-9d56-d581496e56c9.png)
在对这个图回头看代码1，结合下面的例子来理解. 
这里有一个Hello.java
```java
public Class Hello {
    public void test() {
        String str="Hello";
    }
}
```

当编译完成后，有了一个Hello.class的文件。现在要加载这个字节码文件：

* 通过AppclassLoader来加载，因为它是当前应用下的classpath下的类的的加载器（默认没有使用自定义加载器），在运行到AppclassLoader的loadClass()方法时,根据代码1的分析，它要去它的父加载器ExtentionClassLoader中的loadClass()方法中去，同样在执行过程中，也要去父加载器中走一圈，它的父加载器是bootStrapClassLoader（它没有父加载器了）
* 通过findLoadedClass方法看是否加载，如果没有加载，试着用findClass去加载，如果不能加载，返回null，到子加载器中加载，一直进行到AppclassLoader，发现可以加载，于是开始运行。
* 运行到了String str=”Hello”;要加载String类了，（事实上，我并不清楚它在启动时候是否由bootStrap加载器加载过吗-。-，我推测这种系统自定义的类，应该是加载过了。这里假设没有），于是又进行了一轮向上委派，委派到BootStrapClassLoader的时候，正好它可以加载。

这里回答一下，关于String类是否加载引起的思考，到底类加载是在什么时候呢？ 资料上的回答是：什么情况下需要开始类加载过程的第一个阶段:”加载”。虚拟机规范中并没强行约束，这点可以交给虚拟机的的具体实现自由把握。 没有答案就自己找答案吧:
```java
public class StringDemo {
    //空的
}

public class IntDemo {
    private Integer a=3;
    private Integer a1=3;
    public static void main(String[] args){
        IntDemo intDemo=new IntDemo();
    //  StringDemo stringDemo=new StringDemo();
    }
}
```

在idea中加入jvm参数 -XX:+TraceClassLoading 追踪类加载的顺序。  
第一次运行，StringDemo相关被注释掉： 
[Loaded sun.launcher.LauncherHelper from D:\jdk\jre\lib\rt.jar]  
[Loaded sun.misc.URLClassPathFileLoader1 from D:\jdk\jre\lib\rt.jar]   
[Loaded java.net.Inet6Address from D:\jdk\jre\lib\rt.jar]  
[Loaded IntDemo from file:/F:/code/JVM/out/production/JavaString/]  
[Loaded java.net.URI from D:\jdk\jre\lib\rt.jar]  
[Loaded sun.launcher.LauncherHelper$FXHelper from D:\jdk\jre\lib\rt.jar]  
[Loaded java.lang.Void from D:\jdk\jre\lib\rt.jar]  

第二次运行，去掉注释:

[Loaded sun.misc.URLClassPathFileLoader1 from D:\jdk\jre\lib\rt.jar]   
[Loaded IntDemo from file:/F:/code/JVM/out/production/JavaString/]  
[Loaded sun.launcher.LauncherHelper$FXHelper from D:\jdk\jre\lib\rt.jar]   
[Loaded java.lang.Void from D:\jdk\jre\lib\rt.jar]  
[Loaded StringDemo from file:/F:/code/JVM/out/production/JavaString/]  

两段信息对比可以看出两点：

bootStrap和Extclassloader 两个加载器在启动后会加载所有规定路径下的类。即：启动后会加载java核心包。  
appclassloader 加载器对应的应用类，只有在被用到的时候才会被加载。比如上面的StringDemo类。  

**为什么要设计双亲委派模型呢？**
假设，整个系统中没有这个设计，还是上面的程序，没有双亲委派模型，那么String这个类就由AppclassLoader加载了，然后可能在有另一个自定义的加载器加载String这个类了，那么程序就乱套了！！到处都是真假难辨的String。而有了双亲委派模型，不管是哪个加载器加载的String，这个类都是由BootStrapClassLoader加载。双亲委派模型最主要的作用是保证java核心类的安全性。

*注意：一个类可以被多次加载，但是一个加载器只能加载一次，而且判断一个类是不是相同的，是比较 包名+类名+类加载器id 
当某个classloader加载的所有类实例化的对象都被GC回收了，那么这个加载器就会被回收掉。*

### .双亲委派模型是不可改变的吗？
首先，明确双亲委派模型是一种规范，在自定义类加载器的时候完全可以重写loadClass()方法中的逻辑。这里回答一下前面的问题，为什么不用loadClass()实现类加载的功能，而是用findClass()。这是为了把委派模型的逻辑和类加载器要实现的逻辑分离开了。所以一般自定义类加载器loadClass()一般不动，而是重写findClass()。   
什么时候要破坏委派模型呢？

* 第一种情况  
来看一个例子java需要连接数据库，但是数据库的品种这么多，每家厂商写的程序都不同，为了让每个厂商统一，java制定了一系列规范的接口，放在BootStrapClassLoader可以加载的路径中。所有厂商只要按照这些接口规范写就好了，并且统一有一个管理的类在DriverManager。这样java制定者和厂商都开开心心，这时候出现了一个问题。 
根据委派模型，A类中引用了B类
```java
public class A{
    private B = new B();
}
```
那么默认的B的加载器也是由A的加载器加载的！！ 所以通过DriverManager统一来得到Driver的话，那么BootStrapClassLoader默认是加载 java.sql 包下的Driver接口。但实际上必须要加载它的实现类。  
可是，根据委派模型，父类加载器去调用子类加载器是不可能完成的。   
必须由BootStrapClassLoader来加载，这就难办了，BootStrapClassLoader说我的工作就是加载 %JRE_HOME%\lib 下的，要我加工作量，不干！！况且我也找不到在哪啊，我只知道它的接口啊，并不知道它的实现类。  
于是，出现了一个设计,在线程Tread类中内置一个:  
```java
/* The context ClassLoader for this thread 默认是appclassloader ，可以自己设置*/
    private ClassLoader contextClassLoader;
```
因此在父类加载器需要调用子类加载器的时候，就可以通过
```java
Thread.currentThread().getContextClassLoader();
```
来获取想要的类加载器。   
下面来看JDBC的源码：   
在java.sql.DriverManager中  
```java
//从Thread.currentThread中取出appclassloader
 ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }

 是怎么使用这个classLoader的呢？

  private static boolean isDriverAllowed(Driver driver, ClassLoader classLoader) {
        boolean result = false;
        if(driver != null) {
            Class<?> aClass = null;
            try {
       //是通过Class.forName()并指定类加载器。
                aClass =  Class.forName(driver.getClass().getName(), true, classLoader);
            } 
            。。。。
        return result;
    }
```
* 第二种情况
java web中的热部署，指的是热部署是在不重启 Java 虚拟机的前提下，能自动侦测到 class 文件的变化，更新运行时 class 的行为。在编写java程序的时候，每一次改动源代码，就必须重启一次应用程序，那java的这个缺陷是由什么导致的呢？  
1) java类只能由类加载器加载一次，这样在改动代码后，如果不重启就不能被重新加载。  
2) java类一般被系统自带的appclassloader 来加载。  
然而热部署对于开发来说实在太重要了，想想当你调试js代码的时候，一个屏幕源代码，一个屏幕浏览器，随时随地的观察代码带来的变化。而java对于这一点，由于它语言的特性导致很难做到这一点。   
如果想要实现热部署，需要做哪些工作呢？  
* 销毁ClassLoader。  
* 创建新的ClassLoader去加载更新后的class类文件。  
第一步销毁ClassLoader，如果要做到这一步，那么这个类文件一定不能是appclassloader加载的，因此要自定义ClassLoader。   
第二步，要做到的话，首先必须有一个监听器去监听类发生变化，然后才能相应的创建ClassLoader去加载。   
这也是Tomcat服务器在处理类加载的时候进行的做法，每一个web应用都有一个相应的类加载器。原理图如下：   
![4bfd2e34-b46f-35fc-bc60-054010f2a980](https://user-images.githubusercontent.com/6982311/44327495-61091b00-a491-11e8-8f3e-f022cac08379.png) 
并且Tomcat自定义的类加载器并不遵循双亲委派模型，而是先检查自身可不可以加载这个类，而不是先委派。前面也提到了，BootStrap，System和Common 这三个类加载器遵守委派模型同时加载java以及tomcat本身的核心类库。这样做的好处是更好的实现了应用的隔离，但是坏处就是加大了内存浪费，同样的类库要在不同的app中都要加载一份。（当然可以配置使得不是所有的app都要被加载，默认是全部加载）   
Tomcat中当类发生改变时，监听器监听到触发StandardContext.reload()，然后销毁以前的类加载器，重新创造一个类加载器。   
在使用idea开发的时候，可以在debug模式下，配置下图两个地方：   
<img width="492" alt="20171031205419560" src="https://user-images.githubusercontent.com/6982311/44327500-64040b80-a491-11e8-9f46-d6a7d20f933f.png">
就可以实现方法层面的修改，即你修改了方法中的代码，不需要tomcat重新发布，也不需要重启服务器。就可以实现热部署了。  

这是因为jdk1.6增加了agentmain方式，实现了运行时动态性（通过The Attach API 绑定到具体VM）。其基本实现是通过JVMTI的retransformClass/redefineClass进行method body级的字节码更新，ASM、CGLib之类基本都是围绕这些在做动态性。
