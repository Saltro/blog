---
title: 记一次使用 SpringBoot+Sping Security+Vue 的前后端分离权限认证
date: 2021-04-12
tags:
    - Java
    - Spring
    - 后端
    - Vue
    - 前端
---

# 项目介绍

该项目是针对“权限认证”这一事务的简单实验，主要用于学习前后端分离的开发过程，以及权限认证的方式。项目使用前后端分离架构，后端使用目前流行的 Spring 框架进行开发，使用 SpringBoot 进行快速配置，通过 Spring Security 进行傻瓜式权限认证功能。前端使用 Vue3 进行开发，利用 ElmentPlus 框架设计 UI。前后端交互通过前端使用 axios 向后端 RESTapi 接口发送 ajax 请求，后端响应返回 json 数据实现。最主要的认证功能基于 Spring Security，基本没有需要手动造轮子的部分。关于数据库部分，数据库使用 MySQL 数据库，后端使用 Spring JDBCTemplate 方便数据库操作。该项目仅用于开发、演示权限认证功能，因此没有具体事务功能，只有演示用的简单事务接口和测试接口。

# 数据库

在此项目中，数据库基本只用于存取用户信息与权限信息。数据库包含两张表，分别为存放用户基本信息的 sys\_user 表，存放用户权限信息的 user\_authority 表。sys\_user 表中有三个字段，用户唯一标识符 id、用户名 name 和密码 pwd。user\_authority 表中有两个字段，用户标识符 user\_id 和对应用户的权限（角色） role。除 id 外字段均为 varchar 类型。

需要注意的是，id 尽量不要设置为 unsigned int 类型，因 Java 中不存在 unsigned int 类型，因而在读取 id 时会发生解码错误。

# 后端部分

## 环境

首先用 SpringBoot 搭建起一个简单的 SpringBoot 项目框架，在此我们需要用到的依赖有：web、Spring security、Springdata jdbc、lombok（减少代码量）、MySQL connector、junit（用于单元测试）。利用 Maven 导入依赖后就基本搭建好了后端开发的环境。此时项目中包括 SpringBootAplication 启动类，即整个项目的入口。

```java
@SpringBootApplication
public class AlephApplication {

    public static void main(String[] args) {
        SpringApplication.run(AlephApplication.class, args);
    }

}
```

同时还有 Spring 的配置文件 aplication.properties。在配置文件中配置项目所需要用到的数据库和端口。

```.properties
server.port=8700
```

```yaml
Spring:
  datasource:
    url: jdbc:MySQL://localhost/mis_aleph
    username: mis
    password: mismis
    driver-class-name: com.MySQL.cj.jdbc.Driver
```

## 实现 UserDetailsService

为了更好地使用 Security 帮助我们进行自动认证，我们可以自定义用户详情服务，通过实现 UserDetailsService 接口，创建自定义的 user 信息服务。

```java
public class UserDetailsServiceImpl implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 通过用户名查找用户信息，返回用户信息实体
    }
}
```

