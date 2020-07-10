# SpringMVC学习日记02

## SpringMVC中的注解

传统的`JavaWeb`开发充斥着大量的配置文件以及各种各样的`Servlet`的配置，开发起来非常的复杂。从之前的`SpringMVC`的`Hello World`我们可以看到，我们可以简单的使用几个注解大大简化开发。`SpringMVC`是一个复杂的框架，这也就意味着我们开发起来更加的简单舒心。

### `@Controller`

将类实例加入到`IOC`容器中，一般用于控制器。该注解需要配置包扫描来使用。

```xml
<context:component-scan base-package="com.sher"/>
```

又或许在配置类中使用注解配置。

### `@RequestMapping`

`SpringMVC`通过`@RequestMapping`注解进行`URL`和业务方法的映射。该注解可以加载类和方法的定义处。在一个项目中可能有不止一个`Handler`，如果我们将`@RequestMapping`加在`Handler`类的定义上，就相当于给其所有的方法添加了一个`URL`前缀。如：

```java
@Controller
@RequestMapping("/hello")
public class HelloHandler {

    @RequestMapping("/index")
    public String index() {
        System.out.println("Visit index ...");
        return "index";
    }
}
```

此时我们需要访问`/hello/index`才可以。

#### 其他常用参数

默认参数`value`是指定映射的`URL`，该注解还有其他的参数供使用。

#### method

**指定请求的方式**，最常用的两种方式是`GET`和`POST`，在`Rest`风格的`URL`中还有`PUT`和`DELETE`这两种请求方式。

```java
@Controller
@RequestMapping("/hello")
public class HelloHandler {

    @RequestMapping(value = "/index", method = RequestMethod.GET)
    public String index() {
        System.out.println("Visit index ...");
        return "index";
    }
}
```

需要注意的是指定请求方式不是通过字符串，而是通过枚举类`RequestMethod`。我们使用浏览器输入网址访问就是发送`GET`请求，我们可以使用`PostMan`等工具发送其他的请求。

上面我们指定该`URL`只可以使用`POST`方法，如果我们使用其他方法会报`405 Method Not Allowed`错误。当然我们也可以指定可以使用多个方法访问，如

```java
@RequestMapping(value = "/index", method = {RequestMethod.POST, RequestMethod.GET})
```

还有一点需要注意的是，默认情况是，我们是只可以使用`GET POST HEAD`这三种方法的，因为其他的请求方法存在着安全隐患。**即使我们在 method中指定其他方法可以访问，也是无济于事的。**需要后面使用其他方法解决。

#### params

***指定请求中必须要携带给定参数，不然无法访问。***如

```java
@RequestMapping(value = "/index", params = {"id", "name"})
```

此时访问`/hello/index`的时候，需要要携带`id name`这两个参数，否则会报`400 Bad Request`错误。当然我们也可以指定某个参数必须是某一个值，如

```java
@RequestMapping(value = "/index", params = {"id=007", "name"})
```

指定`id`必须要是`007`才可以访问。需要注意的是不能在等号左右加空格，如写成了`id = 007`，这个相等于我们必须要携带参数`id空格`，其值必须要是`空格007`。

`GET`方法携带的参数可以直接放在地址中，如

```java
http://localhost:8080/hello/index?id=007&name=sher
```

`POST`等其他方法可以使用`Postman`或者表单。

### 获取参数

### `@RequestParam`

上面我们设置了需要携带参数，那么该如何获取参数呢？最简单的方法如下

```java
@RequestMapping(value = "/index", params = {"id", "name"})
public String index(int id, String name) {
    System.out.println("Visit index ...");
    System.out.println(id + name);
    return "index";
}
```

直接讲请求参数名作为方法的参数，类型会进行自动转换。如果发现方法参数并没有对应一个请求参数的话就会给方法参数赋`null`，如

```java
@RequestMapping(value = "/index", params = {"id", "name"})
public String index(int id1, String name) {
    System.out.println("Visit index ...");
    System.out.println(id + name);
    return "index";
}
```

如果请求参数中没有`id1`，那么参数就会报错，因为我们无法将`null`赋给`int`类型。如果是`Integer id1`的话就不会报错。

