# Spring学习日记06

## JDBC in Spring

之前我们已经简单的学习了`Spring`中的两大最核心的模块，`IOC`和`AOP`。`Spring`中还有其他很多的模块，其中`JDBC`就属于`Spring`中的模块之一。`Spring`框架对`JDBC`做了完整的封装，写成了一个类`JdbcTemplate`，通过这个类我们可以很容易的操作数据库。再也不需要之前的`JDBCUtils`那些乱七八糟的东西了。

## 引入依赖

既然是需要操作数据库，相应的数据库的驱动以及一个合适的连接池都是需要准备的，还有就是`spring-jdbc`相关的依赖。只需要通过下面的`Maven`引入依赖就可以了。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.2.6.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.22</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.18</version>
</dependency>
```

## 配置数据库和`JdbcTemplate`

配置数据库的方法之前已经配置过一次了，写外部的`jdbc.properties`然后在`Spring`中引入外部依赖，通过`$()$`引用就可以了。具体的`Spring`配置文件如下。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context.xsd">

    <context:property-placeholder location="classpath:jdbc.properties"/>

    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${prop.driverClassName}"/>
        <property name="url" value="${prop.url}"/>
        <property name="username" value="${prop.userName}"/>
        <property name="password" value="${prop.password}"/>
    </bean>

    <context:component-scan base-package="com.sher.jdbc"/>
</beans>
```

下面我们需要做的就是将`DataSource`注入到`JdbcTemplate`中，但是`JdbcTemplate`并不是我们写的类，我们不可以使用注解进行自动注入。此时可以通过`ref`简单的引入。

```xml
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

这样我们就可以使用这个`JdbcTemplate`进行数据库的操作了。

## 使用`JdbcTemplate`进行增删改

首先我们需要创建一个类和数据库中的一行数据进行对应。

```java
package com.sher.entiry;

import lombok.Data;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private int id;
    private String name;
    private int age;
}
```

创建`UserDao`

```java
public interface UserDao {
    void add(User user);
    void delete(int id);
    void update(User user);
    int userNum();
    User queryByName(String name);
    User queryById(int id);
    List<User> queryOverAge(int age);
}
```

上面简单了些需要实现的方法。创建简单的实现类，然后其加入到容器当中，自动注入`JdbcTemplate`。

```java
@Repository
public class UserDaoImpl implements UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void add(User user) {
        String sql = "insert into user values(?, ?, ?)";
        jdbcTemplate.update(sql, user.getId(), user.getName(), user.getAge());
    }

    @Override
    public void delete(int id) {
        String sql = "delete from user where id = ?";
        jdbcTemplate.update(sql, id);
    }

    @Override
    public void update(User user) {
        String sql = "update user set name = ?, age = ? where id = ?";
        jdbcTemplate.update(sql, user.getName(), user.getAge(), user.getId());
    }
    ...
    ...
}
```

这里使用`@Repository`比使用`@Component`更好一些。可以看到的是，增删改的方法基本上都是一样的，都是调用`update`方法。这个和我们之前自己使用`JDBCUtils`实现的方式差不多。

## 使用`JdbcTemplate`进行查询

查询就比增删改要复杂一点点。

### 查询单个值

查询的结果是一个字符串或者说查询的结果是一个数字，比如说上面的查询`User`的数量，返回值是一个`int`类型。

```java
@Override
public int userNum() {
    String sql = "select count(id) from user";
    return jdbcTemplate.queryForObject(sql, Integer.class);
}
```

只需要使用`queryForObject`，然后在后面指定我们需要的类型就可以了。

### 查询单个对象

和查询单个值一样，我们还是使用`queryForObject`，但是不可以举一反三，直接将`Integer.class`变成`User.class`，而是通过这种方式。

```java
@Override
public User queryByName(String name) {
    String sql = "select * from user where name = ?";
    return jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(User.class), name);
}

@Override
public User queryById(int id) {
    String sql = "select * from user where id = ?";
    return jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(User.class), id);
}
```

使用`BeanPropertyRowMapper`将查询得到的数据包装成我们说需要的对象。

### 查询对象的集合

和查询单个对象相似，但是这里使用`query`方法。

```java
@Override
public List<User> queryOverAge(int age) {
    String sql = "select * from user where age > ?";
    return jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(User.class), age);
}
```

## 总结

使用`JdbcTemplate`可以非常容易的操作数据库。