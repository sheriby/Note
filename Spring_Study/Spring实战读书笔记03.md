# Spring实战读书笔记03

## 使用数据，JdbcTemplate

在第二章中我们完成了一个简单的`Web`应用，但是还有一个很严重的问题没有处理。那就是我们没有数据库，无法从数据库中获取数据，无法向数据库中储存数据。

之前学`Spring`的时候，其实就已经学到了`Spring`中对`JDBC`的一个封装，那就是`JdbcTemplate`。所以这里就将使用`JdbcTemplate`和`Mysql`搭建网站的数据库。

### 引入依赖

首先我们需要引入`Spring Boot`中的`jdbc started`。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

然后因为我们需要操作的数据库是`Mysql`，因此还需要引入`Mysql`的驱动。

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

之前我们连接数据库的都是都需要一个连接池，`DataSource`，一般采用的是`Druid`。在`Spring Boot`中默认使用的连接池是`Hikari`，其性能高过其他的数据库连接池，所以这里也不更换成`Druid`了。

### 配置连接池

之前我们都是使用`jdbc.properties`这种方式，然后要么是使用`xml`配置连接池，要么是使用配置类配置。在`Spring Boot`中，我们只需要在`application.yml`中配置就可以了。

```yml
spring:
  datasource:
    url: jdbc:mysql:///springinaction
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.zaxxer.hikari.HikariDataSource
```

在`Spring Boot`中默认使用`Mysql`驱动和`Hikari`连接池，所以后面的两行我们可以省略掉。

```yml
spring:
  datasource:
    url: jdbc:mysql:///springinaction
    username: root
    password: root
```

然后我们需要想的就是将`DataSource`和`JdbcTemplate`注入到容器当中，然后将`DataSource`注入到`JdbcTemplate`当中，这里是否是要在`Spring Boot`配置类中做呢？其实不用，因为上面的这些事情，`Spring Boot`都已经自动的帮我们完成了。我们只需要在需要使用的时候`@Autowired`就行了。

### 修改`Entity`

之前的两个`Entity`，也就是`Taco`和`Order`中，甚至没有`id`属性，不适合持久化，这里我们需要对其进行一点修改。

#### `Taco`

```java
@Data
public class Taco {

    private Integer id;
    private Date createdAt;

    @NotNull
    @Size(min = 5, message = "Name must be at least 5 characters long")
    private String name;

    @NotNull
    @Size(min = 1, message = "You must choose at least 1 ingredient")
    private List<String> ingredient;
}
```

这里还添加了创建`Taco`的时间这个属性。

#### `Order`

```java
@Data
public class Order {

    private Integer id;
    private Date placedAt;

    private List<Taco> tacos = new ArrayList<>();

    public void addDesign(Taco taco) {
        tacos.add(taco);
    }

    @NotBlank(message = "Name is required")
    private String name;

    // .....
}
```

这里也同样的添加了`id`和创建的时间这个属性。不过还添加了存放一个`Order`中所点的`Taco`的`List`。因为一个`Order`中可以点多个`Taco`，所以这里用`List`。

### `IngredientRepo`

之前我们使用`Ingredient`是直接创建一个`List`，然后向其中存放数据的。这里我们将把数据放在`Mysql`中，然后从其中获取数据。

首先创建操作数据库的接口。

```java
package com.sher.tacos.repository;

import com.sher.tacos.entity.Ingredient;

public interface IngredientRepository {

    Iterable<Ingredient> findAll();

    Ingredient findOne(String id);

    Ingredient save(Ingredient ingredient);
}
```

`findAll`是gongzuo返回所有的`Ingredient`， `findOne`根据`id`返回某一个`Ingredient`， `save`是存放一个`Ingredient.`

然后创建接口的实现类。

```java
@Repository
public class JdbcIngredientRepository implements IngredientRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public JdbcIngredientRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Iterable<Ingredient> findAll() {
        String sql = "select id, name, type from Ingredient";
        return jdbcTemplate.query(sql, this::mapRowToIngredient);
    }

    @Override
    public Ingredient findOne(String id) {
        String sql = "select id, name, type from Ingredient where id = ?";
        return jdbcTemplate.queryForObject(sql, this::mapRowToIngredient, id);
    }

    @Override
    public Ingredient save(Ingredient ingredient) {
        String sql = "insert into Ingredient (id, name, type) values (?, ?, ?)";
        jdbcTemplate.update(sql, ingredient.getId(), ingredient.getName(),
                ingredient.getType().toString());
        return ingredient;
    }

    private Ingredient mapRowToIngredient(ResultSet resultSet, int rowNum) throws SQLException {
        return new Ingredient(
                resultSet.getString("id"),
                resultSet.getString("name"),
                Ingredient.Type.valueOf(resultSet.getString("type"))
        );
    }
}
```

