如果需要导入的module的名字是m1，则解释器必须找到m1.py，它首先在当前目录查找，然后是在环境变量PYTHONPATH中查找。然后是python的安装设置相关的默认路径。
> PYTHONPATH可以视为系统的PATH变量一类的东西，其中包含若干个目录。如果PYTHONPATH没有设定，或者找不到m1.py，则继续搜索 与python的安装设置相关的默认路径，在Unix下，通常是/usr/local/lib/python。

正因为存在这样的顺序，如果当前路径或PYTHONPATH中存在与标准module同样的module，则会覆盖标准module。也就是说，如果当前目录下存在xml.py，那么执 行import xml时，导入的是当前目录下的module，而不是系统标准的xml。

Python中的package定义很简单，其层次结构与程序所在目录的层次结构相同，这一点与Java类似，唯一不同的地方在于，python中的package必须包含一个`__init__.py`的文件。

例如，我们可以这样组织一个package:

```
package1/
    __init__.py
    subPack1/
        __init__.py
        module_11.py
        module_12.py
        module_13.py
    subPack2/
        __init__.py
        module_21.py
        module_22.py
    ……
```
`__init__.py`可以为空，只要它存在，就表明此目录应被作为一个package处理。

在顶层目录（也就是package1，或将package1放在解释器能够搜索到的地方）运行python

```Python
>>>from package1.subPack1.module_11 import funcA
>>>funcA()
```
有时在import语句中会出现通配符*，导入某个module中的所有元素，答案就在__init__.py中。在subPack1的__init__.py文件中写

```Python
__all__ = ['module_13', 'module_12']
```
然后进入python

```Python
>>>from package1.subPack1 import *
>>>module_11.funcA()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named module_11
```
也就是说，以*导入时，package内的module是受`__init__.py`限制的。

最后来看看，如何在package内部互相调用。如果希望调用同一个package中的module，则直接import即可。也就是说在`module_12.py`中，可以直接使用`import module_11`

如果不在同一个package中，例如我们希望在module_21.py中调用module_11.py中的FuncA，则应该这样：

```Python
from module_11包名.module_11 import funcA
```

自定义模块加入编译路径方法：在Python安装目录的Lib\site-packages下添加*.pth文件，将模块路径添加到文件中。