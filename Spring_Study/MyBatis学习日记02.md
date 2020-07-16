# MyBatis学习日记02

## 延迟加载

以之前的`findById2`方法为例，有时候我们只是想得到`User`的`name`，但是通过这个方法我们强制的查询了`books`属性，查询了三个表，做了很多多余的事情。我们可以通过延迟加载的操作，知道用到`books`属性的时候再去加载，没用到的时候不加载。

### 开启延迟加载

需要在`config.xml`中配置一下。

```xml
<configuration>
    <settings>
        <setting name="logImpl" value="STDOUT_LOGGING"/>
        <setting name="lazyLoadingEnabled" value="true"/>
    </settings>
</configuration>
```

这里开启了两个配置，第一个`logImpl`的作用是开启执行的`sql`的打印功能，第二个`lazyLoadingEnabled`设置为`true`就是开启延迟加载的功能。

### 使用延迟加载

所谓的延迟加载需要将一条使用多个表的`sql`语句分解成多个使用单独表的`sql`语句，由于`user_book`表是中间表，所以我们可以将

```sql
select
	user.id id, user.name name, user.age age,
	book.id bookid, book.name bookname,
	book.author bookauthor, book.price bookprice
from user, book, user_book
where
	user.id = #{id} and
	user.id = user_book.userid and
	book.id = user_book.bookid
```

这种联结多个表的`sql`语句拆分成以下的两个简单一点的`sql`语句。

```sql
select * from use where id = #{id}

select
	book.id id, book.name name, book.author author, book.price price
from book, user_book
where
	user_book.userid = #{id} and
	user_book.bookid = book.id
```

当我们不需要用到`books`属性的时候，就只执行第一条`sql`，什么时候要用到`books`属性了再去执行第二条`sql`语句。

#### 编写`BookRepository`接口

```java
package com.sher.repository;

import com.sher.entity.Book;

import java.util.List;

public interface BookRepository {

    List<Book> findByUserId(Long id);
}
```

这里的`findByUserId`其实就是上面的第二条`sql`的功能。

#### 实现`BookRepository.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.sher.repository.BookRepository">
    <select id="findById" parameterType="java.lang.Long" resultType="com.sher.entity.Book">
        select * from book where id = #{id}
    </select>

    <select id="findByUserId" parameterType="java.lang.Long" resultType="com.sher.entity.Book">
        select
            book.id id, book.name name, book.author author, book.price price
        from book, user_book
        where
            user_book.userid = #{id} and
            user_book.bookid = book.id
    </select>
</mapper>
```

#### 实现延迟加载方法

```xml
<resultMap id="userMapLazy" type="com.sher.entity.User">
    <id column="id" property="id"/>
    <result column="name" property="name"/>
    <result column="age" property="age"/>
    <collection property="books" ofType="com.sher.entity.Book"
       select="com.sher.repository.BookRepository.findByUserId" column="id"/>
</resultMap>

<select id="findByIdLazy" resultMap="userMapLazy">
    select * from user where id = #{id}
</select>
```

同样的还是需要使用到`resultMap`。在`books`属性中指定`select`和`column`。这里的`select`指向的是`BookRepository`中的那个方法名，`column`是给定的参数。

### 测试延迟加载

```java
@Tes@Test
public void test8() {
    BookRepository bookRepository = sqlSession.getMapper(BookRepository.class);
    List<Book> books = bookRepository.findByUserId(1L);
    System.out.println(books);
}
```

观察打印的`sql`语句可以看出来，两个`sql`语句都被调用，因为我们打印`user`对象，需要使用到`books`属性。

```java
@Test
public void test9() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user = userRepository.findByIdLazy(1L);
    System.out.println(user.getName());
}
```

此时只打印了一条`sql`语句，因为我们只使用到了`name`属性，并没有使用到`books`属性。

## 缓存

除了延迟加载可以提高数据库的效率，还有一个种方法就是使用缓存。使用缓存可以减少程序和数据库的交互的次数，提高效率。在`MyBatis`中的缓存分为两种类型，一级缓存和二级缓存。

### 一级缓存

一级缓存是`SqlSession`级别的，对于同一个`SqlSession`，我们执行同样的查询，后面的会直接从缓存中获取。如果执行了`insert update delete`语句，缓存中的内容会立马清空，因为此时无法保证缓存中的内容还是准确的。

`MyBatis`中的一级缓存是默认开启的，而且是无法关闭的。

```java
@Test
public void test10() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user1 = userRepository.findById(1L);
    System.out.println(user1);
    User user2 = userRepository.findById(1L);
    System.out.println(user2);
}
```

执行了两次查询方法，但是只会执行一次`sql`语句。

```java
@Test
public void test11() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user1 = userRepository.findById(1L);
    System.out.println(user1);
    sqlSession.close();

    sqlSession = sqlSessionFactory.openSession();
    userRepository = sqlSession.getMapper(UserRepository.class);
    User user2 = userRepository.findById(1L);
    System.out.println(user2);
}
```

使用的不是同一个`SqlSession`，还是执行两次`sql`语句。

### 二级缓存

二级缓存是`Mapper`级别的。也就是即使是不同的`SqlSession`，只要是对同一个`Mapper`的操作的数据都是可以共享的。

二级缓存默认是关闭的，可以开启。

#### `MyBatis`自带的二级缓存

##### `config.xml`

首先在`config.xml`中配置，开启二级缓存。

```xml
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

