# Intermediate Python 02

## 装饰器

装饰器是`Python`当中对于新手来说比较难以理解的知识点，但是对于现如今学过装饰器设计模式甚至`Spring`当中的`AOP`面向切面编程的我们来说，非常容易理解。

### Python当中的函数

`Python`当中的函数和`Javascript`中的函数非常的类似，函数就是一种对象。函数可以作为函数的参数，函数的返回值，我们甚至还可以在函数当中定义函数。

这和`Java`这种纯面向对象的语言是非常不一样的，函数必须要依赖于对象。就算是`Java8`中引入了`Lambda`表达式，看起来我们可以将函数作为参数传递，然而底层这个函数还是依赖于一个匿名对象的。

### 装饰函数与装饰器

在`Spring`的开发中，`AOP`的一个比较重要的应用就是日志。通过面向切面编程，我们可以动态的装饰目标方法。如方法执行前和结束后要输入，方法抛出异常之后要输入，甚至修改方法的执行和返回结果等等等。这一切的一切都是通过动态代理的方式来实现的。

在动态语言`Python`当中，这样的业务实现起来就非常的简单了。

```python
def log(func):
    def wrap_function():
        func_name = func.__name__
        print('{} function begin'.format(func_name))
        res = func()
        print('{} function end'.format(func_name))
        print('{} function return {}'.format(func_name, res))
        
        return res
    return wrap_function

def test_function():
    return 1 + 1

test_function = log(test_function)
test_function()
```

上面其实就是一个最简单的装饰器。通过`log`函数传入一个函数，返回被装饰之后的函数。

`Python`给我们提供了一个语法糖用以简化装饰器。

```python
def log(func):
    def wrap_function():
        func_name = func.__name__
        print('{} function begin'.format(func_name))
        res = func()
        print('{} function end'.format(func_name))
        print('{} function return {}'.format(func_name, res))
        
        return res
    return wrap_function

@log
def test_function():
    return 1 + 1

test_function()
```

在函数的声明之前添加`@装饰函数`，这和上面手动的去调用装饰函数去装饰其实是等价的。

不过上面的装饰器还是有点儿问题的，如

```python
@log
def test_function():
    return 1 + 1

print(test_function.__name__)
```

这里输出的是`wrap_function`而不是`test_function`。这很容易理解，因为这里的`test_function`已经完全被`wrap_function`重写了，幸运的是，`Python`提供了函数`functools.wraps`来解决这个问题，使用方式如下。

```python
from functools import wraps

def log(func):
    @wraps(func)
    def wrap_function():
        func_name = func.__name__
        print('{} function begin'.format(func_name))
        res = func()
        print('{} function end'.format(func_name))
        print('{} function return {}'.format(func_name, res))
        
        return res
    return wrap_function

@log
def test_function():
    return 1 + 1

test_function()
print(test_function.__name__)
```

还有一个问题便是函数的参数问题。上面我们用来举例的被修饰函数并不需要传递任何参数，如果需要传递参数又该如何呢。

```python
@log
def test_function(a, b):
    return a * b
```

装饰函数本身是不知道需要装饰的函数需要什么参数的，这就需要使用到之前我们说过的不定参数`*args`和`**kwargs`

```python
from functools import wraps

def log(func):
    @wraps(func)
    def wrap_function(*args, **kwargs):
        func_name = func.__name__
        print('{} function begin'.format(func_name))
        res = func(*args, **kwargs)
        print('{} function end'.format(func_name))
        print('{} function return {}'.format(func_name, res))
        
        return res
    return wrap_function

@log
def test_function(a, b):
    return a * b

test_function(3, 7)
```

到此为止，上面的装饰器就比较的完善了。

### 带参数的装饰器

也许你已经发现了，其实`functool.wraps`也是一个装饰器，使用方式为`@wraps(func)`，比较特别的这个装饰器是有参数的。

考虑这样的业务场景，对于不同的函数，我们需要将日志写入到不同的文件当中，此时就需要使用带参数的装饰器，如下。

```python
from functools import wraps

def log(logfile = 'out.log'):
    def deco_function(func):
        @wraps(func)
        def wrap_function(*args, **kwargs):
            func_name = func.__name__
            print('{} function begin'.format(func_name))
            res = func(*args, **kwargs)
            print('{} function end'.format(func_name))
            print('{} function return {}'.format(func_name, res))
            print('write to {}'.format(logfile))
            
            return res
        return wrap_function
    return deco_function

@log()
def test_function(a, b):
    return a * b

@log(logfile = 'out2.log')
def test_function2(a, b):
    return a + b

test_function(3, 7)
test_function2(3, 7)
```

这看起来非常的乱，`log`函数返回的是一个装饰器。我们可以先不看语法糖来分析上面的代码。

```python
@log(logfile = "out2.log")
def test_function2(a, b):
    return a + b

test_function2(3, 7)
```

就相当于

```python
def test_function2(a, b):
    return a + b

# 对于没参数的装饰器是, test_function2 = log(test_function2)
test_function2 = log('out2.log')(test_function2)
test_function2(3, 7)
```

这就很容易理解了。

### 装饰器类

不仅是函数，类也可以用来构建装饰器。使用类来构建装饰器不仅比使用嵌套函数的方式更加的简洁，而且使用方法还和之前一样。最重要的是，我们可以通过类的继承更好的装饰器进行拓展。

使用装饰器类改写上面的装饰器，如下。

```python
from functools import wraps

class log(object):
    def __init__(self, logfile = 'out.log'):
        self.logfile = logfile
        
    def __call__(self, func):
        @wraps(func)
        def wrap_function(*args, **kwargs):
            func_name = func.__name__
            print('{} function begin'.format(func_name))
            res = func(*args, **kwargs)
            print('{} function end'.format(func_name))
            print('{} function return {}'.format(func_name, res))
            print('write to {}'.format(self.logfile))
            
            self.after()
            return res
        return wrap_function
    
    def after(self):
        pass

@log()
def test_function(a, b):
    return a * b

@log(logfile = 'out2.log')
def test_function2(a, b):
    return a + b

test_function(3, 7)
test_function2(3, 7)
```

我们还在`__call__`方法中留下一个`self.after()`方法的位置，对于继承该类的之类，可以重写`after`方法以决定要在日志打印完毕之后需要做什么。具体的拓展，这里不多说了。

