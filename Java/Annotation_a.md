**java.lang.annotation**

Annotation（注解）就是Java提供了一种元程序中的元素关联任何信息和着任何元数据（metadata）的途径和方法。Annotion(注解)是一个接口，程序可以通过反射来获取指定程序元素的Annotion对象，然后通过Annotion对象来获取注解里面的元数据。

**基本的规则**:Annotation不能影响程序代码的执行，无论增加、删除 Annotation，代码都始终如一的执行。

#### 系统内置标准注解：

注解的语法比较简单，除了@符号的使用外，他基本与Java固有的语法一致，JavaSE中内置三个标准注解，定义在java.lang中：

- @Override：用于修饰此方法覆盖了父类的方法;
- @Deprecated：用于修饰已经过时的方法;
- @SuppressWarnnings:用于通知java编译器禁止特定的编译警告。

SuppressWarnings注解的常见参数值的简单说明：
1. deprecation：使用了不赞成使用的类或方法时的警告；
2. unchecked：执行了未检查的转换时的警告，例如当使用集合时没有用泛型 (Generics) 来指定集合保存的类型;
3. fallthrough：当 Switch 程序块直接通往下一种情况而没有 Break 时的警告;
4. path：在类路径、源文件路径等中有不存在的路径时的警告;
5. serial：当在可序列化的类上缺少 serialVersionUID 定义时的警告;
6. finally：任何 finally 子句不能正常完成时的警告;
7. all：关于以上所有情况的警告。

#### 元注解

元注解的作用就是负责注解其他注解。Java定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。
a. `@Target`

@Target说明了Annotation所修饰的对象范围：Annotation可被用于 packages、types（类、接口、枚举、Annotation类型）、类型成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环变量、catch参数）。在Annotation类型的声明中使用了target可更加明晰其修饰的目标。

作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）

取值(ElementType)有：
1. CONSTRUCTOR:用于描述构造器
2. FIELD:用于描述域
3. LOCAL_VARIABLE:用于描述局部变量
4. METHOD:用于描述方法
5. PACKAGE:用于描述包
6. PARAMETER:用于描述参数
7. TYPE:用于描述类、接口(包括注解类型) 或enum声明
8. ANNOTATION_TYPE:注解类型说明

b. `@Retention`

@Retention定义了该Annotation被保留的时间长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的Annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取（请注意并不影响class的执行，因为Annotation与class在使用上是被分离的）。使用这个meta-Annotation可以对 Annotation的“生命周期”限制。有唯一的value作为成员，它的取值来自java.lang.annotation.RetentionPolicy的枚举类型值。

作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）

取值（RetentionPoicy）有：
1. SOURCE:在源文件中有效（即源文件保留）
2. CLASS:在class文件中有效（即class保留）
3. RUNTIME:在运行时有效（即运行时保留）

属性值是RUTIME的注解处理器可以通过反射，获取到该注解的属性值，从而去做一些运行时的逻辑处理。

c. `@Documented`

@Documented用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化。Documented是一个标记注解，没有成员。

d. `@Inherited`
@Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

*注意*：@Inherited annotation类型是被标注过的class的子类所继承。类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation。

当@Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层。
![Mind Map](../_attach/Java/annotation.jpg)
