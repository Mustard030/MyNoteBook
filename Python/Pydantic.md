由于FastAPI与Pydantic深度绑定，因此在示例中会涉及到一部分FastAPI的内容。

`BaseModel`
FastAPI 现在底层依赖的是 **Pydantic V2**。
- **`class Config`**: 这是 **Pydantic V1** 时代的写法。虽然 V2 为了兼容旧代码仍然支持它，但它属于“二等公民”。
- **`model_config`**: 这是 **Pydantic V2** 引入的官方推荐写法。
优势：类型提示 (IDE Autocomplete)
- **`class Config` (旧写法)** 当你写 `class Config:` 时，它只是一个普通的 Python 类。IDE **根本不知道**里面能写什么属性。如果你把 `orm_mode` 拼写成了 `orm_mod`，编辑器不会报错。
- **`model_config` (新写法)** 你需要导入 `ConfigDict`。这是一个 TypedDict。当你写代码时，**IDE 会自动补全所有可用的配置项**。如果你写了一个不存在的配置，编辑器会直接标红报错。

```python
from pydantic import BaseModel, ConfigDict

class UserNew(BaseModel):
    id: int
    name: str
    
    # V1写法
    class Config: 
	    orm_mode = True # V2 中这其实叫 from_attributes，旧名是为了兼容保留的 allow_population_by_field_name = True

	# V2写法
    model_config = ConfigDict(
        from_attributes=True,          # 原 orm_mode
        populate_by_name=True,         # 原 allow_population_by_field_name
        str_max_length=100             # V2 新增了很多方便的配置
    )
```

**常用配置项名称变更对照表**

|**旧版参数 (class Config)**|**新版参数 (model_config)**|**作用**|
|---|---|---|
|`orm_mode`|**`from_attributes`**|允许从对象属性读取数据 (ORM 必备)|
|`allow_population_by_field_name`|**`populate_by_name`**|允许通过别名赋值 (比如 `_id` 和 `id`)|
|`anystr_strip_whitespace`|**`str_strip_whitespace`**|自动去除字符串首尾空格|
|`schema_extra`|**`json_schema_extra`**|自定义 Swagger 文档示例|


`HttpUrl`
`HttpUrl` 是 **Pydantic** 库提供的一个特殊数据类型，专门用于**验证和处理 URL**。
核心作用：
当你声明一个字段类型为 `HttpUrl` 时，Pydantic 会自动做两件事：
1. 严格校验(Validation)
	- 它必须是一个合法的 URL。
	- 它必须以 `http://` 或 `https://` 开头（这也是为什么它叫 **Http**Url）。
	- 域名必须合法（不能是乱码）。
	- 最大长度限制（默认 2083 字符）。
2. 自动解析(Parsing)
	- 它不仅仅是一串字符，它会被解析成一个对象，你可以轻松获取它的 `host`（主机名）、`port`（端口）、`path`（路径）等。

在 FastAPI 现在的版本（基于 Pydantic V2）中，`HttpUrl` 的行为和以前不一样了：
- **它不再是字符串的子类**。
- 不能直接拿它和字符串拼接（比如 `'Link: ' + url` 会报错）。
- **解决方法**：在使用时，必须显式地把它转为字符串：**`str(url)`**。
```python
# 假设 url 是 HttpUrl 类型
print(url)           # 输出: Url('https://google.com/') (它是对象)
print(str(url))      # 输出: "https://google.com/" (这才是字符串)
```
类似地，还有`WebsocketUrl`, `FileUrl`, `FtpUrl`, `PostgresDsn`, `RedisDsn`等等，大多数时候只是他们的约束不同，比如协议头。

**注意**: 它是一个对象，存入数据库或发送请求前，记得用 `str(url)` 转为字符串。

