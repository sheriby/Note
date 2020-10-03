# Function In GO

## Go中的函数

函数任何语言当中都非常核心的概念，而`Go`语言当中函数作为一等公民，具有更加强大的作用。（`Go`语言中的函数类似`Python`和`JavaScript`这种动态脚本语言，函数可以作为函数的参数，可以作为函数的返回值，可以在函数中定义函数等，但是`Go`语言是一门不折不扣的**静态语言**。）

## 函数定义

任何语言中函数的定义都是三个要素，**函数名，返回值类型，参数类型**，无非的定义的方式不一样以及使用上面细微的差别罢了。在`Go`中，类型写在变量名的后面，函数的返回值类型同样写在函数名后面。下面就是一个简单的函数的定义。

```go
func Add(a int, b int) int {
    return a + b
}
// 连续的参数为同一个类型的话可以省略一个，如
func Add(a, b int) int {
    return a + b
}
```

如果函数没有参数或者返回值的话，可以省略，如

```go
func sayHello() {
    fmt.Println("Hello")
}
```

### 有名的返回值

可以为函数的返回值起一个名字，参数名就相当于函数最外层的局部遍历，函数结束后return可以不带参数名直接返回。如

```go
func Add(a, b int) (sum int) { // 这里使用了有名的返回值需要加上括号
    sum = a + b // 这里不能使用 sum := a + b 这样相当于重新创建sum变量覆盖了上面的
    return // 不必带参数名字返回，这里的return不可以省略
}
```

### 多个返回值

函数的返回值可以有不止一个，虽然在其他语言中可以轻松的使用元组或者结构体实现，但是`Go`直接嵌入语法还是很不错的。

```go
func calculate(a, b int) (sum, mul int) { // 连续具名返回值的类型也可以省略
    sum = a + b
    mul = a * b
    return // 只有没有返回值的函数可以省略return
}
```

使用有名的返回值看起来就非常的清爽。如果不使用，需要这样写

```go
func calculate(a, b int) (int, int) { // 多个返回值需要加上括号
    return a + b, a * b // 返回的时候多个值逗号分隔即可
}
```

接收多个返回值的函数的返回值和`return`类似。

```go
s, m := calculate(1, 2)
```

如果有的返回值不需要，可以使用匿名变量，如。

```go
_, m := calculate(1, 2) // 只需要第二个返回值
```

### 函数的嵌套

在`Go`当中不支持`Python`，`JavaScript`那种在函数中直接定义函数的方式，如下面的代码是非法的。

```go
func a() {
    func b() {
        // ...
    }
}
```

我们可以使用**匿名函数**的方式。

```go
func Add(a, b int) (sum int) { // 具名函数
    anonymous := func(x, y int) int { // 匿名函数
        return x + y
    }
    
    return anonymous(a, b)
}
```

这有点类似下面的`Javascript`代码。

```js
function Add(a, b) { //具名函数
    const anonymous = function(x, y) { // 匿名函数
        return x + y
    }
    
    return anonymous(a, b)
}
```

匿名函数将在后面着重介绍。

### 函数定义的其他注意点

-   `Go`语言当中**不支持默认值参数**。
-   `Go`语言当中**不支持函数的重载**。

则两大特点和`Java`是完全一样，和`C++`是完全相反。

## 函数的参数传递

`Go`中的函数传递方式和`C`非常类似，只有值传递。也就是说我们传递的形参永远是实参的副本。

```go
func inc(a int) {
    a = a + 1
}

func main() {
    a := 1
    inc(a)
    fmt.Println(a)
}
```

有点`C`基础的都知道`a`的值还是1，没有被改变。

虽然值传递只是传递副本，但是这并不代表我们无法修改函数的实参。在`Go`中也可以使用指针，只不过`Go`中的指针是安全的指针，不支持加减等操作。

```go
func inc(a *int) { // ×在类型的前面，同样也是反的
    *a = *a + 1 // *a 获取指针指向的内存地址的值
}

func main() {
    a := 1
    inc(&a) // 调用的时候需要传入地址
    fmt.Println(a)
}
```

指针传递虽然看起来像是引用传递，但是传递的也还是指针的副本，然后通过指针副本中的地址修改真正的实参。

## 不定参数

`Go`函数支持不定数量的形参，其使用格式为`param ...type`，和`C++`中是类似的。不定参数的使用需要注意以下几点。

-   不定参数的类型必须都是一样的，就是上面指定的`type`类型。

-   不定参数必须要是函数形参的最后一个。

    ```go
    func foo(param1 int, param2 float32, params ...int) {
        // ...
    }
    ```

### 不定参数的使用

不定参数在函数中可以当作切片来使用，如

```go
func sum(nums ...int) (sum int) {
    for _, num := range nums {
       sum += num // 这里的sum是具名返回值sum,不是函数名，因为局部变量覆盖全局变量。
    }
    return
}

func main() {
    s := sum(1, 2, 3, 4)
    fmt.Println(s)
}
```

调用函数的时候同样可以使用切片，格式为`slice...`，但是无法使用数组的方式调用，如

```go
func main() {
    slice = []int{1, 2, 3, 4}
    s := sum(slice...) // 使用切片调用
    fmt.Println(s)
    
    array = [...]int{1, 2, 3, 4}
    s = sum(array...) // 不可以使用数组调用
}
```

## 匿名函数

匿名函数可以看作是函数字面量，所有直接使用函数类型变量的地方都可以使用匿名函数进行代替。匿名函数可以当做是实参，返回值，也可以直接被调用。

```go
// 匿名函数直接赋值给函数变量
var sum = func(a, b int) int {
    return a + b
}

// 匿名函数可以作为函数的返回值
func warp(op string) func(int, int) int { // 返回值类型为 func(int, int) int，是一个函数
    switch op {
    case "add":
        return func(a, b int) int {
            return a + b
        }
    case "mul":
        return func(a, b int) int {
            return a * b
        }
    }
}

// 函数的参数为函数
func doinput(f func(int, int) int, a, b int) int {
    return f(a, b)
}

func main() {
    // 匿名函数可以直接被调用
    func(str string) {
        fmt.Println(str)
    }("Hello World")
    
    // 匿名函数可以作为函数的实参
    a := doinput(func(a, b int) int {
        return a + b
    }, 1, 2)
    fmt.Println(a)
}
```

## 闭包

闭包是函数以及其相关引用环境组合而成的实体，一般通过匿名函数中引用外部局部变量或包全局变量构成。

```go
func getCounter(start int) func() {
	return func() {
		fmt.Println(start, &start)
		start++
	}
}

func main() {
	count := getCounter(5)
	count()
	count()
	count()
}
```

上面是一个非常典型的使用闭包实现的计数器的例子。`start`虽然是`getCounter`函数的局部变量，但是在该函数调用结束之后并没有销毁，因为有闭包引用了这个变量。编译器检测到这种情况，将`start`转移到堆上，以便之后我们可以一直引用。

我们还可以将多个函数引用同一个局部变量。

```go
type opFn func(int) int // 定义函数类型的别名，类似C中的typedef

func calc(start int) (opFn, opFn) {
	add := func(num int) int {
		start += num
		return start
	}

	sub := func(num int) int {
		start -= num
		return start
	}

	return add, sub
}

func main() {
	add, sub := calc(10)
	fmt.Println(add(5))
	fmt.Println(sub(3))
}
```

### 闭包的作用

闭包最初的作用是减少全局变量，在函数调用的时候隐式地传递，不过这种共享数据的方式并不清晰。闭包仅仅是锦上添花的东西，并不是必不可少。