这里使用的注解是`@Reposity`注解，因为该类是持久化层的。

需要注意的是，这里没有使用

```java
@Autowired
private final JdbcTemplate jdbcTemplate;
```

这种注入方式。虽然也是可以的，而且很简洁，但是官方不推荐这种注入方式，因为很容易导致依赖关系混乱。

对于强制的依赖，我们可以使用构造器的方式注入。对于非强制的依赖，也就是可有可无的依赖，我们使用`setter`的方式注入。

使用构造器的方式注入。

```java
@Autowired
public JdbcIngredientRepository(JdbcTemplate jdbcTemplate) {
    this.jdbcTemplate = jdbcTemplate;
}
```

使用`setter`的方式注入。

```java
@Autowired
public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
    this.jdbcTemplate = jdbcTemplate
}
```

后面我们尽量少的使用`field`的方式注入依赖。

上面的实现的方式都易懂，就是给定一个`sql`语句，然后使用`JdbcTemplate`来执行，但是有一点需要注意的是查询方法的第二个参数`this::mapRowToIngredient`，这是一个方法引用，也可以写成`lambda`表达式的方式。

之前我们想要返回一个对象，第二个参数写的参数都是`BeanPropertyRowMapper<>(xxx.class)`的方式，那么这里为啥要手动写呢？

因为`BeanPropertyRowMapper`绑定都是一一赋值的方式，比如数据库表中有`id name age`，对象中也有`id name age`，于是就讲表中的值赋值个对象中的值。但是需要注意的是，这里的`Ingredient`中的`type`属性是`Type`类型的，但是数据库中的只能是`String`类型的，所以就需要我们手动的进行包装，将`Spring`转成`Type`类型的。

上面使用方法引用是很好的写法，当然我们也可以使用`lambda`表达式。

```java
RowMapper<Ingredient> rowToIngredient = (rs, n) -> {
    return new Ingredient(
        resultSet.getString("id"),
        resultSet.getString("name"),
        Ingredient.Type.valueOf(resultSet.getString("type"))
    );
}

@Override
public Iterable<Ingredient> findAll() {
    String sql = "select id, name, type from Ingredient";
    return jdbcTemplate.query(sql, rowToIngredient;
}
```

当然，如果不使用`java8`的话，可以使用匿名内部类来做。这里不想写那么赘余的代码。

### `DesignController`中加载`Ingredient`

上面我们没有提如何创建数据库的，因为比较简单，无非是一些`create`，然后添加一些中间表，在中间表中添加外键，下面使用到表的时候会稍微说明。

```java
@Slf4j
@Controller
@RequestMapping("/design")
public class DesignTacoController {

    private final IngredientRepository ingredientRepo;

    @ModelAttribute(name = "design")
    public Taco taco() {
        return new Taco();
    }

    @Autowired
    public DesignTacoController(
            IngredientRepository ingredientRepo) {
        this.ingredientRepo = ingredientRepo;
    }

    @GetMapping
    public String showDesignForm(Model model) {
        List<Ingredient> ingredients = new ArrayList<>();
        ingredientRepo.findAll().forEach(ingredients::add);


        Type[] types = Type.values();
        for (Type type : types) {
            model.addAttribute(type.toString().toLowerCase(), filterByType(ingredients, type));
        }

        return "design";
    }
    
    // ......
}

```

我们讲`IngredientReposity`注入到该控制器中，这里也是采用的构造器注入的方式。然后使用`findAll`方法获取数据。这里的

```java
ingredientRepo.findAll().forEach(ingredients::add);
```

也使用到了`java8`新特性，不过都已经`2020`年了，不会连`java8`还算新吧，不会吧，不会吧？上面使用`Iterator`接口的`forEach`方法对其中的所有的数据进行迭代，然后使用`ingredients::add`方法引用将其加入到上面创建的`ingredients`中。这个也可以换成`lambda`表达式的形式。

