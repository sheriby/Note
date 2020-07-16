# Spring实战读书笔记04

## 保护Spring

在前一章当中还有一点介绍`Spring Data JPA`的内容，在本章中也有使用。但是我认为这种`ORM`的框架应该是不成气候，现在不了解也罢，还是直接使用`JdbcTemplate`吧，我感觉也蛮方便的。

在本章中主要讨论的是网站的安全问题，已经登录等，使用的主要是`Spring Security`。

### 启用`Spring Security`

很简单，只需要引入相关的依赖就行了。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

不过引入这个依赖我也遇到了点问题，他一直告诉我说阿里云的仓库找不到这个`jar`，但是我的阿里云仓库是根据网上配的，应该没问题啊。后来查阅了一点资料，换了个配置，才成功导入。

```xml
<mirror>
    <id>nexus-aliyun</id>
    <mirrorOf>*</mirrorOf>
    <name>Nexus aliyun</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror> 
```

之前的配置是这样的，在大部分情况下都是可用的，个别不可用不知道为啥。

```xml
<mirror>
    <id>alimaven</id>
    <mirrorOf>central</mirrorOf>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
</mirror>
```

将`Spring Security`导入到项目中之后，由于`Spring Boot`的自动配置，我们不需要写一行代码，就可以使用一些初始化的安全配置。

此时我们登录网站的任意一个`url`的时候，都会跳出一个`http basic`认证的对话框提示我们进行认证。默认的用户名为`user`，默认的密码会在启动的时候输出在控制台。

这个默认的安全配置实在是太简陋，不符合我们的期望，因此我们需要显示地进行配置。我们希望实现如下的功能。

-   登录界面不要使用`http basic`对话框，使用我们自定义的界面。
-   提供多个用户，以及注册界面。
-   对不同的请求，执行不同的安全规则。如，主页和注册页面不需要进行安全认证。

### 配置`Spring Security`

书的前面介绍了什么基于内存，基于`JDBC`，基于`LDAP`的用户储存，不过是简单介绍罢了，后面也不会使用这些简单的方式。我们的用户存储一定是基于数据库自定义的。这里简单的介绍一下基于内容的用户存储。

#### 基于内存的用户存储

配置`Spring Security`需要编写配置类，可以选择覆盖`WebSecurityConfigurerAdapter`中的`configure`方法进行配置。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception 		 {
        auth.inMemoryAuthentication()
                    .withUser("sher")
                    .password("sher")
                    .authorities("ROLE_USER")
                .and()
                    .withUser("hony")
                    .password("hony")
                    .authorities("ROLE_USER");
    }
}
```

在该类上需要添加两个注解，其中`@Configuration`是使其成为`Spring`的配置类，`@EnableWebSecurity`开启`Web`安全。具体作用要深入底层，简单的来说要想成为`Spring Security`的配置就需要加上这两个注解。

在`configure`方法中，基于内存，我们两天了两个用户以及设置了他们的密码。此时重启项目我们就可以使用这两个账户进行登录了。

可以看出来这种方式真的是太简陋了，所以也就是简单介绍介绍。

### 自定义用户认证

#### 创建实体`User`

我们第一步需要做的是，创建用户实体，然后在数据库创建对应的表。因为我们需要使用到这些数据进行用户认证。

```java
@Data
@RequiredArgsConstructor
@NoArgsConstructor(access = AccessLevel.PRIVATE, force = true)
public class User implements UserDetails {
    private static final long serialVersionUID = 1l;

    private Integer id;