同时我们需要实现 UserDetails 接口以创建用户实体类。其中我们必须实现获取用户名、密码、用户权限方法，这样 Security 才能够接管用户服务。其他方法主要用于对用户账号进行注销等操作，此处可以忽略。代码中@Data @NoArgsConstructor @AllArgsConstructor 注解为 lombok 注解，自动实现 setter 和 getter 方法，同时配置两个构造器。

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserEntity implements UserDetails {
    private int id;
    private String username;
    private String userPwd;
    private List<String> roles;

    public UserEntity(String username, String userPwd, List<String> roles) {
        this.username = username;
        this.userPwd = userPwd;
        this.roles = roles;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();
        for (String role: roles) {
            authorities.add(new SimpleGrantedAuthority(role));
        }
        return authorities;
    }

    @Override
    public String getPassword() {
        return userPwd;
    }

    @Override
    public String getUsername() {
        return username;
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

有了 User 实体类后，我们需要一个 repository 帮助我们进行数据库的增删改查操作，方便创建、查找 User。使用 Spring 的@Repository 注解将 UserRepository 注册到 Spring 上下文中，这样可以在其他类中进行自动配置。通过@Autowired 自动注入 jdbc 模板加速开发，之后只需要在数据库中依据用户名查找用户即可。查找到全部的用户信息后，返回一个 UserDetails 对象。

需要注意的是如果数据库中没有该用户，则会抛出 EmptyResultDataAccessException 异常。

```java
@Repository
public class UserRepository {
    private final JdbcTemplate jdbc;

    @Autowired
    public UserRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public UserEntity findUserByUsername(String username) throws EmptyResultDataAccessException {
        String SQL = "select * from sys_user where name=?";
        Map<String, Object> user = jdbc.queryForMap(SQL, username);
        SQL = "select role from user_authority where user_id=?";
        List<Map<String, Object>> user_authes = jdbc.queryForList(SQL, user.get("id"));
        List<String> roles = new ArrayList<>();
        for (Map<String, Object> auth: user_authes) {
            roles.add((String) auth.get("role"));
        }
        return new UserEntity(
                (int) user.get("id"),  // 数据库中 id 不为 unsigned int 才能取出为 int 类型，否则为 long
                (String) user.get("name"),
                (String) user.get("pwd"),
                roles);
    }
}
```

有了 UserRepository 后，便可以将其注入到 UserDetailsService 中。直接调用 repository 的方法便可直接返回用户实体。

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    private final UserRepository userRepository;

    @Autowired
    public UserDetailsServiceImpl(UserRepository userRepository) {
        // 注入 UserRepository
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        // 通过用户名查找用户信息，返回用户信息实体
        return userRepository.findUserByUsername(s);
    }
}
```

最后我们将实现好的 UserDetailsService 注入到 SecurityConfig 配置中去，这样我们自定义的用户信息服务便会生效。在注入时用@Qualifier 注解标注我们要注入哪一个实现。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    private final UserDetailsService userDetailsService;

    //  推荐使用构造器进行注入
    public SecurityConfig(@Qualifier("userDetailsServiceImpl") UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 配置认证方式等
        auth
                .userDetailsService(userDetailsService);  // 将用户详情服务配置到 Spring Security 上
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // http 相关的配置，包括登入登出、异常处理、会话管理等
    }
}
```

### 密码加密

新版本的 Spring security 规定必须设置一个默认的加密方式，不允许使用明文。这个加密方式是用于在登录时验证密码、注册时需要用到。在此我们选择标准加密。通过@Bean 注解，我们可以只将其写在 SecurityConfig 中就完成注册。之后同上在 SecurityConfig 中完成配置，这样 Security 在认证时就会自动采用加密认证。

```java
@Bean
public PasswordEncoder encoder() {
    return new StandardPasswordEncoder("7t89yo");
}

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 配置认证方式等
    auth
            .userDetailsService(userDetailsService)  // 将用户详情服务配置到 Spring Security 上
            .passwordEncoder(encoder());  // 在全局中使用加密
}
```

## 用户登入登出

Spring Security 默认的登入路径是/login 登出路径是/logout，向路径发送 post 请求传入用户名 username 和密码 password 即可实现登入或登出。但 Spring Security 默认设置了 csrf 拦截，导致 post 请求默认被拦截，因此我们需要对几个路径配置 csrf token 准许 post 请求通过。这样前端可以向后端的注册、登入、登出路径发送请求。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    // http 相关的配置，包括登入登出、异常处理、会话管理等
    // 设置 csrf token，防止 post 请求拦截
    http
            .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .ignoringAntMatchers("/register", "/login", "/logout");
}
```

之后为了处理登入的反馈，我们需要实现权限认证成功处理器和失败处理器。处理器会在登入后依据认证情况进行响应，如果成功则会执行成功处理器，反之执行失败处理器。因此需要实现 AuthenticationSuccessHandler 和 AuthenticationFailureHandler 接口。

```java
@Component  // 注册 Spring 上下文
public class AuthSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
    }
}
```

以 AuthSuccessHandler 为例，需要实现 onAuthenticationSuccess 方法，参数有 http 请求体、响应体和权限。在此我们需要做的应该是，在登入成功后后端应该返回标志着登录成功的 json 响应数据。

为此首先我们要统一状态码。

```java
public enum ReturnStatus {
    // 响应正常
    SUCCESS(1000, "服务器响应正常"),
    RESPONSE_SUCCESS(1001, "服务器响应正常且数据操作正常"),

    // 事务异常
    TRANSACTION_ERROR(2000, "事务操作异常"),

    // 服务器非正常
    SERVER_ERROR(3000, "服务器异常"),

