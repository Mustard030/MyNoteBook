SpringSecurity中有很多个过滤器，在默认配置下启用15个。其中比较关键的是：

- UsernamePasswordAuthenticationFilter：表单登录处理逻辑
- DefaultLoginPageGeneratingFilter：内置表单登录页面生成
- DefaultLogoutPageGeneratingFilter：内置表单登出页面生成
- BasicAuthenticationFilter：与上面**UsernamePasswordAuthenticationFilter**基本认证流程一样，唯一不同之处在于获取用户名和密码不同，是通过AuthenticationConverter获取的
- AuthorizationFilter：用于鉴权




# 认证
SpringSecurity中的用户类是`User`，它实现了`UserDetails`接口。主要用来将登录用户保存到安全上下文`SecurityContextHolder`中。



## 用户登录

### SrpingSecurity默认的用户登录流程
主要依赖`UsernamePasswordAuthenticationFilter`这个过滤器，它的抽象父类`AbstractAuthenticationProcessingFilter`的`doFilter`方法中执行了`Authentication authenticationResult = this.attemptAuthentication(request, response);`，而这个过滤器重写了这个实现
```java
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {  
    if (this.postOnly && !request.getMethod().equals("POST")) {  
        throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());  
    } else {  
        String username = this.obtainUsername(request);  
        username = username != null ? username.trim() : "";  
        String password = this.obtainPassword(request);  
        password = password != null ? password : "";  
        UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username, password);  
        this.setDetails(request, authRequest);  
        return this.getAuthenticationManager().authenticate(authRequest);  
    }  
}
```
其中的变量`authRequest`就保存了前端传过来的用户名和密码，然后调用了`this.getAuthenticationManager().authenticate(authRequest)`方法。
然后可以看到`AuthenticationManager`接口，这个接口规定了认证的规范，去找它的实现类`ProviderManager`，并调用`ProviderManager`的`authenticate`方法，这个方法里的核心是，它会遍历所有实现了`AuthenticationProvider`接口的类，并调用它们的`authenticate`方法

```java
@Override  
public Authentication authenticate(Authentication authentication) throws AuthenticationException {  
    //省略
    for (AuthenticationProvider provider : getProviders()) {  
        if (!provider.supports(toTest)) {  
            continue;  
        }  
        //省略
        try {  
            //这里调用了authenticate方法
            result = provider.authenticate(authentication);  
            //省略
        }  
        //省略
    }
```
而`AbstractJaasAuthenticationProvider`这个抽象类实现了`AuthenticationProvider`接口，并且`DaoAuthenticationProvider`继承了这个抽象类并实现了最关键的两个方法`retrieveUser`和`additionalAuthenticationChecks`，然后调用这两个方法。
其中`retrieveUser`方法调用了实现了`UserDetailsService`接口的`loadUserByUsername`方法获取并返回实现了`UserDetails`接口的类，其中最重要的就是security自己的`User`类。
`additionalAuthenticationChecks`方法则是在 `retrieveUser` 方法加载用户信息后调用进行密码验证。



### 自定义用户登录流程（整合jwt）

前后端分离的开发模式中我们不会采用SpringSecurity自带的登录表单，因此**第一步是禁用formLogin**
```java
@Bean  
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {  
    http  
        	.authorizeRequests(authorize -> authorize
                        .antMatchers(WHITE_LIST).permitAll() // 允许白名单接口无需认证
                        .antMatchers("/login").permitAll() // 登录接口无需认证
                        .anyRequest().authenticated() // 其他请求需认证
            )
            .csrf(AbstractHttpConfigurer::disable)  // 禁用 CSRF            
        	.formLogin(AbstractHttpConfigurer::disable)  // 禁用SpringSecurity内置的表单登录  
            .logout(AbstractHttpConfigurer::disable)  // 登出也禁用
        	.httpBasic(AbstractHttpConfigurer::disable)
        	.addFilterBefore(jwtAuthenticationFilter, AuthorizationFilter.class);  //注入的
      
    // 构建 SecurityFilterChain    
    return http.build();  
}
```

当禁用了`formLogin`之后注册的过滤器会少4个，分别是`UsernamePasswordAuthenticationFilter`、`DefaultLoginPageGeneratingFilter`、`DefaultLogoutPageGeneratingFilter`、`BasicAuthenticationFilter`。

**第二步：创建认证的Service**

由于在默认的登录流程中，SpringSecurity通过`getAuthenticationManager()`获得认证管理器，但是我们自己没办法获得，所以要自己创建。

在配置类中

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    @Autowired
    private UserDetailsService myUserDetailsService;  // 自己实现的UserDetailsService
    
    @Bean
    public AuthenticationManager authenticationManager() throws Exception {
        // 匹配合适的AuthenticationProvider(DaoAuthenticationProvider)
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        // 配置自定义的UserDetailsService
        provider.setUserDetailsService(myUserDetailsService);
        // 创建并返回认证管理器对象（实现类ProviderManager）
        return new ProviderManager(provider);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // 使用 BCrypt 密码编码器
    }

}
```

然后自定义登录服务流程

```java
@Service
public class LoginService {
    @Autowired
    private AuthenticationManager authenticationManager;  // 在配置类中的Bean
    
