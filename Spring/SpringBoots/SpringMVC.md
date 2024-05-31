# 获取单个&多个参数
> URL传参：http://127.0.0.1:8080/web/hello?age=18&name="小明"

```java
@GetMapping("/test")  //不推荐
public String test(HttpServletRequest request, HttpServletResponse response){  
    Integer age = request.getParameter("age");  
    return "OK";  
}

//推荐，顺序不要求，只要变量名能对上就行，且里面建议使用包装类，否则为空会报错
@GetMapping("/test")  
public String test(String name, Integer age){  
    return "OK";  
}

//推荐，用实体类接收，Spring MVC 会去这个实体类里解析它所有的属性, 然后调用属性的 setter 方法, 将传递过来的值根据 key 值设置进去.
//参数名是大小写敏感的, 如果大小写不一样, 依然获取不到参数. 建议全部使用小写, 不要使用小驼峰
@GetMapping("/test")  
public String test(User user){  
    return "OK";  
}
```

前端表单发送时，或者后端获取ajax传递的参数时，上面的方法也能获取到值
当后端返回Map类型时，会自动识别键值对格式并转换成json格式返回给前端，因为加了`@ResponseBody`，当类上加了`@RestController`时，相当于给所有方法加上了`@ResponseBody`



# 接收请求体内的信息
当请求在Body里附带了json信息时，则必须在参数中使用`@RequestBody`才能接收到，并且需要Json数据键名与形参对象里面的属性名相同该对象才可以接收
```json
{
	"id": 1,
	"name": "小明",
	"age": 18,
	"sex": "男"
}
```

```java
@GetMapping("/test")  
public String test(@RequestBody User user){  
	HashMap<String, Object> map = new HashMap<>();
	map.put("id", user.getId());
	...
    return map;  
}
```

> 1. 当前端传递过来 JSON 格式的数据时, 后端如果还使用前面的 "接收多个参数" 的形式来接收, 又或者是通过 "接收普通对象" 的方式来接收, 此时就接收不到数据了, 此时通过 postman 发送请求时, 响应里的键对应的 value 值就都为 null 了.
> 2. 接收 JSON 格式的数据时, 后端需要使用 @RequestBody 注解来实现, @RequestBody 注解就可以从 body 中拿到 JSON 格式的数据. 并且接收 JSON 对象时, 只能是以对象的形式来接收, 不能写成多个参数的形式.
> 3. @requestBody 注解只能接收 JSON 对象, 不能用于接收普通对象, 不能混着用, 需要配套使用.


# 接收文件@RequestPart
```java
@PostMapping("/upload")  
public String upload(String name, @RequestPart("myfile") MultipartFile file) throws IOException {  
    //保存文件  
    boolean result = false;
	//得到原图片的名称
	String filename = file.getOriginalFilename();
	//得到图片后缀（png)
	filename = filename.substring(filename.lastIndexOf("."));
	//生成不重名的文件名
	filename = UUID.randomUUID().toString() + filename;
	try {
		//将图片保存
		file.transferTo(new File(imgpath + filename));
		result = true;
	} catch (IOException e) {
		log.error("图片上传失败!" + e.getMessage());
	}
	return result;
}
```
`@RequestPart("myfile")`对应的是请求体中文件的key
在开发中应使用配置文件分生产和测试环境写不同的文件保存路径，比如在配置文件中
```yml 
# 文件保存路径
img:
	path: d:\\aa\\

spring:
	servlet:
		multipart:
			max-file-size=10MB
			max-request-size=10MB
```


# 获取Cookies @CookieValue
`@CookieValue("xxx") String value`获取key为xxx的Cookie的值，获取多个就加多个参数

# 获取Session @SessionAttribute
`@SessionAttribute("xxx", required=false) String value`获取key为xxx的Session的值，可以添加required参数

# 获取Header @RequestHeader
`@RequestHeader("xxx") String value`

# 后端参数重命名 @ReuqestParam
`@ReuqestParam("t") String time`将传入的t重命名为time

# 获取URL里的参数 @PathVariable
```java
@GetMapping("/path/{id}")  
public String pathParam(@PathVariable("id") Integer id) {  
    return "OK";  
}
```
若同名可写为`@PathVariable Integer id`


# 转发
```java
public String jump() {  
    return "forward:/index.html";  //服务器端转发，请求转发地址不改变
    return "redirect:/index.html";  //将请求重新定位，请求重定向地址会改变
}
```