其中第三个配置就是开启二级缓存的配置，前面的两个配置是之前使用的。

##### `mapper.xml`

然后在需要开启缓存的`Mapper`中配置。

```xml
<mapper namespace="com.sher.repository.UserRepository">
    <cache/>
    ....
</mapper>
```

添加一个`cache`标签就行了。

##### 实体类序列化

`Mapper`对应的那个实体类，需要实现`Serializable`接口，如

```java
@Data
@NoArgsConstructor
public class User implements Serializable {
    private static final long serialVersionUID = -859484160068145658L;

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

这里因为对象中还有`Book`类型，所以`Book`也需要实现`Serializable`接口。

```java
@Data
public class Book implements Serializable {
    private static final long serialVersionUID = 4390108442428843142L;

    private Long id;
    private String name;
    private String author;
    private double price;
}
```

##### 测试二级缓存

```java
@Test
public void test12() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user1 = userRepository.findByIdLazy(1L);
    System.out.println(user1);
    sqlSession.close();

    sqlSession = sqlSessionFactory.openSession();
    userRepository = sqlSession.getMapper(UserRepository.class);
    User user2 = userRepository.findByIdLazy(1L);
    System.out.println(user2);
}
```

虽然是不同的`SqlSession`，但是`sql`语句只执行了一次。

#### `ehcache`二级缓存

##### 添加依赖

`ehcache`是一个专门的缓存框架，我们可以将其集成在`MyBatis`中。首先在`pom.xml`中添加相关的依赖。

```xml
<dependency>
    <groupId>org.mybatis.caches</groupId>
    <artifactId>mybatis-ehcache</artifactId>
    <version>1.2.1</version>
</dependency>

<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>3.8.1</version>
</dependency>
```

##### 添加`ehcache.xml`

需要使用额外的配置文件来配置`ehcache`缓存。下面是一个简单的常用配置。

```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false">

    <diskStore path="java.io.tmpdir"/>

    <defaultCache maxElementsInMemory="1000"
                  eternal="false"
                  timeToIdleSeconds="3600"
                  timeToLiveSeconds="0"
                  overflowToDisk="true"
                  maxElementsOnDisk="10000"
                  diskPersistent="false"
                  diskExpiryThreadIntervalSeconds="120"
                  memoryStoreEvictionPolicy="FIFO"
    />

    <cache
            name="cloud_user"
            eternal="false"
            maxElementsInMemory="5000"
            overflowToDisk="false"
            diskPersistent="false"
            timeToIdleSeconds="1800"
            timeToLiveSeconds="1800"
            memoryStoreEvictionPolicy="LRU"
    />

</ehcache>
```

##### `config.xml`

和上面同样的方法

```xml
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

##### `mapper.xml`

还是使用`<cache>`标签配置，但是因为不是使用`MyBatis`自带的缓存，所以需要额外的配置。

```xml
<cache type="org.mybatis.caches.ehcache.EhcacheCache">
    <property name="timeToIdleSeconds" value="3600"/>
    <property name="timeToLiveSeconds" value="3600"/>
    <property name="memoryStoreEvictionPolicy" value="LRU"/>
</cache>
```

##### 测试方法

和官方提供的二级缓存的很大的不同就是，使用`ehcache`缓存的话实体类并不需要实现`Serilizable`接口。

```java
@Test
public void test12() {
    UserRepository userRepository = sqlSession.getMapper(UserRepository.class);
    User user1 = userRepository.findByIdLazy(1L);
    System.out.println(user1);
    sqlSession.close();

    sqlSession = sqlSessionFactory.openSession();
    userRepository = sqlSession.getMapper(UserRepository.class);
    User user2 = userRepository.findByIdLazy(1L);
    System.out.println(user2);
}
```

结果一样，虽然使用了不同的`SqlSession`，同样的`SQL`语句只执行了一次。