    public ResponseResult login(LoginDTO dto) {
        //将前端传来的用户名密码封装成authRequest对象
        UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(dto.getUsername(), dto.getPassword());
        //发起认证得到认证后的authentication对象
        try{
            Authentication authentication = authenticationManager.authenticate(authRequest);
            // 返回认证结果 如果成功，则将认证后的对象authentication存到安全上下文供后续过滤器取用（重要！）
            if (authentication.isAuthenticated()) {
                SecurityContextHolder.getContext().setAuthentication(authentication);
                //生成jwt
                String token = ""; //自行实现
                //用户标识信息写入到redis中，自行实现
                
                return ResponseResult.ok(token);  // 返回token
            }
        } catch (Exception e){
         	// 处理异常，或者交给Advice	   
        }
        return ResponseResult.error();  // 返回需要的信息
    }
}
```

**第三步：创建JWT过滤器**

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 放行登录请求
        if ("/login".equals(request.getRequestURI())) {
            filterChain.doFilter(request, response);
            return;
        }
        // 获取请求头中的jwt
        // 按照规范，jwt在请求头中的Authorization字段中，并且以"Bearer "开头
        String jwt = request.getHeader("Authorization");
        // 判断是否为空，为空则抛出异常
        if (jwt == null || jwt.isEmpty()) {
            // 抛出项目中定义好的异常
        }
        // 获取token中的用户标识，注意解析的异常处理
        try{
            final JWT parseToken = JWTUtil.parseToken(jwt);
            parseToken.getPayload("sub");
            //做自己需要的操作
        } catch (Exception e){
            // 抛出项目中定义好的异常，可能是解析错误或者token过期
        }
        
        // 从Redis中通过用户标识（一个UserDetails类型），获取用户认证信息，注意redis返回null的异常处理
        MyUserDetails userDetails = (MyUserDetails) redisTemplate.opsForValue().get("user:" + userId); // 或者自己的缓存工具类获取
        if (userDetails == null) {
            // 抛出项目中定义好的异常，可能是redis中获取不到用户信息
        }
        // 将用户信息存入安全上下文
        Authentication authentication = UsernamePasswordAuthenticationToken.authenticated(userDetails, null, userDetails.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(authentication);
        // 过滤放行
        filterChain.doFilter(request, response);
    }
}
```

**第四步：创建登出的Service**

由于不再使用SpringSecurity提供的登录和登出，改为用jwt和redis维护，因此登出也要自行实现

从上面的登录过滤器不难看出，只需让前端的token失效或是redis中的token失效即可让过滤器不通过，从而实现登出

```java
@Service
public class LogoutService {
    public ResponseResult logout() {
        // 清空redis数据
        String redisKey;
        MyUserDetails principal = (MyUserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (principal != null) {
            // 根据自己的逻辑获取redisKey
            // 删除redis数据
        }
        // 清空安全上下文
        SecurityContextHolder.clearContext();
        return ResponseResult.ok();
    }
}
```



最后自定义一个Controller端点调用登录服务即可。





## 管理用户相关接口

### 认证接口
用于认证用户的接口有两个，分别是`UserDetailsManager`和`UserDetailsService`，`UserDetailsManager`继承了`UserDetailsService`，`UserDetailsService`仅有`loadUserByUsername`方法，而`UserDetailsManager`有以下方法：
- `void createUser(UserDetails user)`
- `void updateUser(UserDetails user)`
- `void deleteUser(String username)`
- `boolean userExists(String username)`
- `void changePassword(String oldPassword, String newPassword)`

例如：

```java
@Service
public class MyUserDetailsService implements UserDetailsService {
    @Autowired
    private UserMapper userMapper;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserEntity userEntity = userMapper.selectOne(
                new LambdaQueryWrapper<UserDo>()
                        .eq(UserDo::getUserName, username)
        );
        // 这里用的是security的User，也可以自定义User类，实现UserDetails接口即可
        /*UserDetails userDetails = User.builder()
                .username(userDo.getUserName())
                .password(userDo.getPassword())
                .build();*/
        // 例如这里可以修改为
        UserDetails userDetails = new MyUserDetails(userEntity);
        return userDetails;
    }
}
```



### 密码升级
在每次登录成功后，SpringSecurity会调用`DaoAuthenticationProvider`中的`createSuccessAuthentication`
```java
@Override  
protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,  
       UserDetails user) {  
    boolean upgradeEncoding = this.userDetailsPasswordService != null  
          && this.passwordEncoder.upgradeEncoding(user.getPassword());  
    if (upgradeEncoding) {  
       String presentedPassword = authentication.getCredentials().toString();  
       String newPassword = this.passwordEncoder.encode(presentedPassword);  
       user = this.userDetailsPasswordService.updatePassword(user, newPassword);  
    }  
    return super.createSuccessAuthentication(principal, authentication, user);  
}
```
其中会调用`this.userDetailsPasswordService.updatePassword`，如果我们实现`UserDetailsPasswordService`中的`UserDetails updatePassword(UserDetails user, String newPassword)`方法，例如：
```java
@Override
public UserDetails updatePassword(UserDetails user, String newPassword){
	// 这里的newPassword是加密过的密码，如果没有指定PasswordEncoder，则默认使用BC
	userMapper.updatePassword(user.getUserName(), newPassword);// 自行实现数据库更新
	return user;
}
```
则会在每次登录成功后进行密码升级
注意：
- 原来未指定密码加密算法，就升级为指定的
- 原来指定了密码加密算法的，将对比已有加密算法与指定的密码加密算法，采用强度更大的算法



# 鉴权

使用注解`@EnableMethodSecurity`开启方法鉴权

@PreAuthorize("hasAuthority('权限字符串')")

