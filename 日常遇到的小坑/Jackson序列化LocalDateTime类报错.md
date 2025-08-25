在Config中创建自定义的ObjectMapper Bean
[全局ObjectMapper](../Spring/SpringBoots/Jackson.md#springboot中注册自定义的全局objectmapper)


定义序列化器
```java
@JsonComponent
public class MyLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getText();
        try {
            if (text == null || "null".equals(text) || text.isEmpty() || "undefined".equals(text)) {
                return null;
            }
            else if (text.matches("^-?\\d+$")) {
                // 时间戳转LocalDateTime
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(Long.parseLong(text)), ZoneId.systemDefault());
            } else {
                // 字符串转LocalDateTime
                try {
                    // 尝试按完整的日期时间格式解析
                    return LocalDateTime.parse(text, DATE_TIME_FORMATTER);
                } catch (DateTimeParseException e) {
                    // 尝试按日期格式解析
                    try {
                        LocalDate date = LocalDate.parse(text, DATE_FORMATTER);
                        // 将日期转换为带有时间部分的 LocalDateTime
                        return date.atStartOfDay();
                    } catch (DateTimeParseException ex) {
                        // 如果仍然解析失败，抛出异常
                        throw new IllegalArgumentException("Invalid date or datetime format: " + text, ex);
                    }
                }
            }
        } catch (Exception e) {
            throw new IOException("Could not parse '" + text + "'");
        }
    }
}
```

LocalDate反序列化器

```java
@JsonComponent
public class MyLocalDateDeserializer extends JsonDeserializer<LocalDate> {
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    @Override
    public LocalDate deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getText();
        try {
            if (text == null || "null".equals(text) || text.isEmpty() || "undefined".equals(text)) {
                return null;
            }
            else if (text.matches("^-?\\d+$")) {
                // 时间戳转LocalDateTime
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(Long.parseLong(text)), ZoneId.systemDefault()).toLocalDate();
            } else {
                // 字符串转LocalDateTime
                try {
                    // 尝试按正确的日期格式解析
                    return LocalDate.parse(text, DATE_FORMATTER);
                } catch (DateTimeParseException e) {
                    // 尝试按日期格式解析
                    try {
                        LocalDateTime date = LocalDateTime.parse(text, DATE_TIME_FORMATTER);
                        // 将日期转换为带有时间部分的 LocalDateTime
                        return date.toLocalDate();
                    } catch (DateTimeParseException ex) {
                        // 如果仍然解析失败，抛出异常
                        throw new IllegalArgumentException("Invalid date or datetime format: " + text, ex);
                    }
                }
            }
        } catch (Exception e) {
            throw new IOException("Could not parse '" + text + "'");
        }
    }
}
```





手动使用`objectMapper.readValue`：

```java
@Resource  
ObjectMapper objectMapper;

String json = "{\n" +  
        "  \"create_time\": \"2020-08-06 08:06:06\"\n" +  
        "}";  
  
Person person = objectMapper.readValue(json, Person.class);
```

自动转换不需要做自动注入

字段上不需要加上`@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")`