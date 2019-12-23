spring boot的启动过程

一、从SpringApplication.run说起

静态方法，一般是传入主类的类名和应用的启动参数，然后创建SpringApplication的实例，再调用这个实例的run方法，入参是应用的启动参数。

接着Spring要做一些初始化工作：

1.加载Listener

默认加载EventPublishingRunListener，SpringBoot使用当前线程的类加载器去实例化这个类（ClassUtils.forName(name,classLoader)），自定义的Listerner也必须是SpringApplicationRunListener的子类.

resolvetype
