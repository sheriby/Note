# SpringBoot&MyBatis

## 使用`Spring Boot`整合`MyBatis`

之前我们已经使用过注解版的`Spring`整合`MyBatis`了，这里使用`Spring Boot`基本上也是一样的。这里我们就将之前的`SpringInAction`的`Taco Cloud`项目中的持久化技术从`JdbcTemplate`换成`MyBatis`。

### 引入依赖

只需要引入`MyBatis`的`starter`就行了。

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.3</version>
</dependency>
```

### 配置`MyBatis`

在`application.yml`中我们已经配置好了数据库连接的各种配置，`DataSource`也会自动注入到容器当中。有了`MyBatis`的`starter`，`SqlSessionFactoryBean`也会自动注入到容器当中。也就是说我们只需要写`Mapper`，要用的时候直接从容器中获取就行了，真的是非常的方便。

#### 配置`MapperScan`

在`Spring Boot`的引导类添加`MapperScan`的注解指定需要扫描的包。

```java
@SpringBootApplication
@MapperScan(basePackages = "com.sher.tacos.repository", annotationClass = Mapper.class)
public class TacoCloudApplication implements WebMvcConfigurer {

    public static void main(String[] args) {
        SpringApplication.run(TacoCloudApplication.class, args);
    }
    ...
}
```

#### 开启`MyBatis`的`SQL`语句打印功能

在调试的时候，需要看看`SQL`语句的执行情况。在`application.yml`中添加。

```yml
mybatis:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

### 编写`Repository`

这里的`Mapper`，很多时候还被称为`Repository`或者`DAO`什么的。就像上面的`Entity`实体类还有时候被称为`Bean Pojo Demain`什么什么的。按照自己习惯的来就好了，名字什么的无所谓。

#### `IngredientRepository`

```java
@Mapper
@Repository
public interface IngredientRepository {

    @Select("select * from Ingredient")
    List<Ingredient> findAll();

    @Select("select * from Ingredient where id = #{id}")
    Ingredient findOne(String id);

    @Insert("insert into Ingredient values(#{id}, #{name}, #{type})")
    int save(Ingredient ingredient);
}
```

这里我们给接口添加了一个`@Repository`注解，其实这个添加不添加效果都是一样。不添加的话，运行也不会报错（在`Spring`注解版的时候我们没有添加），但是`Idea`会提醒我们无法`Autowired`，为了消除这个误报，我才添加了这个注解。

返回的结果可以自动的封装，真的是非常的智能。看第三个方法，`Ingredient`的`type`属性是一个`Ingredient.Type`类型的，但是这里会自动的转换为`String`类型，也就是自动调用了`toString`方法。

#### `UserRepository`

```java
@Mapper
@Repository
public interface UserRepository {

    @Select("select * from User where username = #{username}")
    User findByUsername(String username);

    @Insert("insert into User (username, password, fullname, street, city, state, zip, phoneNumber)" +
            "values (#{username}, #{password}, #{fullname}, #{street}, #{city}, #{state}, #{zip}, #{phoneNumber})")
    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    int save(User user);
}
```

这里的`@Insert`虽然长，但是并不复杂，这是插入的属性多了点。这种情况我也不太清楚如何省略。

主要的看点是`@Options`注解，在`User`表中，我们设置了主键自增，所以我们没有插入主键`id`。使用

```java
@Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
```

可以将主键`id`返还给对象的`id`属性。

#### `TacoRepository`

```java
@Mapper
@Repository
public interface TacoRepository {

    @Insert("insert into Taco (name, createdAt) values(#{name}, #{createdAt})")
    @Options(useGeneratedKeys = true, keyColumn = "id", keyProperty = "id")
    int save(Taco design);


    @Insert("<script>" +
            "insert into Taco_Ingredients (taco, ingredient) values " +
            "<foreach collection='ingredient' item='inid' separator=','>" +
            "(#{id}, #{inid})" +
            "</foreach></script>")
    int saveIngredient(Taco design);
}
```

这里将原本的一个`save`方法拆分成了两个方法。因为一个方法只可以执行一条`SQL`语句。在原本的方法的内容，调用了多条`SQL`语句操作了多条表，我认为是不合理的。

那么该如何调用方法呢？难道是按顺序调用这两条方法。其实不然，我们需要编写`Service`层来操作`Repository`，我们在`Controller`或者其他地方的时候注入`Service`而不是`Repository`。这也是一个很基本的结构——`@Controller <- @Service <- @Repository`。在书中忽略了`Service`，其实是不应该的。