如果我们真的需要将方法参数`id1`绑定请求参数`id`的话，就需要使用`@RequestParam`注解了。

```java
@RequestMapping(value = "/index", params = {"id", "name"})
public String index(@RequestParam("id") Integer id1, String name) {
    System.out.println("Visit index ...");
    System.out.println(id1 + name);
    return "index";
}
```

在参数前面加上注解，并指定绑定的请求参数名。

#### defaultValue

如果没有给定该请求参数的话，给其一个默认值。原来的默认值是`null`。可以给定默认值的参数那肯定不是必须要携带的，所以在`parms`中不能有。

```java
@RequestMapping(value = "/index", params = {"id", "name"})
public String index(@RequestParam(value = "id2", defaultValue = "17") Integer id1, String name) {
    System.out.println("Visit index ...");
    System.out.println(id1 + name);
    return "index";
}
```

这里如果我们没有携带`id2`参数的话，默认会赋为`17`。

### `@PathVariable`

有时候我们使用的是`Rest`风格的`URL`，如`/rest/7/sher`，参数在`URL`当中，并不在`？`后面的参数中，使用`@ResquestParam`无法获取到。此时我们需要使用`@PathVariable`，当然也要修改一下`@ResquestMapping`中的内容。如：

```java
@RequestMapping("/rest/{id}/{name}")
public String rest(@PathVariable("id") int id, @PathVariable("name") String name) {
    System.out.println("Visit rest");
    System.out.println(id + name);
    return "index";
}
```

在`@RequestMapping`中指定参数的位置，然后在方法的参数中使用`@PathVariable`获取。需要注意的是，虽然这里参数名字都是相同的，但是我们不可以省略`@PathVariable`，因为省略了就代表获取请求参数`@RequestParam`中的内容了。如

`http://localhost:8080/rest/7/sher?id=1&name=hony`，此时获取到的是`16和hony`。

### `@CookieValue`

除了获取请求参数中的值，我们还可以获取`Cookie`中的值，使用`@CookieValue`注解。

```java
@RequestMapping("cookie")
public String cookie(@CookieValue("JSESSIONID") String sessionId) {
    System.out.println(sessionId);
    return "index";
}
```

### `@RequestBody @RestController`，转发和重定向

通过情况下处理方法的处理结果将交由视图解析器处理然后得到相应的`HTML`数据返回给浏览器。如上面的方法返回`index`经过解析之后就返回了`index.jsp`中的内容。

不过并不是所有的返回都会经过视图解析器的解析，只有转发的请求才会。跳转页面有两种方式，一种是**转发**，另一种是**重定向**。转发的服务器内部的跳转，只产生一次请求，而重定向是浏览器的跳转，产生了两次请求。两者可见的最大的区别就，转发不会改变浏览器的地址栏，而重定向会改变浏览器的地址栏。

在`SpringMVC`中，默认的跳转方式是转发。

```java
@RequestMapping("/cookie")
public String cookie(@CookieValue("JSESSIONID") String sessionId) {
    System.out.println(sessionId);
    return "index";
}
```

相当于，加上转发的前缀`forward:`

```java
@RequestMapping("/cookie")
public String cookie(@CookieValue("JSESSIONID") String sessionId) {
    System.out.println(sessionId);
    return "forward:/index.jsp";
}
```

需要注意的是，这里不可以写成`forward:index`，因为补全了之后视图解析器是不会处理的。

重定向的前缀是`redirect:`

```java
@RequestMapping("/cookie")
public String cookie(@CookieValue("JSESSIONID") String sessionId) {
    System.out.println(sessionId);
    return "redirect:/index.jsp";
}
```

有时候，我们并不需要返回页面，我们只需要返回一些数据。如返回`json`数据或者返回字符串之类的。这时候，需要给方法添加`@ResponseBody`注解，指定方法的返回值就是需要返回的数据。

```java
@RequestMapping("/index")
@ResponseBody
public String index() {
    return "index";
}
```

此时直接返回一个字符串`index`，并不是`HTML`页面。

如果一个`Handler`中的所有的方法都是`@ResponseBody`方法，那么可以将`@Controller`修改称为`@RestController`，然后去掉所有方法中的`@ResponseBody`。`@RestControoler`的`Handler`中的所有的方法都是直接返回数据的。

