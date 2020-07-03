# Spring学习日记01

## Spring的简单介绍

`Spring`是当前`Java`企业级应用开发的最主流的框架，可以说是`Spring`早就了`Java`如今的辉煌。那么这么牛逼的框架到底是用来做什么的呢？

`Spring`是一个轻量级的底层框架，他并不是面向某一个具体的应用或者过程来做的，这就决定了他的应用面非常的广泛。其最重要的两个特性，`IOC`和`AOP`极大的简化了企业级应用的开发。

## Spring的Hello World

`Spring`既然是一个框架，我们必然是需要额外下载的，但是作为一个框架也不然不是只有一个jar包，应该是很多个jar包的集合体。虽然我们可以通过手动的下载这些jar包然后将其一个个导入到项目中，先且不谈这么多jar包的版本是否是一致的，是否还有其他的依赖，光是从网上找包就要花费不少的时候，因而这里使用现在Java中比如常见的项目依赖管理工具`Maven`来构建`Spring`项目。

-   打开`IDEA`创建`Maven`项目。

-   添加`Spring context`依赖

    ```xml
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.2.6.RELEASE</version>
    </dependency>
    ```

这样`Spring`的核心的几个`jar`包就会添加到项目中了。

### IOC初识

IOC翻译成中文的意思是控制反转，目的是减少类与类之间的耦合，讲对象的创建交给`IOC`容器来管理。其内部的主要实现方式是`xml配置 + 工厂模式 + 反射机制`。

下面简单的介绍一下如果通过`Spring`来创建对象，而不是直接通过`new`来创建对象。

### 创建一个简单的`Java Bean`

```java
package com.sher;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@AllArgsConstructor // 提供所有参数的构造器
@NoArgsConstructor // 提供无参构造器
@Data // 提供getter，setter， toString(), equals(), hashcode()等各种方法
public class User {
    private String name;
    private Integer age;
    private Book book;
}
```

上面使用了`lombok`简化了我们的代码。

### 在`resources`文件夹下创建`Spring`的配置文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="user" class="com.sher.User"></bean>

</beans>
```

该配置文件的最大的标签是`beans`，在该标签下我们使用`bean`标签来配置我们想要的对象。如上面所示的`<bean id="user" class="com.sher.User"></bean>`，这里的`id`和html中的id作用相似，是该对象的唯一的标识，后面在容器中我们可以通过`id`来获取该对象。`class`属性是告诉`Spring`我们要创建什么对象，需要写类的全类名。

`Spring`在帮我们创建对象的时候会调用类的无参构造器，所以在类中已经写了有参构造器的情况下，我们一定要补上一个无参构造器。

### 创建测试类，启动

这里可以导入`Junit`用于测试，在`Maven`的配置文件`pom.xml`中添加`junit`依赖。

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13</version>
</dependency>
```

当然也可以不使用`Junit`直接使用`main`函数进行测试。

```java
package com.sher;

import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MainTest {

    @Test
    public void test() {
        // 读取Spring的配置文件
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("bean.xml");
        // 通过id获取我们在配置文件中配置的bean
        User user = (User) applicationContext.getBean("user");
        System.out.println(user);
    }
}
```

运行单元测试，输出的结果为`User{name='null', age='null'}`。

上面就是一个最简单的`Spring`的`HelloWorld`的`Demo`了。

## 如何向`Bean`中注入属性

上面我们虽然创建了对象，但是有一个问题，我们调用的无参构造器，无法赋予属性值，所以得到的属性值全都是`null`，那么如何向`Bean`中注入属性呢？

### 通过`set`的方法注入属性

我们可以在对象创建完成之后，调用对象的`setXXX`方法来对象的属性进行设置，当然这些步骤并不是我们需要做的。我们需要做的就是在`Bean`中自动生成好所有属性的`setter`方法，然后在`xml`文件中配置就行了。

