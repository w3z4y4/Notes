## Java类加载机制进阶---20180830

### 双亲委派模型是不可改变的吗？

首先，明确双亲委派模型是一种规范，在自定义类加载器的时候完全可以重写loadClass()方法中的逻辑。这里回答一下前面的问题，为什么不用loadClass()实现类加载的功能，而是用findClass()。这是为了把委派模型的逻辑和类加载器要实现的逻辑分离开了。所以一般自定义类加载器loadClass()一般不动，而是重写findClass()。 


### 什么时候要破坏委派模型呢？
#### jdbc驱动的类加载
一句话概括：Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。
这些 SPI 的接口由 Java 核心库来提供，而这些 SPI 的实现代码则是作为 Java 应用所依赖的 jar 包被包含进类路径（CLASSPATH）里。SPI接口中的代码经常需要加载具体的实现类。那么问题来了，SPI的接口是Java核心库的一部分，是由启动类加载器(Bootstrap Classloader)来加载的；SPI的实现类是由系统类加载器(System ClassLoader)来加载的。引导类加载器是无法找到 SPI 的实现类的，因为依照双亲委派模型，BootstrapClassloader无法委派AppClassLoader来加载类。
而线程上下文类加载器破坏了“双亲委派模型”，可以在执行线程中抛弃双亲委派加载链模式，使程序可以逆向使用类加载器。

先来看平时是如何使用mysql获取数据库连接的：
```java
// 加载Class到AppClassLoader（系统类加载器），然后注册驱动类
// Class.forName("com.mysql.jdbc.Driver"); 
String url = "jdbc:mysql://localhost:3306/testdb";    
// 通过java库获取数据库连接
Connection conn = java.sql.DriverManager.getConnection(url, "name", "password"); 
```

两个疑问：传统写法为什么要写Class.forName("com.mysql.jdbc.Driver");？为什么要通过DriverManager去获取连接，为什么不用Driver的实例获取？

第一个问题分析：

先看传统写法，com.mysql.jdbc.Driver 类的源码：
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
      //
      // Register ourselves with the DriverManager
      //
      static {
          try {
              java.sql.DriverManager.registerDriver(new Driver());
          } catch (SQLException E) {
              throw new RuntimeException("Can't register driver!");
         }
     }
 
     /**
      * Construct a new driver and register it with DriverManager
      * 
      * @throws SQLException
      *             if a database error occurs.
      */
     public Driver() throws SQLException {
         // Required for Class.forName().newInstance()
     }
 }
```
static语句做了一件事：生成驱动实例，并向DriverManager注册。所谓注册，就是将driver的信息保存起来，以便后来取用。第一个问题解决。

```java
public static Connection getConnection(String url,
          String user, String password) throws SQLException {
          java.util.Properties info = new java.util.Properties();
  
          if (user != null) {
              info.put("user", user);
          }
          if (password != null) {
              info.put("password", password);
         }
 
         return (getConnection(url, info, Reflection.getCallerClass()));
     }
     
private static Connection getConnection(
         String url, java.util.Properties info, Class<?> caller) throws SQLException {
         /*
          * When callerCl is null, we should check the application's
          * (which is invoking this class indirectly)
          * classloader, so that the JDBC driver class outside rt.jar
          * can be loaded from here.
          */
         ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
         synchronized(DriverManager.class) {
             // synchronize loading of the correct classloader.
             if (callerCL == null) {
                 callerCL = Thread.currentThread().getContextClassLoader();
             }
         }
 
         if(url == null) {
             throw new SQLException("The url cannot be null", "08001");
         }
 
         println("DriverManager.getConnection(\"" + url + "\")");
 
         // Walk through the loaded registeredDrivers attempting to make a connection.
         // Remember the first exception that gets raised so we can reraise it.
         SQLException reason = null;
 
         for(DriverInfo aDriver : registeredDrivers) {
             // If the caller does not have permission to load the driver then
             // skip it.
             if(isDriverAllowed(aDriver.driver, callerCL)) {
                 try {
                     println("    trying " + aDriver.driver.getClass().getName());
                     Connection con = aDriver.driver.connect(url, info);
                     if (con != null) {
                         // Success!
                         println("getConnection returning " + aDriver.driver.getClass().getName());
                         return (con);
                     }
                 } catch (SQLException ex) {
                     if (reason == null) {
                         reason = ex;
                     }
                 }
 
             } else {
                 println("    skipping: " + aDriver.getClass().getName());
             }
 
         }
 
         // if we got here nobody could connect.
         if (reason != null)    {
             println("getConnection failed: " + reason);
             throw reason;
         }
 
         println("getConnection: no suitable driver found for "+ url);
         throw new SQLException("No suitable driver found for "+ url, "08001");
     }
