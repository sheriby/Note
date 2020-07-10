# Spring学习日记04

## 基于注解的开发

之前听过一个笑话，说的是有一个开发了几十万行的`C++`破项目，代码十分臃肿而且结果混乱。`Python`程序员说，这种项目用我们大`Python`八万行就可以搞定。`Java`程序员表示不屑，说使用我们`Java`只需要六万行代码。周围的程序员都笑了，因为六万行代码还附带了几十万行`xml`配置文件。

虽然这只是一个笑话，但是就我们之前的`Spring`学习来说，`xml`配置文件确实是挺多的，然而，我们也可以完全不用`xml`文件，`Spring`给我们提供了注解开发的方式。

注解开发不要是太方便！之前我们创建类时候使用的`lombok`插件，仅仅使用了一个`@Data`注解就简化了几十行的代码。

### 使用注解创建对象

我们可以使用一下的四个注解创建对象。

-   `@Component`；通用
-   `@Service` ：一般用于服务层
-   `@Controller`：一般用于Web层
-   `@Repository` ： 一般用于持久化层（DAO）

上面的四个注解作用相同，都可以将类实例加入到`IOC`容器中，一般情况需要挑选一个合适的使用。

### 组件扫描

不是添加了上面的注解，就可以讲其添加到`IOC`容器中，我们需要告诉`Spring`，去哪儿添加组件。这里还是以`xml`配置的方式举例。

首先我们需要引入`context`名称空间，然后配置组件扫描。

```xml
<context:component-scan base-package="com.sher.bean"/>
```

其中的`base-package`用于配置扫描哪个包下面的类。这里我的`com.sher.bean`包下面有`Emp`和`Dept`两个类，将他们加上`@Component`注解，就会被自动加入到`IOC`容器中。如果我们需要扫描多个包，则可以使用逗号分隔，如：

```xml
<context:component-scan base-package="com.sher.bean,com.sher.factorybean"/>
```

当然，这两个包都在`com.sher`下，我们也可以写成

```xml
<context:component-scan base-package="com.sher"/>
```

当组件被加入到`IOC`容器中，他们的`id`默认是类名的第一个字母小写，如`Emp`变成`emp`。当然，我们也可以通过注解设置，如

```java
@Data
@Component(value = "sher")
public class Emp {
    private String name;
    private double salary;
    private Dept dept;
}
```

这时候加入`IOC`容器的对象的`id`就会被设置称为`sher`了。我们来测试一下。

```java
@Test
public void test5() {
    ApplicationContext context = new ClassPathXmlApplicationContext("bean3.xml");
    Emp emp = context.getBean(Emp.class);
    Dept dept = context.getBean(Dept.class);
    System.out.println(emp);
    System.out.println(dept);
    System.out.println(emp.getBeanName());
}
```

这里我们通过类型来获取`Bean`，然后输入获取到的`Bean`的`id`是多少。如果想要得到`Bean`的`id`，我们需要进行额外的操作，让其实现`BeanNameAware`接口，实现其中的方法即可。

```java 
@Data
@Component(value = "sher")
public class Emp implements BeanNameAware {
    private String name;
    private double salary;
    private Dept dept;
    private String beanName;
}
```

其抽象方法为`setBeanName`，但是这个方法已经被`lombok`实现了，后面我们使用`getBeanName`方法就可以得到`Bean`的`id`。

#### 组件扫描的配置

我们还可以对组件扫描进行额外的配置，如

```xml
<context:component-scan base-package="com.sher.bean" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

指定扫描的包为`com.sher.bean`，然后设置`use-default-filters`为`false`。默认是扫描上面说的那四个注解，现在取消了这个默认设置，然后在子标签中设置扫描`@Controller`。也就是我们只添加`com.sher.bean`包下面带了`@Controller`注解的类。

```xml
<context:component-scan base-package="com.sher.bean">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

