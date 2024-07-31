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