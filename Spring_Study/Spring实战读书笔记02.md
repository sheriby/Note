# Spring实战读书笔记02

## 开发`Web`应用

通过第一章的学习，我们已经简单的搭建了`Spring`项目的基础。通过这一章的学习，我们将进一步的完善这个项目。

### 构建实体类

#### `Taco`

本网站的目的是在线订购`Taco`，所以我们第一步需要做的就是定义这个实体类。

```java
package com.sher.tacos.entity;

import lombok.Data;
import java.util.List;

@Data
public class Taco {
    
    private String name;
    private List<String> ingredient;
}
```

其中`ingredient`表示顾客需要订购的`Taco`的成分，因为不止一个，所以这里使用`List`储存。这里的`@Data`注解，之前学习`Spring`的时候就已经用到了，需要引入`lombok`依赖。

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.12</version>
    <scope>provided</scope>
</dependency>
```

这里的范围写成`provided`并不是必须的，但是最好还是这样做。因为`lombok`在`java`文件编译之后就不再有作用了。

#### `Ingredient`

上面定义了`Taco`类之后，我们需要定义配置，也就是上面属性的`ingredient`。虽然在上面是使用`String`类型来储存的，但是也还是有必要将其做成一个实体类的。

```java
package com.sher.tacos.entity;

import lombok.Data;
import lombok.RequiredArgsConstructor;

@Data
@RequiredArgsConstructor
public class Ingredient {

    private final String id;
    private final String name;
    private final Type type;

    public static enum Type {
        WRAP, PROTEIN, VEGGIES, CHEESE, SAUCE
    }

}
```

配料的类型用的是一个内部的枚举类。上面用到了`@RequiredArgsConstructor`注解，其作用是生成类中所有的的`final`属性的构造器。

#### `Order`

除了`Taco`和`Ingredient`之外，我们还需要一个实体类。因为我们是网上订餐的网页，所以我们需要一个类来表示订单的各种信息。

```java
package com.sher.tacos.entity;

import lombok.Data;

@Data
public class Order {

    private String name;
    private String street;
    private String city;
    private String state;
    private String zip;
    
    private String ccNumber;
    private String ccExpiration;
    private String ccCVV;
}
```

### 编写控制器和视图

所谓的控制器(`Controller`)，也就是`MVC`中的`C`，主要是用来处理前端的`Http`请求的类。在`SpringMVC`中我们已经接触了非常多了。

#### 只处理视图映射关系的请求

在访问了`http://localhost:8080/design`，之后，我们需要给这个请求以回应。在前一章中，我们访问`http://localhost:8080/`页面之后，`return "home"`。只是返回了一个视图。

这种体现不出控制器的真正作用，而且对于上面的这种只是做一个视图映射，而不做任何数据处理的话，我们甚至可以不用写一个控制器。可以直接让一个类实现`WebMvcConfigurer`接口，然后在其中配置就行了。

一般情况下，我们可以让`Spring Boot`引导类来做这件事，如

```java
@SpringBootApplication
public class TacoCloudApplication implements WebMvcConfigurer {

    public static void main(String[] args) {
        SpringApplication.run(TacoCloudApplication.class, args);
    }

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
```

重写`addViewControllers`方法，然后在方法中配置映射关系。

#### 处理`/design`请求

因为`html`页面的数据并不是静态的，所以我们不可以直接展示视图，要通过后端讲视图渲染到前端中去，也就是使用视图模板。之前，一般都是使用`JSP`这种老掉牙的技术，不过在`Spring`中，还是最推荐`thymeleaf`视图模板技术的。

```java
@Slf4j
@Controller
@RequestMapping("/design")
public class DesignTacoController {

    @GetMapping
    public String showDesignForm(Model model) {
        List<Ingredient> ingredients = Arrays.asList(
                new Ingredient("FLTO", "Flour Tortilla", Type.WRAP),
                new Ingredient("FLTO2", "Flour Tortilla2", Type.WRAP),
                new Ingredient("FLTO3", "Flour Tortilla3", Type.PROTEIN),
                new Ingredient("FLTO4", "Flour Tortilla4", Type.VEGGIES),
                new Ingredient("FLTO5", "Flour Tortilla5", Type.CHEESE),
                new Ingredient("FLTO6", "Flour Tortilla6", Type.SAUCE),
                new Ingredient("FLTO7", "Flour Tortilla7", Type.VEGGIES)
        );

        Type[] types = Type.values();
        for (Type type : types) {
            model.addAttribute(type.toString().toLowerCase(), filterByType(ingredients, type));
        }

        model.addAttribute("design", new Taco());

        return "design";
    }

    public List<Ingredient> filterByType(
            List<Ingredient> ingredients, Type type) {
        return ingredients
                .stream()
                .filter(x -> x.getType().equals(type))
                .collect(Collectors.toList());
    }
}
```