上面这个设置了一个`exclude-filter`，不扫描带`@Controller`注解的，也就是说只扫描另外的三个。

### 使用注解进行组件扫描

上面我们进行组件扫描都是通过写`xml`配置的方式进行实现的。现在我们可以不需要配置文件，`Spring`提供了一个配置类，我们可以在配置类中配置组件扫描。

```java
@Configuration // 设置为Spring配置类
@ComponentScan(value = "com.sher.bean") // 设置组件扫描
public class SpringConfig {
}
```

将一个类加上`@Configuration`注解，表明其是一个`Spring`配置文件。加上`@ComponentScan`注解就可以配置组件的扫描了。

同样的使用注解也可以达到上面的两种较为复杂的`xml`配置。

只扫描`@Controller`注解的类。

```java
@Configuration
@ComponentScan(value = "com.sher.bean",
        useDefaultFilters = false,
        includeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,
                classes = {Controller.class})})
public class SpringConfig {
}
```

不扫描`@Controller`注解的类

```java
@Configuration
@ComponentScan(value = "com.sher.bean",
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,
                classes = {Controller.class})})
public class SpringConfig {
}
```

这种情况下，似乎使用`xml`更加的简洁呢。

### 在注解类中创建对象

我们使用`@Controller`这些注解创建对象，一个类只能创建一个对象在`IOC`容器中。那么该怎么创建多个对象呢？这就需要使用注解类了。在注解类中使用`@Bean`来创建对象。

```java
@Configuration
public class SpringConfig {

    @Bean("configBean")
    public User user() {
        User user = new User();
        user.setName("sher");
        user.setAge(18);
        return user;
    }
}
```

加上了`@Bean`标签的注解，方法名默认就是对象的`id`名，但是我们也可以通过上面的方式手动的设置对象的`id`。该方法返回的对象会被加入到`IOC`容器当中。

### 使用注解进行自动注入

还有一点我们没有从`xml`配置文件向注解迁移，那就是自动注入功能。自动注入使用注解实现可谓是非常的简单。

我们可以使用一下的四个注解，下面我们一一介绍区别。

-   `@AutoWire`， 根据类型自动注入
-   `@Qualifier`， 根据给定`id`自动注入
-   `@Resource`， 根据类型或者`id`自动注入
-   `@Value`， 注入基本类型的数据

首先是`@AutoWire`注解，上面我们没有对`Ept`当中注入`Dept`对象。因为`IOC`容器中只有一份`Dept`，所以我们可以直接通过类型自动注入。其做法是：

```java
@Data
@Component(value = "sher")
public class Emp implements BeanNameAware {
    private String name;
    private double salary;
    @Autowired
    private Dept dept;
    private String beanName;
}
```

在需要注入的属性前面加上`@Autowired`注解。

如果想要根据`id`进行自动注入，需要同时使用`@Autowired`和`@Qualifier`注解，如

```java
@Data
@Component(value = "sher")
public class Emp implements BeanNameAware {
    private String name;
    private double salary;
    @Autowired
    @Qualifier(value = "dept")
    private Dept dept;
    private String beanName;
}
```

至于`@Resource`注解，其属于`javax`包，并不是`Spring`的标准注解，所以不推荐使用。我的`jdk`中甚至都没有这个注解。使用方式如下。

```java
@Data
@Component(value = "sher")
public class Emp implements BeanNameAware {
    private String name;
    private double salary;
 // @Resource     							通过类型自动注入
 // @Resource(name="dept")		 通过名称自动注入
    private Dept dept;
    private String beanName;
}

```

最后一个`@Value`注解用来注入基本数据，如

```java
@Data
@Component
public class Emp implements BeanNameAware {
    @Value("sher")
    private String name;
    @Value("8888")
    private double salary;
    @Autowired
    private Dept dept;
    private String beanName;
}
```

## 总结

至此，我们的`Spring`中已经完全不需要`xml`配置文件了，只需要简单的几个注解就可以配置好。