# Intermediate Python 01

## \*args & \*\*kwargs

在很多函数的声明中，我们都会看到上面这种形式的参数放在方法参数的最后面。例如。

```python
def myfunction(a, b, *args, **kwargs):
    pass
```

使用`*args`和`**kwargs`之后，我们可以将不定数量的参数传递个函数，当然这里的`args, kwargs`名字并不是必须如此，只不过这是一种比较普通的写法。

### 使用\*args

```python
def test_var_args(f_arg, *argv):
    print("first normal arg:", f_arg)
    for arg in argv:
        print("another arg through *argv:", arg)

test_var_args('yasoob', 'python', 'eggs', 'test')
```

这个函数最少需要接收一个参数，如果我们还传递的了多余的参数，会被`*args`接收，可以通过`for`迭代参数的值。说实话，这个就有点`javascript`那个味道了，但是更有点儿像`java`当中的不定长参数。

```java
public update(String sql, Object... args)
```

执行`sql`语句的时候，需要的参数是不定的，可以使用数组，但是使用这种不定长数组的形式可以让函数的调用更加的简单。

我们使用的最常用的不定长参数的函数就是`print`函数，其函数的声明如下。

```python
print(value, ..., sep=' ', end='\n', file=sys.stdout, flush=False)
```

后面的都是具名参数，可以不用管。前面的`value, ...`其实有点没看懂，`print`函数实际上是可以不传递任何参数的。如果让我们自己来实现`print`函数的话，会使用如下的函数声明。

```python
print(*args, sep=' ', end='\n', file=sys.stdout, flush=False)
```

### 使用\*kwargs

所谓的`kwargs`就是`key word args`，如果我们需要处理不定长的具名参数，可以使用这个。

```python
def greet_me(**kwargs):
    for key, value in kwargs.items():
        print("{0} == {1}".format(key, value))


>>> greet_me(name="yasoo")
name == yasoo
```

### 函数调用时使用

除了在函数的声明中可以使用他们，在函数的调用中也可以使用他们。

```python
def test_args_kwargs(arg1, arg2, arg3):
    print("arg1:", arg1)
    print("arg2:", arg2)
    print("arg3:", arg3)
```

我们手中有如下的元祖和字典。

```python
lst = [3, 5, 9]
dst = {'arg2': 3, "arg1": 5, "arg3": 9}
```

可以通过如下的形式调用上面这个函数。

```python
test_args_kwargs(*lst)
test_args_kwargs(**dst)
```

其实说到底，非常的简单。`*`不仅仅是乘号，其实还是一种解包的操作符，可以对任何的可迭代对象使用。

```python
a = [1, 2, 3]
print(*a) // 1 2 3
a = (1, 2, 3)
print(*a) // 1, 2, 3
a = {1, 2, 3}
print(*a) // 1 2 3
a = {'a': 1, 'b': 2}
print(*a) // a b
```

当我们对字典使用迭代，迭代的是他的键。而`**`是专门针对字典的解包操作，不可以对list使用。

### 不定参数的使用顺序

不定参数的使用顺序为

```python
func(fargs, *args, **kwargs)
```

解释起来也很简单，首先普通参数放在前面这是毋庸置疑的，主要是`args`和`kwargs`的位置问题。`×args`一般情况下用于接收列表类型的参数，`**kwargs`用于接收不定长的具名参数，然而python解析参数的时候会先解析`keyword argument`，然后再解析`position argument`，所以具名参数需要放在参数列表的最后面，因而`**kwargs`也必须要放在参数的最后面。

