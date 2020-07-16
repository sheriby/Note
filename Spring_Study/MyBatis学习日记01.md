# MyBatis学习日记01

## 基本介绍

现如今`Java`最流行的框架就是`SSM`，分别是`Spring SpringMVC MyBatis`。`MyBatis`是一个持久化层的框架，是半自动的`ORM`框架。

要学习`MyBatis`，先从`MaBatis`的基础开始学起，然后使用`MyBatis`集成`Spring`，`Spring Boot`。

## `Hello World`

### 引入依赖

要使用`MyBatis`，首先需要在`pom.xml`中引入相关的依赖。

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.5</version>
</dependency>
```

### 配置数据库

创建`config.xml`文件作为`MyBatis`的配置文件。

```java
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <environments default="dev">
        <environment id="dev">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql:///mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```

在`configuration`中配置数据库的`environments`，里面可以指定多个数据库，然后通过`default`选择默认的数据库，这里只指定一个。

### 创建数据表及其对应的实体类

这里简单的创建一个`user`表用来学习。

```sql
create table user (
	id bigint not null primary key auto_increment,
  name varchar(20) not null,
  age varchar(20) not null
)
```

然后相对应的实体类`User`

```java
package com.sher.entity;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private int age;
}
```

### 使用`MyBatis`操作数据库。

#### 使用原生的接口

##### 创建`mapper.xml`

