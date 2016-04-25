####定义

在《设计模式》一书中，作者这样来叙述单例模 式的：确保一个类只有一个实例并提供一个对它的全局访问指针。单例模式适合于一个类只有一个实例的情况，比如窗口管理器，打印缓冲池和文件系统，它们都是原型的例子。典型的情况是，那些对象的类型被遍及一个软件系统的不同对象访问，因此需要一个全局的访问指针，这便是众所周知的单例模式的应用。当然这只有在你确信你不再需要任何多于一个的实例的情况下。

单例模式看起来是最简单的设计模式之一，但是使用不当的话，会存在很多的缺陷。

经典的单例模式如下：
```Java
public final class ClassicSingleton {
    public static ClassicSingleton instance=null;
    public static ClassicSingleton getInstance(){
        if(instance==null){
            instance=new ClassicSingleton();
        }
        return instance;
    }
}
```
将ClassicSingleton类声明为final的，确保类不能被随意实例化，另一种可行的方案是把构造函数声明为private，保证不能被客户 端实例化，但是不可以声明为protected的，因为如果有其他类对其进行了继承，则仍可对其进行实例化，创建新的对象。

下面引用David Geary的一段话：
>关于ClassicSingleton的感兴趣的地方是，如果单例由不同的类装载器装入，那便有可能存在多个单例类的实例。 假定不是远端存取，例如一些servlet容器对每个servlet使用完全不同的类装载器，这样的话如果有两个servlet访问一个单例类，它们就都 会有各自的实例。另外，如果ClasicSingleton实现了java.io.Serializable接口，那么这个类的实例就可能被序列化和复 原。不管怎样，如果你序列化一个单例类的对象，接下来复原多个那个对象，那你就会有多个单例类的实例。

####线程安全

问题来了：就是例1中的ClassicSingleton类不是线程安全的。如果两个线程，我们称它们为线程1 和线程2，在同一时间调用`ClassicSingleton.getInstance()`方法，如果线程1先进入if块，然后线程2进行控制，那么就会有 ClassicSingleton的两个的实例被创建。
分析一下：
```Java
if(instance==null){
    instance=new ClassicSingleton();
}
```
当线程1刚进入if语句时一直到当`new ClassicSingleton()`刚结束，但还没有赋值给instance之前，这段时间里，一旦发生了线程切换，则线程2就会进入if语句，这是 instance仍然是null，所以线程1和线程2都会创建一个ClassicSingleton实例，也就是整个系统会有两个 ClassicSingleton实例被创建。

- 解决方法1： 可以使用线程同步的方法，用关键字synchronized来使其同步，这样当线程1进入同步块时，线程2则无法进入。
```Java
public static synchronized  ClassicSingleton getInstance(){
    if(instance==null){
        instance=new ClassicSingleton();
    }
    return instance;
}
```
- 解决方法2：用一个更简单、快速而又是线程安全的单例模式实现
```Java
public final class ClassicSingleton {
    public final static ClassicSingleton instance= new ClassicSingleton();
    public static synchronized  ClassicSingleton getInstance(){
        return instance;
    }
}
```

这段代码是线程安全的是因为静态成员变量一定会在类被第一次访问时被创建。你得到了一个自动使用了懒汉式实例化的线程安全的实现。

总结一下:Singleton单例模式为一个面向对象的应用程序提供了对象唯一的访问点，不管它实现何种功能，此种模式都为设计和开发团队提供了共享的概念。

单例模式的要点有三个：一是某个类只能有一个实例；二是它必须自行创建这个实例；三是它必须自行向整个系统提供这个实例。

来源： [cnblogs](http://www.cnblogs.com/coser/archive/2011/04/16/2017724.html)
