# Spring学习日记03

## 自动注入

自动注入又被称为是自动装配，指定是`Spring`自己自动的将一个对象注入到另一个对象当中去。之前我们进行对象的注入的时候，使用的是外部`Bean`加上`ref`属性的方式，又或者是内部`Bean`的方式。但是如果`IOC`容器中只有一份我们需要的对象，又或者说我们指定了需要注入对象的`id`，`Spring`就可以帮助我们自动注入。

想要开启自动注入，需要在`bean`标签中添加属性`autowire`，它有两个取值，一个是`byType`，按照所需的类型进行注入（一般都是这种），另一个是`byName`，根据`id`进行注入。

如`Emp`表中有一个属性是`dept`，需要注入一个`Dept`对象，而且`IOC`容器中只有一个`Dept`对象，我们指定`autowire="byType"`，该对象就可以自动注入。

```xml
<bean id="dept" class="com.sher.bean.Dept">
    <property name="name" value="Tech"/>
</bean>

<bean id="emp" class="com.sher.bean.Emp" autowire="byType">
    <property name="name" value="sher"/>
    <property name="salary" value="8888"/>
</bean>
```

当`IOC`容器中有多`Dept`对象时，根据类型进行注入就会报错，因为`Spring`不知道该注入哪一个对象，此时我们可以根据`id`进行注入。设定`autowire="byName"`，`Spring`会寻找和当前属性名相同的`id`的对象进行注入。这里就是找`id`为`dept`的`Dept`对象进行注入。

```xml
<bean id="dept" class="com.sher.bean.Dept">
    <property name="name" value="Tech"/>
</bean>

<bean id="dept2" class="com.sher.bean.Dept">
    <property name="name" value="Tech2"/>
</bean>

<bean id="emp" class="com.sher.bean.Emp" autowire="byName">
    <property name="name" value="sher"/>
    <property name="salary" value="8888"/>
</bean>
```

不过这种用法真的不常见，`idea`也不推荐我们使用这种方式。因为上面的配置完全等同于

```xml
<bean id="dept" class="com.sher.bean.Dept">
    <property name="name" value="Tech"/>
</bean>

<bean id="dept2" class="com.sher.bean.Dept">
    <property name="name" value="Tech2"/>
</bean>

<bean id="emp" class="com.sher.bean.Emp">
    <property name="name" value="sher"/>
    <property name="salary" value="8888"/>
    <property name="dept" ref="dept"/>
</bean>
```

何必要使用自动装配来绕一下呢？我们使用自动注入的时候，一般都是根据**类型**自动注入。

## 引入外部依赖文件

自动注入是非常的常见的，我们常常会遇到这种情况。`UserDao`需要注入`DataSource`，`UserService`又需要注入`UserDao`等等，这些对象都是包含关系，而且一般情况下`IOC`容器中只会有一份，使用自动注入就非常的方便，简化了配置文件。

不过说到`DataSource`，就要提到配置数据库的连接了。之前我们在连接`JDBC`的时候，一般都是使用一个`jdbc.properties`，然后引用其中的值。那么在`Spring`的配置文件中又该如何引入外部的依赖文件呢？下面以使用`Druid`配置`JDBC`为例。

-   首先使用`Maven`引入`MySQL`的`jdbc`驱动还有`Druid`依赖。

    ```xml
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.18</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.1.22</version>
    </dependency>
    ```

-   然后编写`jdbc.properties`配置文件。

    ```properties
    prop.driverClassName=com.mysql.cj.jdbc.Driver
    prop.url=jdbc:mysql:///test
    prop.userName=root
    prop.password=root
    ```

-   在`Sprnig`的`xml`配置文件中引入`context`名称空间和之前引入`p, util`是相似的。

    ```xml
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:context="http://www.springframework.org/schema/context"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context.xsd">
    ```

    主要看第三行和第五行。

-   使用`context`名称空间引入外部依赖文件。

    ```xml
    <context:property-placeholder location="classpath:jdbc.properties"/>
    ```

    此时我们就可以在`xml`配置文件中，使用在`properties`中配置的信息了。

-   配置`DataSource`

    ```xml
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${prop.driverClassName}"/>
        <property name="url" value="${prop.url}"/>
        <property name="username" value="${prop.userName}"/>
        <property name="password" value="${prop.password}"/>
    </bean>
    ```

    使用`${}`这种表达式来引入外部的`properties`中的值。

-   简单的连接一下数据库，查询数据。

    ```java
    @Test
    public void test4() throws SQLException {
        ApplicationContext context = new ClassPathXmlApplicationContext("bean3.xml");
        DataSource dataSource = context.getBean("dataSource", DataSource.class);
        Connection connection = dataSource.getConnection();
        PreparedStatement statement = connection.prepareStatement("SELECT * FROM user");
        ResultSet resultSet = statement.executeQuery();
        while (resultSet.next()) {
            System.out.print(resultSet.getInt("id") + "\t");
            System.out.print(resultSet.getString("name") + "\t");
            System.out.println(resultSet.getInt("age"));
        }
        connection.close();
    }
    ```

    看来`Spring`使用起来真的是简单又顺手呀！