创建`UserMapper.xml`，在其中配置需要进行的`sql`操作。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sher.mapper.UserMapper">
    <insert id="save" parameterType="com.sher.entity.User">
        insert into user (id, name, age) values (#{id}, #{name}, #{age})
    </insert>
</mapper>
```

这里的`namespace`并不是强制的，可以随便写，但是一般写成这样子。

`insert`标签表示插入操作，同样的还有`updata delete select`标签。

`parameterType`表示传入方法的参数的类型，这里是实体类`User`。

通过`#{id}`这种方式我们就可以获取传入的对象的`id`属性的值。

##### 在`config.xml`中配置`Mapper`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <environments default="dev">
        <environment id="dev">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql:///mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="com/sher/mapper/UserMapper.xml"/>
    </mappers>
</configuration>
```

这里的`resource`写的是`xml`文件所在的类路径。

##### 执行方法

```java
@Before
public void before() {
    InputStream resource = MapperTest.class.getClassLoader().getResourceAsStream("config.xml");
    sqlSessionFactory = new SqlSessionFactoryBuilder().build(resource);
    sqlSession = sqlSessionFactory.openSession();
}
```

这里使用的是`junit`进行测试，`@Before`表示所有的测试方法之前都需要执行。

首先将上面的配置文件读取为一个流，这里使用类加载器的`getResourceAsStream`的方法可以在类路径下面读取文件。

然后通过`SqlSessionFactoryBuilder`的`build`方法创建`SqlSessionFactory`对象，需要传入上面读取的配置流。

最后通过`SqlSessionFacotry`的`openSession`方法可以获取一个`SqlSession`。通过这个`SqlSession`对象我们就可以执行在上面`mapper.xml`中定义的sql方法。

```java
@Test
public void test() {
    String statement = "com.sher.mapper.UserMapper.save";
    User user = new User(2L, "hony", 18);
    sqlSession.insert(statement, user);
}
```

使用`insert`方法执行插入语句，第一个参数是需要执行的语句的全名，相当于`namespace.id`，第二个参数就是我们需要插入的对象。

`SqlSession`使用结束之后需要`commit`，不然不会自动提交。不使用之后还需要`close`。

```java
@After
public void after() {
    sqlSession.commit();
    sqlSession.close();
}
```

#### 使用自定义接口

上面的方法似乎有点过于复杂了，我们可以通过`Mapper`代理的方式自定义接口。

-   自定义接口，写业务方法
-   编写相关的`mapper.xml`

创建`UserRepository`接口。

```java
public interface UserRepository {

    int save(User user);

    int deleteById(Long id);

    int update(User user);

    User findById(Long id);

    List<User> findAll();
}
```

编写对象的`UserRepository.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sher.repository.UserRepository">
    <insert id="save" parameterType="com.sher.entity.User">
        insert into user (id, name, age) values (#{id}, #{name}, #{age})
    </insert>

    <delete id="deleteById" parameterType="java.lang.Long">
        delete from user where id = #{id}
    </delete>

    <update id="update" parameterType="com.sher.entity.User">
        update user set name = #{name}, age = #{age} where id = #{id}
    </update>

    <select id="findById" parameterType="java.lang.Long" resultType="com.sher.entity.User">
        select * from user where id = #{id}
    </select>

    <select id="findAll" resultType="com.sher.entity.User">
        select * from user
    </select>
</mapper>
```

`MyBatis`会根据我们写的这个`mapper.xml`动态的生成自定义接口的实现类，我们只需要从`SqlSession`中获取就可以了。

不过既然是`MyBatis`自动给我们生成的，我们就需要遵循一定的规范。

-   `namespace`为自定义接口的全类名。
-   `id`和接口中的业务方法相同
-   `parameterType`和方法中的参数相同
-   对于`select`操作，`resultType`也要相对应。需要注意的是，这里的`findAll`方法返回的类型是`List<User>`，但是在`mapper.xml`中，我们只需要写`User`类型就可以了，`MyBatis`查询到多条数据之后，会自动的合并到一个`List`当中返回。

##### 在`cnofig.xml`中配置`Mapper`

同样的在`config.xml`中配置。

```xml
<mappers>
    <mapper resource="com/sher/mapper/UserMapper.xml"/>
    <mapper resource="com/sher/mapper/UserRepository.xml"/>
</mappers>
```

##### 获取动态生成对象，执行方法

通过`SqlSession`的`getMapper`方法获取`MaBatis`为我们动态生成的自定义接口的实例对象，然后使用该对象执行方法。

```java
@Test
public void test2() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user = new User(3L, "sheriby", 12);
    int res = userRepository.save(user);
    System.out.println(res);
}
```

对于同一个接口，我们不可以使用两个`Mapper`。

其他的方式使用方式同理。到这里，我们就已经会使用`MyBatis`的基本的功能了。

## 级联查询

### 创建实体类

之前的`Hello World`中完成了一些简单的查询操作。如果实体类中有属性为对象，此时又该如何查询呢？

```java
package com.sher.entity;

import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@NoArgsConstructor
public class User {
    private Long id;
    private String name;
    private int age;

    List<Book> books;

    public User(Long id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }
}
```

这里的`User`对象中多了一个`books`的属性，其中`Book`也是一个实体类。

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

如果我们想要通过`findById2`方法查询到`User`的信息，同时还需要查询到其拥有的`Book`，此时就需要通过级联查询的方式。

### 创建数据表

除了创建`book`表之外，我们还需要创建`user_book`表来关联这两个表。

```sql
create table book (
    id bigint auto_increment primary key,
    name   varchar(20) not null,
    author varchar(20) not null,
    price  float       not null
);
```

```sql
create table user_book (
    userid bigint references user(id),
    bookid bigint references book(id)
);
```

### 级联查询

```xml
<select id="findById2" parameterType="java.lang.Long" resultMap="userMap">
    select
    user.id id, user.name name, user.age age,
    book.id bookid, book.name bookname,
    book.author bookauthor, book.price bookprice
    from user, book, user_book
    where
    user.id = #{id}
    and user.id = user_book.userid
    and book.id = user_book.bookid
</select>
```

这里没有指定`resultType`而是指定`resultMap`，这里的`userMap`是我们在`xml`文件中自定义的`ResultMap`。

```xml
<resultMap id="userMap" type="com.sher.entity.User">
    <id column="id" property="id"/>
    <result column="name" property="name"/>
    <result column="age" property="age"/>
    <collection property="books" ofType="com.sher.entity.Book">
        <id column="bookid" property="id"/>
        <result column="bookname" property="name"/>
        <result column="bookauthor" property="author"/>
        <result column="bookprice" property="price"/>
    </collection>
</resultMap>
```

`type`指定需要将其包装成什么对象。`column`表示数据库中的字段，`property`表示实体类中的属性名。我们使用`<id>`标签注入主键，使用`<result>`标签注入非主键内容。

因为`books`属性的类型为`List<Book>`是一个集合，这里使用`collection`标签，使用`ofType`属性指定集合中元素的类型。

通过这个`ResultMap`，`MyBatis`就可以将查询到的结果包装成一个完整的`User`对象。

如果属性不是集合类型，就是一个对象，如`Book`对象，此时需要使用`association`标签，使用`javaType`属性指定对象的类型。如

```xml
<association property="book" javaType="com.sher.entity.Book">
	<id column="bookid" property="id"/>
	<result column="bookname" property="name"/>
	<result column="bookauthor" property="author"/>
	<result column="bookprice" property="price"/>
</association>
```

### 测试方法

```java
@Test
public void test6() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user = userRepository.findById2(2L);
    System.out.println(user);
}
```