`@Slf4j`注解由`lombok`提供，在这里他的作用就相当于在类中创建了一个静态属性。

```java
private static final org.slf4j.logger log = 
    org.slf4j.LoggerFactory.getLogger(xxx.class)
```

这里的`xxx`就是`DesignTacoController`。

这里的`@GetMapping`没用任何参数，这就相当于`value=""`，其处理的类上面的`/design`请求。在该方法中，我们手动的创建了一些配料的数据。然后通过`filterByTyper`方法将其按`Type`分类好。该方法中使用到了`Java8`中的`StreamAPI`以及`lambda`表达式，很简单易懂。

我们想要数据来渲染页面，就要通过一定的途经将数据带过去。在学`SpringMVC`的时候已经学过了，一般是使用`Model`或者`Map`，也有时候可以使用`ModelAndView`或`@ModelAttribute`等。

如上面的第20行和第23行绑定了数据。第20行绑定的`type`很容易理解，用来在前端显示给我们看。但是第23行的绑定`Taco`对象却不是用来显示的。而是用来回传。我们通过`design`页面进行选择，可以创建这样的一个`Taco`对象，然后回传到后端来处理。

#### `design.html`

所有的模板文件都需要放在`resources/templates`中。

```html
<!DOCTYPE html>
<html lang="en"
      xmlns="http://www.w3.org/1999/xhtml"
    xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Taco Cloud</title>
</head>
<body>
    <h1>Design Your Taco</h1>
    <img th:src="@{images/avator.jpg}" alt="image" style="width: 100px;height: 100px;">

    <form method="post" th:object="${design}">
        <div>
            <div>
                <h3>Designate your warp:</h3>
                <div th:each="ingredient : ${wrap}">
                    <input name="ingredient" type="checkbox" th:value="${ingredient.id}">
                    <span th:text="${ingredient.name}"></span><br/>
                </div>
            </div>
            <div>
                <h3>Designate your protein:</h3>
                <div th:each="ingredient : ${protein}">
                    <input name="ingredient" type="checkbox" th:value="${ingredient.id}">
                    <span th:text="${ingredient.name}"></span><br/>
                </div>
            </div>
            <div>
                <h3>Designate your cheese:</h3>
                <div th:each="ingredient : ${cheese}">
                    <input name="ingredient" type="checkbox" th:value="${ingredient.id}">
                    <span th:text="${ingredient.name}"></span><br/>
                </div>
            </div>
            <div>
                <h3>Designate your veggies:</h3>
                <div th:each="ingredient : ${veggies}">
                    <input name="ingredient" type="checkbox" th:value="${ingredient.id}">
                    <span th:text="${ingredient.name}"></span><br/>
                </div>
            </div>
            <div>
                <h3>Designate your sauce:</h3>
                <div th:each="ingredient : ${sauce}">
                    <input name="ingredient" type="checkbox" th:value="${ingredient.id}">
                    <span th:text="${ingredient.name}"></span><br/>
                </div>
            </div>
            <h3>Name your taco creation:</h3>
            <input type="text" th:field="*{name}"/>
            <br/>

            <button>Submit your taco</button>
        </div>
    </form>

</body>
</html>
```

我们需要在`html`标签的属性中添加`xmlns:th="http://www.thymeleaf.org"`以说明这是一个`thymeleaf`模板文件。

可以看到的是，在该`html`文件中有很多不属于`html`的表达式语言，就像`el`表达式一样。`thymeleaf`的所有的表达式都是以`th:`开头的。

#### Thymeleaf

##### `@{}`

