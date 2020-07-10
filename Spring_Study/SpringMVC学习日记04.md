# SpringMVC学习日记04

## SpringMVC模型数据解析

所谓的模型数据解析其实就是将数据绑定在`JSP`的四大域作用对象上，其中最重要的作用域就是`request`，其次是`session`。其他两个要么是太大了要么是太小了，基本不常用。

模型数据的绑定是`ViewResolver`来完成的，我们只需要添加想应的模型数据，然后交由`ViewResolver`来绑定就行了。

### 将数据绑定到`request`中

首先要编写`jsp`来判断是否绑定成功。

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page isELIgnored="false" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
${requestScope.user}
</body>
</html>
```

#### `Model`

将`Model`作为方法的参数传入方法，然后讲需要绑定的参数放到`model`中。

```java
@RequestMapping("/model")
public String model(Model model) {
    User user = new User();
    user.setId(1);
    user.setName("sher");

    model.addAttribute("user", user);

    return "view";
}
```

#### `Map`

将`Map`作为方法的参数传入方法，然后将需要绑定的参数放到`map`中。

```java
@RequestMapping("/map")
public String map(Map map) {
    User user = new User();
    user.setId(1);
    user.setName("sher");

    map.put("user", user);

    return "view";
}
```

可以发现的是，上面的两个方法非常的像。其实其本质上操作的还是同一个对象，`ModelMap`。`Model`和`Map`都是接口，而`ModelMap`实现了这两个接口，因此我们写`Model`或者是写`Map`都是一样的。当然，我们也可以写`ModelMap`。

#### `ModelAndView`

这个就比较特别了，这个对象绑定了数据和视图。在方法中返回这个对象就完成了数据的绑定和视图的处理。

```java
@RequestMapping("modelAndView")
public ModelAndView modelAndView() {
    User user = new User();
    user.setId(1);
    user.setName("sher");

    ModelAndView modelAndView = new ModelAndView();
    modelAndView.addObject("user", user);
    modelAndView.setViewName("view");

    return modelAndView;
}
```

#### `@ModelAttribute`

这个注解和上面的稍有不同了，此注解给类中的某一个方法添加，该方法的返回会被绑定在该类的所有的业务方法的`request`中。也就是说，不单对某一个方法生效，对类中的所有方法生效。

```java
@ModelAttribute
public User getUser() {
    User user = new User();
    user.setId(1);
    user.setName("sher");

    return user;
}

@RequestMapping("modelAttribute")
public String modelAttribute() {
    return "view";
}
```

在`@ModelAttribute`注解的方法中处理`Model`，在业务方法中只需要页面跳转就行了。当然我们也也可以不返回，手动地进行绑定，如。

```java
@ModelAttribute
public void getUser(ModelMap modelMap) {
    User user = new User();
    user.setId(1);
    user.setName("sher");

    modelMap.addAttribute("user", user);
}

@RequestMapping("modelAttribute")
public String modelAttribute() {
    return "view";
}
```

这里使用到了上面没有使用到的`ModelMap`进行绑定。

#### `HttpServletRequest`

使用原生的`API`，将`HttpServletRequest`作为参数，然后在方法中进行绑定。

### 将数据绑定到`session`中

#### `HttpSession`

使用原生的`API`，将`HttpSession`作为参数，然后在方法中进行绑定。

#### `@SessionAttributes`

此注解只能给类添加，可以在注解中指定将那些参数添加到`request`的同时，再添加到`session`中去。对类中的所有的方法生效。如

```java
@Controller
@RequestMapping("/model")
@SessionAttributes(value = {"user", "student"})
public class ModelHandler {
    ......
}
```

上面我们将`user`添加到`request`中的所有的方法都会同时将`user`添加到`session`当中去。

## `SpringMVC`自定义数据转换器

数据转换器的功能是将`Http`中的请求参数转换称为业务方法的形参。`SpringMVC`默认给我们提供了`HandlerAdapter`，完成了一些通用的转换，如`String`转`int`，`String`转`double`等。我们根据自己的需要可以自定义数据转换器。

如下面将自定义转化器将`Http`请求参数中`2020-07-06`这种字符串转换称为`java`当中的``Date`类型。

### 创建`DateConverter`转换器，实现`Converter`接口

自定义的数据转换器都需要`Converter`接口。

```java
import org.springframework.core.convert.converter.Converter;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class DateConverter implements Converter<String, Date> {
    private final String pattern;

    public DateConverter(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public Date convert(String s) {
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat(pattern);
        Date date = null;
        try {
            date = simpleDateFormat.parse(s);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return date;
    }
}
```

需要注意的是这里的`Converter`最好要加上泛型。

### 配置数据转换器

写完了转换器之后需要到配置文件`springmvc.xml`中进行配置。

```xml
<bean id="conversionServices" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <list>
            <bean class="com.sher.converter.DateConverter">
                <constructor-arg name="pattern" value="yyyy-MM-dd"/>
            </bean>
        </list>
    </property>
</bean>

<mvc:annotation-driven conversion-service="conversionServices">
	....
</mvc:annotation-driven>
```

主要看上面的那个`ConversionServiceFactoryBean`的配置，里面可以放多个我们自定义的数据转换器，需要使用`list`标签。然后还需要在`<mvc:annotation-driven>`标签的`conversion-service`属性的配置。

### 数据转换器测试

编写`Handler`和`jsp`用于测试。

```java
@RestController
@RequestMapping("/converter")
public class ConverterHandler {

    @RequestMapping("date")
    public String date(Date date) {
        return date.toString();
    }
}
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
    <form action="/converter/date" method="post">
        <input type="text" name="date"/>
        <input type="submit"/>
    </form>
</body>
</html>
```