回到第二个`@Insert`中，使用`<script>`为了使用动态`SQL`，也就是后面需要使用的`<foreach>`，`Taco`对象的`ingredient`是`List<String>`类型的，添加的需要遍历。其生成的语句类似如下。

```sql
insert into xx (xx, xx) values (xx, xx), (xx, xx), (xx, xx)
```

#### `OrderRepository`

```java
@Mapper
@Repository
public interface OrderRepository {

    @Insert("insert into Taco_Order (name, street, city, state, zip, ccNumber, " +
            "ccExpiration, ccCvv, placedAt) values(" +
            "#{name}, #{street}, #{city}, #{state}, #{zip}, #{ccNumber}, " +
            "#{ccExpiration}, #{ccCVV}, #{placedAt})")
    @Options(useGeneratedKeys = true, keyColumn = "id", keyProperty = "id")
    int save(Order order);

    @Insert("<script>" +
            "insert into Taco_Order_Tacos (tacoOrder, taco) values " +
            "<foreach collection='tacos' item='taco' separator=','>" +
            "(#{id}, #{taco.id})" +
            "</foreach></script>")
    int saveTaco(Order order);
}
```

同样将一个方法分成了两个方法，同样的内容就叭说了。唯一需要看看的就是第二个`@Insert`中，我们也使用了动态`SQL`的`<foreach>`，不过这次遍历的是`List<Taco>`，得到的`item`也是对象，获取对象的属性直接通过`xx.xx`的方式。也就是说，如果属性是一个对象，想要获取属性对象的属性，可以通过`属性.属性`的方式。

### 编写`Service`

#### `IngredientService`

```java
@Service
public class IngredientService {

    private final IngredientRepository ingredientRepository;

    @Autowired
    public IngredientService(IngredientRepository ingredientRepository) {
        this.ingredientRepository = ingredientRepository;
    }

    public List<Ingredient> findAll() {
        return ingredientRepository.findAll();
    }

    public Ingredient findOneById(String id) {
        return ingredientRepository.findOne(id);
    }

    public Ingredient saveIngredient(Ingredient ingredient) {
        ingredientRepository.save(ingredient);
        return ingredient;
    }
}
```

#### `UserService`

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findUserByName(String name) {
        return userRepository.findByUsername(name);
    }

    public User saveUser(User user) {
        userRepository.save(user);
        return user;
    }
}
```

#### `TacoService`

```java
@Service
public class TacoService {

    private final TacoRepository tacoRepository;

    @Autowired
    public TacoService(TacoRepository tacoRepository) {
        this.tacoRepository = tacoRepository;
    }

    public Taco saveTacoDesign(Taco taco) {
        taco.setCreatedAt(new Date());
        tacoRepository.save(taco);
        tacoRepository.saveIngredient(taco);

        return taco;
    }
}
```

#### `OrderService`

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order saveOrder(Order order) {
        order.setPlacedAt(new Date());
        orderRepository.save(order);
        orderRepository.saveTaco(order);

        return order;
    }
}
```

### 编写测试

现在我们就可以直接删除之前写的那些`Jdbc`的`impl`了。将注入`Repository`换成注入`Service`，然后修改一下方法名字，项目就可以完美的运行起来。

但是我们肯定是写一步测试一步，不可能是直接写完跑的，不然发生问题调试起来会很难搞。使用`Spring Boot`的测试，要比`Spring`难搞一点。

首先引入`spring-boot-starter-test`

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

编写`Test`类做测试，如

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class MapperTest {
    
    @Autowired
    IngredientRepository ingredientRepository;

    @Autowired
    UserRepository userRepository;

    @Autowired
    TacoRepository tacoRepository;

    @Autowired
    OrderRepository orderRepository;

    @Test
    public void testIngredient() {
        System.out.println(ingredientRepository.findAll());
        Ingredient ingredient = new Ingredient("abc", "sher", Ingredient.Type.PROTEIN);
        int save = ingredientRepository.save(ingredient);
        System.out.println(save);
    }
    
    .......
}
```

需要添加上面的两个测试用的注解，测试用的组件直接注入就好了。

至此我们就将`Taco Cloud`的持久化层技术换成了`MyBatis`，简洁了不少呢。