一般和`th:src`配置使用，如上面的`th:src="@{image/avator.jpg}"`，这种`@{xx}`表示的是链接。我们同样可以在其中使用一个网络链接，如

```text
th:src="@{https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=1089267199,3108441249&fm=26&gp=0.jpg}"
```

这里我们引用的资源没有前缀，也就是相当与本项目的链接

```text
th:src="@{http://localhost:8080/images/avator.jpg}"
```

其在项目中的位置可以是`src/main/resources/static/images/avator.jpg`

##### `${}`

获取容器中的上下文对象，也就是在我们从`Controller`中通过`Model`传递过来的数据都可以通过这种方法获取。`"${design}"`获取了我们传递过来的`Taco`对象。

##### `#{}`

一般和`th:text`一起使用，表明标签中的信息是`#{}`中的`key`对应的`value`，原标签中的信息将不会被显示。如。

```html
<span th:text="#{key}">Will not display</span>
```

其`key-value`一般来自于静态的`properties`文件，常常涌来做国际化。

##### `*{}`

从选择的对象中获取属性，而不是整个环境中。所谓的选定的对象就是`th:object`绑定的对象。

##### `@{}`和`${}`的嵌套

```html
//错误的写法
<img th:src="@{/getImage/${attDesc.identityImgN}">

//正确的写法
<img th:src="@{/getImage/{filename}(filename=${attDesc.identityImgN})}">
```

##### `th:object`

用来接收从后台传递过来的对象。

##### `th:field`

用法：`th:field="*{name}"`，用来绑定后台对象和表单数据。需要注意的是必须要配合`th:ojbect`使用。

##### `th:value`

用法如上面的`th:value="${ingredient.id}"`，替换原标签的`value`属性。

上面的`th:field`就相当于`th:name`加上`th:value`的效果。如

```html
<input type="text" name="name", th:value="${design.name}">
```

二者的效果是一致的。

##### `th:each`

遍历。如

```html
<div th:each="ingredient : ${wrap}">
	<input name="ingredient" type="checkbox" th:value="${ingredient.id}">
	<span th:text="${ingredient.name}"></span><br/>
</div>
```

这里的`${wrap}`是后台传递过来的一个数组，如果该数组中有三个元素，就会遍历生成三个这样的`div`。虽然很难说明，但是很容易理解。

##### `th:if`

如果`th:if`中的值为`true`，则该标签会显示，不然该标签不会显示。

##### 表单的提交

在上面的表单`form`标签中，并没有指明`action`属性，其表示还是发送原请求，也就是`/design`。但是这里指明了使用`post`的方式，一开始我们使用的是`get`的方式。这里就有点儿`Rest`风格的味道了，同一个`url`，发送`get`请求表示获取数据，发送`post`请求提交数据。这里我们需要在`DesignController`中新建方法处理这个`post`请求。

#### 处理`/design`的`post`请求

```java
@PostMapping
public String processDesign(Taco design) {
    // save the taco design in chapter 3

    log.info("Processing design: " + design);
    return "redirect:/orders/current";
}
```

因为第二章的重点没有数据持久化，所以这里没有保存数据，而是直接输出了。

上面还有一点需要注意的是，接收的参数是`Taco`类型的，这个之前在`SpringMVC`中也已经说过了，数据适配器`HandlerAdapter`将数组包装成了`Taco`。

最后返回的是`redirect:/order/current`，表示重定向到`/order/current`请求。

#### 处理`/order`

通过`/design`请求完成了选择购买什么样的`Taco`的任务，此时我们就需要进行提交订单处理。创建`OrderController`处理`/order`请求，形式和`DesignTacoController`是差不多的。

```java
@Slf4j
@Controller
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/current")
    public String orderForm(Model model) {
        model.addAttribute("order", new Order());
        return "orderForm";
    }

    @PostMapping
    public String processOrder(Order order) {

        log.info("Order submitter: " + order);
        return "redirect:/";
    }
}
```

也是通过`Get`方法填写订单数据，然后通过`Post`方法交由`processOrder`方法进行处理。

在`Get`方法中，我们添加了一个`Order`对象，然后转到`orderForm`视图。

#### `orderForm.html`