    private final String username;
    private final String password;
    private final String fullname;
    private final String street;
    private final String city;
    private final String state;
    private final String zip;
    private final String phoneNumber;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"));
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

和普通的实体类不同的是，这个类实现了`UserDetail`接口。接口中除了需要实现下面的五个方法之后，还需要实现`getUsername`和`getPassword`方法，只不过这些方法我们使用`lombok`实现了。

通过实现`UserDetail`方法，我们可以暴露更多的信息给框架，如后面的各种`is`方法，表示的是用户是否可用和过期等。`getAuthorities`方法返回用户可以被授予权限的集合，这里我们只选择了普通用户，或许后面会添加管理员用户，用于添加配料之类的。

#### `UserRepository`

然后我们就需要做数据的持久化工作了，和第三章使用方法一样。

先是创建相应的接口。

```java
package com.sher.tacos.repository;

import com.sher.tacos.entity.User;

public interface UserRepository {
    User findByUsername(String username);

    User save(User user);
}
```

这里暂时就写了两个方法，因为其余的方法还没有用到。等用到了再写也不迟。

实现该接口。

```java
@Repository
public class JdbcUserRepository implements UserRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public JdbcUserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    RowMapper<User> userRowMapper = (rs, n) -> {
        return new User(
                rs.getString("username"),
                rs.getString("password"),
                rs.getString("fullname"),
                rs.getString("street"),
                rs.getString("city"),
                rs.getString("state"),
                rs.getString("zip"),
                rs.getString("phoneNumber")
        );
    };

    @Override
    public User findByUsername(String username) {
        String sql = "select * from User where username = ?";
        return jdbcTemplate.queryForObject(sql, userRowMapper, username);
    }

    @Override
    public User save(User user) {
        System.out.println(user);
        String sql = "insert into User (username, password, fullname, street, city, state, zip, phoneNumber) " +
                "values (?, ?, ?, ?, ?, ?, ?, ?)";
        jdbcTemplate.update(sql, user.getUsername(), user.getPassword(), 
                            user.getFullname(), user.getStreet(), 
                            user.getCity(), user.getState(),
                            user.getZip(), user.getPhoneNumber());
        return user;
    }
}
```

还是熟悉的配方，熟悉的味道。这里本可以使用`SimpleJdbcInsert`来做的，但是想了想似乎没什么必要。

这里专门地实现了`findByUsername`方法，是要在后面的服务中使用。

#### 创建用户详情服务

这个服务也是交由框架使用，在我们登录的时候，框架通过登录的用户名获取到`UserDetail`对象，然后使用`UserDetail`对象的`getPassword`方法就可以得到密码，这些操作就需要通过此服务。

```java
@Service
public class UserRepoUserDetailService implements UserDetailsService {

    private final UserRepository userRepository;

    @Autowired
    public UserRepoUserDetailService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(s);
        System.out.println(user);
        if (user != null) {
            return user;
        }
        throw new UsernameNotFoundException("User " + s + " Not Found!");
    }
}
```

只需要实现一个很简单的方法，注入`UserRepository`就可以轻松实现。

我们写了这样的一个服务类，当然还是需要让框架知道的，只需要在`configure`方法中配置一下就可以了。

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userDetailsService);
}
```

为了进一步确保数据的安全，我们在数据中存放的密码不能是明文的，需要使用一定的方式进行加密处理，此时就需要使用一个`PasswordEncoder`。

首先在容器中添加一个`PasswordEncoder`，这里使用`Spring`官方推荐使用的`BCryptPasswordEncoder`。

```java
@Bean
PasswordEncoder encoder() {
    return new BCryptPasswordEncoder();
}
```

然后在`configure`方法中配置。

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    PasswordEncoder encoder = encoder();

    auth.userDetailsService(userDetailsService)
        .passwordEncoder(encoder);
}
```

### 用户注册

#### `RegistrationController`

这时候需要使用第二章学到的`Spring MVC`的相关知识。首先，我们需要编写控制器，处理视图逻辑。

```java
@Controller
@RequestMapping("/register")
public class RegistrationController {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Autowired
    public RegistrationController(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @GetMapping
    public String registerForm() {
        return "registration";
    }

    @PostMapping
    public String processRegistration(RegistrationForm registrationForm) {
        userRepository.save(registrationForm.toUser(passwordEncoder));

        return "redirect:/login";
    }
}
```

使用`Get`方法跳转到`registration.html`中注册，然后使用`Post`方法回传注册的数据。

#### `registration.html`

`registration.html`内容如下，非常简单，毕竟我们不是做前端的。

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
            margin-top: 10px;
        }

        form input:last-child{
            margin-left: 90px;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>Registration</h1>
    <img th:src="@{/images/avator.jpg}" style="width: 100px;height: 100px;"/>

    <form method="post" th:action="@{/register}">
        <label for="username">Username: </label>
        <input type="text" name="username" id="username"/><br/>

        <label for="password">Password: </label>
        <input type="password" name="password" id="password"/><br/>

        <label for="confirm">Confirm Password: </label>
        <input type="password" name="confirm" id="confirm"/><br/>

        <label for="fullname">Fullname: </label>
        <input type="text" name="fullname" id="fullname"/><br/>

        <label for="street">Street: </label>
        <input type="text" name="street" id="street"/><br/>

        <label for="city">City: </label>
        <input type="text" name="city" id="city"/><br/>

        <label for="state">State: </label>
        <input type="text" name="state" id="state"/><br/>

        <label for="zip">Zip: </label>
        <input type="text" name="zip" id="zip"/><br/>

        <label for="phone">Phone: </label>
        <input type="text" name="phone" id="phone"/><br/>

        <input type="submit" value="Register">
    </form>

</body>
</html>
```

当使用`Post`方法回传的时候，我们将其包装成一个`RegisterForm`对象，方便使用。

```java
package com.sher.tacos.entity;

import lombok.Data;
import org.springframework.security.crypto.password.PasswordEncoder;

@Data
public class RegistrationForm {

    private String username;
    private String password;
    private String fullname;
    private String street;
    private String city;
    private String state;
    private String zip;
    private String phone;

    public User toUser(PasswordEncoder passwordEncoder) {
        return new User(username, passwordEncoder.encode(password),
                fullname, street, city, state, zip, phone);
    }

}
```

其中的`toUser`方法是根据注册的表单提供的信息，封装成一个`User`对象，其中最主要的就是通过`PasswordEncoder`将密码加密。

注册成功之后，我们就重定向到`/login`。

在`Spring Boot`的引导类我们对其进行了配置。

```java
@Override
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("home");
    registry.addViewController("/login").setViewName("login");
}
```

#### `login.html`

```html
<!DOCTYPE html>
<html lang="en"
    xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Taco Cloud</title>