    // 用户非正常
    USER_AUTH_DENY(4000, "用户权限不足"),
    USER_PASSWORD_ERROR(4001, "用户密码错误"),
    USER_ACCOUNT_EXPIRED(4002, "用户账号过期"),
    USER_PASSWORD_EXPIRED(4003, "用户密码过期"),
    USER_ACCOUNT_DISABLE(4004, "用户账号不可用"),
    USER_ACCOUNT_LOCKED(4005, "用户账号锁定"),
    USER_ACCOUNT_NOT_EXIST(4006, "用户不存在"),
    USER_NOT_LOGIN(4007, "用户未登录"),

    // 其他错误
    COMMON_ERROR(9000, "未知错误");

    private int statusCode;
    private String statusMsg;

    ReturnStatus(int statusCode, String statusMsg) {
        this.statusCode = statusCode;
        this.statusMsg = statusMsg;
    }

    public int getStatusCode() {
        return statusCode;
    }

    public void setStatusCode(int statusCode) {
        this.statusCode = statusCode;
    }

    public String getStatusMsg() {
        return statusMsg;
    }

    public void setStatusMsg(String statusMsg) {
        this.statusMsg = statusMsg;
    }

    public static String getMsgByCode(int statusCode) {
        for (ReturnStatus rs: ReturnStatus.values()) {
            if (rs.getStatusCode() == statusCode) return rs.getStatusMsg();
        }
        return "";
    }
}
```

之后构造统一的返回体工具类，辅助构建 json 响应数据。我们预期的返回体构成如下。

```json
{
    "statusCode": "xxx",
    "statusMsg": "xxx",
    "data": {
        "xxx": "xxx"
    }
}
```

因此构建 ReturnJsonUtil 类。

```java
@Data
public class ReturnJsonUtil<T> {
    private int statusCode;
    private String statusMsg;
    private T data;

    public ReturnJsonUtil() {}

    public ReturnJsonUtil(ReturnStatus rs) {
        statusCode = rs.getStatusCode();
        statusMsg = rs.getStatusMsg();
    }

    public ReturnJsonUtil(int statusCode, T data) {
        this.statusCode = statusCode;
        statusMsg = ReturnStatus.getMsgByCode(statusCode);
        this.data = data;
    }

    public ReturnJsonUtil(int statusCode) {
        this.statusCode = statusCode;
        statusMsg = ReturnStatus.getMsgByCode(statusCode);
    }

    public void setStatusCode(int statusCode) {
        this.statusCode = statusCode;
        statusMsg = ReturnStatus.getMsgByCode(statusCode);
    }

    public void setStatus(ReturnStatus rs) {
        statusCode = rs.getStatusCode();
        statusMsg = rs.getStatusMsg();
    }

    public void setData(T data) {
        this.data = data;
    }
}
```

之后便可以在登入成功处理器中使用返回体工具来向响应体中写入 json 格式的数据，以此返回前端。此处使用 jackson 的 objectmapper 将对象解析为 json 字符串写入 response 响应体。

```java
@Component  // 注册 Spring 上下文
public class AuthSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
        httpServletResponse.setContentType("text/json;charset=utf-8");  // 设置返回数据格式为 json
        ObjectMapper objectMapper = new ObjectMapper();
        ReturnJsonUtil<Object> rj = new ReturnJsonUtil<>(ReturnStatus.SUCCESS);
        objectMapper.writeValue(httpServletResponse.getWriter(), rj);
    }
}
```

同理实现 AuthenticationFailureHandler 接口响应登入失败状况。

```java
@Component
public class AuthFailureHandler implements AuthenticationFailureHandler {
    @Override
    public void onAuthenticationFailure(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
        ReturnJsonUtil<Object> rj = null;
        if (e instanceof AccountExpiredException) {
            //账号过期
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_ACCOUNT_EXPIRED);
        } else if (e instanceof BadCredentialsException) {
            //密码错误
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_PASSWORD_ERROR);
        } else if (e instanceof CredentialsExpiredException) {
            //密码过期
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_PASSWORD_EXPIRED);
        } else if (e instanceof DisabledException) {
            //账号不可用
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_ACCOUNT_DISABLE);
        } else if (e instanceof LockedException) {
            //账号锁定
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_ACCOUNT_LOCKED);
        } else if (e instanceof InternalAuthenticationServiceException) {
            //用户不存在
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_ACCOUNT_NOT_EXIST);
        }else{
            //其他错误
            rj = new ReturnJsonUtil<>(ReturnStatus.COMMON_ERROR);
        }

        httpServletResponse.setContentType("text/json;charset=utf-8");
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.writeValue(httpServletResponse.getWriter(), rj);
    }
}
```

最后将写好的处理器配置到 SecurityConfig 中去，完成登入配置。同时要先将处理器注入。

```java
private final AuthSuccessHandler authSuccessHandler;
private final AuthFailureHandler authFailureHandler;

