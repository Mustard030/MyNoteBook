**依赖**
```xml title:pow.xml
<dependency>  
    <groupId>com.auth0</groupId>  
    <artifactId>java-jwt</artifactId>  
    <version>4.3.0</version>  
</dependency>
```

生成范例
```java
@Test  
void contextLoads() {  
    String jwtKey = "abcdefghijklmn";  
    Algorithm algorithm = Algorithm.HMAC256(jwtKey);  
    String jwtToken = JWT.create()  
            .withClaim("id",1)  
            .withClaim("name", "lbw")  
            .withClaim("role", "nb")  
            .withExpiresAt(new Date(2024, Calendar.FEBRUARY,1))  
            .withIssuedAt(new Date())  
            .sign(algorithm);  
    System.out.println(jwtToken);  
}
```


Utils
```java title:JWTUtils.java
public class JWTUtils {  
    //密钥  
    private static final String key = "abcdefghijklmn";  
  
    private static final HashSet<String> blackList = new HashSet<>();  
  
    //加入黑名单方法  
    public static boolean invalidate(String token){  
        Algorithm algorithm = Algorithm.HMAC256(key);  
        JWTVerifier jwtVerifier = JWT.require(algorithm).build();  
        try {  
            DecodedJWT verify = jwtVerifier.verify(token);  
            Map<String, Claim> claims = verify.getClaims();  
            return blackList.add(verify.getId());  
        } catch (JWTVerificationException e){  
            return false;  
        }    }  
  
    //根据用户信息创建JWT令牌  
    public static String createJwt(UserDetails user) {  
        Algorithm algorithm = Algorithm.HMAC256(key);  
        Calendar calendar = Calendar.getInstance();  
        Date now = calendar.getTime();  
        calendar.add(Calendar.SECOND, 3600 * 24 * 7);  
        return JWT.create()  
                .withJWTId(UUID.randomUUID().toString())  
                .withClaim("name", user.getUsername())  
                .withClaim("authorities", user.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList())  
                .withExpiresAt(calendar.getTime())  
                .withIssuedAt(now)  
                .sign(algorithm);  
    }  
  
    public static UserDetails resolveJwt(String token){  
        Algorithm algorithm = Algorithm.HMAC256(key);  
        JWTVerifier jwtVerifier = JWT.require(algorithm).build();  
        try {  
            DecodedJWT verify = jwtVerifier.verify(token);  //对JWT令牌进行验证，看看是否被修改  
            if (blackList.contains(verify.getId())) return null;    //判断是否存在于黑名单中  
            Map<String, Claim> claims = verify.getClaims();  //获取令牌中内容  
            if(new Date().after(claims.get("exp").asDate())) //如果是过期令牌则返回null  
                return null;  
            else  
                //重新组装为UserDetails对象，包括用户名、授权信息等  
                return User  
                        .withUsername(claims.get("name").asString())  
                        .password("")  
                        .authorities(claims.get("authorities").asArray(String.class))  
                        .build();  
        } catch (JWTVerificationException e) {  
            return null;  
        }    }  
}
```

Filter
请求时需要携带token否则由其他filter处理无授权重新登录
```java title:JwtAuthenticationFilter.java
public class JwtAuthenticationFilter extends OncePerRequestFilter {  
    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {  
        String authorization = request.getHeader("Authorization");  
        if (authorization != null && authorization.startsWith("Bearer ")) {  
            String token = authorization.substring(7);  
            UserDetails user = JWTUtils.resolveJwt(token);  
            if (user != null) {  
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());  
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));  
                SecurityContextHolder.getContext().setAuthentication(authentication);  
            }  
        }  
  
        filterChain.doFilter(request, response);  
    }  
}
```

登陆成功时在Writer中写入token信息
```java
private void handleProcess(HttpServletRequest request,  
                               HttpServletResponse response,  
                               Object exceptionOrAuthentication) throws IOException {  
        response.setContentType("application/json;charset=utf-8");  
        PrintWriter writer = response.getWriter();  
        if (exceptionOrAuthentication instanceof AccessDeniedException exception) {  
            writer.write(RestBean.failure(403, exception.getMessage()).asJsonString());  
        } else if (exceptionOrAuthentication instanceof Exception exception) {  
            writer.write(RestBean.failure(401, exception.getMessage()).asJsonString());  
        } else if (exceptionOrAuthentication instanceof Authentication authentication) {  
//            writer.write(RestBean.success(authentication.getName()).asJsonString());  
            writer.write(RestBean.success(JWTUtils.createJwt((User) authentication.getPrincipal())).asJsonString());  
        }    }
```

退出登录时将token放入黑名单 
```java
void onLogoutSuccess(HttpServletRequest request,  
                     HttpServletResponse response,  
                     Authentication authentication) throws IOException {  
    response.setContentType("application/json;charset=utf-8");  
    PrintWriter writer = response.getWriter();  
    String authorization = request.getHeader("Authorization");  
    if (authorization != null && authorization.startsWith("Bearer ")) {  
        String token = authorization.substring(7);  
        if (JWTUtils.invalidate(token)) {  
            writer.write(RestBean.success("退出登录成功").asJsonString());  
            return;  
        }  
    }  
    writer.write(RestBean.failure(400, "退出登录失败").asJsonString());  
}
```

