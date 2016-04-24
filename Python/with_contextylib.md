- 平常Coding过程中，经常使用到的with场景是（打开文件进行文件处理，然后隐式地执行了文件句柄的关闭）:
```Python
with file('test.py','r') as f :
    print f.readline()
```

- with的作用，类似`try...finally...`，提供一种上下文机制，要应用with语句的类，其内部必须提供两个内置函数 `__enter__`以及`__exit__`。前者在主体代码执行前执行，后则在主体代码执行后执行。as后面的变量，是在`__enter__`函数中返回的。 通过下面这个代码片段以及注释说明，可以清晰明白`__enter__`与`__exit__`的用法：
```Python
#!encoding:utf-8
class echo :
    def output(self) :
        print 'hello world'
    def __enter__(self):
        print 'enter'
        return self #返回自身实例，当然也可以返回任何希望返回的东西
    def __exit__(self, exception_type, exception_value, exception_traceback):
        #若发生异常，会在这里捕捉到，可以进行异常处理
        print 'exit'
        #如果可以处理该异常,则通过返回True告知该异常不必传播，否则返回False
        if exception_type == ValueError :
            return True
        else:
            return False

with echo() as e:
    e.output()
    print 'do something inside'
print '-----------'
with echo() as e:
    raise ValueError('value error')
print '-----------'
with echo() as e:
    raise Exception('can not detect')
```

- contextlib是为了加强with语句，提供上下文机制的模块，它是通过Generator实现的。通过定义类以及写__enter__ 和__exit__来进行上下文管理虽然不难，但是很繁琐。contextlib中的contextmanager作为装饰器来提供一种针对函数级别的上下文管理机制。常用框架如下：
```Python
from contextlib import contextmanager

@contextmanager
def make_context() :
    print 'enter'
    try :
        yield {}
    except RuntimeError, err :
        print 'error' , err
    finally :
        print 'exit'

with make_context() as value :
    print value
```

- contextlib还有连个重要的东西，一个是nested，一个是closing，前者用于创建嵌套的上下文，后则用于帮你执行定义好的close函数。但是nested已经过时了，因为with已经可以通过多个上下文的直接嵌套了。下面是一个例子：
```Python
from contextlib import contextmanager
from contextlib import nested
from contextlib import closing
@contextmanager
def make_context(name) :
    print 'enter', name
    yield name
    print 'exit', name

with nested(make_context('A'), make_context('B')) as (a, b) :
    print a
    print b

with make_context('A') as a, make_context('B') as b :
    print a
    print b

class Door(object) :
    def open(self) :
        print 'Door is opened'
    def close(self) :
        print 'Door is closed'

with closing(Door()) as door :
    door.open()
```

来源： <http://www.cnblogs.com/coser/archive/2011/12/11/2284245.html>
