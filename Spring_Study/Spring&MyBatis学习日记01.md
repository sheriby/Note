# Spring&MyBatis学习日记01

## 使用`Spring`整合`MyBatis`

现在先使用原生`Spring`的方式尝试一下整合`MyBatis`框架，之后再使用`Spring Boot`进行整合。一种非常简单的整合方式是使用`xml`的方式进行整合，但是我个人是不太喜欢`xml`的，而且现在都`Spring5`还在使用`xml`配置确实是有点儿捞，难道注解配置不香吗？

### 引入相关依赖

#### `Spring`

首先是`Spring`相关的依赖需要引入。(`maven`的这个`xml`配置其实也蛮头疼的，一直想要`gradle`，但是似乎有点问题，之后再尝试使用吧)

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
```

虽然只是引入了`context`和`jdbc`两个包，但是相关的`aop core tx expression`等包都会因为依赖被导入。

#### `MyBatis`

除了`MyBatis`的核心依赖之外，因为要整合`Spring`，还要导入和`Spring`相关的依赖。

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.5</version>
</dependency>

<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.5</version>
</dependency>
```

#### 数据库

数据库方面需要导入数据库的驱动，以及数据库连接池，这里使用`Spring Boot`中默认的数据库连接池`HikariCP`。

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.20</version>
</dependency>

<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>3.4.5</version>
</dependency>
```

#### 其他依赖

其他基本上都是工具类的依赖了。

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.12</version>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.30</version>
</dependency>

<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13</version>
    <scope>test</scope>
</dependency>
```

### `MyBatis`操作流程

在之前我们没有整合`Spring`的时候，使用`MyBatis`的基本的操作流程如下。

-   配置`MyBatis`的配置文件`config.xml`，里面主要是配置数据源以及注册`Mapper`。
-   编写实体类
-   编写自定义的`Mapper`接口
-   编写`mapper.xml`文件，配置自定义接口中方法。

其中第一步和第四步都是使用`xml`的方式，这里我将使用全注解开发，所以需要改变的就是第一步和第四步。第二步和第三步基本上都是一样的。

### 配置数据源

之前我们也做过使用`xml`配置`DataSource`的操作，这里将使用注解。

-   首先需要编写`jdbc.properties`

```properties
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql:///mybatis
jdbc.username=root
jdbc.password=root
```

-   编写`Spring`配置类

```java
package com.sher.config;

@Configuration
@PropertySource("classpath:jdbc.properties")
public class SpringConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.driver}")
    private String driver;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(url);
        dataSource.setDriverClassName(driver);
        dataSource.setUsername(username);
        dataSource.setPassword(password);

        return dataSource;
    }
}
```

使用`@PropertySource`注解指定我们需要将什么配置文件加载到`Spring`环境当中。

使用`@Value`注解注入基本数据类型，里面使用的是`${xx}`的方式，和我们在`xml`中使用的方式相同。

使用`@Bean`注解将配置好的`DataSource`加入到`Spring`容器中。

至此，配置数据源的工作就完成了。


### `SqlSessionFactoryBean`

之前使用`MyBatis`的操作分为以下流程。

-   使用`SqlSessionFacotryBuilder`读取配置文件流，`build`一个`SqlSessionFactory`。
-   使用`SqlSessionFactory`的`openSession`生成一个`SqlSession`。
-   使用`SqlSession`的`getMapper`获取动态代理生成的`Mapper`。
-   使用`Mapper`执行业务方法。

这里我们没有配置文件流，而且也不再使用`SqlSessionFactoryBuilder`，而且使用`SqlSessionFactroyBean`，它实现了`FactoryBean`接口。如果我们将其通过`xml`配置文件直接加入到容器中的话，会返回`SqlSessionFactory`对象。同样的，我们可以通过`SqlSessionFactory`做一些原本在配置文件中事情，如上面说的配置数据源，注册`Mapper`。

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());
    
    return factoryBean.getObject();
}
```

注册`Mapper`的操作可以留到后面再说。

### 创建实体类和对应`Mapper`接口

#### `Book`

```java
package com.sher.entity;

import lombok.Data;

@Data
public class Book {

    private Long id;
    private String name;
    private String author;
    private double price;
}
```

#### `User`

```java
package com.sher.entity;

import lombok.Data;

import java.util.List;

@Data
public class User {

    private Long id;
    private String name;
    private int age;

    List<Book> books;
}
```

#### `BookMapper`

```java
package com.sher.mapper;

import com.sher.entity.Book;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;

import java.util.List;

public interface BookMapper {
    
    List<Book> findByUserId(Long id);
}
```

#### `UserMapper`

```java
package com.sher.mapper;

import com.sher.entity.User;
import org.apache.ibatis.annotations.Many;
import org.apache.ibatis.annotations.Result;
import org.apache.ibatis.annotations.Results;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.mapping.FetchType;

public interface UserMapper {
    
    User findById(Long id);
}
```

### 注册和使用`Mapper`

一个比较麻烦的方式是通过`SqlSessionFactory`添加`Mapper`。

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());

    SqlSessionFactory factory = factoryBean.getObject();
    factory.getConfiguration().addMapper(BookMapper.class);
    return factory;
}
```