//  推荐使用构造器进行注入
public SecurityConfig(
    @Qualifier("userDetailsServiceImpl") UserDetailsService userDetailsService,
    AuthSuccessHandler authSuccessHandler,
    AuthFailureHandler authFailureHandler) {
    this.userDetailsService = userDetailsService;
    this.authSuccessHandler = authSuccessHandler;
    this.authFailureHandler = authFailureHandler;
}
......
http
        .formLogin()  // 配置登入
        .loginProcessingUrl("/login")  // 设置登录接口路径，登录方式为 post 请求，字段为用户名 username 及密码 password
        .successHandler(authSuccessHandler)  // 注入成功处理器
        .failureHandler(authFailureHandler)  // 注入失败处理器
        .permitAll();  // 允许所有用户访问（包括匿名用户
```

登出配置基本同理，响应登出成功需要实现 LogoutSuccessHandler 接口。从用户体验角度出发，没有登入的用户是不能进行登出，因此我们判断此处权限是否为空，若为空则代表用户没有权限，返回用户未登录状态码。

```java
@Component
public class AccountLogoutSuccessHandler implements LogoutSuccessHandler {
    @Override
    public void onLogoutSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
        httpServletResponse.setContentType("text/json;charset=utf-8");
        ObjectMapper objectMapper = new ObjectMapper();
        ReturnJsonUtil<Object> rj;
        if (authentication != null) {
            rj = new ReturnJsonUtil<>(ReturnStatus.SUCCESS);
        } else {
            rj = new ReturnJsonUtil<>(ReturnStatus.USER_NOT_LOGIN);
        }
        objectMapper.writeValue(httpServletResponse.getWriter(), rj);
    }
}
```

在配置到 SecurityConfig 时，我们需要在用户登出时删除 JSESSIONID 的 Cookie，以确保用户正常登出。

```java
http
        .formLogin()  // 配置登入
            .loginProcessingUrl("/login")  // 设置登录接口路径，登录方式为 post 请求，字段为用户名 username 及密码 password
            .successHandler(authSuccessHandler)  // 注入成功处理器
            .failureHandler(authFailureHandler)  // 注入失败处理器
            .permitAll()  // 允许所有用户访问（包括匿名用户
        .and()
            .logout()
            .logoutSuccessHandler(accountLogoutSuccessHandler)  // 注入登出处理器
            .deleteCookies("JSESSIONID");  // 登出时删除 JSESSIONID
```

## 权限设置与匿名用户

在 SecurityConfig 中我们可以自由地配置不同路径下的权限。如下代码中我们对/api/user/**路径设置了角色权限，只有拥有 USER 角色的用户才能够正常访问，而/api/admin/**路径只允许 ADMIN 角色访问。/register 和/register/下的所有路径允许所有用户访问，而除此之外的路径至少先通过认证才能访问。

```java
http
        .authorizeRequests()
            .antMatchers("/api/user/**").hasRole("USER")  // 使用 /api/user/** 而不是 /api/user  ** 为通配符
            .antMatchers("/api/admin/**").hasRole("ADMIN")
            .antMatchers("/register", "/register/**").permitAll()
            .antMatchers("/**").authenticated();