```
Reflection.getCallerClass()是一个native方法，返回调用的类，这里是DriverManager。什么是native方法暂且不提，题外话：从 jdk 7u40 开始，Oracle 已经弃用了 sun.reflect.package 包里不易理解的 Reflection.getCallerClass（int）方法。在 Java 7 中，通过设置 Java 命令行选项 Djdk.reflect.allowGetCallerClass，可以继续使用该方法。但在 Java 8 及以后的版本中，该方法将被彻底删除，调用它会导致 UnsupportedOperationException 异常。类似的， JDK8 中的 @java.lang.Class 的 forName(className) 方法也加了@Deprecated 这个注解，因为forName方法里面也调用给到了Reflection.getCallerClass()。Oracle建议开发人员不要使用sun包，但是有一系列的项目都依赖于getCallerClass()方法，如Jigsaw和Lambda，Intellij IDEA。在spring-loaded的源码这里也调用了getCallerClass()方法，作为临时的方案去兼容JDK 7u25，需要添加系统属性：jdk.reflect.allowGetCallerClass。

caller.getClassLoader()获取DriverManager的类加载器是Bootstrap Classloader，无法获取，也就是null。因此通过Thread.currentThread().getContextClassLoader()获取线程上下文类加载器。查看Thread类的源码可以知道，Thread有一个私有变量：
```java
/* The context ClassLoader for this thread */
    private ClassLoader contextClassLoader;
```

contextClassLoader由第一个线程的类加载器设置，即主线程main。那第一个启动的线程（包含main方法的那个线程）里面的contextClassLoader是谁设置的呢？？？

这就要看 sun.misc.Launcher 这个类的源码。Launcher是JRE中用于启动程序入口main()的类。
```java
loader = AppClassLoader.getAppClassLoader(extcl);

Thread.currentThread().setContextClassLoader(loader);
```

这里截取的两行代码出自 Launcher 的构造方法。第一行用一个扩展类加载器extcl构造了一个系统类加载器loader，第二行把loader设置为当前线程（包含main方法）的类加载器。所以，我们启动一个线程的时候，如果之前都没有调用 setContextClassLoader 方法明确指定的话，默认的就是系统类加载器。

回到isDriverAllowed方法：
```java
private static boolean isDriverAllowed(Driver driver, ClassLoader classLoader) {
    boolean result = false;
    if(driver != null) {
        Class<?> aClass = null;
        try {
        // 传入的classLoader为调用getConnetction的线程上下文类加载器，从中寻找driver的class对象
            aClass =  Class.forName(driver.getClass().getName(), true, classLoader);
        } catch (Exception ex) {
            result = false;
        }
    // 注意，只有同一个类加载器中的Class使用==比较时才会相等，此处就是校验用户注册Driver时该Driver所属的类加载器与调用时的是否同一个
    // driver.getClass()拿到就是当初执行Class.forName("com.mysql.jdbc.Driver")时的应用AppClassLoader
        result = ( aClass == driver.getClass() ) ? true : false;
    }

    return result;
}
```

可以看到这儿TCCL（ThreadContextClassLoader）的作用主要用于校验注册的driver是否属于调用线程的Classloader。例如tomcat多个webapp都有自己的Classloader，如果它们都自带 mysql-connect.jar包，那底层Classloader的DriverManager里将注册多个不同类加载器的Driver实例，想要区分只能靠TCCL了。


回到DriverManager的getConnection 方法：
```java
for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info);
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }
```
在getConection方法中，会不断尝试的connect方法中，传入的URL也将会被先判定是不是自己可接受的URL，然后再执行，各个厂商提供的驱动自己编写是否支持这个URL的代码。也就是说，当我们注册了多个数据库驱动，mysql，oracle等；DriverManager都帮我们管理了，它会取出一个符合条件的driver，就不用我们在程序里自己去控制了。这就是第二个问题的答案：DriverManager会帮你匹配注册的driver和URL，决定使用哪个驱动。

#### JDBC4.0自动加载驱动器类
从JDK1.6开始，Oracle就将修改了添加了新的加载JDBC驱动的方式。即JDBC4.0。在启动项目或是服务时，会判断当前classspath中的所的jar包，并检查META-INF目录下，是否包含services文件夹，如果包含，就会将里面的配置加载成相应的服务。

如Oracle11g的ojdbc6.jar包：

META-INF/services/jdbc.sql.Driver文件内容只有一行，即实现java.sql.Driver的类：
oracle.jdbc.OracleDriver

Oracle在随即发布的mysql-connector-java-5.1.8.jar中，也同样添加了上述的特性：
里面的内容，也是一句，即：
com.mysql.jdbc.Driver

为了验证是否会自动加载数据库驱动类，我们书写一段Java代码：
```java
 public static void main(String[] args) throws Exception {
         //从DriverManager中获取所有驱动类，遍历并输出
         Enumeration<java.sql.Driver> en = DriverManager.getDrivers();
         while(en.hasMoreElements()){
             java.sql.Driver d = en.nextElement();
             System.err.println(d.toString());
         }
 }