上面就注册了一个`BookMapper.class`。

然后可以通过如下的方式获取到这个`Mapper`。

```java
@Test
public void test2() {
    SqlSession sqlSession = applicationContext.getBean(SqlSessionFactory.class).openSession();
    BookMapper bookMapper = sqlSession.getMapper(BookMapper.class);
    List<Book> books = bookMapper.findByUserId(1L);
    System.out.println(books);
}
```

说实话有点儿复杂了。整合`Spring`之后我们可以有更加简单的方式，直接从`Spring`容器中获取`Mapper`。

使用`@MapperSacn`注解，指定我们`Mapper`所在的包。

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
@MapperScan("com.sher.mapper")
public class SpringConfig {
	......
}
```

使用的时候直接从`Spring`容器中获取，当然也可以直接`@Autowire`，非常的方便。

```java
@Test
public void test2() {
    BookMapper bookMapper = applicationContext.getBean(BookMapper.class);
    List<Book> books = bookMapper.findByUserId(1L);
    System.out.println(books);
}
```

但是`@MapperScan`注解是将包下面的所有的接口都实例化为`Bean`，如果有一些接口并不是`Mapper`呢，就会导致一些问题。我们可以指定需要扫描的接口上要有`@Mapper`注解，`@Mapper`注解只是一个标记注解，没其他额外的作用。

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
@MapperScan(value = "com.sher.mapper", annotationClass = Mapper.class)
public class SpringConfig {
	......
}
```

### 实现`Mapper`中的方法

这也是我们没有说的最后一步了。之前我们都是在`xml`文件中配置的这一步，这里使用注解。

```java
@Mapper
public interface BookMapper {

    @Select("select book.id id, book.name name, book.author author, book.price price " +
            "from book, user_book " +
            "where user_book.userid = #{id} and " +
            "user_book.bookid = book.id")
    List<Book> findByUserId(Long id);
}
```

这里的`@Select`注解就相当于`xml`文件中的`<select>`标签，这里我们不需要再去指定什么方法名，参数类型，返回类型了。

上面的返回类型是`List<Book>`类型的，`MyBatis`会自动的封装成`Book`对象，然后将其放入`List`中，非常的智能。

如果想要指定方法中参数在`sql`语句中的名字，可以使用`@Param`注解。

```java
@Select("select book.id id, book.name name, book.author author, book.price price " +
        "from book, user_book " +
        "where user_book.userid = #{hello} and " +
        "user_book.bookid = book.id")
List<Book> findByUserId(@Param("hello") Long id);
```

说实话，没想到这个有什么适用的场景。

上面的返回`List<Book>`还不够复杂，如果我们返回的是级联类型的呢？在`xml`中，我们需要编写`resultMap`，这里我们需要编写`@Results`注解。

```java
@Mapper
public interface UserMapper {

    @Select("select * from user where id = #{id}")
    @Results({
            @Result(id = true, column = "id", property = "id"),
            @Result(column = "name", property = "name"),
            @Result(column = "age", property = "age"),
            @Result(column = "id", property = "books",
                    many = @Many(select = "com.sher.mapper.BookMapper.findByUserId",
                            fetchType = FetchType.LAZY))
    })
    User findById(Long id);
}
```

这里的`User`的`books`属性为`List<Book>`类型的，这里使用`@Many`注解与其对应。和`xml`中同样的`select`属性，指向另一个查询的方法名，`FetchType.LAZY`表示的是懒加载的方式。只有用到了`books`属性才会去加载这个`sql`。在`xml`中对应的是`<collection>`标签。

如果是`Book`类型，不是集合类型，我们需要使用`one=@One(select="..")`的方式，在`xml`中对应`<association>`标签。

### 开启二级缓存

默认的`MyBatis`缓存，使用`xml`的话，是在`mapper.xml`中配置`<cache/>`标签就行了。这里使用注解也很简单，只需要使用`@CacheNamespace`注解就行了。

```java
@Mapper
@CacheNamespace
public interface BookMapper {

    @Select("select book.id id, book.name name, book.author author, book.price price " +
            "from book, user_book " +
            "where user_book.userid = #{id} and " +
            "user_book.bookid = book.id")
    @Options(useCache = true)
    List<Book> findByUserId(Long id);
}
```

上面的`@Options`中的`userCache`默认就是`true`，如果对于某一个方法不想缓存可以设置为`false`。

如果想要选择其他的缓存方案，可以配置`CacheNamespace`中的`implemention`，这里不再介绍。

当然`MyBatis`的懒加载和二级缓存默认都是关闭的，我们需要配置一下开启。

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());

    SqlSessionFactory factory = factoryBean.getObject();
    factory.getConfiguration().setLazyLoadingEnabled(true);
    factory.getConfiguration().setCacheEnabled(true);

    return factory;
}
```

### 总结

至此，我们已经可以在`Spring`中使用纯注解的方式开发`MyBatis`了。不过仅仅是基础，还有更多内容和注解没有介绍到。





​    