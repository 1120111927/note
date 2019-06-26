# 装饰器（Decorator） #

装饰器本质上是一个python函数或类，可以让其他函数或类在不需要做任何代码修改的前提下增加额外功能，装饰器的返回值也是一个函数或类对象。经常用于有切面需求的场景，比如插入日志、性能测试、事务处理、缓存、权限校验等。使用装饰器可以抽离出大量与函数功能本身无关的雷同代码到装饰器中并重用。简言之，装饰器的作用就是为已经存在的对象添加额外的功能。

简单装饰器实现：

```python
def use_logging(func):

    def wrapper():
        logging.warn("%s is running" % func.__name__)
        return func()
    # 返回函数对象
    return wrapper

def foo():
    print('I am foo')

foo = use_logging(foo)
foo()
```

使用@语法糖：

```python
def use_logging(func):

    def wrapper():
        logging.warn("%s is running" % func.__name__)
        return func()
    return wrapper

@use_logging
def foo():
    print("I am foo")

foo()
```

带参数的装饰器：

```python
def use_logging(level):
    def decorator(func):
        def wrapper(*args, **kwargs):
            if level == "warn":
                logging.warn("%s is running" % func.__name__)
            elif level == "info":
                logging.info("%s is running" % func.__name__)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@use_logging(level="warn")
def foo(name='foo'):
    print("i am %s" % name)

foo()
```

可以在定义`wrapper`函数的时候指定参数，使用`*arg`来表示普通参数，使用`**kwargs`标识关键字参数。

类装饰器

```python
class Foo(object):
    def __init__(self, func):
        self._func = func
    
    def __call__(self):
        print('class decorator runing')
        self._func()
        print('class decorator ending')

@Foo
def bar():
    print('bar')

bar()
```

使用类包装器主要依靠类的`__call__`方法，当使用`@`形式将装饰器附加到函数上时，就会调用此方法。

使用functools.wraps保留原函数的元信息：

```python
from functools import wraps

def logged(func):
    @wraps(func)
    def with_logging(*args, **kwargs):
        print func.__name__
        print func.__doc__
        return func(*args, **kwargs)
    return with_logging

@logged
def f(x):
    return x + x * x

f(5)
```

wraps本身也是一个装饰器，它能把原函数的元信息复制到装饰器里面的func函数中，使得装饰器里面的func函数和原函数foo有相同的元信息。

装饰器的顺序：一个函数可以同时定义多个装饰器，执行顺序是从里到外，最先调用最里层的装饰器，最后调用最外层的装饰器。