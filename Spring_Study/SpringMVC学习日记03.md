# SpringMVC学习日记03

## `SpringMVC`中的参数绑定

之前我们已经说过了基本的数据类型以及其包装类的数据的绑定了，其实还有其他的绑定的方式。

### `JavaBean`绑定参数

有时候我们想要将传递过来的请求参数封装成一个对象，虽然我们可以在方法中做这件事，但是还有更简单的方法。比如说我们讲`id name`封装成一个`User`对象。

```java
package com.sher.entiry;

import lombok.Data;

@Data
public class User {
    private int id;
    private String name;
    private boolean sex;
}
```

首先创建`User`实体类。

```java
@RequestMapping("/bean")
public String bean(User user) {
    System.out.println("Visit bean");
    System.out.println(user);
    return "index";
}
```

直接将其作为方法的参数，`id name`就会自动绑定到`User`对象中。

### 数组绑定参数

当请求的参数携带的值不止一个的时候，我们可以使用数组进行接收。如:

```
http://localhost:8080/bind/array?name=sher&name=hony&name=sheriby&id=1...
```

一个`name`对应了不止一个值。

```java
@RequestMapping("/array")
public String array(String[] name, int id) {
    System.out.println(Arrays.toString(name));
    return "ok";
}
```

### `List`绑定参数

`SpringMVC`中不可以直接个`List`赋值，需要创建相应的包装类。

```java
package com.sher.entiry;

import lombok.Data;

import java.util.List;

@Data
public class UserList {
    private List<User> userList;
}
```

发送请求的时候的表单如下图所示。

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>List</title>
</head>
<body>
<form action="/bind/list" method="post">
    <input type="text" name="userList[0].name"/><br>
    <input type="text" name="userList[0].id"/><br>
    <input type="submit"/>
</form>

</body>
</html>
```

但是我们在使用`Postman`等工具或者直接使用地址栏发送请求的时候，不能直接使用`[]`，需要进行转义。`[=> %5B ]=>%5D`，如。

```
http://localhost:8080/bind/list?userList%5B0%5D.name=sheri&userList%5B0%5D.id=123
```

```java
@RequestMapping("/list")
public String list(UserList userList) {
    System.out.println(userList.getUserList());
    return "ok";
}
```

### `Map`绑定参数

和`List`绑定参数相似，也需要写相应的封装类，这里省略。表单的格式为:

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>List</title>
</head>
<body>
<form action="/bind/list" method="post">
    <input type="text" name="userMap['a'].name"/><br>
    <input type="text" name="userMap['a'].id"/><br>
    <input type="submit"/>
</form>

</body>
</html>
```

### `JSON`数据

有时候前端通过`Ajax`给我们发送的是`json`格式的数据，这种情况下也是可以封装的。

我们需要将`json`转换称为`java`中的对象，需要引入一些依赖，常用的是`fastjson`。

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.62</version>
</dependency>
```

然后在`springmvc.xml`文件中配置`fastjson`。

```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="supportedMediaTypes" value="text/html;charset=UTF-8"/>
        </bean>
        <bean class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter"/>
    </mvc:message-converters>
</mvc:annotation-driven>
```

和我们配置从后端到前端的乱码的位置是相同的。

这里通过`Jquery`发送`Jquery`请求，那么就需要引入`jquery.js`。这里采用本地引入的方法，下载`jquery.js`放置在`js`文件夹下。

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Ajax</title>
</head>
<script type="text/javascript" src="js/jquery.js"></script>
<script>
    $(function () {
        const user = {
            name: 'sher',
            id: 777
        }
        alert('ok')

        $.ajax({
            url: '/bind/json',
            data: JSON.stringify(user),
            type: 'POST',
            dataType: 'JSON',
            contentType: 'application/json;charset=UTF-8',
            success: function (data) {
                alert(data.id + '===' + data.name)
            }
        })
    })
</script>
<body>

</body>
</html>
```

在`Handler`中处理这个`Ajax`请求。

```java
@RequestMapping("/json")
public User json(@RequestBody User user) {
    System.out.println(user);
    user.setName("hony");
    user.setId(2);
    return user;
}
```

我们需要将参数指定为`@RequestBody`，因为传过来的是`Json`格式的字符串。然后`Spring`会使用`fastjson`对该字符串进行解析，然后转换成`Java`对象`user`。需要注意的是该方法的返回值是`User`，这个对象将通过`fastjson`转成`json`数据，然后返回给前端`ajax`请求。前端又会将这个`json`转成称为`javascript`对象进行使用。