```

这样一个普通的用户便无法访问到/api/admin/下的任何路径了。但如果未登录用户访问高权限数据，请求会被重定向至登入界面，这并不符合我们的设计。我们需要用户在访问被拒绝时返回统一的 json 数据。因此我们需要屏蔽重定向页面，并在在 httpconfig 中设置匿名用户的异常处理。实现这一功能的核心就是实现 AuthenticationEntryPoint 接口。

```java
@Component
public class AuthEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
        ReturnJsonUtil<Object> rj = new ReturnJsonUtil<>(ReturnStatus.USER_NOT_LOGIN);
        httpServletResponse.setContentType("text/json;charset=utf-8");
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.writeValue(httpServletResponse.getWriter(), rj);
    }
}
```

最后配置到 SecurityConfig 中，同样 authEntryPoint 也需要注入。

```java
http
    .authorizeRequests()
        .antMatchers("/api/user/**").hasRole("USER")  // 使用 /api/user/** 而不是 /api/user
        .antMatchers("/api/admin/**").hasRole("ADMIN")
        .antMatchers("/register", "/register/**").permitAll()
        .antMatchers("/**").authenticated()
    .and()
        .formLogin()
        .loginProcessingUrl("/login")  // 设置登录接口路径，登录方式为 post 请求，字段为用户名 username 及密码 password
        .successHandler(authSuccessHandler)  // 注入成功处理器
        .failureHandler(authFailureHandler)  // 注入失败处理器
        .permitAll()
    .and()
        .logout()
        .logoutSuccessHandler(accountLogoutSuccessHandler)  // 注入登出处理器
        .deleteCookies("JSESSIONID")  // 登出时删除 JSESSIONID
    .and()
        .exceptionHandling()  // 异常处理
        .authenticationEntryPoint(authEntryPoint);
```

这样前端的匿名用户在访问高权限数据时也能够接收到 json 响应了。

## 用户注册

Sping Security 没有支持用户注册，因此我们需要手动编写用户注册接口。本质上用户注册接口就是接收前端传来的数据，将其保存到数据库中，再向前端响应。因此我们只是需要实现一个 RestController 控制器。

当然首先我们需要能够保存数据至数据库，即在 UserRepository 中实现 saveWithoutId 方法。首先判断用户名是否已存在。当用户名不存在时，我们需要先将用户名用户密码存入 sys\_user 表中，然后再得到它自动创建的 id，再使用 id 将用户权限信息存入 user\_authority 表中。因此我们需要 KeyHolder 帮助我们在插入数据时保留插入行的数据，以此得到用户 id。而为了让 jdbc 能够支持 KeyHolder，我们又需要创建 PreparedStatementCreator，这样才能够在 PreparedStatement 执行中使用 KeyHolder。

```java
public void saveWithoutId(UserEntity user) throws UsernameAlreadyExistsException {
    try {
        findUserByUsername(user.getUsername());
        throw new UsernameAlreadyExistsException();  // 如果用户名已存在抛出 UsernameAlreadyExistsException 异常
    } catch (EmptyResultDataAccessException ignored) {
    }

    PreparedStatementCreator psc = new PreparedStatementCreator() {
        @Override
        public PreparedStatement createPreparedStatement(Connection connection) throws SQLException {
            PreparedStatement ps = connection.prepareStatement("insert into sys_user(name, pwd) values(?,?)", Statement.RETURN_GENERATED_KEYS);  // Statement.RETURN_GENERATED_KEYS) 确保能够使用 KeyHolder 返回值
            ps.setString(1, user.getUsername());
            ps.setString(2, user.getPassword());
            return ps;
        }
    };
    KeyHolder keyHolder = new GeneratedKeyHolder();
    jdbc.update(psc, keyHolder);
    int userId = keyHolder.getKey().intValue();
    String SQL = "insert into user_authority values(?,?)";
    List<String> roles = user.getRoles();
    for (String role: roles) {
        jdbc.update(SQL, userId, role);
    }
}
```

能够存储用户信息还不够，我们需要能够接收前端传来的用户注册信息数据，即用户名和用户密码。对于接收数据，我们只需要建立一个 form 实体对象，Spring 会自动进行对象映射。

```java
@Data
@NoArgsConstructor
public class RegistrationForm {
    private String username;
    private String userPwd;

