# Spring实战读书笔记01

## `Tacos Cloud Example`

### 新建`Spring Boot`项目

可以选择在`start.spring.io`中或者是在`IDEA`的`Spring Initializr`(也是基于`start.spring.io`)中创建相应版本的`Spring Boot`项目。选择我们需要使用到的`web thymeleaf test`依赖

这里不知道是我的网速的原因还是连这个网站都被墙了的原因，我无法通过以上的方式直接创建`Spring Boot`项目，只好创建`Maven`项目，手动配置`pom.xml`。

`pom.xml`中的内容如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.sher</groupId>
    <artifactId>SpringInAction</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>taco-cloud</name>
    <description>Taco Cloud Example</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.1.RELEASE</version> 
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>htmlunit-driver</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### `pom.xml`的一些说明

```xml
<packaging>jar</packaging>
```

指定项目打包为`jar`包而不是`war`包，打包为`jar`包也是`Spring initializr`的默认选择。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.1.RELEASE</version> 
    <relativePath/>
</parent>
```

指定我们需要使用的`Spring Boot`的版本，后面的库都需要据此依赖管理。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

我们构建`Spring Boot`项目的时候选择的三大模块。

-   `thymeleaf`是一个类似`JSP`的模板引擎
-   `web`用来编写`web`应用
-   `test`用来编写测试

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

`Spring Boot`的`Maven`插件。

### 引导类

引导类又被称为是主类，一般用来配置`Spring Boot`和引导`Spring Boot`的启动。本案例中一个最小的配置如下。

```java
package com.sher.tacos;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TacoCloudApplication {

    public static void main(String[] args) {
        SpringApplication.run(TacoCloudApplication.class, args);
    }
}
```

`@SpringBootApplication`是一个非常强大的注解，是以下三个注解的组合。

-   `@SpringBootConfiguration`：和我们使用`Spring`注解版时相似，当时我们使用`@Configuration`表明这是一个`Spring`配置类。这里的`SpringBootCOnfiguration`其实就是`@Configration`注解的特殊形式。
-   `@EnableAutoConfiguration`：开启`Spring Boot`自动配置。
-   `@ComponentScan`：`Spring`注解版用到过，启用组件的自动扫描。

在`main`方法中使用`SpringApplication.run()`执行应用的引导，也就是创建`Spring`的应用上下文。一般来说，第一个参数就写引导类（也可以写其他的类）。

### 测试类

```java
package com.sher.tacos;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class TacoCloudApplicationTest {

    @Test
    public void contextLoads() {

    }
}
```

上面是最简单的一个测试类，只有一个空的方法，但是基本如此，测试类也会运行以确保应用上下文创建成功。

`@RunWith`是`JUnit`的注解，使用`SpringRunner`引导`JUnit`如何进行测试。

`@SpringBootTest`给`JUnit`在启动测试的时候添加上`Spring Boot`的功能，相当于该测试类在`main`方法中调用`SpringApplication.run()`。

### 处理请求

编写一个简单的`Handler`处理`web`请求。

```java
package com.sher.tacos.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller // 添加到IOC容器中，交给Spring管理
public class HomeHandler {

    @GetMapping("/")
    public String home() {
        return "home";
    }
}
```

和`SpringMVC`中编写的相似，但是这里我们不用写任何的视图解析器或者是其他的配置了。

在`SpringMVC`中设置`URI`映射使用的注解是`@RequestMapping`，这里的`@GetMapping`其实就是`@RequestMapping(method = RequestMethod.GET)`的缩写。同样的还有`@PostMapping`等。

该方法返回`home`字符串，在`SpringMVC`中我们通过视图解析器可能会将这个逻辑视图处理为`/home.jsp`这种形式。这里视图的具体实现还要取决于多种因素。

### 定义视图

这里使用`thymeleaf`作为模板引擎，编写`home.html`页面。

```html
<!DOCTYPE html>
<html lang="en"
      xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Taco Cloud</title>
    <style>
        body > img {
            width: 100px;
            height: 100px;
        }
    </style>
</head>
<body>
    <h1>Welcome come to <strong>Tacos Cloud</strong>!</h1>
    <img th:src="@{/images/avator.jpg}"/>
</body>
</html>
```

这里使用到了`thymeleaf`的表达式`th:src="@{/images/avator.jpg}"`来引用图片，我们需要在相应的位置添加图片。

### 测试控制器

上面控制器和视图都写好了，需要写一个测试类来测试其功能。（这一步并不是必须的，我们可以直接打开浏览器测试）

```java
package com.sher.tacos;

import com.sher.tacos.controller.HomeHandler;
import org.hamcrest.Matchers;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;


@RunWith(SpringRunner.class)
@WebMvcTest(HomeHandler.class)
public class HomeHandlerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testHomePage() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/"))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.view().name("home"))
                .andExpect(MockMvcResultMatchers.content()
                        .string(Matchers.containsString("Welcome come to")));

    }
}
```

这里没有使用`@SpringBootTest`而是使用`@WebMvcTest`注解。`@WebMvcTest`不仅可以创建应用上下文，还可以将其中的参数`HomeHandler`注册到`SpringMVC`中以供测试。

自动注入`MocMvc`类实例以供测试，在`testHomePage`中的测试，看一下方法名就知道测试了啥。

### 启动应用

会到引导类中，运行`main`方法，该项目就运行起来了。此时可以打开浏览器访问`http://localhost:8080`查看运行是否正常。

一个较为完整的应用已经运行起来了，但是似乎我们一行配置都没有写，甚至我们的电脑连难搞的`tomcat`都没有安装，这就是`Spring Boot`的强大之处，提供了强大的自动配置，已经内嵌`tomcat`。

一个人一旦熟悉起`Spring Boot`的开发，就再也不想使用原生的`Spring SpringMVC`了。