```html
<!DOCTYPE html>
<html lang="en"
    xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Taco Cloud</title>
    <style>
        label {
            display: inline-block;
            width: 90px;
            text-align: center;
        }
    </style>
</head>
<body>
    <form method="post" th:action="@{/orders}" th:object="${order}">
        <h1>Order your taco creations!</h1>

        <img th:src="@{/images/avator.jpg}" style="width: 100px;height: 100px;"/>
        <a th:href="@{/design}">Design another taco</a>

        <h3>Deliver my taco masterpieces to ...</h3>
        <label for="name">Name: </label>
        <input type="text" th:field="*{name}"/>

        <label for="street">Street: </label>
        <input type="text" th:field="*{street}"/>

        <label for="city">City: </label>
        <input type="text" th:field="*{city}"/>

        <label for="state">State: </label>
        <input type="text" th:field="*{state}"/>

        <label for="zip">Zip: </label>
        <input type="text" th:field="*{zip}"/>

        <h3>Here's how I'll pay ...</h3>

        <label for="ccNumber">Credit Card: </label>
        <input type="text" th:field="*{ccNumber}"/>

        <label for="ccExpiration">Expiration: </label>
        <input type="text" th:field="*{ccExpiration}"/>

        <label for="ccCVV">CVV: </label>
        <input type="text" th:field="*{ccCVV}"/>

        <input type="submit" value="submit order"/>
    </form>

</body>
</html>
```

在熟悉了`design.html`中的`thymeleaf`之后，这里的代码还是非常容易理解的，然后使用同样的方式`post`回订单数据，交由后台服务器处理。

### 校验表单数据

上面我们虽然完成了一个简单的`Taco`订购网站，但是有一点是我们需要考虑的，如果用户传来的数据是非常该怎么样，又或者说用户传来的数据的空的。这都是我们需要避免的。

检验表单数据可以在后端做也可以在前端做，我们熟悉的`JQuery`就有前端校验的工具，但是我们更倾向的是放在后端做。前端主要是做数据的展示，不要靠`Javascript`来做这些逻辑。比如说，用户一禁用`JavaScript`又或者是无法使用`Javascript`，我们的前端校验就无法进行。

要进行校验，要引入校验的依赖，这里只需要引入`Spring Boot`的`Validation`启动器，里面包含了基本上所有的校验需要使用到的库。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

#### 声明校验的规则

##### `Taco`

```java
package com.sher.tacos.entity;

import lombok.Data;


import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;
import java.util.List;

@Data
public class Taco {

    @NotNull
    @Size(min = 5, message = "Name must be at least 5 characters long")
    private String name;

    @NotNull
    @Size(min = 1, message = "You must choose at least 1 ingredient")
    private List<String> ingredient;
}
```

其中`@NotNull`注解，表示该属性的值不可以为`null`值。`@Size`注解是限定值的长度。

`message`属性用来设置出错的时候反馈的信息。

##### `Order`

```java
package com.sher.tacos.entity;

import lombok.Data;

import javax.validation.constraints.Digits;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.Pattern;

@Data
public class Order {

    @NotBlank(message = "Name is required")
    private String name;

    @NotBlank(message = "Street is required")
    private String street;

    @NotBlank(message = "City is required")
    private String city;

    @NotBlank(message = "State is required")
    private String state;

    @Pattern(regexp = "^\\d{6}$", message = "Invalid zip")
    private String zip;

//    @CreditCardNumber(message = "Not a credit card number")
    @Pattern(regexp = "^\\d{10}$", message = "Invalid creadit crad number")
    private String ccNumber;

    @Pattern(regexp = "^(0[1-9]|1[0-2])([\\/])([1-9][0-9])$",
            message = "Must be formatted MM/YY")
    private String ccExpiration;

    @Digits(integer = 3, fraction = 0, message = "InValid CVV")
    private String ccCVV;
}
```

这里的`@NotBlank`和`@NotNull`不一样，`@NotBlank`表示的是不能为空。比如说`"   "`这种都是空白的，但是不是`null`。`@Pattern`使用正则表达式进行匹配。`@DIgits`表示值一定要是数字，`integer`指定整数的最多位数，`fraction`指定小数的最多位数。

不过无论怎么匹配，使用正则表达式都是可以匹配的。

#### 表单绑定，执行校验

