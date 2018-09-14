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

类加载又可以分为三个大类：Loading(加载)、Linking(连接)、Initialization(初始化)，验证、准备、解析属于连接阶段。  

在这五个阶段中，加载、验证、准备和初始化这四个阶段发生的顺序是确定的，而解析阶段则不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定。另外这里的几个阶段是按顺序开始，而不是按顺序进行或完成，通常在一个阶段执行的过程中调用或激活另一个阶段。 

**系统可能在第一次使用某个类时加载该类，也可能采用预加载机制来加载某个类，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（LinkageError错误）,如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误.当运行某个java程序时，会启动一个java虚拟机进程，两次运行的java程序处于两个不同的JVM进程中，两个jvm之间并不会共享数据。**

![20140317163048593](https://user-images.githubusercontent.com/6982311/44572680-2ff15900-a7b7-11e8-9857-138ef16ac41d.png)

![141101100595683](https://user-images.githubusercontent.com/6982311/44570608-07fef700-a7b1-11e8-913c-9d2c78567ae1.jpg)

1、加载阶段(Loading)  
这个流程中的加载是类加载机制中的一个阶段，这两个概念不要混淆，这个阶段需要完成的事情有：  
1）通过一个类的全限定名来获取定义此类的二进制字节流。(.class文件可能来自本地、远程、数据库、zip、jar)  
2）将这个字节流所代表的静态存储结构转化为**方法区**的运行时数据结构（Data Segment）。  
3）在java堆中生成一个代表这个类的Class对象(java.lang.Class对象)，作为访问方法区中这些数据的入口。  

关于java内存结构：
java的内存管理分为：  
栈（stack 虚拟机栈 本地方法栈）  
堆（heap）  
程序计数器  
方法区（Code Segment-代码段 Data Segment-数据段都是属于方法区）  
比如最通用的tomcat启动一个war包服务，tomcat的类加载器将war中的class字节码加载到jvm的方法区（习惯称之为永久代，实际也在堆中，此处不扩展），存储字节码的的位置被称为Code Segment ，存储静态常量和字符串常量的位置被称为Data Segment。  

2、验证阶段(Verification)  
验证是保证二进制字节码在结构上的正确性，具体来说，工作包括检测类型正确性，接入属性正确性（public、private），检查final class 没有被继承，检查静态变量的正确性等。验证阶段是非常重要的，但不是必须的，如果所引用的类经过反复验证，那么可以考虑采用-Xverify:none参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

3、准备阶段(Preparation)  
这个阶段正式为静态变量（类变量，也即被static修饰的变量，静态方法则成为类方法）分配内存并设置类变量默认值，这个内存分配是发生在方法区中。  
1)、注意这里并没有对实例变量进行内存分配，实例变量将会在对象实例化时随着对象一起分配在JAVA堆中。  
2)、这里设置的初始值，通常是指数据类型的零值，引用类型全部为null。(问题：Java的数据类型和引用类型有哪些？初始值的设置？如何存储？)  
![20180119092157123](https://user-images.githubusercontent.com/6982311/45545403-b9db9180-b84c-11e8-8117-c912acca5ca6.png)

```java
private static int a = 3;
```
这个类变量a在准备阶段后的值是0，将3赋值给变量a是发生在初始化阶段。  
如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。 假设上面的类变量value被定义为： public static final int value = 3； 编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为3。被final和static修饰的变量必须在声明时被显示赋值。

4、解析阶段(Resolution)  
解析的过程就是对类中的接口、类、方法、变量的符号引用进行解析并定位，解析成直接引用（符号引用就是编码是用字符串表示某个变量、接口的位置，直接引用就是根据符号引用翻译出来的地址），并保证这些类被正确的找到。解析的过程可能导致其它的类被加载。需要注意的是，根据不同的解析策略，这一步不一定是必须的，有些解析策略在解析时递归的把所有引用解析，这是early resolution，要求所有引用都必须存在；还有一种策略是late resolution，这也是Oracle 的JDK所采取的策略，即在类只是被引用了，还没有被真正用到时，并不进行解析，只有当真正用到了，才去加载和解析这个类。

5、初始化(Initialization)  
初始化是类加载机制的最后一步，**这个时候才正真开始执行类中定义的JAVA程序代码**。在前面准备阶段，类变量(类变量指的是静态变量，java的非静态变量也即类成员变量在实例化对象时分配内存空间，优先于构造块和构造方法执行之前。)已经赋过一次系统要求的初始值，在初始化阶段最重要的事情就是对类变量进行初始化，关注的重点是父子类之间各类资源初始化的顺序。  
java类中对类变量指定初始值有两种方式：1、声明类变量时指定初始值；2、使用静态初始化块为类变量指定初始值。  
初始化的时机(主动)：  
1）创建类实例的时候，分别有：1、使用new关键字创建实例；2、通过反序列化方式创建实例。
```java
new Test();
```

2）反射，没有创建实例。

```java
Class.forName(“com.mengdd.Test”);
```

3）调用某个类的静态方法
```java
Test.doSomething();
```
4）访问某个类或者接口的静态变量，或者对该静态变量赋值（如果访问静态编译时常量(即编译时可以确定值的常量)不会导致类的初始化）
```java
int b=Test.a;
Test.a=b;
```
5）初始化某个类的子类。当初始化子类的时候，该子类的所有父类都会被初始化。不过直接通过子类引用父类元素，不会引起子类的初始化。  
6）直接使用java.exe命令来运行某个主类。  
除了上面几种方式会自动初始化一个类，其他访问类的方式都称不会触发类的初始化，称为被动引用。 

被final修饰静态字段在操作使用时，不会使类进行初始化，因为在编译期已经将此常量放在常量池。

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
最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。（所有的java.XXX开头的类均被Bootstrap ClassLoader加载，启动类加载器是无法被Java程序直接引用的。）另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变BootstrapClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。  
* ExtentionClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。  
* AppclassLoader也称为SystemAppClass 加载当前应用的classpath的所有类。  

每个Class Loader有自己的命名空间，命名空间由被此Class Loader加载的类组成，不同命名空间的两个类是相互不可见的。不过如果能得到（例如通过反射）另一命名空间中的类所对应的Class对象的引用，则还是可以访问该类的成员变量。

应用程序都是由这三种类加载器互相配合进行加载的，如果有必要，我们还可以加入自定义的类加载器。因为JVM自带的ClassLoader只是懂得从本地文件系统加载标准的java class文件，因此如果编写了自己的ClassLoader，便可以做到如下几点：

1）在执行非置信代码之前，自动验证数字签名。

2）动态地创建符合用户特定需要的定制化构建类。

3）从特定的场所取得java class，例如数据库中和网络中。

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
JVM类加载机制

* 全盘负责，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入

* 父类委托，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类

* 缓存机制，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效  

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

* 通过AppclassLoader来加载，因为它是当前应用下的classpath下的类的的加载器（默认没有使用自定义加载器），在运行到AppclassLoader的loadClass()方法时,根据代码1的分析，它要去它的父加载器ExtentionClassLoader中的loadClass()方法中去，同样在执行过程中，也要去父加载器，它的父加载器是bootStrapClassLoader（它没有父加载器了）。
* 通过findLoadedClass方法看是否加载，如果没有加载，试着用findClass去加载，如果不能加载，返回null，到子加载器中加载，一直进行到AppclassLoader，发现可以加载，于是开始运行。
* 运行到了String str=”Hello”;要加载String类了，它在启动时候是否由bootStrap加载器加载过了。这里假设没有，于是又进行了一轮向上委派，委派到BootStrapClassLoader的时候，正好它可以加载。

注意：  
1.bootStrap和Extclassloader 两个加载器在启动后会加载所有规定路径下的类。即：启动后会加载java核心包。
2.appclassloader 加载器对应的应用类，只有在被用到的时候才会被加载。

那么为什么要设计双亲委派模型呢？  
假设，整个系统中没有这个设计，还是上面的程序，没有双亲委派模型，那么String这个类就由AppclassLoader加载了，然后可能在有另一个自定义的加载器加载String这个类了，那么程序就乱套了！！到处都是真假难辨的String。而有了双亲委派模型，不管是哪个加载器加载的String，这个类都是由BootStrapClassLoader加载。双亲委派模型最主要的作用是保证java核心类的安全性。

**注意：一个类可以被多次加载，但是一个加载器只能加载一次，而且判断一个类是不是相同的，是比较 包名+类名+类加载器id 
当某个classloader加载的所有类实例化的对象都被GC回收了，那么这个加载器就会被回收掉。**

#### Class.forName()和ClassLoader.loadClass()区别
```java
package com.neo.classloader;
public class loaderTest { 
        public static void main(String[] args) throws ClassNotFoundException { 
                ClassLoader loader = HelloWorld.class.getClassLoader(); 
                System.out.println(loader); 
                //使用ClassLoader.loadClass()来加载类，不会执行初始化块 
                loader.loadClass("Test2"); 
                //使用Class.forName()来加载类，默认会执行初始化块 
//                Class.forName("Test2"); 
                //使用Class.forName()来加载类，并指定ClassLoader，初始化时不执行静态块 
//                Class.forName("Test2", false, loader); 
        } 
}

public class Test2 { 
        static { 
                System.out.println("静态初始化块执行了！"); 
        } 
}
```
Class.forName()：将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块；

ClassLoader.loadClass()：只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。

注：

Class.forName(name, initialize, loader)带参函数也可控制是否加载static块。并且只有调用了newInstance()方法调用构造函数，才创建类的对象 。

#### ClassNotFoundException
1、当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。

2、当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。

3、如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；

4、若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。



### 自定义类加载器
通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java 类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。自定义类加载器一般都是继承自 ClassLoader 类，从上面对 loadClass 方法来分析来看，我们只需要重写 findClass 方法即可。下面我们通过一个示例来演示自定义类加载器的流程：

```java
package com.neo.classloader;
 
import java.io.*;
 
public class MyClassLoader extends ClassLoader {
 
    private String root;
 
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }
 
    private byte[] loadClassData(String className) {
        String fileName = root + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
        try {
            InputStream ins = new FileInputStream(fileName);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, length);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
 
    public String getRoot() {
        return root;
    }
 
    public void setRoot(String root) {
        this.root = root;
    }
 
    public static void main(String[] args)  {
 
        MyClassLoader classLoader = new MyClassLoader();
        classLoader.setRoot("E:\\temp");
 
        Class<?> testClass = null;
        try {
            testClass = classLoader.loadClass("com.neo.classloader.Test2");
            Object object = testClass.newInstance();
            System.out.println(object.getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```
自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。由于这里只是演示，我并未对class文件进行加密，因此没有解密的过程。这里有几点需要注意：

1、这里传递的文件名需要是类的全限定性名称，即com.paddx.test.classloading.Test格式的，因为 defineClass 方法是按这种格式进行处理的。

2、最好不要重写loadClass方法，因为这样容易破坏双亲委托模式。

3、这类Test 类本身可以被 AppClassLoader 类加载，因此我们不能把 com/paddx/test/classloading/Test.class 放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由AppClassLoader 加载，而不会通过我们自定义类加载器来加载。

### 题目
```java
class SingleTon {
	private static SingleTon singleTon = new SingleTon();
	public static int count1;
	public static int count2 = 0;
 
	private SingleTon() {
		count1++;
		count2++;
	}
 
	public static SingleTon getInstance() {
		return singleTon;
	}
}
 
public class Test {
	public static void main(String[] args) {
		SingleTon singleTon = SingleTon.getInstance();
		System.out.println("count1=" + singleTon.count1);
		System.out.println("count2=" + singleTon.count2);
	}

```
错误答案:

count1=1
count2=1

正确答案:

.
.
.

count1=1
count2=0

为什么呢？

1:SingleTon singleTon = SingleTon.getInstance();调用了类的SingleTon调用了类的静态方法，触发类的初始化。  
2:类加载的时候在准备过程中为类的静态变量分配内存并初始化默认值 singleton=null count1=0,count2=0。  
3:类初始化，为类的静态变量赋值和执行静态代码快。singleton赋值为new SingleTon()调用类的构造方法。  
4:调用类的构造方法后count=1;count2=1。  
5:继续为count1与count2赋值,此时count1没有赋值操作,所以count1为1,但是count2执行赋值操作就变为0。  

### 参考文章
https://blog.csdn.net/mooneal/article/details/78397751

http://www.importnew.com/23742.html

https://www.cnblogs.com/dongguacai/p/5860241.html

https://www.cnblogs.com/cz123/p/6867345.html

http://www.importnew.com/16799.html

https://blog.csdn.net/yangcheng33/article/details/52631940

https://segmentfault.com/a/1190000013504913

https://blog.csdn.net/vking_wang/article/details/12379333

http://www.javafxchina.net/blog/2016/07/osgi-00-what/

https://blog.csdn.net/lovebuzhidao/article/details/80326698

https://www.cnblogs.com/wxd0108/p/6681618.html

### 疑问
https://gitbook.cn/gitchat/activity/5a751b1391d6b7067048a213

* 与线程相关类加载器
* 如何使用类加载器实现简单的模块间隔离，阿里的类隔离是如何做的
