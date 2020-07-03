# Spring学习日记02

## `IOC`中的`Bean`管理

### 工厂`Bean`

在`Spring`中，有两种`Bean`，一种是普通`Bean`，另一种是工厂`Bean`。所谓的普通`Bean`就是我们之前一直在接触的那种，我们在`bean`标签中指定`class`，IOC给我产生相应的对象，我们从`IOC`容器中获取到指定的对象。如：

```xml
<bean id="user" class="com.sher.User">
	...
</bean>
```

我们指定的`class`为`com.sher.User`，我们在`getBean`的时候得到的就是`User`类型的对象。

还有一种`Bean`叫做工厂`Bean`，我们在`IOC`容器中存放生产某一个类的工厂，然后通过该工厂获取相应的对象。工厂模式是一种非常常见的设计模式，一般用于创建复杂的对象或者解耦合等。这里使用创建书的工厂进行举例，现在我们不直接在`IOC`容器中放置`Book`对象，而是放入生产`Book`的工厂`BookFactory`。

```xml
<bean id="bookFactory" class="com.sher.factorybean.BookFactory">
	...
</bean>
```

但是如果只是这样不进行其他相关的设置的话，使用`getBean`还是会反正这个工厂对象。想要成为工厂`Bean`需要实现`FactoryBean`这个接口。

```java
package com.sher.factorybean;

import com.sher.Book;
import org.springframework.beans.factory.FactoryBean;

public class BookFactory implements FactoryBean<Book> {
    @Override
    public Book getObject() throws Exception {
        Book book = new Book();
        book.setName("learn python3");
        book.setPrice(49.98);
        return book;
    }

    @Override
    public Class<?> getObjectType() {
        return Book.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

-   `getObject`方法用于返回创建的对象。
-   `getObjectType`方法返回创建的对象的类型。
-   `isSingleton`用于设置该工厂创建的对象是否是单例。

```java
ApplicationContext context = new ClassPathXmlApplicationContext("bean2.xml");
Book book = context.getBean("bookFactory", Book.class);
System.out.println(book);
```

虽然我们在`xml`中配置的`Bean`是`BookFactory`类型的，但是我们得到的`Bean`的类型却是`Book`，这就是工厂`Bean`的作用，简化了工厂模式在`Spring`中的使用。如果没有工厂`Bean`，我们通过`getBean`得到工厂对象，然后还需要手动的再去创建目标对象，这样就麻烦多了。

### `Bean`的作用域

所谓的`Bean`的作用域值得就是在`Spring`当中`Bean`是单实例的还是多实例的。在`Spring`当中，**所有的`Bean`都是单实例的**。也就是说我们通过`getBean`多次获取同一个`id`得到的对象都是同一个对象。

那么`Spring`中为什么会把`Bean`设计成为默认单例呢？主要就是一句话，为了高性能，如果一个对象是单例的，那么在系统启动的时候，我们就可以将这些对象全部都创建然后放置到缓存当中，这样避免了系统在运行时间创建对象，而且因为只有一个实例，`jvm`的垃圾回收也会相应的减少。但是单例也会有一定的缺点，比如线程可能存在一些不安全的因素，或者有些时候我们确实需要多个实例等。

可以通过`bean`标签中的`scope`属性设置该`bean`是单实例还是多实例。

-   `scope: singleton`，默认，单实例。
-   `scope: prototype`，多实例。

### `Bean`的生命周期

#### 对象的创建

上面也提到了对象的创建，所有的单例`Bean`会在加载`xml`文件之后立刻进行初始化，而所有的原型对象（也就是多实例）会在使用到的时候创建。如：

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("bean.xml");
// 如果是单例——xml文件加载完毕，创建所有的单例对象
//....
//....其他代码
//....
// 如果是原型——下面要用到user对象，创建该对象。
User user = (User) applicationContext.getBean("user");
```

对象的创建一般是调用**无参沟站器**（除非我们指定了使用有参构造器进行属性的注入），然后调用属性的`set`方法进行属性的注入以及对其他的`bean`的引用。

#### 调用对象的初始化方法

我们可以在类中编写一个初始化方法，然后在`<bean>`标签中指定该方法为初始化方法，这样在对象创建完成之后，该方法就会被调用。如我们在`Book`类中编写了`initMethod`用于初始化工作。

```xml
<bean id="book" class="com.sher.Book" init-method="initMethod"></bean>
```

在`init-method`中指定初始化方法的名字，如果不指定的话就不调用啦。

#### 对象被使用

使用`getBean`获取到`IOC`容器中的对象。

#### 调用对象的销毁方法

当`IOC`容器关闭的时候，调用对象的销魂方法，和调用初始化方法类似，需要指定`destroy-method`属性

```xml
<bean id="book" class="com.sher.Book" destroy-method="destroyMethod"></bean>
```

#### 后置处理器

我们可以使用后置处理器来插手`Bean`的生命周期。创建一个后置处理器需要实现`BeanPostProcessor`方法，然后现在其中的两个方法。如：

```java
package com.sher;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

public class MyPost implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BookPost.postProcessBeforeInitialization");
        return null;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BookPost.postProcessAfterInitialization");
        return null;
    }
}
```

-   `postProcessBeforeInitialization`：在`Bean`调用初始化方法之前调用。
-   `postProcessAfterInitialization`： 在`Bean`调用初始化方法之后调用。

创建完后置处理器之后，只需要将其在设置`IOC`容器中，就可以使用，和普通的`bean`的方式是一样的。如：

```xml
<bean id="myPost" class="com.sher.MyPost"/>
```

需要注意的是，后置处理器会对`IOC`中所有的`Bean`的创建都有作用。