由于我们是后端的数据校验，所以校验只能放在前端将数据发送到后端，后端接收到数据这个步骤。也就是上面我们写的两个处理`post`请求的方法。我们需要将要校验的参数加上`@valid`注解。如下所示。

```java
@PostMapping
public String processDesign(@Valid Taco design, Errors errors) {
    // save the taco design in chapter 3
    if (errors.hasErrors()) {
        return "home";
    }

    log.info("Processing design: " + design);
    return "redirect:/orders/current";
}
```

```java
@PostMapping
public String processOrder(@Valid Order order, Errors errors) {
    if (errors.hasErrors()) {
        return "orderForm";
    }

    log.info("Order submitter: " + order);
    return "redirect:/";
}
```

参数中还多了一个`Errors`类型的参数，如果检验失败， `Errors`对象中会存放`Error`，此时`errors.hasErrors()`就会返回`true`，我们就无法进入下一步。

#### 前端展现校验错误

上面的校验都是后端进行的，就算是校验失败了，前端也看不到任何的错误提示信息。`Thymeleaf`提供了非常方便的访问`Errors`的方法，通过`fields`对象和`th:errors`。如

```html
<label for="ccNumber">Credit Card: </label>
<input type="text" th:field="*{ccNumber}"/>
<span th:if="${#fields.hasErrors('ccNumber')}"
      th:errors="*{ccNumber}">Error
</span><br/>
```

上面我们使用`$(#fields.hasErrors('ccNumber'))`，如果有这个名字的错误将返回`true`，否则返回`false`。然后配置`th:if`，如果有错误，使用`th:errors`得到错误的提示内容。（在`message`中指定的文字），然后显示。

`orderForm.html`添加展示校验错误的全部代码如下。

```html
<!DOCTYPE html>
<html lang="en"
    xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Taco Cloud</title>
    <style>
        label {
            display: inline-block;
            width: 90px;
            text-align: center;
        }
    </style>
</head>
<body>
    <form method="post" th:action="@{/orders}" th:object="${order}">
        <h1>Order your taco creations!</h1>

        <img th:src="@{/images/avator.jpg}" style="width: 100px;height: 100px;"/>
        <a th:href="@{/design}">Design another taco</a>

        <div th:if="${#fields.hasErrors()}">
            <span>Please correct the problem below and resubmit.</span>
        </div>

        <h3>Deliver my taco masterpieces to ...</h3>
        <label for="name">Name: </label>
        <input type="text" th:field="*{name}"/>
        <span th:if="${#fields.hasErrors('name')}"
              th:errors="*{name}">Error</span>
        <br/>

        <label for="street">Street: </label>
        <input type="text" th:field="*{street}"/>
        <span th:if="${#fields.hasErrors('street')}"
              th:errors="*{street}">Error</span>
        <br/>

        <label for="city">City: </label>
        <input type="text" th:field="*{city}"/>
        <span th:if="${#fields.hasErrors('city')}"
              th:errors="*{city}">Error</span>
        <br/>

        <label for="state">State: </label>
        <input type="text" th:field="*{state}"/>
        <span th:if="${#fields.hasErrors('state')}"
              th:errors="*{state}">Error</span>
        <br/>

        <label for="zip">Zip: </label>
        <input type="text" th:field="*{zip}"/>
        <span th:if="${#fields.hasErrors('zip')}"
              th:errors="*{zip}">Error</span>
        <br/>

        <h3>Here's how I'll pay ...</h3>

        <label for="ccNumber">Credit Card: </label>
        <input type="text" th:field="*{ccNumber}"/>
        <span th:if="${#fields.hasErrors('ccNumber')}"
                th:errors="*{ccNumber}">Error</span>
        <br/>

        <label for="ccExpiration">Expiration: </label>
        <input type="text" th:field="*{ccExpiration}"/>
        <span th:if="${#fields.hasErrors('ccExpiration')}"
              th:errors="*{ccExpiration}">Error</span>
        <br/>

        <label for="ccCVV">CVV: </label>
        <input type="text" th:field="*{ccCVV}"/>
        <span th:if="${#fields.hasErrors('ccCVV')}"
              th:errors="*{ccCVV}">Error</span>
        <br/>

        <input type="submit" value="submit order"/>
    </form>

</body>
</html>
```