```java
ingredientRepo.findAll().forEach(i -> ingredients.add(i));
```

对于既有方法来说，`lambda`表达式就没有方法引用那么简洁了。

还有一点我们需要注意的是，我们去掉了`showDesignForm`方法中的

```java
model.addAttribute("design", new Order());
```

转而添加了

```java
@ModelAttribute(name = "design")
public Order order() {
    return new Order();
}
```

上面的`@ModelAttribute(name = "design")`也可以改成`@ModelAttribute(value = "design")`，或者直接`@ModelAttribute("design")`。

在`SpringMVC`中我们学到了，`@ModelAttribute`和`Model`都可以将对象添加了`request`中，但是`@ModelAttribute`是对类中的所有的业务方法都生效的。

此时我们已经可以完美的访问`/design`页面了，但是我们还没有保存`Taco`和`Order`，所以还需要做一些工作。

### `TacoReposity`

首先还是创建一个接口。

```java
package com.sher.tacos.repository;

import com.sher.tacos.entity.Taco;

public interface TacoReposity {

    Taco save(Taco design);
}
```

只有一个方法，那就是保存一个`Taco`。然后创建实现类。

```java
@Repository
public class JdbcTacoRepository implements TacoReposity {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public JdbcTacoRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Taco save(Taco taco) {
        int tacoId = saveTacoInfo(taco);
        taco.setId(tacoId);

        for (String ingredient : taco.getIngredient()) {
            saveIngredientToTaco(ingredient, tacoId);
        }

        return taco;
    }

    private int saveTacoInfo(Taco taco) {
        taco.setCreatedAt(new Date());
        String sql = "insert into Taco (name, createdAt) values (?, ?)";
        Timestamp timestamp = new Timestamp(taco.getCreatedAt().getTime());
        jdbcTemplate.update(sql, taco.getName(), timestamp);
        String sql2 = "select id from Taco where name = ? and createdAt = ?";
        return jdbcTemplate.queryForObject(sql2, int.class, taco.getName(), timestamp);
    }

    private void saveIngredientToTaco(String ingredient, int tacoId) {
        String sql = "insert into Taco_Ingredients (taco, ingredient) values (?, ?)";
        jdbcTemplate.update(sql, tacoId, ingredient);
    }
}
```

同样地使用构造器将`JdbcTemplate`实例注入进来。

在`save`方法中，首先调用的是`saveTacoInfo`的方法，该方法将我们`design`的`Taco`添加到`Taco`表中。

在`saveTacoInfo`方法中，需要注意的是，此时的`Taco`对象中的`id`和`createdAt`属性都是`null`，所以我们先将`createdAt`属性复制。至于`id`这个值，因为在表中我们设置了主键自增`auto_increment`，所以我们不必添加。但是我们还是要获取这个`id`的值，所以又使用了一个`select`语句获取这个`id`。

然后后面通过`saveIngredientToTaco`方法将`Taco`中的`ingredients`添加到`Taco_Ingredients`这个表中。这个表中的两个字段是`Taco`和`Ingredient`的外键。

### `DesignController`中保存`Taco`

```java
@Slf4j
@Controller
@RequestMapping("/design")
@SessionAttributes("order")
public class DesignTacoController {

    private final IngredientRepository ingredientRepo;
    private final TacoReposity tacoReposity;

    @ModelAttribute(name = "order")
    public Order order() {
        return new Order();
    }

    @Autowired
    public DesignTacoController(
            IngredientRepository ingredientRepo,
            TacoReposity tacoReposity) {
        this.ingredientRepo = ingredientRepo;
        this.tacoReposity = tacoReposity;
    }

    // ...

    @PostMapping
    public String processDesign(@Valid Taco design, Errors errors,
                                @ModelAttribute Order order) {
        if (errors.hasErrors()) {
            return "home";
        }

        log.info("Processing design: " + design);
        Taco saved = tacoReposity.save(design);
        order.addDesign(saved);
        log.info("Order Tacos" + order.getTacos());

        return "redirect:/orders/current";
    }
}
```

同样的注入不必多说。这里我们添加了

```java
@ModelAttribute(name = "order")
public Order order() {
    return new Order();
}
```

