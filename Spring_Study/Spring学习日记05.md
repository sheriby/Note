# Spring学习日记05

## 初识AOP

之前我们已经简单的学习了什么是`IOC`，已经在`Spring`中我们该如何使用`IOC`。`IOC`是为了解决企业级开发中多个类之间复杂的依赖关系，将其类与类之前的耦合，其主要的实现方式是**反射和工厂模式**。

而现在要说的`AOP`，中文翻译成**面向切面编程**，是面向对象编程的一种延生。其主要的目的是在不改变源代码的情况下，如何对已有的类进行增强，其主要的实现方式是**反射和动态代理**。可见，反射和设计模式真的是非常的重要。

## 代理

先说说什么是代理，代理这个词大概意思就是替代xxx做某件事。比如说我们常说的`VPN`就是一种代理。我们本身并不能访问国外的网络，但是我们可以访问代理`A`，代理`A`可以通过某种方式访问国外的网络。此时我们向代理`A`发送`Http`请求，代理`A`将同样的`Http`请求发往国外网站，然后国外的网站返回数据给代理`A`，代理`A`再将数据返回给我们。在我们的角度看来，似乎没有代理的存在，我们是直接和国外网站进行通信的。

也就是说，我们原本没有访问国外网站的能力，但是通过了代理，我们获得了这种能力。

### 静态代理

要说明白什么是动态代理，我们还需要先说说什么是静态代理。

下面我们有一个计算机的接口已经它的一个实现类。

```java
package com.sher.cal;

public interface Calculator {
    int add(int a, int b);
    int minus(int a, int b);
    int multiply(int a, int b);
}
```

```java
package com.sher.cal;

public class SherCalculator implements Calculator{
    @Override
    public int add(int a, int b) {
        return a + b;
    }

    @Override
    public int minus(int a, int b) {
        return a - b;
    }

    @Override
    public int multiply(int a, int b) {
        return a * b;
    }
}
```

现在我们可以使用计算机类`SherCalculator`进行计算。但是现在我们要对计算器的功能进行拓展，使其在计算之前输出计算的表达式，计算之后输入结果。

如果修改源代码的话，很简单直接修改三个方法就行了。但这种做法是企业级开发的大忌，一是改动源代码可能会导致原来的程序出错，二是改动源代码之前的程序都需要重新进行编译。三是改动的内容可能之后我们还想要可以简单的去掉。

此时我们就可以使用静态代理模式在不改动源代码的情况下，对原计算器进行功能商的拓展。

我们需要创建一个代理类，其实现计算器的接口`Calculator`，然后通过某种方式原对象放入到代理对象中进行功能上的拓展。（如通过构造函数的方式或者通过`set`方法传入参数）

```java
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
public class CalculatorProxy implements Calculator {
    private Calculator calculator;

    @Override
    public int add(int a, int b) {
        System.out.printf("%d + %d\n", a, b);
        int res = calculator.add(a, b);
        System.out.println("result: " + res);
        return res;
    }

    @Override
    public int minus(int a, int b) {
        System.out.printf("%d - %d\n", a, b);
        int res = calculator.minus(a, b);
        System.out.println("result: " + res);
        return res;
    }

    @Override
    public int multiply(int a, int b) {
        System.out.printf("%d * %d\n", a, b);
        int res = calculator.multiply(a, b);
        System.out.println("result: " + res);
        return res;
    }
}
```

这里为了使代码更加的简洁使用了`lombok`注解。

然后可以通过如下的方式使用代理类对原有的类进行扩充。

```java
@Test
public void test() {
    Calculator calculator = new SherCalculator();
    calculator = new CalculatorProxy(calculator); // 使用代理
    calculator.add(1, 3);
}
```

可以看到这就和我们使用`VPN`一样，实际的功能是代理帮我们做的，如果我们去掉代理，代码还是照样可以运行，但是加上代理类我们就可以得到拓展的效果。

如果我们需要拓展的类没有接口该咋办呢？这时候只能将代理类继承该类，然后在子类中调用父类的方法了，这里不做深入的介绍。

### 动态代理

那么什么又是动态代理呢？所谓的动态代理就是在程序运行的过程中，使用反射的机制动态的创建代理对象。`jdk`当中为有接口的类提供了专门的动态代理的方法，`Proxy`。

首先我们创建一个类用于扩展方法，其需要实现`InvocationHandler`接口。

