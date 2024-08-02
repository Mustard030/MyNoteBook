更简单地，直接注册ObjectMapper，并调用`findAndRegisterModules()`即可
```java
@Configuration  
public class JacksonConfig {  
    @Bean  
	@Primary  
	@ConditionalOnMissingBean(ObjectMapper.class)  
	public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder)  {  
	    // 保留Springboot中的自动配置  
	    ObjectMapper objectMapper = builder.createXmlMapper(false).build();  
	    objectMapper.findAndRegisterModules();  // 核心，下面的按需添加
	      
	    SimpleModule module = new SimpleModule();  
	    //Long类型的反序列化器，防止丢失精度  
	    module.addSerializer(Long.class, ToStringSerializer.instance);  
	    module.addSerializer(Double.class, ToStringSerializer.instance);  
	    module.addSerializer(Double.TYPE, ToStringSerializer.instance);  
	      
	    module.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
	    module.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
	    module.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
	    module.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
	    module.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));  
	    module.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));  
	      
	    objectMapper.registerModule(module);  
	      
	    // 在序列化时，忽略值为 null 的属性  
	    objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);  
	  
	    // 在反序列化时，忽略在 json 中存在但 Java 对象不存在的属性。  
	    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  
	      
	    return objectMapper;  
	}
}
```


创建Config
```java
@Configuration  
//@Slf4j  
public class JacksonConfig {  
  
    public static JavaTimeModule buildJavaTimeModule() {  
        JavaTimeModule javaTimeModule = new JavaTimeModule();  
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DatePattern.NORM_TIME_FORMATTER));  
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DatePattern.NORM_DATE_FORMATTER));  
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DatePattern.NORM_DATETIME_FORMATTER));  
  
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DatePattern.NORM_TIME_FORMATTER));  
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(DatePattern.NORM_DATE_FORMATTER));  
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DatePattern.NORM_DATETIME_FORMATTER));  
        return javaTimeModule;  
    }  
  
    @Bean  
    @ConditionalOnMissingBean    public Jackson2ObjectMapperBuilderCustomizer customizer() {  
        // log.info("[Jackson2ObjectMapperBuilderCustomizer][初始化customizer配置]");  
        return builder -> {  
            builder.locale(Locale.CHINA);  
            builder.timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()));  
            builder.simpleDateFormat(DatePattern.NORM_DATETIME_PATTERN);  
            builder.serializerByType(Long.class, ToStringSerializer.instance);  
            builder.modules(buildJavaTimeModule());  
        };  
    }  
}
```

使用：
```java
@Resource  
ObjectMapper objectMapper;

String json = "{\n" +  
        "  \"create_time\": \"2020-08-06 08:06:06\"\n" +  
        "}";  
  
Person person = objectMapper.readValue(json, Person.class);
```

其中字段上需要加上`@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")`