    public UserEntity toUser(PasswordEncoder passwordEncoder) {
        List<String> role = Arrays.asList("ROLE_USER");
        return new UserEntity(username, passwordEncoder.encode(userPwd), role);
    }
}
```

在前端发送数据后，会将数据一一映射形成对象，controller 只需处理对象即可。使用@RestController 可以注册为 RestController，使得控制器不再操作视图，使用@RequestMapping(path = “/register”) 注解标注要监听的路径，使用@CrossOrigin(origins = “\*“) 注解来允许跨域请求。

```java
@RestController
@RequestMapping(path = "/register")
@CrossOrigin(origins = "*")
public class RegisterRESTController {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Autowired
    public RegisterRESTController(UserRepository userRepository,
                                  PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @PostMapping
    public ReturnJsonUtil<Object> userRegistration(RegistrationForm form) {
        ReturnJsonUtil<Object> rj = new ReturnJsonUtil<>();
        try {
            userRepository.saveWithoutId(form.toUser(passwordEncoder));
            rj.setStatus(ReturnStatus.RESPONSE_SUCCESS);
        } catch (UsernameAlreadyExistsException e) {
            rj.setStatus(ReturnStatus.TRANSACTION_ERROR);
        }
        return rj;
    }
}
```

需要注意的是，因为我们在认证中使用了加密认证，因此我们在存储密码时也需要先将用户密码进行加密存储，因此在 form 对象的 toUser 方法中，传入之前定义过的 PasswordEncoder 对象进行加密，生成加密后的用户对象。

## 简单的事务处理

至此我们基本完成了 Spring Security 的全部配置，因此我们写一个最简单的控制器来测试我们的配置。只要用户请求，服务器便会记录下用户名和用户权限，并返回成功信息。

```java
@RestController
@RequestMapping(path = "/api")
@CrossOrigin(origins = "*")
public class CommonApiRESTController {
    @GetMapping("/user/common")
    public ReturnJsonUtil<Object> userCommonRequest(@AuthenticationPrincipal UserEntity user) {
        System.out.println("User: "+user.getUsername()+" Authorities: "+user.getAuthorities());
        return new ReturnJsonUtil<>(ReturnStatus.SUCCESS);
    }