```java
@AllArgsConstructor
public class CalProxy implements InvocationHandler {
    private final Object obj;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.print(args[0] + " ");
        switch (method.getName()) {
            case "add":
                System.out.print("+");
                break;
            case "minux":
                System.out.print("-");
                break;
            case "multiply":
                System.out.print("*");
                break;
        }
        System.out.println(" " + args[1]);
        Object res = method.invoke(obj, args);
        System.out.println("result: " + res);
        return res;
    }
}
```

可以看到的是，这个`InvocationHandler`与我们需要拓展的类是没有依赖关系的，所以说其可拓展性，可复用性极高。

然后我们使用`JDK`内置的方法创建动态代理。

```java
@Test
public void test2() {
    Calculator calculator = new SherCalculator();
    calculator = (Calculator) Proxy.newProxyInstance(CalTest.class.getClassLoader(),
             new Class[]{Calculator.class}, new CalProxy(calculator));
    calculator.add(1, 3);
}
```

使用的是`Proxy.newProxyInstance`方法，其第一个参数是类加载器，第二个参数是实现的接口的类型的数组，第三个参数是我们拓展的方法，方法的返回值就是`JDK`为我们创建的动态代理对象。

将静态代理和动态代理放在一起进行比较，我们可以很明显的发现动态代理的优势，我们只使用了一个方法就可以处理需要代理中的所有的方法。而静态代理只可以一个一个实现，代码比较的赘余。

## AOP中的一些术语

-   连接点：类中可以被增强的方法被称为连接点。

-   切入点：类中实际被增强的方法

-   通知（增强）：实际增强的逻辑部分被称为通知

    通知有以下的几种类型

    -   前置通知
    -   后置通知
    -   环绕通知
    -   异常通知
    -   返回通知

-   切面：将通知应用到切入点的过程被称为切面。

## AOP的Hello World

下面我们就使用`Spring`中的`AOP`实现上面的动态代理的效果。在`Spring AOP`中可以完全使用`xml`语法来配置，但是更常用的是使用注解的方式。使用注解的话，`Spring`框架一般是基于`AspectJ`实现`AOP`操作。`AspectJ`是一个`AOP`框架，是`AOP`的老祖宗。

### 导入`AspectJ`依赖

因为需要用到`AspectJ`的注解，切入点表达式，通知等，我们可以通过`Maven`引入`AspectJ`。

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.4</version>
</dependency>
```

注：如果完全使用`xml`不使用注解的话，是不需要`AspectJ`的。

### 切入点表达式

切入点表达式的作用是指定我们需要对哪个类中的哪个方法进行增强。

>   语法格式：`execution([权限修饰符][返回类型][类全路径].[方法名](方法参数))`

其中使用`*`可以表示匹配任意，使用`..`表示匹配任意参数列表。

比如说我们要对`SherCalculation`中的add方法进行增强，可以写

```java
execution(* com.sher.cal.SherCalculation.add(..))
```

对其中的所有的方法进行增强，可以写

```java
execution(* com.sher.cal.SherCalculation.*(..))
```

### 编写增强类

比如这里我们给`add`方法添加一个前置通知（在方法执行之前执行），需要这么写

```java
@Component
@Aspect
public class SpringProxy {

    @Before("execution(* com.sher.cal.SherCalculator.add(..))")
    public void before() {
        System.out.println("SpringProxy.before");
    }
}
```

我们需要将`SherCalculator`和`SpringProxy`都放入到`IOC`容器当中去，所以都要加上`@Component`注解。下面的`@Aspect`注解表示需要根据此类创建一个代理类。

`@Before`注解表示前置通知，其中需要写一个切入点表达式，这里匹配的是`add`方法，则在`add`方法执行之前该方法`before`会被执行。

### 配置类

```java
@Configuration
@ComponentScan("com.sher.cal")
@EnableAspectJAutoProxy
public class CalConfig {
}
```

在配置类中，除了之前说的`@Configuration`和`@ComponentScan`注解之外，还添加了一个`@EnableAspectJAutoProxy`，其用于扫描`@Aspect`注解并且创建代理对象。

### 测试运行

```java
@Test
public void test4() {
    ApplicationContext context = 
        new AnnotationConfigApplicationContext(CalConfig.class);
    Calculator calculator = context.getBean(SherCalculator.class);
    calculator.add(3, 5);
}
```

这段代码运行之后，你会发现报错了，原因是`IOC`容器中没有`SherCalculator`类型的`Bean`，但是我们不是加入进去了吗？那是因为该类被代理了之后被移除出了`IOC`容器中，`IOC`容器中只有代理对象，这个代理对象实现了`Calculator`接口，并且`id`变成了增强之前的对象的`id`。

我们可以通过如下的方式获取代理对象。

```java
@Test
public void test4() {
    ApplicationContext context = 
        new AnnotationConfigApplicationContext(CalConfig.class);
    Calculator calculator = context.getBean("sherCalculator", Calculator.class);
    calculator.add(3, 5);
    System.out.println(calculator.getClass());
    // class com.sun.proxy.$Proxy20
}
```

不过如果`SherCalculator`没有任何接口呢？此时可以使用第一种方式获取到代理类，因为此时代理类就是该类的之类，通过`SherCalculator.class`是可以获取到的。

### 通知执行的顺序

上面已经介绍了有五种通知，现在我们写一个小的`demo`看看通知的执行顺序是如何的。

```java
@Component
@Aspect
public class SpringProxy {