```

这就是为什么不用写Class.forName("com.mysql.jdbc.Driver")的原因。

TCCL破坏了双亲委派模型，根据委派模型，A类中引用了B类
```java
public class A{
    private B = new B();
}
```
那么默认的B的加载器也是由A的加载器加载的！！ 

所以通过DriverManager统一来得到Driver的话，那么BootStrapClassLoader默认是加载 java.sql 包下的Driver接口。但实际上必须要加载它的实现类。 
可是，根据委派模型，父类加载器去调用子类加载器是不可能完成的，必须由BootStrapClassLoader来加载，这就难办了，BootStrapClassLoader说我的工作就是加载 %JRE_HOME%\lib 下的，要我加工作量，不干！！况且我也找不到在哪啊，我只知道它的接口啊，并不知道它的实现类。

于是，出现了一个设计,在线程Tread类中内置一个
```java
/* The context ClassLoader for this thread 默认是appclassloader ，可以自己设置*/
    private ClassLoader contextClassLoader;
```

因此在父类加载器需要调用子类加载器的时候，就可以通过
```java
Thread.currentThread().getContextClassLoader();
```
来获取想要的类加载器。 

#### Tomcat 7 的类加载机制
java web中的热部署，指的是热部署是在不重启 Java 虚拟机的前提下，能自动侦测到 class 文件的变化，更新运行时 class 的行为。在编写java程序的时候，每一次改动源代码，就必须重启一次应用程序，那java的这个缺陷是由什么导致的呢？

java类只能由类加载器加载一次，这样在改动代码后，如果不重启就不能被重新加载。
java类一般被系统自带的appclassloader 来加载。
然而热部署对于开发来说实在太重要了，想想当你调试js代码的时候，一个屏幕源代码，一个屏幕浏览器，随时随地的观察代码带来的变化。而java对于这一点，由于它语言的特性导致很难做到这一点。 
如果想要实现热部署，需要做哪些工作呢？

销毁ClassLoader
创建新的ClassLoader去加载更新后的class类文件。
第一步销毁ClassLoader，如果要做到这一步，那么这个类文件一定不能是appclassloader加载的，因此要自定义ClassLoader。 
第二步，要做到的话，首先必须有一个监听器去监听类发生变化，然后才能相应的创建ClassLoader去加载。 
这也是Tomcat服务器在处理类加载的时候进行的做法，每一个web应用都有一个相应的类加载器。原理图如下： 

![4bfd2e34-b46f-35fc-bc60-054010f2a980](https://user-images.githubusercontent.com/6982311/44792895-83104500-abd7-11e8-8dac-335ec0473c86.jpg)

* Bootstrap 

这个类加载器包含Java虚拟机提供的基本运行时类，以及来自System Extensions目录 
（$JAVA_HOME/jre/lib/ext）中的JAR文件的所有类。注意：有些JVM可能将其实现为多个类加载器，比如HotSpot就分为启动类加载器（Bootstrap ClassLoader）和扩展类加载器（Extension ClassLoader）。

* System 

这个类加载器通常是通过CLASSPATH环境变量的内容初始化的。 所有这些类对于Tomcat和Web应用程序都是可见的。但是，标准Tomcat启动脚本（$CATALINA_HOME/bin/catalina.sh或％CATALINA_HOME％\bin\catalina.bat）完全忽略了CLASSPATH环境变量本身的内容，而是从以下存储库（repositories）构建System类加载器:

1.$CATALINA_HOME/bin/bootstrap.jar：包含用于初始化Tomcat服务器的main()方法以及它所依赖的类加载器实现类。

2.$CATALINA_BASE/bin/tomcat-juli.jar或者$CATALINA_HOME/bin/tomcat-juli.jar：日志实现类。其中包括称为Tomcat JULI的java.util.loggingAPI的增强类以及由Tomcat内部使用的ApacheCommons Logging库的软件包重命名副本。

* Common

这个类加载器包含了对Tomcat内部类和所有Web应用程序都可见的其他类。通常，应用程序类别不应放置在此处。这个类加载器搜索的位置由$CATALINA_BASE/conf/catalina.properties中的common.loader属性定义。默认设置将按照它们列出的顺序搜索以下位置：

1.$CATALINA_BASE/lib下没打包的classes和资源文件

2.$CATALINA_BASE/lib下的jar

3.$CATALINA_HOME/lib下没打包的classes和资源文件

4.$CATALINA_HOME/lib下的jar

* WebappX 为每个部署在单个Tomcat实例中的Web应用程序创建类加载器。

Web应用程序的/WEB-INF/classes目录中的所有未打包的类和资源，以及Web应用程序的/WEB-INF/lib目录下的JAR文件中的类和资源都可以被此Web应用程序访问，但不能访问到其他的。

如上所述，Web应用程序类加载器与默认的Java委托模型不同(即双亲委派)。当处理从Web应用程序的WebappX类加载器加载类的请求时，该类加载器将首先在本地存储库中查找，而不是在查找之前进行委托。 但是有例外，作为JRE基类的一部分的类不能被覆盖。例如J2SE 1.4+中的XML解析器组件，以及Java 8将会使用的类。最后，包含Servlet API类的任何JAR文件将被类加载器显式忽略 - 你的Web应用程序中不应该包含这些类。 Tomcat中的所有其他类加载器都遵循通常的代理模式。

因此，从Web应用程序的角度来看，类或资源加载按以下顺序在以下存储库中查找：

Bootstrap classes of your JVM

web应用的/WEB-INF/classes

web应用的/WEB-INF/lib/*.jar

System class loader classes (如上所述)

Common class loader classes (如上所述)

如果Web应用程序类加载器配置为<Loader delegate ="true"/>，那么顺序将变为：

Bootstrap classes of your JVM

System class loader classes (如上所述)

Common class loader classes (如上所述)

web应用的/WEB-INF/classes

web应用的/WEB-INF/lib/*.jar

并且Tomcat自定义的类加载器并不遵循双亲委派模型，而是先检查自身可不可以加载这个类，而不是先委派。前面也提到了，BootStrap，System和Common 这三个类加载器遵守委派模型同时加载java以及tomcat本身的核心类库。这样做的好处是更好的实现了应用的隔离，但是坏处就是加大了内存浪费，同样的类库要在不同的app中都要加载一份。（当然可以配置使得不是所有的app都要被加载，默认是全部加载） 
Tomcat中当类发生改变时，监听器监听到触发StandardContext.reload()，然后销毁以前的类加载器，重新创造一个类加载器。 

在使用idea开发的时候，可以在debug模式下，配置下图两个地方： 

![20171031205419560](https://user-images.githubusercontent.com/6982311/44792924-958a7e80-abd7-11e8-9070-a6879600b173.jpg)

就可以实现方法层面的修改，即你修改了方法中的代码，不需要tomcat重新发布，也不需要重启服务器。就可以实现热部署了。

这是因为jdk1.6增加了agentmain方式，实现了运行时动态性（通过The Attach API 绑定到具体VM）。其基本实现是通过JVMTI的retransformClass/redefineClass进行method body级的字节码更新，ASM、CGLib之类基本都是围绕这些在做动态性。

#### osgi的类加载
OSGi（Open Service Gateway Initiative）：是OSGi联盟指定的一个基于Java语言的动态模块化规范，这个规范最初是由Sun、IBM、爱立信等公司联合发起，目的是使服务提供商通过住宅网管为各种家用智能设备提供各种服务，后来这个规范在Java的其他技术领域也有不错的发展，现在已经成为Java世界中的“事实上”的模块化标准，并且已经有了Equinox、Felix等成熟的实现。OSGi在Java程序员中最著名的应用案例就是Eclipse IDE。

开发者们所开发的OSGi组件都称为Bundle，Bundle是一组Java包、类、描述文件等资源的集合，一般都以JAR格式进行封装。Java模块化不足问题的根源是那个全局的、扁平化的classpath；OSGi采取了一个完全不同的方法：每个模块都有自己独立的classpath。

如何实现这一点呢？OSGi采取了不同的类加载机制：
* OSGi为每个bundle提供一个类加载器，该加载器能够看到bundle Jar文件内部的类和资源；
* 为了让bundle能互相协作，可以基于依赖关系，从一个bundle类加载器委托到另一个bundle类加载器。

![20131006171453671](https://user-images.githubusercontent.com/6982311/44795399-2879e780-abdd-11e8-8c8e-3482e387366d.png)

Java和J2EE的类加载模型都是层次化的，只能委托给上一层类加载器；
而OSGi类加载模型则是网络图状的，可以在bundle间互相委托。——这样更合理，因为bundle间的依赖关系并不是层次化的。
 
例如bundleA、B都依赖于bundleC，当他们访问bundleC中的类时，就会委托给bundleC的类加载器，由它来查找类；如果它发现还要依赖bundleE中的类，就会再委托给bundleE的类加载器。

优点

* 找不到类时的错误提示更友好。假如bundleE不存在，则bundleC就不会被解析成功，会有错误消息提示为何未能解析；而不是报错ClassNotFoundException或NoClassDefFoundError。
* 效率更高。在标准Java类加载模型中，总是会在classpath那一长串列表中进行查找；而OSGi类加载器能立即知道去哪里找类。

解决模块化问题

1.OSGi可以帮助你先确保代码满足依赖关系，然后才允许执行代码；避免类路径错误

2.OSGi会对类路径上的依赖集进行一致性检查（如：版本）；

3.不必担心由于层次化的类加载模式隐含的限制；

4.OSGi可以把程序打包逻辑上独立的JAR文件，并且只部署指定的部分、动态部署；——OSGi生命周期层实现动态部署

5.OSGi可以声明JAR中的哪些代码可以被其他JAR访问，强化可见性； ——OSGi中，只有那些被显式导出的包才能被其他bundle使用。也就是说默认情况下，所有包都是bundle private的，不能被其他包看到。

6.OSGi为程序定义了一个插件式的扩展机制。

一个Bundle可以声明它所依赖的Java Package（通过Import-Package描述），也可以声明他允许导出发布的Java Package（通过Export-Package描述）。在OSGi里面，Bundle之间的依赖关系从传统的上层模块依赖底层模块转变为平级模块之间的依赖（至少外观上如此），而且类库的可见性能得到精确的控制，一个模块里只有被Export过的Package才可能由外界访问，其他的Package和Class将会隐藏起来。除了更精确的模块划分和可见性控制外，引入OSGi的另外一个重要理由是，基于OSGi的程序很可能可以实现模块级的热插拔功能，当程序升级更新或调试除错时，可以只停用、重新安装然后启动程序的其中一部分，这对企业级程序开发来说是一个非常有诱惑性的特性

　　OSGi之所以能有上述“诱人”的特点，要归功于它灵活的类加载器架构。OSGi的Bundle类加载器之间只有规则，没有固定的委派关系。例如，某个Bundle声明了一个它依赖的Package，如果有其他的Bundle声明发布了这个Package，那么所有对这个Package的类加载动作都会为派给发布他的Bundle类加载器去完成。不涉及某个具体的Package时，各个Bundle加载器是平级关系，只有具体使用某个Package和Class的时候，才会根据Package导入导出定义来构造Bundle间的委派和依赖

　　另外，一个Bundle类加载器为其他Bundle提供服务时，会根据Export-Package列表严格控制访问范围。如果一个类存在于Bundle的类库中但是没有被Export，那么这个Bundle的类加载器能找到这个类，但不会提供给其他Bundle使用，而且OSGi平台也不会把其他Bundle的类加载请求分配给这个Bundle来处理。
  
  OSGi类加载时的查找规则如下：

　　　　1）以java.*开头的类，委派给父类加载器加载

　　　　2）否则，委派列表名单内的类，委派给父类加载器加载

　　　　3）否则，Import列表中的类，委派给Export这个类的Bundle的类加载器加载

　　　　4）否则，查找当前Bundle的ClassPath，使用自己的类加载器加载

　　　　5）否则，查找是否在自己的Fragment Bundle中，如果是，则委派给Fragment bundle的类加载器加载

　　　　6）否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载

　　　　7）否则，查找失败

　　从之前的图可以看出，在OSGi里面，加载器的关系不再是双亲委派模型的树形架构，而是已经进一步发展成了一种更复杂的、运行时才能确定的网状结构。
  
### init和clinit

* 类型初始化方法<clinit>：
  
JVM通过Classload进行类型加载时，如果在加载时需要进行类型初始化操作时，则会调用类型的初始化方法。类型初始化方法主要是对static变量进行初始化操作，对static域和static代码块初始化的逻辑全部封装在<clinit>方法中。
  
Java类型初始化过程中对static变量的初始化操作依赖于static域和static代码块的前后关系，static域与static代码块声明的位置关系会导致java编译器生成<clinit>方法字节码。类型的初始化方法<clinit>只在该类型被加载时才执行，且只执行一次。

* 对象实例化方法<init>：
  
Java对象在被创建时，会进行实例化操作。该部分操作封装在<init>方法中，并且子类的<init>方法中会首先对父类<init>方法的调用。Java对象实例化过程中对实例域的初始化赋值操作全部在<init>方法中进行，<init>方法显式的调用父类的<init>方法，实例域的声明以及实例初始化语句块同样的位置关系会影响编译器生成的<init>方法的字节码顺序，<init>方法以构造方法作为结束。

####  init和clinit区别
①init和clinit方法执行时机不同

init是对象构造器方法，也就是说在程序执行 new 一个对象调用该对象类的 constructor 方法时才会执行init方法，而clinit是类构造器方法，也就是在jvm进行类加载—–验证—-解析—–初始化，中的初始化阶段jvm会调用clinit方法。

②init和clinit方法执行目的不同

init is the (or one of the) constructor(s) for the instance, and non-static field initialization. 
clinit are the static initialization blocks for the class, and static field initialization. 
上面这两句是Stack Overflow上的解析，很清楚init是instance实例构造器，对非静态变量解析初始化，而clinit是class类构造器对静态变量，静态代码块进行初始化。看看下面的这段程序就很清楚了。

```java
class X {

