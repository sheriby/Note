# Spring实战读书笔记05

## 使用配置属性

这一章节的内容和`Taco Cloud`这个项目基本无关，主要是讨论`Spring Boot`的配置问题的。之前我们使用`Spring`，都需要大量复杂的`xml`配置，但是现在使用`Spring Boot`之后，一行配置文件都看不到，如果我们真的想要自定义配置一些属性，又该怎么做呢？

### `Spring`的环境抽象

其实我们也并不是一行配置都没添加过，我们在配置数据库的时候，使用到了`application.yml`这个文件进行配置。但这也只是`Spring`配置的一种方式。

在`Spring`启动的时候，会将属性源中的所有的属性值聚合到同一个源中，供`Spring`容器中的所有的`Bean`来使用。属性源包括一下的几个部分。

-   `JVM`系统属性
-   操作系统环境变量
-   命令行参数
-   `application.yml`
-   `application.properties`

需要注意的是`application.yml`和`application.properties`只能使用一种，个人倾向于使用`yml`格式，有结构很美观。

如，`Tomcat`的默认启动端口为`8080`，如果我们想要使用`8888`端口启动，可是选择设置`server.port`这个属性。

#### 使用`JVM`系统属性

```java
java -Dserver.port=8888 -jar xxx.jar
```

使用`java -D`命令可以设置`JVM`系统属性。

#### 使用命令行参数

```java
java -jar xxx.jar --server.port=8888
```

和使用`JVM`系统属性类似，在命令行的最后使用`--xx=xx`。

#### 使用`application.yml`

```yml
server:
  port: 8888
```

#### 使用系统环境变量

系统环境变量的命名方式和上面有所不同。

```bash
export SERVER_PORT=8888
```

不过`Spring`读取到这个配置后，会将其转换为`server.port=8888`。

对于启动的端口这个属性，如果我们设置`server.port=0`，服务器并不会真的在`0`端口上面启动，而是随机使用一个没有被使用的端口启动。在我们需要保证并发测试的时候端口不能冲突，而且不关心在哪个端口启动的时候，可以这样配置。

### 配置日志

设置显示的日志的级别。

```yml
logging:
  level:
    root: WARN
    org.springframework.security: DEBUG
```

设置日志保存的位置。

```yml
logging:
  path: /var/logs/
  file: TacoCloud.log
```

### 使用自定义配置属性

在`OrderController`中显示数据的时候，需要使用到`pageSize`属性来设置一次显示多少数量。为了之后可以方便的修改，可以在`application.yml`中配置这个属性。

```yml
taco:
  orders:
    page-size: 10
```

这里的`page-size`和`pageSize`是一样的。

然后我们需要在`OrderController`中开启配置属性的注解，`@ConfigurationProperties`。

```java
@Controller
@RequestMapping("/orders")
@SessionAttribute("order")
@ConfigurationProperties(prefix="taco.orders")
public class OrderController {
    
    private int pageSize = 20;
    
    public setPageSize(int pagSize) {
        this.pageSize = pageSize;
    }
    ....
}
```

上面我们设置了前缀为`taco.orders`，属性中的`pageSize`就会被自动注入为上面的设置的`taco.orders.page-size`中的值。

### 定义属性持有者

上面这种配置有点让`Controller`的代码不那么简洁了，比如我们还想对属性`pageSize`做一些约束，或者是添加其他的一些属性，又要在`Controller`中添加代码。这些本不是`Controller`应该做的功能。我们可以专门写一个类处理这些属性。

```java
@Component
@ConfigurationProperties(prefix="taco.orders")
@Data
public class OrderProps {
    private int pageSize = 20;
}
```

想要获取属性，这个对象必须要加入`IOC`容器，所有需要加上`@Component`注解。之后我们只需要讲该属性类注入到`OrderController`中使用就可以了。

这里对属性的限制和验证的工作也可以放到属性中来做。如

```java
@Component
@ConfigurationProperties(prefix="taco.orders")
@Data
@Validated
public class OrderProps {
    
    @Min(value=5, message="must be between 5 and 25")
    @Max(value=25, message="must be between 5 and 25")
    private int pageSize = 20;
}
```

### 使用`profile`

有时候对于不同的环境我们需要不同的配置，此时可以创建另外的一个配置文件。要遵守一下的约定`application-xxx.yml`。如开发环境的配置放在`application-dev.yml`中，生产环境的配置放在`application-pro.yml`。通用的配置可以放在`application.yml`中。

```yml
spring:
  profiles:
    actives:
      - pro
```

当然也可以选择激活多个`profile`。

```yml
spring:
  profiles:
    actives:
      - pro1
      - pro2
      - pro3
```

### 使用`profile`条件化地创建`bean`

不同的环境配置不同的配置文件，也可以根据激活的环境条件化的创建`bean`。只需要使用`@Profile`注解。

```java
@Bean
@Profle("dev")
public xxx xxx{
    
}
```

该`bean`只会在`dev`环境被激活的时候创建。

```java
@Profile({"dev", "dev2"})
```

该`bean`会在`dev`或者`dev2`激活的时候才加载。

```java
@Profile("!pro")
```

该`bean`会在`pro`没被激活的时候被加载。

```java
@Profile({"!pro", "!qa"})
```

该`bean`会在`pro`并且`qa`都没有被激活的时候被加载。