    @Before("execution(* com.sher.cal.SherCalculator.add(..))")
    public void before() {
        System.out.println("Before...");
    }

    @After("execution(* com.sher.cal.SherCalculator.add(..))")
    public void after() {
        System.out.println("After");
    }

    @AfterThrowing("execution(* com.sher.cal.SherCalculator.add(..))")
    public void afterThrowing() {
        System.out.println("AfterThrowing...");
    }

    @Around("execution(* com.sher.cal.SherCalculator.add(..))")
    public Object arroud(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Before Around");
        Object res = pjp.proceed();
        System.out.println("After Around");
        return res;
    }

    @AfterReturning("execution(* com.sher.cal.SherCalculator.add(..))")
    public void afterRuturning() {
        System.out.println("AfterRuturning");
    }
}
```

输出的结果为：

```text
Before Around   环绕前置
Before...				 前置通知
Add...						 方法执行
After Around		 环绕后置
After							后置通知
AfterRuturning   最终通知
```

如果该方法会引发异常的话，程序输出结果如下。

```text
Before Around   环绕前置
Before...				前置通知
Add...						方法执行
After						 后置通知
AfterThrowing...异常通知
```

环绕后置通知和返回通知都没有执行。

了解了通知的执行之后，我们再来看看其中一个最特殊的通知，环绕通知。这个通知的方法既有返回值也有参数。其参数为`ProceedingJoinPoint`类型，调用其`proceed`方法，被增强的方法就会得到执行，可以得到方法的返回值。这个环绕通知的返回值就是最终调用方法得到的返回值。

### 提取切入点表达式

上面的代码可以看出一个很明显的问题，同样的切入点表达式写了五遍。可以通过`@Pointcut`注解对切入点进行抽取。

```java
@Pointcut("execution(* com.sher.cal.SherCalculator.add(..))")
public void pointDemo() {
}
```

写一个空的方法，然后上面加上`@Pointcut`和我们需要抽取的切入点表达式，然后我们就可以使用该方法来引用该切入点表达式了。如

```java
@Before("pointDemo()")
public void before() {
    System.out.println("Before...");
}
```

### 多个代理设置优先级

如果我们对一个类设置多个代理的话，可以通过`@Order`注解设置代理的优先级。格式为`@Order(int)`，其中数字越小优先级越高。

多层代理也比较容易理解，将第一次代理之后的代理对象作为第二次代理的原对象。

### 完成之前的简单的计算器增强

这个增强非常的简单，可以只是用环绕通知，而且必须要使用环绕通知。因为只有在环绕通知中我们才可以获取到`ProceedingJoinPoint`对象，然后获取的方法的名字以及方法的参数。具体的实现如下所示

```java
@Component
@Aspect
public class CalAopProxy {

    @Around("execution(* com.sher.cal.SherCalculator.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        Object[] args = pjp.getArgs(); // 获取方法的参数
        System.out.print(args[0] + " ");
        switch (pjp.getSignature().getName()) { // 获取方法的名字
            case "add":
                System.out.print("+");
                break;
            case "minus":
                System.out.print("-");
                break;
            case "multiply":
                System.out.print("*");
                break;
        }
        System.out.println(" " + args[1]);
        Object res = pjp.proceed();
        System.out.println("result: " + res);
        return res;
    }
}
```