   static Log log = LogFactory.getLog(); // <clinit>

   private int x = 1;   // <init>

   X(){
      // <init>
   }

   static {
      // <clinit>
   }

}
```

#### clinit详解
在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，或者可以从另外一个角度来表达：初始化阶段是执行类构造器＜clinit＞（）方法的过程。

①＜clinit＞（）方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问如下代码

```java
public class Test{
static{
i=0；//给变量赋值可以正常编译通过
System.out.print（i）；//这句编译器会提示"非法向前引用"
}
static int i=1；
}
```

②虚拟机会保证在子类的＜clinit＞（）方法执行之前，父类的＜clinit＞（）方法已经执行完毕。 因此在虚拟机中第一个被执行的＜clinit＞（）方法的类肯定是java.lang.Object。由于父类的＜clinit＞（）方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作，如下代码中，字段B的值将会是2而不是1。

```java
static class Parent{
    public static int A=1；
    static{
    A=2；}
    static class Sub extends Parent{
    public static int B=A；
    }
    public static void main（String[]args）{
    System.out.println（Sub.B）；
    }
}
```

③接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成＜clinit＞（）方法。 但接口与类不同的是，执行接口的＜clinit＞（）方法不需要先执行父接口的＜clinit＞（）方法。 只有当父接口中定义的变量使用时，父接口才会初始化。 另外，接口的实现类在初始化时也一样不会执行接口的＜clinit＞（）方法。 

注意：接口中的属性都是static final类型的常量，因此在准备阶段就已经初始化话。