还有就是`@SessionAttribute("order")`，同时将这个`Order`对象添加到`session`中。然后就是`processDesign`方法参数上面的`@ModelAttribute`注解，这个注解添加的目的是确保不要用请求参数中获取包装。

为什么要将`Order`对象放在`Session`当中呢？因为一个用户也就是一个订单中可能会包含多个`Taco`也就是说访问了`/design`多次，此时我们只能将`Order`放在`Session`中，因为我们需要多个请求共享这个数据。

此时我们已经可以将定做的`Taco`存放在数据库中了，还有一个`Order`没有进行处理。

### `OrderRepository`

创建接口类

```java
package com.sher.tacos.repository;

import com.sher.tacos.entity.Order;

public interface OrderRepository {

    Order save(Order order);
}
```

实现接口类

```java
@Repository
public class JdbcOrderRepository implements OrderRepository {

    private final SimpleJdbcInsert orderInsert;
    private final SimpleJdbcInsert orderTacoInsert;
    private final ObjectMapper objectMapper;

    @Autowired
    public JdbcOrderRepository(JdbcTemplate jdbcTemplate) {
        orderInsert = new SimpleJdbcInsert(jdbcTemplate).
                withTableName("Taco_Order").usingGeneratedKeyColumns("id");
        orderTacoInsert = new SimpleJdbcInsert(jdbcTemplate).withTableName("Taco_Order_Tacos");
        objectMapper = new ObjectMapper();
    }


    @Override
    public Order save(Order order) {
        order.setPlacedAt(new Date());
        int orderId = saveOrderDetails(order);
        order.setId(orderId);

        List<Taco> tacos = order.getTacos();
        for (Taco taco : tacos) {
            saveTacoToOrder(taco, orderId);
        }

        return order;
    }

    private int saveOrderDetails(Order order) {
        Map<String, Object> values = objectMapper.convertValue(order, Map.class);
        values.put("placedAt", order.getPlacedAt());

        return orderInsert.executeAndReturnKey(values).intValue();
    }

    private void saveTacoToOrder(Taco taco, int orderId) {
        Map<String, Object> values = new HashMap<>();
        values.put("tacoOrder", orderId);
        values.put("taco", taco.getId());
        orderTacoInsert.execute(values);
    }
}
```

`Order`中有太多的属性，所以这里我们没有使用普通的方法添加，而是使用`SimpleJdbcInsert`来帮助我们处理。

首先通过`JdbcTemplate`生成我们需要处理的两个表的`SimpleJdbcInsert`。

在方法`saveOrderDetails`中，我们使用到了`Jackson`提供的`ObjectMapper`，一般是用来进行`Json`的处理。不过这里我们使用他将一个类的属性变成`Map`的形式，然后使用`orderInsert`插入。比如`{"name": "sher", "age": 15}`这种的`Map`使用`SimpleJdbcInsert`就相当于使用了`insert into xxx (name, age) values('sher', 15)`。

在`orderInsert`的定义的后面还加上了`usingGeneratedKeyColumns("id")`，他可以返回自动生成的`id`。

### `OrderController`中保存`Order`

这是我们要做的最后一步了，在控制器中处理`Order`保存的逻辑。

```java
@Slf4j
@Controller
@RequestMapping("/orders")
@SessionAttributes("order")
public class OrderController {

    private final OrderRepository orderRepository;

    @Autowired
    public OrderController(JdbcOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping("/current")
    public String orderForm(Model model) {
        return "orderForm";
    }

    @PostMapping
    public String processOrder(@Valid Order order, Errors errors,
                               SessionStatus sessionStatus) {
        if (errors.hasErrors()) {
            return "orderForm";
        }

        log.info("Order submitter: " + order);
        orderRepository.save(order);
        sessionStatus.setComplete();
        return "redirect:/";
    }
}
```

同样是`@SessionAttribute("order")`确保`Order`对象被放入到`Session`中。其次是通过构造器的形式将`OrderRepository`注入到其中，因为这是强制依赖。

变的方法就只有`processOrder`了，其方法参数中添加了`SessionStatus`，用来后面对`Session`的重置。当我们已经保存了`order`之后，当前的`Order`对象就没有存在的必要了，所以最后我们调用`sessionStatus.setComplete()`重置`Session`中的内容。





