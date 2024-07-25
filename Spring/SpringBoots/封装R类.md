创建状态码枚举类
```java
@Getter  
@AllArgsConstructor  
public enum ResponseConstantEnum {  
    /**  
     * 操作成功  
     */  
    SUCCESS(200,"操作成功"),  
  
    /**  
     * 错误请求  
     */  
    ERROR(400,"错误请求"),  
  
    /**  
     * 请求页面未找到  
     */  
    NOT_FIND_ERROR(404,"请求页面未找到"),  
  
    /**  
     * 系统异常  
     */  
    SYS_ERROR(-1, "系统异常"),  
  
    /**  
     * 参数校验失败  
     */  
    VALIDATE_ERROR(1002,"参数校验失败"),  
    ;  
  
    //请求响应码  
    private final Integer resultCode;  
  
    //请求响应消息  
    private final String resultMessage;  
  
  
    public static ResponseConstantEnum get(Integer code) {  
        //value()方法可以将枚举类转变为一个枚举类型的数组  
        for (ResponseConstantEnum responseConstantEnum : ResponseConstantEnum.values()) {  
            if (Objects.equals(code, responseConstantEnum.getResultCode())) {  
                return responseConstantEnum;  
            }  
        }  
        return ResponseConstantEnum.ERROR;  
    }  
  
}
```
R类
```java
@Data  
@NoArgsConstructor  
public class R<T> implements Serializable {  
  
    @Serial  
    private static final long serialVersionUID = 1L;  
  
    //响应码  
    private Integer code;  
  
    //返回消息  
    private String message;  
  
    //返回数据  
    private T data;  
  
    //整合TraceID  
    //public String traceId = TraceIdUtils.getTraceId();   
      
      
    /**  
     * 操作成功ok方法  
     */  
  
    public static <T> R<T> ok() {  
  
        R<T> response = new R<>();  
        response.setCode(ResponseConstantEnum.SUCCESS.getResultCode());  
        response.setMessage(ResponseConstantEnum.SUCCESS.getResultMessage());  
        return response;  
    }  
    public static <T> R<T> ok(T data) {  
  
        R<T> response = new R<>();  
        response.setCode(ResponseConstantEnum.SUCCESS.getResultCode());  
        response.setMessage(ResponseConstantEnum.SUCCESS.getResultMessage());  
        response.setData(data);  
        return response;  
    }  
  
    /**  
     * 编译失败方法  
     */  
    public static <T> R<T> error() {  
        R<T> response = new R<>();  
        response.setCode(ResponseConstantEnum.SYS_ERROR.getResultCode());  
        response.setMessage(ResponseConstantEnum.SYS_ERROR.getResultMessage());  
        return response;  
    }  
  
  
    public static <T> R<T> error(ResponseConstantEnum errorEnum){  
  
        R<T> response = new R<>();  
        response.setCode(errorEnum.getResultCode());  
        response.setMessage(errorEnum.getResultMessage());  
        return response;  
    }  
  
    public static <T> R<T> error(ResponseConstantEnum errorEnum, String errorMsg){  
  
        R<T> response = new R<>();  
        response.setCode(errorEnum.getResultCode());  
        response.setMessage(errorMsg);  
        return response;  
    }  
  
  
    public static <T> R<T> error(String s) {  
        R<T> response = new R<>();  
        response.setCode(ResponseConstantEnum.ERROR.getResultCode());  
        response.setMessage(s);  
        return response;  
    }  
}
```
针对需要返回字符串的接口，添加一个注解，统一返回拦截中会忽略打了这个注解的接口，不包装为R类
```java
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface NotRestController { }
```

统一错误拦截Advice
```java
@Slf4j  
@RestControllerAdvice  
public class ControllerExceptionAdvice {  
	//自行添加需要拦截的错误类型和R类信息
    @ExceptionHandler(BindException.class)  
    public R<?> MethodArgumentNotValidException(BindException e) {  
        ObjectError error = e.getBindingResult().getAllErrors().getFirst();  
        return R.error(ResponseConstantEnum.VALIDATE_ERROR, error.getDefaultMessage());  
    }  
  
    @ExceptionHandler(Exception.class)  
    public R<?> Exception(Exception e) {  
        log.error("未知异常！", e);  
        return R.error(ResponseConstantEnum.ERROR);  
    }  
}
```
统一返回拦截
```java
@RestControllerAdvice(basePackages = "org.example.springbootstudy.controller")  
public class ControllerResponseAdvice implements ResponseBodyAdvice<Object> {  
    @Resource  
    private ObjectMapper objectMapper;  
  
    @Override  
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {  
        //如果是R类型或者注释了@NotControllerResponseAdvice都不进行包装  
        return !(returnType.getParameterType().isAssignableFrom(R.class)  
                || returnType.hasMethodAnnotation(NotRestController.class));  
    }  
  
    @Override  
    public Object beforeBodyWrite(Object body,  
                                  MethodParameter returnType,  
                                  MediaType selectedContentType,  
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,  
                                  ServerHttpRequest request,  
                                  ServerHttpResponse response) {  
        //String类型不能直接包装  
        if (body instanceof String){  
            try {  
                //将数据包装在R里后转换为json串返回  
                return objectMapper.writeValueAsString(R.ok(body));  
            } catch (JsonProcessingException e){  
                e.printStackTrace();  
            }  
        }  
        //否则直接包装成R返回  
        return R.ok(body);  
    }  
}
```