    @GetMapping("/admin/common")
    public ReturnJsonUtil<Object> adminCommonRequest(@AuthenticationPrincipal UserEntity user) {
        System.out.println("User: "+user.getUsername()+" Authorities: "+user.getAuthorities());
        return new ReturnJsonUtil<>(ReturnStatus.SUCCESS);
    }
}
```

后端搭建基本完毕，运行 SpringBootApplication 类便可启动后端。

# 前端部分

## 环境

前端使用 VueCLI 搭建开发环境，使用 vue 指令便可快速搭建好前端开发环境，同时在项目目录中安装 ElementPlus 和 axios 模块。配置环境在项目目录下新建 vue.config.js 文件，配置如下。最核心的配置是 proxy 代理，将对当前服务器的请求代理到后端服务器，以此实现开发环境下的前后端交互（生产环境一般采用 Nginx 配置）。

```javascript
module.exports = {
  devServer: {
    open: true,  // 自动打开浏览器
    host: 'localhost',
    port: 8600,
    proxy: {
      '/api': {
        target: 'http://localhost:8700',
        changeOrigin: true,
        ws: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
  }
}
```

组件开发



之后便是各个页面（组件）的开发，单以登录组件为例，首先在 Login.vue 中依靠 ElementPlus 写一个模板。

```html
<template>
  <el-card class="index-card" shadow="always">
    <h3> 登录 </h3>
    <el-form :model="form">
      <el-form-item>
        <el-input v-model="form.username" maxlength="30" placeholder="用户名"></el-input>
      </el-form-item>
      <el-form-item>
        <el-input v-model="form.userPwd" maxlength="30" placeholder="密码" show-password></el-input>
      </el-form-item>
      <el-button class="login-submit" type="primary" @click="submit"> 登录 </el-button>
    </el-form>
  </el-card>
</template>
```

之后配置其绑定的 data 和 methods，即 form 数据和 submit 方法，form 数据同表单绑定，而 submit 方法在登录按钮点击时触发。

```javascript
export default {
  name: 'Login',
  data() {
    return {
      form: {
        username: "",
        userPwd: ""
      }
    }
  },
  methods: {
    submit() {
    }
  }
}
```

之后实现 submit 方法。submit 首先需要判断用户名、密码是否符合要求，之后将用户名密码发送到后端相应的登录路径，并对后端响应进行操作，进行相应的响应。向后端发送请求时需要使用 axios 进行请求发送，使用 qs 进行对象解析，否则无法发送数据。在 axios 发送 post 请求后，可以使用 then 进行数据的异步响应，此处的 response 即包括后端所发送的 json 对象。如果正确响应并且状态码为 1000 或者 1001，则表示登录成功，使用 ElementPlus 的 ElMessage 弹出一条“登录成功”提示。否则弹出错误提示。如果服务器没有能够正确响应，则使用 catch 对错误进行响应和记录。

```javascript
submit() {
    if (this.form.username.length < 4) {
        ElMessage.warning("用户名长度至少 4 位");
        return;
    }
    if (this.form.userPwd.length < 6) {
        ElMessage.warning("密码长度至少 6 位");
        return;
    }

    // 直接向后端请求，后端传入参数为 null，需要通过 qs 进行对象解析传递
    let loginData = {
        username: this.form.username,
        password: this.form.userPwd
    }
    axios.post('/api/login', qs.stringify(loginData))
        .then(response => {
        if (response.data.statusCode == 1000  response.data.statusCode == 1001) {
            ElMessage.success("登录成功");
        } else {
            ElMessage.error(response.data.statusMsg);
        }
    })
        .catch(function (error) {
        ElMessage.error("未知错误");
        console.log(error);
    });
}
```

注册组件基本相同在此不再赘述。在事务处理组件中，如果用户权限不足，则后端不会正常响应，即返回的 http 状态码为 403 权限不足。此时 axios 会将 response 交给 catch 处理。因此如果在 catch 中发现 http 状态码为 403，则表示用户权限不足，弹出错误提示。

```javascript
axios.get('/api/api/user/common')
    .then(response => {  // 直接使用函数会使得 this 指向丢失，此时可以使用 => 使得 this 指向 vue
    if (response.data.statusCode == 1000) {
        ElMessage.success("请求成功");
        this.userReturn = "用户你好";
    } else if (response.data.statusCode == 4007) {
        ElMessage.error("用户未登录");
        this.userReturn = "";
    } else {
        ElMessage.error("请求失败");
        this.userReturn = "";
    }
})
    .catch(function (error) {
    const response = error.response;
    if (response.status == 403) {
        ElMessage.error("用户权限不足");
    } else if (response.status == 500) {
        ElMessage.error("服务器错误");
    } else {
        ElMessage.error("未知错误");
    }
})
}
```

## 整合

组件全部开发完成后，在 App.vue 中完成组件整合。在模板中使用 tabs 来选择组件，以此实现“假路由”。

```html
<template>
  <el-tabs class="top-navigation">
    <el-tab-pane label="登录">
      <login></login>
    </el-tab-pane>
    <el-tab-pane label="注册">
      <register></register>
    </el-tab-pane>
    <el-tab-pane label="事务">
      <main-office></main-office>
    </el-tab-pane>
    <el-tab-pane label="登出">
      <logout></logout>
    </el-tab-pane>
  </el-tabs>
</template>
```

在 js 中导入组件进行引入，从而在模板中使用组件。

```javascript
import Login from './components/Login.vue'
import Register from './components/Register.vue'
import MainOffice from './components/MainOffice'
import Logout from './components/Logout'

export default {
  name: 'App',
  components: {
    Login,
    Register,
    MainOffice,
    Logout
  }
}
```

最后使用`npm run serve`指令启动 vue 项目，完成前端开发。

# 总结与不足

总体来看项目已经完成了“权限认证”的功能，能够在整体上做到前后端分离，在后端做到权限认证、用户操作、处理请求、响应数据等基本功能，在前端做到发送请求、响应数据等。同时项目过程中也确实学习到一些新的技术，更主要的是学习到一些前后端分离、权限认证的思想和方法。

但项目还有很多不完善的地方，有很多方面值得去努力。在后端上，jdbc 模板操作数据库的方式较为低效且工作量大，代码复用低，没有面向接口编程，没有投入过真正的生产环境等等。在前端上，由于技术实力原因没有采用 Vue Router 动态路由组件导致用户体验不良，界面仍较为简陋，代码复用低，对前端项目理解浅薄等等。因此，未来的发展方向是：1、学习 MyBatis，简化数据库操作；2、学习设计模式；3、尝试将该项目投入真正的生产环境进行测试；4、学习 Vue Router，更好地开发单页面应用；5、加深 JavaScript 理解。

# 写在最后

终于写完了，不管是项目本体还是这一份文档（实验报告）。总的来说这一次项目内容确实非常的少，但学到的东西确实很多。抛开框架怎么用这些纯粹没什么含量的技能，至少在前后端分离思想、单页面应用的逻辑、RESTful 接口、权限认证的作用上我确实学到很多（也明白了为什么全栈不可能）。这是一个好的开始，不管是从成功上还是不足上，它都有很多亮点。所以就以此为基开始漫长的全栈之旅吧，哪怕不为了什么“屠龙”什么“基本功”。
