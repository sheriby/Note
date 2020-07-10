# SpringMVC学习日记01

## 什么是`SpringMVC`

`MVC`的全名是`Model View Controller`，是一种讲业务逻辑，数据，界面分离的一种设计典范。`SpringMVC`就是当前最流行的`WebMVC`框架。下面是对各层作用的一些说明。

-   **Model（模型）** - 模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。
-   **View（视图）** - 视图代表模型包含的数据的可视化。
-   **Controller（控制器）** - 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

## `SpringMVC`的`Hello World`

`SpringMVC`其实就是`Spring`框架中的`Web`模块，之所以单独的学习`SpringMVC`是因为其中细节比较多也相对比较复杂。

既然要开发的是`JavaWeb`项目，首先使用`Maven`创建一个`webapp`的项目。和普通的`Java`项目不同，`JavaWeb`项目，多了个`webapp`文件夹，其中有`web`相关的一下配置和文件。

其次，需要配置`Web`容器，其中最著名的就是`Tomcat`了。

### 引入`SpringMVC`依赖

只需要在`pom.xml`中简单的配置一下。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.2.6.RELEASE</version>
</dependency>
```

### 配置`DispatcherServlet`

`DispatcherServlet`是前置控制器，是我们配置`SpringMVC`需要做的第一步。他可以拦截匹配的请求，然后将其交给相应的`Controller`来处理。

我们需要在`web.xml`中写如下的配置。

```xml
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

其中我们使用`servlet`标签，注册`SpringMVC`给我们提供的`DispatcherServlet`，然后通过`servlet-mapping`标签设置该`servlet`需要拦截的请求。因为我们只有被`DispatcherServlet`拦截到的请求才能被我们处理，所以这里我们设置拦截`/`根请求，也就是拦截所有的请求。

#### 配置静态资源可以访问

所有的请求都会被`DispatcherServlet`拦截，当我们访问静态资源的时候如`xx/jquery.js`，该请求也会被`DispatcherServlet`拦截，但是在控制器中并没有相关的映射，就会报`404`错误。

为了防止静态资源无法访问，我们需要让`DispatcherServlet`不拦截这些请求，只需要设置这些资源给`DefaultServlet`来处理就行了。

```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.js</url-pattern>
</servlet-mapping>
```

这里我们不需要配置`jsp`，因为`jsp`文件就是动态的资源，其和`Servlet`本质上是一致的。

### 配置`SpringMVC`包扫描和视图解析器

在上面的`DispatcherServlet`中有一个`init-param`，加载了一个配置文件，那个就是我们`SpringMVC`的配置文件。可以在`resources`文件夹下面创建`springmvc.conf`进行配置。

首先我们需要配置的就是包扫描，然后需要配置的就是一个视图解析器。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.sher"/>

    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

视图解析器作用其实很简单，将逻辑视图更好的变成物理视图。比如我们要访问`http://localhost:8080/index`，这里的`index`就可以是一个物理视图，通过视图解析器，我们将`index`加上前缀和后缀，就相当于访问了`http://localhost:8080/index.jsp`。

### 配置`CharacterEncodingFilter`

除了配置`DispatcherServlet`之外，一般情况下还需要配置`CharacterEncodingFilter`，用于解决非常常见的中文乱码问题。

在`web.xml`中添加如下的配置。

```xml
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <url-pattern>/</url-pattern>
</filter-mapping>
```

配置的方式和`servlet`非常的相似。`filter`标签中注册一个`filter`，然后在`filter-mapping`中设置其需要处理的请求。这里也是设置为所有的请求。

### 配置`StringHttpMessageConvert`

上面的`CharacterEncodingFilter`解决的乱码问题是从前端到后端数据的乱码，此外一般情况下我们还需要解决一下从后端到前端数据的乱码。

引入`mvc`名称空间，在`springmvc.xml`配置文件中添加。

```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="supportedMediaTypes" value="text/html;charset=UTF-8"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>
```

### 编写一个简单的控制器

现在编写一个简单的控制器来处理请求，在`Web`里面，一般被称为`Handler`。

```java
package com.sher.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class HelloHandler {
    @RequestMapping("/index")
    public String index() {
        System.out.println("Visit index ...");
        return "index";
    }
}
```

上面就是一个简单的`Handler`。我们通过`@Controller`注解将其加入到`IOC`容器中交由`Spring`进行管理。需要注意的是，这里我们一般不写其余的三个注解，虽然也是可以的。

`Handler`中的方法都可以用来处理请求，通过在方法上使用`@RequestMapping`注解指定处理哪一个请求。如这里处理的是`index`请求。方法的返回值就用来展示页面。这里返回`index`，但是之前我们已经配置过一个视图解析器，所以会被解析称为`index.jsp`。

## `Hello World`简单的总结

浏览器访问网页`http://localhost:8080/index`，该请求被`DispatcherServlet`拦截到，发现`HelloHandler`中的方法`index`中的`@RequestMapping`可以匹配`index`，于是执行该方法。该方法返回字符串`index`，经过视图解析器解析加上了前缀和后缀变成了`index.jsp`。返回`index.jsp`中的内容给浏览器进行展示。