</head>
<body>
    <h1>Login</h1>
    <img th:src="@{/images/avator.jpg}" style="width: 100px;height: 100px;"/>

    <div th:if="${Error}">
        Unable to login. Check your username and password.
    </div>

    <p>
        New here? Click <a th:href="@{/register}">here</a> to register.
    </p>

    <form method="post" th:action="@{/login}">
        <label for="username">UserName: </label>
        <input type="text" name="username" id="username"/><br/>

        <label for="password">Password: </label>
        <input type="password" name="password" id="password"/><br/>

        <input type="submit" value="Login">
    </form>

</body>
</html>
```

### 设置拦截请求

如果我们现在登录界面，会发现所有的页面都会被拦截，包括主页和登录页面。我们需要设置这些请求没有登录也可以访问。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/design/**", "/orders/**")
        .hasRole("USER")
        .antMatchers("/", "/**")
        .permitAll()
}
```

同样也是`configure`方法，但是这里的参数是`HttpSecurity`，是处理`Http`请求的重载方法。在上面，我们设置了除了`/design/**`和`/orders/**`这些请求是需要登录才可以访问的，其余的请求都无需登录就可以访问。

### 设置自定义登录界面

虽然我们编写了自定义的登录界面`login.html`但是并没有告诉`Spring`，这个也需要在`configure`方法中进行配置。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/design/**", "/orders/**")
        .hasRole("USER")
        .antMatchers("/", "/**")
        .permitAll()
        .and()
        .formLogin()
        .loginPage("/login")
        .defaultSuccessUrl("/design")
        .and()
        .logout()
        .logoutSuccessUrl("/");
}
```

通过上面的设置之后，`Spring Security`会监听`/login`请求，并且预期用户名和密码的属性名称为`username`和`password`。当然，这些都是可以配置的，只不过遵守框架的约定是最好的。

```java
.formLogin()
.loginPage("/login")
.loginProcessingUrl("/authenticate")
.usernameParameter("user")
.passwordParameter("pwd")
```

通过上面的设置，我们设置了登录的界面为`/login`，监听的请求为`/authenticate`，其中用户名和密码的字段分别为`user pwd`。如果真的要这样的话，我们还需要修改`login.html`中的信息，修改`name`为`user pwd`，修改`th:action`为`@{/authenticate}`。这样纯属多此一举，不是万不得已的话，不要修改默认配置。

上面除了自定义了登录界面，还设置了登录成功之后跳转的页面，已经退出之后跳转的页面。

当登录成功后，短时间内，只要浏览器还没有关系，认证信息会一直存在。不过我们也可以手动的退出登录，只需要向`/logout`发送请求就行了。(前提是，通过上面的`configure`中配置了`.logout()`开启退出的功能)。

可以在合适的页面添加这样的一个表达请求。

```html
<form method="post" th:action="@{/logout}">
    <input type="submit" value="Logout">
</form>
```

### 获取登录的用户的信息

获取登录的用户的信息有多种方法。

#### 向控制器方法中注入`Principal`

```java
User user = UserRepository.findByName(principal.getName());
```

有点儿复杂了。

#### 向控制器方法中注入`Authentication`

```java
User user = (User) authenticastion.getPrincipal();
```

还蛮不错的。

#### 使用`AuthenticationPrincipal`注解的参数

这也是我在本项目中使用的方法，在用户登录之后，我们已经有了用户的基本信息，可以在`/orders/current`页面填充这些信息。此时可以这么做。

```java
@PostMapping
public String processDesign(@Valid Taco design, Errors errors,
                            @ModelAttribute Order order,
                            @AuthenticationPrincipal User user) {
    if (errors.hasErrors()) {
        return "home";
    }

    log.info("Processing design: " + design);
    Taco saved = tacoReposity.save(design);
    order.addDesign(saved);
    userToOrder(user, order);
    log.info("Order Tacos" + order.getTacos());

    return "redirect:/orders/current";
}

private void userToOrder(@NotNull  User user, @NotNull  Order order) {
    order.setName(user.getUsername());
    order.setStreet(user.getStreet());
    order.setCity(user.getCity());
    order.setState(user.getState());
    order.setZip(user.getZip());
}
```

添加参数`@AuthenticationPrincipal User user`就可以直接使用，连类型转换都不需要。

#### 通过`SecurityContextHolder`

上面的三个方法有个局限性，只能在控制器的处理器方法中使用。如果在其他地方也想使用，就需要使用这个方法了。

```java
Authentication authentication = 
    SecurityContextHolder.getContext().getAuthentication();
User user = (User) authentication.getPrincipal();
```