在`bean`标签下配置`property`标签，在其中配置我们需要设置的属性值。其中`name`为属性名，`value`为属性值。

```xml
<bean id="user" class="com.sher.User">
    <property name="name" value="sher"/>
    <property name="age" value="18"/>
</bean>
```

### 通过有参构造器注入属性

我们也可以通过类的有参构造器来设置属性。

```xml
<bean id="user" class="com.sher.User">
    <constructor-arg name="name" value="hony"/>
    <constructor-arg name="age" value="16"/>
</bean>
```

需要注意的是，当我们通过这种方式指定调用有参构造器，无参构造器并不会被调用，但是一般来说一个JavaBean类中，需要有一个无参构造器。

### `p`名称空间对注入的简化

使用`p`名称空间可以简化通过`set`方法注入属性的方式。

#### 在`xml`文件中添加`p`名称空间

只需要在`beans`标签中添加如下的属性

```xml
xmlns:p="http://www.springframework.org/schema/p"
```

#### 通过`p`名称空间注入属性

此时，我们就不需要写前面长长的一串`property`了。

```xml
<bean id="user" class="com.sher.User" p:name="sher" p:age="18"/>
```

只需要一行就可以注入属性。

## 属性注入的一些问题

### 需要注入`null`值

可能这个会有点问题，因为不注入不就是空值吗？但是有些时候是必须要注入的，这时候我们不能写成这样。

```xml
<property name="xxx" value="null"/>
```

这个相当于注入了一个字符串`null`。如果想要注入空值的话，需要使用专门的标签`<null/>`

```xml
<property name="xxx">
	<null/>
</property>
```

### 注入的字符串中有特殊符号

如`<>`这种符号

```xml
<bean id="user" class="com.sher.User">
    <property name="name" value="<<sher>>"/>
</bean>
```

这种方式虽然在`xml`的语法中没什么问题，但是在`Spring`解析这个`xml`文档的时候，会将其当作标签，所以我们需要额外的处理这些特殊字符。

#### 使用转义字符

第一种方式我们之前在html中也是学过的，使用特殊的转义符号来表示这些特殊字符。

```xml
<bean id="user" class="com.sher.User">
    <property name="name" value="&lt;&lt;sher&gt;&gt;"/>
</bean>
```

但是这种方式总归看起来不是那么直观，还有一种方式来处理这些特殊字符。

#### 使用CDATA

第二种方式是将这些特殊符号写到`CDATA`中。如：

```xml
<bean id="user" class="com.sher.User">
    <property name="name">
        <value><![CDATA[<<sher>>]]></value>
    </property>
</bean>
```

这种方式比起上面的转义就非常的直观了。

### 注入对象

上面我们注入的都是一些字面量，那么该如何注入对象呢？在此之前，创建`Book`类，然后为`User`新添加一个新的属性`Book`，并添加相应的`setter`。

#### 内部`Bean`

这种方式就像是之前我们需要注入`null`值，那时候我们使用`<null/>`标签，此时我们需要注入一个对象，使用的就是我们之前一直在使用的`<bean>`标签。在`<bean>`标签中配置需要注入的对象。

```xml
<bean id="user" class="com.sher.User">
    <property name="name" value="sher"/>
    <property name="age" value="18"/>
    <property name="book">
        <bean class="com.sher.Book" p:name="Best Python" p:price="89.99"/>
    </property>
</bean>
```

需要注意的是，内部`bean`是不需要给定`id`属性的，因为外部无法获取该`bean`，`id`也就没有存在的必要了。

#### 外部`Bean`

但是有的时候我们在外部也需要获取这个对象，或者说这个对象要给多个对象共享，此时就不可以使用内部`Bean`了，此时需要创建外部的`Bean`。那么这两个`Bean`该如何关联呢？通过`ref`属性关联。

