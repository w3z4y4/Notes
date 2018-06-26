# （Java设计模式）装饰者模式
父类和子类继承自同一接口，父对象通过构造函数传递给子类，子类动态扩展父对象的方法。
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