```xml
<bean id="book" class="com.sher.Book" p:name="Best Python" p:price="89.99"/>

<bean id="user" class="com.sher.User">
    <property name="name" value="sher"/>
    <property name="age" value="18"/>
    <property name="book" ref="book"/>
</bean>
```

### 注入数组类型的数据

现修改`User`类中的属性，每个人拥有不止一本书，所以这里使用一个数组来存放。

```java
package com.sher;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@AllArgsConstructor
@NoArgsConstructor
@Data
public class User {
    private String name;
    private Integer age;
    private Book[] books;
}
```

我们可以通过`<array>`标签注入数组类型的数据。

```xml
<bean id="book1" class="com.sher.Book" p:name="Best Python" p:price="89.99"/>
<bean id="book2" class="com.sher.Book" p:name="Best Scala" p:price="79.99"/>

<bean id="user" class="com.sher.User">
    <property name="name" value="sher"/>
    <property name="age" value="18"/>
    <property name="books">
        <array>
            <ref bean="book1"/>
            <ref bean="book2"/>
        </array>
    </property>
</bean>
```

因为这里是采用外部`Bean`的注入方式，所以在`<array>`标签中需要使用`<ref>`标签，然后使用`bean`属性指定需要注入的对象。如果是内部注入的话，可以使用`<bean>`标签直接注入。如果只是简单的字面量（如字符串）的话，就更简单了，可以使用`value`标签，如。

```xml
<array>
	<value>sher</value>
  <value>hony</value>
</array>
```

### List类型数据的注入

除了数组类型，我们还经常使用`List`集合来存放数据，如`ArrayList`等，如果将`Book[]`改成`List<Book>`的话，就不可以使用`<array>`标签进行注入了，此时应该使用`<list>`标签。如：

```xml
<bean id="user" class="com.sher.User">
    <property name="name" value="sher"/>
    <property name="age" value="18"/>
    <property name="books">
        <list>
            <ref bean="book1"/>
            <ref bean="book2"/>
        </list>
    </property>
</bean>
```

虽然我们在定义的时候使用的是`List<Book>`，但是`Spring`默认会使用`ArrayList`来实现。如果我们将其更改为`LinkedList<Book>`，发现直接报错了，看来我们只能进行`ArrayList`类型的`List`的注入，不过`List`中也就是`ArrayList`比较常见吧。

### Map类型的注入

`map`的注入使用的是`<map>`标签，配合`<entry>`标签，每一个`<entry>`都是一项。

```xml
<property name="userMap">
    <map>
        <entry key="one" value="1"/>
        <entry key="two" value="2"/>
    </map>
</property>
```

需要注意的是`Map`的默认实现是`LinkedHashMap`，无论我们写的是`Map`还是`HashMap`，最终注入的都是`LinkedHashMap`类型。

### set类型的注入

`set`和`list`的注入基本是相似的，只是将`<list>`改成了`<set>`而已。

```xml
<property name="userSet">
    <set>
        <value>Java</value>
        <value>C++</value>
        <value>Python</value>
    </set>
</property>
```

其实现为`LinkedHashSet`。

### 提取集合类型属性注入

和外部`bean`，内部`bean`一样，有时候我们的集合并不是只给一个对象使用的，那么如何集合写在`bean`的外部然后写入呢？一般来说需要遵循以下的步骤。

#### 引入名称空间`util`

和引入`p`名称空间的功能相似，扩展`xml`语法。

添加完成后，`beans`标签的属性如下。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/util  http://www.springframework.org/schema/util/spring-util.xsd">
```

#### 提取外部集合

利用`util:list`对`List`进行提取。

```xml
<util:list id="list">
    <value>Java</value>
    <value>Python</value>
    <value>C++</value>
</util:list>
```

然后在`bean`的`<property>`中使用`ref`属性进行注入，和注入外部`bean`的方式是相似的。

```xml
<property name="language" ref="list"/>
```

同样的，我们可以使用`util:map`和`util:set`对`Map`和`Set`进行提取。





