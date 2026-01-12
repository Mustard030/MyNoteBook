[官网](https://fastapi.tiangolo.com/zh)
**FastAPI**是一个基于**Starlette**（异步Web框架）和**Pydantic**（数据验证）的异步web框架，拥有强大的性能和完整的类型提示。**FastAPI**的核心优势在于 **类型提示 (Type Hints)**、**异步支持 (AsyncIO)** 和 **自动生成文档**。


# 环境准备与安装
- Python 版本要求 (3.8+)
- 安装 FastAPI 与 Uvicorn (`pip install "fastapi[standard]"`)

```bash
# pip
pip install fastapi[standard]
# uv
uv add fastapi[standard]
```

FastAPI需要使用ASGI服务器进行部署，如：Uvicorn
```
# pip
pip install uvicorn[standard]
# uv
uv add uvicorn[standard]
```




# 数据获取与处理
后端的职责就是获取并处理数据，下面将介绍在`fastapi`中如何获取请求的数据。

首先，从`fastapi`包下可以`import`以下工具
**核心构建与路由 (Core & Routing)：**
- `FastAPI`: 应用程序的主入口类。用于创建 app 实例 (`app = FastAPI()`)。
- `APIRouter`: 路由分组工具。用于将路由拆分到不同的模块中，实现模块化开发。
**参数获取与声明 (Parameter Declaration)：**
- `Path`: 声明参数来自 **URL 路径** (例如 `/users/{id}` 中的 `id`)。
- `Query`: 声明参数来自 **URL 查询字符串** (例如 `/users?page=1` 中的 `page`)。
- `Body`: 声明参数来自 **HTTP 请求体** (通常是 JSON 数据)。
- `Header`: 声明参数来自 **HTTP 请求头**。
- `Cookie`: 声明参数来自 **HTTP Cookie**。
- `Form`: 声明参数来自 **表单数据** (`application/x-www-form-urlencoded`)。 *注意：需要安装 `python-multipart` 库。常用于传统网页表单提交。*
- `File`: 声明参数是 **上传的文件** (`multipart/form-data`)。*注意：通常配合 `UploadFile` 类型一起使用。*
- `UploadFile`: 文件对象类型注解。*用途：相比 `bytes`，它使用“假脱机”文件（SpooledTemporaryFile），适合处理大文件上传，提供 `.read()`, `.write()`, `.save()` 等方法。*
**依赖注入与安全 (Dependencies & Security)：**
- `Depends`: 声明依赖关系。FastAPI 会自动执行该函数，并将其返回值注入到视图函数中。
- `Security`: `Depends` 的高级版。*它不仅像 `Depends` 一样注入依赖，还会在 Swagger UI 文档中定义安全方案（如 OAuth2 按钮、API Key 输入框）。*
**请求与响应对象 (Request & Response)：**
- `Request`: 原始请求对象 (来自 Starlette)。*用途：获取客户端 IP、访问原始请求体、获取完整的 URL 等。*
- `Response`: 原始响应对象。*用途：手动设置 Cookie、Header，或者返回非 JSON 内容。通常作为父类，也有 `JSONResponse`, `HTMLResponse` 等子类。*
**WebSocket 长连接 (WebSockets)：**
- `WebSocket`: WebSocket 连接对象。*用途：用于 `await websocket.accept()`, `await websocket.send_text()`, `await websocket.receive_text()` 等操作。*
- `WebSocketDisconnect`: 断开连接异常。*用途：当客户端断开连接时（如关闭浏览器），`receive` 会抛出此异常，需要在代码中捕获处理。*
- `WebSocketException`: WebSocket 专用异常。*用途：用于在建立连接阶段或通信中主动拒绝/关闭连接（例如认证失败），并附带特定的 WebSocket 关闭码。*
**异常与状态码 (Exceptions & Status)：**
- `HTTPException`: HTTP 错误异常类。*在业务逻辑中直接 `raise HTTPException(status_code=404, detail="...")` 来中断请求并返回错误。*
- `status`: 各种HTTP请求状态码。*避免手写数字。例如使用 `status.HTTP_200_OK` 代替 `200`，`status.HTTP_404_NOT_FOUND` 代替 `404`。*
**后台任务 (Async Tasks)：**
- `BackgroundTasks`: 后台任务管理器。*在返回 HTTP 响应**之后**执行耗时任务（如发送邮件、日志记录），不阻塞用户等待响应。*




## Annotated
在Python3.9+中`typing`提供了一个注解工具`Annotated`，它的作用是允许在变量的类型提示中附加额外的元数据。因为`FastAPI`强依赖于参数类型，曾经单一的类型提示无法满足多元数据的校验。
`Annotated` 的结构如下：$Annotated[T, x]$
- `T`: 真正的类型（如 `str`, `int`, `User`）。
- `x`: 附加的元数据（`FastAPI` 用它来放 `Depends()`, `Query()`, `Path()` 等）。

示例：
在老写法中，类型定义和逻辑校验混在默认值中，IDE会困惑（声明了类型是 `str`，但赋值给它的默认值却是一个 `Query` 对象。）且无法复用
```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
# 这里的 default=None 实际上是在 Query() 里面定义的
# 这里的类型 str | None 仅仅告诉编辑器它可能是 None
def read_items(q: str | None = Query(default=None, max_length=50)):
    return {"q": q}
```
本来应该书写默认值的地方，但现在变成了Query的校验。

在新写法中，类型和校验逻辑全部放在等号左边（类型声明区），等号右边回归纯粹的**默认值**。
```python
from typing import Annotated

@app.get("/items/{item_id}") 
def read_items(
	q: Annotated[str | None, Query(max_length=50)] = None
): 
	return {"q": q}
...
```

更进一步地，`Annotated` 在 FastAPI 中最强大的地方在于：**它可以让你定义可复用的依赖类型。**
```python
from fastapi import Depends, FastAPI

def get_current_user():
    return "User123"

@app.get("/users/me")
def read_users_me(user: str = Depends(get_current_user)):
    return user

@app.get("/users/items")
def read_user_items(user: str = Depends(get_current_user)): # 重复写 Depends
    return [item]
```
**新写法 (DRY 原则 - Don't Repeat Yourself)** 你可以定义一个**类型别名**。
```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

def get_current_user():
    return "User123"

# 🌟 定义一次，到处使用！
# 创建一个自定义类型，只要某个参数标注为此类型，FastAPI 就知道要自动注入 get_current_user
CurrentUser = Annotated[str, Depends(get_current_user)]

@app.get("/users/me")
def read_users_me(user: CurrentUser): # 极其简洁，就像普通参数一样
    return user

@app.get("/users/items")
def read_user_items(user: CurrentUser): # 直接复用
    return ["Item1", "Item2"]
```
示例2：全局数据库会话管理
```python
from typing import Annotated, Generator
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session
# 假设你已经配置好了 database (这里为了演示简化了 import)
# from .database import SessionLocal, engine 

app = FastAPI()

# 1. 传统的依赖函数 (负责创建和关闭连接)
def get_db() -> Generator:
    db = SessionLocal() # 创建连接
    try:
        yield db        # 将连接“借”给路由使用
    finally:
        db.close()      # 路由执行完后，自动“归还”并关闭连接

# ---------------------------------------------------------
# 🌟 2. 使用 Annotated 定义“通用依赖类型” (这是核心！)
# 翻译：SessionDep 是一个 Session 类型，但它不仅是 Session，
# 它还捆绑了 Depends(get_db) 这个获取逻辑。
# ---------------------------------------------------------
SessionDep = Annotated[Session, Depends(get_db)]


# 3. 在路由中使用
@app.post("/users/")
def create_user(
    user_name: str, 
    db: SessionDep  # 👈 看这里！不需要写 = Depends(get_db) 了
):
    # 此时 db 已经是一个打开的数据库会话实例
    # 可以直接使用 SQLAlchemy 的方法
    # new_user = User(name=user_name)
    # db.add(new_user)
    # db.commit()
    return {"message": f"User {user_name} created successfully"}

@app.get("/users/{user_id}")
def read_user(
    user_id: int, 
    db: SessionDep  # 👈 第二次使用，依然无比简洁
):
    # user = db.query(User).get(user_id)
    return {"user_id": user_id}
```


优点：
1. **语义分离**：它将“**这个参数是什么类型**”（Python 关注）和“**如何获取/校验这个参数**”（FastAPI 关注）完美结合但在语法上又解耦了。
2. **减少代码重复**：如上面的 `CurrentUser` 例子，这在大型项目中非常关键（特别是数据库 Session 的注入）。
3. **IDE 友好**：编辑器（VS Code, PyCharm）能完美识别 `Annotated[str, ...]` 就是 `str`，从而提供准确的代码补全，而不会被 `Depends` 等逻辑干扰。
	- **左侧声明，右侧默认**：让函数签名更符合 Python 原生习惯。
	- **类型别名 (Type Alias)**：可以把 `Depends` 封装在类型里，实现 `SessionDep` 或 `UserDep` 的全局复用。






## 核心构建与路由 (Core & Routing)

### FastAPI
它是所有功能的入口：路由管理、中间件、异常处理、文档生成、生命周期管理全部由它统筹。所有的 `APIRouter`、中间件、事件处理最终都要挂载到它身上，才能被 Uvicorn 运行。
#### 初始化参数 (The Constructor)
当你执行 `app = FastAPI(...)` 时，你有很多配置选项。虽然参数很多，但常用的就这几类：

##### 文档元数据 (用于美化 Swagger UI)
这些参数决定了你打开 `http://127.0.0.1:8000/docs` 时看到什么。
```python
app = FastAPI(
    title="我的超级 API",
    description="这是一个用于管理用户的系统...",
    version="1.0.0",
    terms_of_service="http://example.com/terms/",
    contact={"name": "Jerry", "email": "jerry@example.com"},
    license_info={"name": "MIT"}
)
```

##### 文档 URL 配置 (安全性与定制)
你可以修改文档的路径，或者在生产环境中**彻底关闭**文档（为了安全）。
```python
app = FastAPI(
    docs_url="/api-docs",    # 默认是 /docs
    redoc_url="/api-redoc",  # 默认是 /redoc
    openapi_url="/openapi.json" # 默认是 /openapi.json
)

# 🔒 生产环境安全配置：禁用文档
app_prod = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)
```

##### `lifespan`
在 `FastAPI(lifespan=...)` 参数中，你**只能传入一个**函数。你不能像以前写 `@app.on_event("startup")` 那样写十个函数然后指望它们自动依次执行。

**为什么设计成“只能传一个”？**
以前的 `@app.on_event("startup")` 允许定义多个，但有两个严重问题：
1. **执行顺序不明确**：你很难控制哪个先运行，哪个后运行。
2. **状态共享困难**：如何在 `shutdown` 时拿到 `startup` 创建的数据库连接对象？以前只能靠全局变量，很不安全。
现在的 `lifespan` 通过 `yield` 机制，强制你把 **创建** 和 **销毁** 写在同一个函数里，利用上下文管理器的特性，**逻辑上的依赖关系（谁包裹谁）变得非常清晰**。

更多请看[生命周期管理](#生命周期管理)


##### `debug`
设置为 `True` 时，报错会显示详细的堆栈信息（生产环境务必关掉）。


#### 核心方法 (Methods)
##### `include_router` (模块化)
这是最重要的功能。它把分散的 `APIRouter` 组装起来。
```python
from routers import users, items

app.include_router(users.router)
app.include_router(items.router, prefix="/api/v1")
```
##### `add_middleware` (添加中间件)
用于处理跨域 (CORS)、GZip 压缩、或者自定义请求拦截。
```python
from fastapi.middleware.cors import CORSMiddleware

# 允许跨域请求（前端分离必配）
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], # 允许所有来源
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

##### `add_exception_handler` (全局异常捕获)
如果你想把系统的报错（比如 500 错误）变成自定义的 JSON 格式，或者捕获特定的业务异常。
```python
from fastapi import Request
from fastapi.responses import JSONResponse

class MyCustomException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(MyCustomException)
async def unicorn_exception_handler(request: Request, exc: MyCustomException):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} did something wrong."},
    )
```

##### `mount` (挂载静态文件/子应用)
这是用来托管静态资源（CSS, JS, 图片）的。
```python
from fastapi.staticfiles import StaticFiles

# 访问 /static/image.png -> 读取本地 "static" 目录下的文件
app.mount("/static", StaticFiles(directory="static"), name="static")
```


#### 高级配置参数
##### `default_response_class` (性能优化)
如果你确定你的 API 永远只返回 JSON，并且追求极致性能，可以替换默认的 JSON 解析器（例如使用 `ORJSON`）。
```python
from fastapi.responses import ORJSONResponse

# 这样全局所有接口都会默认使用更快的 ORJSONResponse
app = FastAPI(default_response_class=ORJSONResponse)
```

##### `root_path` (代理配置)
当你使用 Nginx 或 Docker 将应用部署在某个子路径下（比如 `https://example.com/api/v1/` 代理到你的 `http://localhost:8000/`）时，Swagger UI 可能会坏掉（找不到 `openapi.json`）。
你需要告诉 FastAPI 它被代理了：
```python
# 告诉 FastAPI："虽然你在根目录运行，但在外界看来，你的前缀是 /api/v1"
app = FastAPI(root_path="/api/v1")
```

##### 全局状态存储 (`app.state`)
`FastAPI` 实例有一个 `state` 属性，它可以作为一个**全局的内存容器**。这和 `request.state` 很像，但 `app.state` 是应用级别的，生命周期更长。
```python
# main.py
app = FastAPI()
app.state.admin_email = "admin@example.com"
app.state.db_pool = None # 稍后在 lifespan 里填充

@app.get("/info")
async def info(request: Request):
    # 可以通过 request.app 访问到主 app
    return {"admin": request.app.state.admin_email}
```




#### 总结

| **功能维度**  | **常用参数/方法**                         | **作用**                 |
| --------- | ----------------------------------- | ---------------------- |
| **文档**    | `title`, `description`, `version`   | 装修 Swagger UI 门面       |
| **安全/隐藏** | `docs_url=None`, `openapi_url=None` | 生产环境隐藏文档接口             |
| **启动/关闭** | `lifespan`                          | 数据库连接、模型加载             |
| **扩展能力**  | `.add_middleware()`                 | 处理 CORS, GZip, Session |
| **模块组装**  | `.include_router()`                 | 挂载子模块                  |
| **静态资源**  | `.mount("/static", ...)`            | 托管图片、前端文件              |
| **错误处理**  | `.add_exception_handler()`          | 自定义报错格式                |
| **部署**    | `root_path`                         | 配合 Nginx 反向代理使用        |



### APIRouter
通常我们不会直接把url挂在`FastAPI`对象下，因为这意味着这个url不属于任何一个模块，是一个完全公共的url。
因此我们通常根据一个模块一个`APIRouter`的规则去规划这个系统。
例如：

```python
# users.py
router = APIRouter(prefix="/users", tags=["User"])

# main.py
app.include_router(router, prefix="/v1", tags=["V1"])

# ---------------------------------------------
# 最终结果：
# 路径 URL:  /v1/users/  (叠加了)
# 文档标签:  ["User", "V1"] (叠加了)
# ---------------------------------------------
```




`APIRouter`的可配置项有很多，这些参数分为三大类：**路由控制**、**文档元数据** 和 **逻辑行为**：

#### 1. 路由控制类参数 (Path & Routing)
##### `prefix` (最常用)
- **类型**: `str`
- **作用**: 给该 Router 下定义的所有路径加上统一的前缀。
- **⚠️ 注意**: 前缀**不要**以 `/` 结尾（FastAPI 会自动处理拼接，写了容易造成双斜杠 `//`）。

##### `redirect_slashes`
- **类型**: `bool`
- **默认值**: `True`
- **作用**: 控制是否自动重定向斜杠。
- **详解**:
    - 如果你定义了接口 `@router.get("/items/")` (带尾部斜杠)。
    - 用户访问 `/items` (不带斜杠)。
    - **默认 (True)**: FastAPI 会返回 `307 Temporary Redirect` 跳转到 `/items/`。
    - **设为 False**: FastAPI 会直接返回 `404 Not Found`。



#### 2.文档与元数据类参数 (Docs & Metadata)
这些参数主要影响 Swagger UI (`/docs`) 和 ReDoc 的显示效果。

##### `tags`
- **类型**: `List[str]` 或 `List[Enum]`
- **作用**: 在文档中对接口进行**分组**。如果不写，接口会混在一起；写了之后，Swagger UI 会把它们折叠在一个大标题下。

##### `deprecated`
- **类型**: `bool` (默认 `False`)
- **作用**: 将该 Router 下的**所有**接口标记为“已过时”。文档中会显示一条删除线，但接口依然可用。适合版本迁移时使用。

##### `include_in_schema`
- **类型**: `bool` (默认 `True`)
- **作用**: 是否在文档中显示该 Router 的接口。如果设为 `False`，这些接口就像“隐形”了一样（通常用于内部管理接口）。

##### `generate_unique_id_function`
- **类型**: `Callable[[APIRoute], str]`
- **默认值**: `Default` (使用函数名)
- **痛点**:
    - 如果你有两个路由：`items.py` 里有个函数叫 `get_list()`，`users.py` 里也有个函数叫 `get_list()`。
    - 生成的 OpenAPI `operationId` 可能会冲突或变得很难看（如 `get_list_items_get_list_get`）。
- **作用**: 自定义生成 `operationId` 的规则。这对前端生成客户端代码（如使用 `openapi-generator`）非常重要。
```python
def custom_id_generator(route: APIRoute):
    # 生成格式例如: "items:get_list"
    return f"{route.tags[0]}:{route.name}"

router = APIRouter(generate_unique_id_function=custom_id_generator)
```
##### `callbacks`
- 类型: `Optional[list[BaseRoute]]`
- 默认值: `None`
- 作用：有时候我们提供的API是一些长时间的任务，这种任务一般会在API调用的时候立刻返回一个“创建任务成功”之类的结果，并开始后台运行这个任务。等到任务结束之后，将会主动向原调用方发送一个回调请求，这叫做**Webhooks**。而这个参数将会帮你在docs文档中显示你的回调请求会发送什么样的数据格式，发送到什么地址上。这样就能方便文档使用者提前预知回调请求的格式并进行开发。
```python
from fastapi import APIRouter, FastAPI
from pydantic import BaseModel, HttpUrl

# 1. 定义 webhook 的数据结构 (发给别人的数据)
class NotificationMsg(BaseModel):
    event_id: str
    status: str

# 2. 创建一个专门用于描述 webhook 的 Router
callback_router = APIRouter()

# 定义 webhook 的样子：比如我们会发一个 POST 到对方的 {$callback_url}
@callback_router.post("{$callback_url}/notify")
def notification_webhook(body: NotificationMsg):
    """
    这里写文档：当任务完成时，我会向你提供的 URL 发送这个 POST 请求。
    """
    pass

# 3. 在主 Router 中引用它
# 注意：这里演示的是 Router 级别的 callbacks，意味着该 Router 下所有接口
# 都会在文档里显示这个 Webhook 说明。
msg_router = APIRouter(
    callbacks=callback_router.routes  # 👈 核心参数
)

@msg_router.post("/start-task/")
def start_task(callback_url: HttpUrl):
    # 业务逻辑：启动任务...
    # 业务逻辑：任务结束后，你需要自己写代码发送 HTTP 请求到 callback_url
    return {"msg": "Task started"}
```

> `{$callback_url}` 这种写法并不是 Python 的语法，而是 **OpenAPI 规范（Runtime Expression）** 的语法。
> 简单来说：**它是写给 Swagger 文档（前端）看的“动态占位符”，而不是给 Python 解释器（后端）执行的代码。**
> 核心含义：OpenAPI 运行时表达式
> 在 FastAPI 中，当你定义 `callbacks` 时，你是在描述一个“未来会发生的请求” 。
> **普通路由**：`@app.get("/items/{id}")` : `{id}` 是 Python 需要解析的变量。
> **Callback 路由**：`@router.post("{$callback_url}/notify")` : `{$callback_url}` 是 **OpenAPI** 的特殊语法。
> **含义**：“去**触发这个回调**（原始请求）的那个接口里，找到一个叫 `callback_url` 的参数，把它的值填到这里。”


OpenAPI 允许通过 `$` 符号引用原始请求的不同部分：

| **写法**                             | **含义**                     | **场景**                 |
| ---------------------------------- | -------------------------- | ---------------------- |
| **`{$url}`**                       | 原始请求的完整 URL                | 回调地址就是原始请求地址本身         |
| **`{$method}`**                    | 原始请求的 HTTP 方法 (GET/POST)   | 动态复用方法                 |
| **`{$request.query.url}`**         | 引用原始请求的**查询参数** `?url=...` | 参数在 URL 问号后面时          |
| **`{$request.body.callback_url}`** | 引用原始请求 **Body 体**内的字段      | 参数在 JSON Body 里时 (最常用) |
| **`{$response.body.token}`**       | 引用原始请求**响应**里的数据           | 用服务器返回的 Token 进行回调     |

**FastAPI 的智能化**： 在 FastAPI 中，如果你直接写 `{$callback_url}`（省略了 `.request.body` 等前缀），FastAPI 会尝试智能匹配参数名。但为了标准起见，在复杂的 OpenAPI 定义中，通常建议写全路径，如 `{$request.body.callback_url}`。


#### 3.逻辑行为类参数 (Logic & Behavior)
这是 `APIRouter` 最强大的地方，可以实现**批量配置**。

##### `dependencies` (核心)
- **类型**: `List[Depends]`
- **作用**: 给该 Router 下的**所有**接口强制加上依赖。
- **场景**: 比如 `/admin` 模块下的所有接口都需要管理员权限，你不需要在每个 `@router.get` 里写 `Depends(check_admin)`，直接在 `APIRouter` 里定义一次即可。

**用法一：在创建 Router 时指定**
例如：
```python
from fastapi import APIRouter, Depends, HTTPException, Header
from typing import Annotated

# 1. 定义一个依赖函数
async def verify_token(x_token: Annotated[str, Header()]):
    if x_token != "secret-password":
        raise HTTPException(status_code=400, detail="Token 无效")

# 2. 创建 Router 时注入 dependencies
# 注意：这里接收的是一个列表 list[Depends]
router = APIRouter(
    prefix="/users",
    tags=["users"],
    dependencies=[Depends(verify_token)]  # 👈 关键！一键应用到下方所有路由
)

# 3. 下面的路由函数，不需要再写 Depends(verify_token) 了
# 它们会自动执行 verify_token，如果不通过会直接报错
@router.get("/")
async def read_users():
    return [{"username": "Rick"}, {"username": "Morty"}]

@router.get("/me")
async def read_me():
    return {"username": "Current User"}
```

**用法二：在 `include_router` 时指定**
这种方式适合**“外部控制”**。 比如 `users.py` 模块本身不想关心权限，但是 `main.py` 想在挂载它的时候，给它加一道“防火墙”。
```python
# users.py (模块里很干净，没有任何权限依赖)
router = APIRouter()

@router.get("/")
async def get_users():
    return ["user1", "user2"]

# -------------------------------------------

# main.py (主入口)
from fastapi import FastAPI, Depends
from .users import router as users_router

app = FastAPI()

async def verify_admin():
    # 模拟管理员检查
    pass

# 挂载时强行加上依赖
app.include_router(
    users_router,
    prefix="/admin",
    dependencies=[Depends(verify_admin)] # 👈 在这里加
)
```
**最大的坑：返回值去哪了？**
当你在 `APIRouter(dependencies=[Depends(func)])` 中使用依赖时，**路由函数拿不到依赖的返回值！**
**普通 Depends**: `user = Depends(get_user)` $\to$ 函数里能拿到 `user` 变量。
**Router Depends**: 仅仅是为了**执行副作用 (Side Effects)**，比如：
- 抛出异常 (401 Unauthorized)。
- 记录日志。
- 修改数据库状态。

如果你既想“批量拦截”，又想“在函数里拿到返回值”（比如当前用户对象），该怎么办？
你需要在具体路由里**再写一遍**，或者使用 `request.state` 传递。
```python
# 1. 如果只是做权限拦截（不需要返回值），放 Router dependencies
router = APIRouter(dependencies=[Depends(verify_token)])

# 2. 如果需要拿到具体的 User 对象，还是要在函数参数里写
@router.get("/")
async def root(user: Annotated[User, Depends(get_current_user)]):
    return user
    
# ==========================================
# 1. 定义依赖：校验 Token 并写入 state
# ==========================================
async def verify_token_and_set_state(request: Request):
    """
    这个函数会被 Router 的 dependencies 调用。
    注意：必须声明 request 参数，才能往 request.state 里塞东西。
    """
    token = request.headers.get("x-token")
    
    # 模拟校验失败
    if token != "secret-jerry":
        raise HTTPException(status_code=401, detail="Token 无效")
    
    # 模拟从数据库查到了用户信息
    user_info = {"id": 10086, "username": "Jerry", "role": "admin"}
    
    # 🔥 核心动作：把数据挂载到 request.state 上
    # 比如 request.state.current_user
    request.state.current_user = user_info
    
    print(f"✅ [依赖层] 用户 {user_info['username']} 已写入 state")

# ==========================================
# 2. 定义 Router (统一挂载依赖)
# ==========================================
# 只要请求进入这个 router，verify_token_and_set_state 就会先执行
router = APIRouter(dependencies=[Depends(verify_token_and_set_state)])

# ==========================================
# 3. 具体的路由函数
# ==========================================
@router.get("/me")
async def read_current_user(request: Request):
    """
    路由函数不需要再写 Depends，但需要接收 request 对象
    """
    # 🔥 核心动作：从 state 里取出来
    user = request.state.current_user
    
    return {
        "msg": "从 state 获取成功",
        "user_data": user
    }

@router.get("/dashboard")
async def read_dashboard(request: Request):
    # 另一个接口，同样可以直接取用，因为安检已经通过了
    user = request.state.current_user
    return {"admin_panel": f"Welcome back, {user['username']}"}

# ==========================================
# 4. 挂载到 App
# ==========================================
app = FastAPI()
app.include_router(router, prefix="/users")
```

虽然这种方法解决了问题，但它有明显的副作用，**这也是为什么官方文档更推荐在每个函数里写 Depends 的原因**。

| **维度**         | **显式 Depends 写法 func(user: User = Depends(get_user))** | **隐式 State 写法 request.state.user**             |
| -------------- | ------------------------------------------------------ | ---------------------------------------------- |
| **代码简洁度**      | ❌ 啰嗦，每个函数都要写一遍参数。                                      | ✅ 清爽，函数签名很短。                                   |
| **类型提示 (IDE)** | ✅ **完美**。IDE 知道 `user` 是什么类型，点得出来属性。                   | ❌ **丢失**。IDE 不知道 `request.state` 里有什么，也没有属性提示。 |
| **重构难度**       | ✅ 容易。改了依赖返回值，静态检查会报错。                                  | ❌ 困难。如果依赖里改了属性名，后面代码运行才会报错。                    |
| **可测试性**       | ✅ 容易。单元测试可以直接传对象。                                      | ⚠️ 麻烦。测试时需要 Mock request 对象。                   |



##### `responses`
- **类型**: `dict`
- **作用**: 定义该 Router 下所有接口可能返回的**通用错误响应**。这会让文档更准确。

##### `default_response_class`
- **类型**: `Response` 子类 (默认 `JSONResponse`)
- **作用**: 改变默认的响应格式。默认情况下，FastAPI 使用 `JSONResponse`（基于 Python 标准库 `json` 模块）。 这个参数允许你替换该 Router 下所有接口的**默认响应处理类**。如果这个 Router 是专门用来返回 HTML 页面的，可以设置为 `HTMLResponse`。


在 `APIRouter` 中定义参数很方便，但在 `main.py` 的 `include_router` 中也可以定义同样的参数。规则是：`include_router` 的参数优先/叠加。
- **Prefix (前缀)**: 会**拼接**。
    - Include定义 `/v1` + Router定义 `/users`  = `/v1/users`

- **Tags (标签)**: 会**追加**。
    - Router定义 `["A"]` + Include定义 `["B"]` = `["A", "B"]`

- **Dependencies (依赖)**: 会**追加** (都执行)。
    - 先执行 Include 的依赖，再执行 Router 自身的依赖。

**总结：**
- **`FastAPI()`** 是**服务器实例**，只在 `main.py` 里实例化一次。它关注文档地址、中间件、全局异常等**基础设施**。
- **`APIRouter()`** 是**业务模块**，在每个功能文件（如 `auth.py`, `item.py`）里实例化。它关注**路径前缀**和**路由分组**。
- **连接点**：`app.include_router(router)` 是将两者联系起来的唯一桥梁。
- 尽量在 `APIRouter` 初始化时定义好属于该模块**内部**的属性（如 `prefix` 和 `tags`）。 而在 `include_router` 时定义属于**架构层级**的属性（如版本号前缀 `/v1` 或全局特定的权限依赖）。





## 参数获取与声明 (Parameter Declaration)
### 路径参数(Path)
正常情况下是不需要主动声明参数默认值为`Path`的。
```python
@app.get("/items/{item_id}") 
async def read_item(item_id: int): 
	return {"item_id": item_id}
```
但是在以下几种情况下，你需要手动定义，以下逐项说明：
1. 默认的类型声明 (`int`, `str`) 无法满足需求时，如需要路径转换：需要捕获带斜杠的路径（文件路径）
2. 需要检验这个参数：需要限制数值范围 (`gt`, `le`) 或 字符串格式 (正则 `pattern`)
3. 提供文档元数据：需要给参数加 `description`、`title` 或 `example` 等文档内容

下面用一段代码举例：
```python
@app.get("/cars/{license_plate}")
async def get_car_info(
    license_plate: Annotated[
        str, 
        Path(
            min_length=8,
            max_length=8,
            # 🔍 核心：正则表达式
            # ^ : 开头
            # [A-Z]{3} : 3个大写字母
            # - : 连字符
            # \d{4} : 4个数字
            # $ : 结尾
            pattern="^[A-Z]{3}-\\d{4}$",
            title="车牌号",
            description="格式必须为：ABC-1234",
            example="AAA-8888" # 注意：这里仅作示例，实际正则可能需要调整以匹配中文
        )
    ]
):
    return {"car": license_plate, "status": "Clean"}
    
    
@app.get("/slots/{slot_id}")
async def control_slot(
    slot_id: Annotated[
        int, 
        Path(
            ge=1,  # 大于等于 1
            le=10, # 小于等于 10
            title="设备插槽 ID",
            description="仅允许访问 1-10 号插槽，超出范围会报错。",
            deprecated=True # 👈 标记为过时，文档里会显示删除线
        )
    ]
):
    return {"slot": slot_id, "action": "activate"}
    
@app.get("/download/{file_path:path}")  # **装饰器里**：使用 `{file_path:path}` 告诉 Starlette 匹配完整路径。
async def download_file(
    file_path: Annotated[
        str, 
        Path(  # **函数参数里**：使用 `Path` 增加文档说明（这一步可选，但推荐）。
            title="文件路径",
            description="从根目录开始的完整文件路径，包含文件名和扩展名。",
            example="images/2023/vacation/photo.jpg"
        )
    ]
):
    if file_path.startswith("private"):
         return {"error": "禁止访问私有目录"}
    return {"file": file_path, "status": "Downloading..."}
```
> 一个容易被忽视的“坑”
> **`alias` (别名) 在 `Path` 中几乎不可用。**
> 在 `Query` 或 `Body` 中，我们可以用 `alias="item-id"` 来把前端传的 `item-id` 映射给 Python 变量 `item_id`。
> 但在 `Path` 中，**Python 函数的参数名必须严格等于路径模板 `{}` 中的名字**。


`Path`虽然有给默认值形参，但是路径参数永远是必填的，类中有默认值的形参只是为了兼容性。

### 查询参数(Query)
在FastAPI中，形参列表里定义了但是路径上没有对应变量的自动归为查询参数。

如果不使用 `Query`，FastAPI 只能做简单的类型转换。但有了 `Query`，你可以：
1. **设置默认值**：用户不传时用什么值？
2. **强制校验**：字符串长度、正则、数值范围。
3. **接收列表**：处理 `?ids=1&ids=2&ids=3` 这种数组传参。
4. **别名映射**：处理 `kebab-case` (URL) 到 `snake_case` (Python) 的转换。
5. **文档元数据**：给 Swagger UI 提供说明。

`Query` 支持的参数和 `Path` 几乎一模一样，但多了一个 **`alias`** (别名)，这在 Query 中非常常用。

| **类别**   | **参数**                     | **说明**        | **示例**                       |
| -------- | -------------------------- | ------------- | ---------------------------- |
| **数值校验** | `ge`, `le`, `gt`, `lt`     | 范围限制          | `ge=1` (页码必须大于1)             |
| **字符校验** | `min_length`, `max_length` | 长度限制          | `min_length=3` (搜索词太短不行)     |
| **正则校验** | `pattern`                  | 正则表达式         | `pattern="^apple-.*$"`       |
| **功能性**  | **`alias`**                | **别名映射 (重点)** | URL用 `user-id`，代码用 `user_id` |
| **文档**   | `title`, `description`     | 文档描述          | `description="搜索关键词"`        |
| **废弃**   | `deprecated`               | 标记过时          | `deprecated=True`            |

以下是几种常用的场景：
```python
from typing import Annotated
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(
    # 1. 设置默认值 = 1
    # 2. 限制必须 >= 1
    page: Annotated[int, Query(ge=1, description="页码")] = 1,
    
    # 1. 设置默认值 = 10
    # 2. 限制最大 100
    size: Annotated[int, Query(le=100, description="每页条数")] = 10
):
    return {"page": page, "size": size}
    
# 用户想要筛选多个标签：`?tags=python&tags=fastapi&tags=code`。
@app.get("/filter/")
async def filter_items(
    # 只需要把类型声明为 list[str]
    # Query() 是必须的，否则 FastAPI 可能会困惑
    tags: Annotated[list[str], Query(description="标签列表")] = []
):
    return {"filter_tags": tags}
    
# 前端/URL 标准习惯用中划线：`item-query`。
# Python 变量名不支持中划线，必须用下划线：`item_query`
@app.get("/alias/")
async def alias_demo(
    # Python 里叫 item_query
    # URL 里寻找 item-query
    item_query: Annotated[
        str | None, 
        Query(alias="item-query") 
    ] = None
):
    return {"query": item_query}

# 必填
@app.get("/required/")
async def required_query(
    # 只要等号右边没有默认值，或者使用了 Query(...)
    # FastAPI 就认为它是必填的
    token: Annotated[str, Query()] 
):
    return {"token": token}
```


### 请求体(Body)
通常我们用 Pydantic 模型（`BaseModel`）来定义请求体，但 `Body` 专门处理那些 **“不值得专门写一个模型”** 或者 **“复杂的混合结构”** 的场景。
如果你在函数参数里写 `name: str` 或 `age: int`，FastAPI 默认认为它们是 **Query 参数**（URL 里的 `?name=foo&age=18`）。如果你想通过 **JSON Body** 传递单一的 `age` 字段，必须显式使用 `Body`。
```python
from typing import Annotated
from fastapi import FastAPI, Body

app = FastAPI()

# 此时客户端发送的 JSON：
# {
#     "importance": 5
# }

@app.post("/items/")
def update_item(
    # 告诉 FastAPI: 别去 URL 找，去 Body 里找 "importance" 字段
    importance: Annotated[int, Body(gt=0)]
):
    return {"importance": importance}
```
或者混合使用：Pydantic 模型 + 额外字段
有时候你的请求体主要是一个对象（比如 `User`），但还需要额外传一个简单的字段（比如 `version`），这时候 `Body` 就很方便。
```python
from pydantic import BaseModel

class User(BaseModel):
    username: str
    password: str
    
# 注意：如果有多个 Body 参数，FastAPI 默认会把它们变成键值对形式
# {
#     "item": {
#         "name": "Iphone",
#         "price": 999
#     },
#     "user": {
#         "username": "jerry",
#         "password": "123"
#     },
#     "count": 1
# }

@app.post("/users/login")
async def login(
	item: Item,  # Pydantic 模型
    user: User,  # Pydantic 模型
    # 额外的单一字段，强制要求在 Body 中
    count: Annotated[int, Body()] 
):
    return {"item": item, "user": user, "count": importance}
```

Body有一个关键的参数 `embed=True` (嵌入)
假设你只有一个 Pydantic 模型参数 `item: Item`。
**默认情况**：客户端直接发送 Item 的字段。
而**使用 `embed=True`**：客户端必须把 Item 包裹在一个 JSON Key 里面。
```python
class Item(BaseModel):
    name: str
    price: float

# --- 场景 A: 默认行为 ---
@app.post("/a")
def create_a(item: Item):
    pass
# 期望 JSON: { "name": "foo", "price": 10 }

# --- 场景 B: 使用 embed=True ---
@app.post("/b")
def create_b(
    item: Annotated[Item, Body(embed=True)]
):
    pass
# 期望 JSON: 
# { 
#   "item": { "name": "foo", "price": 10 } 
# }
```
其他的校验能力和其他的工具类完全一样，不再复述。



### 请求头(Header)
**FastAPI 的魔法**： 当你声明参数 `user_agent` 并使用 `Header()` 时，FastAPI 会自动把下划线 `_` 变成连字符 `-`，去 HTTP 请求头里找对应的值。而且，它**忽略大小写**。
```python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/headers/")
async def read_header(
    # Python 写法: user_agent
    # FastAPI 寻找: User-Agent (或 user-agent)
    user_agent: Annotated[str | None, Header()] = None
):
    return {"User-Agent": user_agent}
```

有时候，你可能有特殊的自定义 Header，它本身就真的包含下划线（虽然不符合 HTTP 标准习惯，但有些内部系统会这么干），比如 `sys_token`。如果此时 FastAPI 还是把它转成 `sys-token`，那就取不到值了。你需要关掉这个功能。
**参数：**`convert_underscores=False`
```python
@app.get("/items/")
async def read_items(
    # 告诉 FastAPI: 别转义！我就要找 "sys_token" 这个 Header
    sys_token: Annotated[
	    str | None, 
		Header(convert_underscores=False)
	] = None
):
    return {"sys_token": sys_token}
```

HTTP 协议允许同一个 Header 出现多次（虽然不常见，但合法）。 例如：
```text
X-Cat-Type: Tabby
X-Cat-Type: Persian
```
如果你想接收所有的值，需要把类型声明为 **`list[str]`**。
```python
@app.get("/cats/")
async def read_cats(
    # 接收列表
    x_cat_type: Annotated[list[str] | None, Header()] = None
):
    return {"x_cat_type": x_cat_type}
```
**返回结果**: `{"x_cat_type": ["Tabby", "Persian"]}`

因为 `Header` 也继承自 `Param`，所以它支持和 `Query`/`Path` 一样的校验参数：
- `default`: 设置默认值（如果不传 Header）。
- `min_length`, `max_length`, `pattern`: 校验字符串格式。
- `description`, `title`: 文档描述。

**场景：强制要求客户端带上特定的 Token**
```python
@app.get("/secure/")
async def secure_data(
    # 1. 没有默认值 -> 必填
    # 2. 长度必须 32 位
    x_token: Annotated[str, Header(min_length=32, max_length=32)]
):
    return {"data": "Secret info"}
```
**注意事项 (Gotchas)**
1. **大小写不敏感**：HTTP Headers 本身就是大小写不敏感的。你在代码里定义 `Header()`，客户端传 `user-agent`、`User-Agent` 甚至 `USER-AGENT`，FastAPI 都能接收到。
2. **不要用它做复杂的鉴权**：虽然你可以用 `Header` 获取 `Authorization` 头，但 FastAPI 提供了更高级的 `Security` 和 `Depends` 工具（如 `OAuth2PasswordBearer`）来专门处理鉴权。`Header` 更适合获取元数据（如设备信息、语言偏好）。
3. **Content-Type 等特殊头**：虽然你可以通过 `Header()` 获取 `Content-Type`，但通常最好让 FastAPI 自动处理（通过 `Body` 解析 JSON）。




### Cookie
FastAPI 中用于从 **HTTP 请求的 Cookie 头部** 中提取特定数据的工具。
核心用途：只读不写 (Read-Only)
- **`Cookie(...)` 参数**：用于 **读取** 客户端发来的 Cookie。
- **`Response` 对象**：用于 **设置 (写入)** 返回给客户端的 Cookie。

比如：
```python
from typing import Annotated
from fastapi import FastAPI, Cookie

app = FastAPI()

@app.get("/items/")
async def read_items(
    # 告诉 FastAPI: 去请求头里找 Cookie: ads_id=...
    ads_id: Annotated[
	    str | None, 
	    Cookie()
	] = None
):
    return {"ads_id": ads_id}
    
@app.get("/user/me")
async def get_current_user(
    # 如果请求里没有 session_id 这个 cookie，直接报错 422
    session_id: Annotated[
	    str, 
	    Cookie(alias="session-id", description="会话ID")
	]
):
    return {"session_id": session_id}
    

@app.post("/login/")
async def login(response: Response):
    # 1. 写入 Cookie (设置给浏览器)
    # httponly=True 是安全最佳实践，防止前端 JS 偷取
    response.set_cookie(key="fakesession", value="fake-cookie-session-value", httponly=True)
    return {"message": "Cookie 已种下"}

@app.get("/logout/")
async def logout(
    response: Response,
    # 2. 读取 Cookie (从浏览器获取)
    fakesession: Annotated[str | None, Cookie()] = None
):
    # 3. 清除 Cookie
    response.delete_cookie("fakesession")
    return {"old_session": fakesession, "message": "Cookie 已清除"}
```
**关键特性与注意事项**
- **安全性 (Security)**： `Cookie` 参数只能拿到**值 (Value)**。你拿不到 Cookie 的属性（比如它是否是 `HttpOnly`，是否 `Secure`，过期时间是多少）。因为浏览器发送请求时，**只发 key=value**，不发元数据。
- **没有自动转换**： 与 `Header` 不同，`Cookie` **不会**自动把下划线转换成连字符。
    - 变量名 `user_id` 对应的 Cookie key 就是 `user_id`。
    - 如果 Cookie key 是 `User-ID`，你需要用 `alias="User-ID"`。
- **GDPR 与 隐私**： 通常建议将 Cookie 参数设为可选（`default=None`）。因为用户可能拒绝了 Cookie 许可，或者浏览器插件拦截了追踪 Cookie。如果设为必填，可能会导致非预期的 422 错误。

### 表单(Form)
FastAPI 中专门用于处理 **表单数据** 的工具。它是处理 HTML 传统表单提交（`<form>`）和 OAuth2 登录接口的专用工具，它和 JSON 是死对头，不能混用。虽然它和 `Body` 很像（都是从请求体拿数据），但它们处理的 **编码格式（Media Type）** 完全不同。表单使用的是`application/x-www-form-urlencoded`

**什么时候必须用 Form？**
除了传统网页，最常见的场景是 **OAuth2 规范**。当你做标准的“用户名密码登录获取 Token”接口时，OAuth2 协议**强制要求**使用表单数据，而不是 JSON。
```python
# 这是一个标准的 OAuth2 获取 Token 的接口样子
@app.post("/token")
async def login_for_access_token(
    # OAuth2PasswordRequestForm 本质上就是封装好的 Form 参数
    username: Annotated[str, Form()],
    password: Annotated[str, Form()]
):
    # 验证逻辑...
    return {"access_token": "xxx", "token_type": "bearer"}
```
其次，如果你想**上传文件**，同时又想**附带一些字段**（比如上传头像，同时修改昵称），这时候 JSON 就无能为力了。 必须使用 `multipart/form-data`。
FastAPI 允许 `Form` 和 `File` 同时出现，此时 `Content-Type` 自动变为 `multipart/form-data`：
```python
from fastapi import FastAPI, File, Form, UploadFile
from typing import Annotated

@app.post("/files/")
async def create_file(
    file: Annotated[UploadFile, File()],
    token: Annotated[str, Form()]  # 接收文本字段 (必须用 Form，不能用 Body)
):
    return {
        "file_size": file.size,
        "token": token,
        "file_content_type": file.content_type,
    }
```

### 文件上传(File)
FastAPI 中用于提取 **上传文件** 的工具。它继承自 `Form`，专门处理 HTTP 协议中最复杂的 `multipart/form-data` 数据格式。
虽然在参数中使用 `File(...)` 标记了来源，但在类型注解（Type Hint）上，有两种选择。
- `file: bytes = File(...)`: 不要用
- `file: UploadFile = File(...)`: 用这个就行

```python
from typing import Annotated
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/upload/")
async def create_upload_file(
    # 核心：类型是 UploadFile，来源是 File()
    file: Annotated[UploadFile, File(description="请上传图片")]
    files: Annotated[list[UploadFile], File(description="多文件上传")]
):
    return {
        "filename": file.filename, 
        "content_type": file.content_type
    }
```
但是特殊的是，虽然他也有各种校验参数，但是基本上不起作用或者行为会有问题，不要用！


## 请求与响应对象 (Request & Response)

`Request` 和 `Response` 是 FastAPI（实际上是 Starlette）暴露给你的 **底层 HTTP 协议接口**

### Request
`Request` 代表了 **从客户端发来的那个 HTTP 请求**。
以下情况必须用 `Request`：
- **获取客户端 IP 地址**。
- **获取完整的当前 URL**。
- **访问 Middleware (中间件) 传下来的数据** (`request.state`)。
- **读取未经解析的原始 Body** (比如读取二进制流或特殊的 WebHook)。

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/info/")
async def get_request_info(request: Request):
    # 1. 获取客户端 IP
    client_host = request.client.host if request.client else "unknown"
    
    # 2. 获取当前完整 URL
    url = str(request.url)
    
    # 3. 获取所有 Headers (不仅仅是某一个)
    # request.headers 是一个类字典对象
    user_agent = request.headers.get("user-agent")
    
    return {
        "client_ip": client_host,
        "url": url,
        "user_agent": user_agent
    }
```
#### 核心属性速查
- `request.method`: 请求方法 (`GET`, `POST`...)
- `request.url`: 完整的 URL 对象。
- `request.headers`: 请求头 (不可变字典)。
- `request.query_params`: 查询参数 (不可变字典)。
- `request.path_params`: 路径参数。
- `request.client`: 包含 `host` (IP) and `port`。
- `request.cookies`: Cookies 字典。
- `request.state`: 用于在中间件和路由之间传递数据的任意对象。
- `await request.body()`: 获取原始 bytes 类型的 body。
- `await request.json()`: 手动解析 JSON。


### Response
`Response` 代表了 **服务器发回给客户端的 HTTP 响应**。
当你需要返回 **非 JSON** (HTML/File) 或执行 **重定向** 时使用。在 FastAPI 中，使用 `Response` 有两种截然不同的方式：**“参数注入模式”** 和 **“直接返回模式”**。

**参数注入** (只修改头/Cookie，内容照旧)：当你依然想利用 FastAPI 的 JSON 序列化功能，但只是想顺手**加个 Cookie** 或 **改个 Header** 时，使用这种方式。
```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.post("/login/")
async def login(response: Response):
    # 1. 设置 Cookie
    response.set_cookie(key="fakesession", value="fake-cookie-session-value")
    
    # 2. 设置 Header
    response.headers["X-Cat-Dog"] = "alone in the world"
    
    # 3. 修改状态码
    response.status_code = 201
    
    # ⚠️ 注意：最后返回的依然是 dict 或 Pydantic 模型
    return {"message": "Logged in"}
```

**直接返回** (完全接管响应)：你想跳过 FastAPI 的序列化过程以提升性能时，可以直接返回一个 `Response` 对象（或其子类）。

**常见的 Response 子类：**
1. **`JSONResponse`**: 手动返回 JSON，常用于自定义错误处理或统一响应结构。
2. **`HTMLResponse`**: 返回 HTML 页面。
3. **`RedirectResponse`**: 执行 HTTP 重定向 (307/301)。
4. **`StreamingResponse`**: 流式响应 (如下载大文件)。
5. **`FileResponse`**: 自动处理文件发送。

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, HTMLResponse, RedirectResponse

app = FastAPI()

# 1. 返回 HTML
@app.get("/html/")
async def get_html():
    html_content = "<h1>Hello FastAPI</h1>"
    return HTMLResponse(content=html_content)

# 2. 重定向
@app.get("/github/")
async def redirect_to_github():
    return RedirectResponse(url="https://github.com")

# 3. 自定义 JSON (跳过 Pydantic 验证，性能更高)
@app.get("/fast-json/")
async def fast_json():
    data = {"k": "v" * 1000}
    # 直接返回 JSONResponse
    return JSONResponse(content=data, status_code=200)
```

**自定义 Response 类 (全局统一处理)**
你可以在创建 `FastAPI` 实例时指定 `default_response_class`，比如你想让所有的 JSON 响应都自动使用更快的 `ORJSONResponse`。
```python
from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)
```



# 依赖注入与生命周期管理 (Dependency Injection & Lifespan)

## 依赖注入
**依赖注入 (Dependency Injection, DI)** 是 FastAPI 的**灵魂**。

依赖注入的核心理念就是，你不需要自己创建对象，只需要声明你想要什么，FastAPI会创建好并送给你。

在没有依赖注入的情况下，对于连接数据库、验证用户Token、记录日志等功能，通常需要自己调用方法，如：
```python
# 糟糕的写法
@app.get("/items/")
def read_items(token: str):
    # 1. 自己手动验证 Token
    user = check_token(token) 
    # 2. 自己手动连数据库
    db = Database() 
    db.connect()
    # 3. 查数据
    items = db.query()
    # 4. 手动关闭数据库 (万一报错了没关怎么办？)
    db.close() 
    return items
```
有依赖注入 (FastAPI 写法)：
```python
@app.get("/items/")
async def read_items(
    # 声明依赖：我需要 user 和 db
    # FastAPI 会自动去执行 get_current_user 和 get_db，把结果传进来
    user: Annotated[User, Depends(get_current_user)],
    db: Annotated[Session, Depends(get_db)]
):
    # 这里只写纯粹的业务逻辑！
    return db.query()
```

并且，FastAPI 的依赖注入系统是一个**有向图**。它支持**无限层级**的嵌套。
比如：
1. **路径操作函数** 依赖 `get_current_user`。
2. `get_current_user` 依赖 `get_token_header`。
3. `get_token_header` 依赖 `Header` 参数。

```python
from typing import Annotated
from fastapi import FastAPI, Depends, Header, HTTPException

app = FastAPI()

# --- 第一层依赖 ---
def verify_token(x_token: Annotated[str, Header()]):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="Token无效")
    return x_token

# --- 第二层依赖 (使用了第一层) ---
def verify_key(
    x_key: Annotated[str, Header()],
    # 依赖注入！
    token: Annotated[str, Depends(verify_token)] 
):
    if x_key != "fake-key":
        raise HTTPException(status_code=400, detail="Key无效")
    return {"token": token, "key": x_key}

# --- 最终路由 (使用了第二层) ---
@app.get("/items/")
async def read_items(
    # 只要声明这一层，FastAPI 会自动顺藤摸瓜把上面全执行一遍
    auth_data: Annotated[dict, Depends(verify_key)]
):
    return auth_data
```

### `Yield` 依赖
这是 FastAPI 中最常用的模式，专门用于需要 **“准备资源 -> 使用资源 -> 清理资源”** 的场景（比如数据库会话）。
它利用了 Python 生成器 (`yield`) 的特性。
```python
# database.py

# 1. 创建生成器函数
def get_db():
    db = SessionLocal()  # A. 建立连接
    try:
        yield db         # B. 暂停，把 db 交给路由函数使用
    finally:
        db.close()       # C. 路由执行完后，自动回来关闭连接
```

### 类作为依赖 (Class as Dependency)
`Depends` 不仅可以接受函数，还可以接受类。如果是类，FastAPI 会自动调用它的 `__init__` 方法。
```python
class CommonQueryParams:
    def __init__(self, q: str | None = None, skip: int = 0, limit: int = 100):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(
    # FastAPI 会调用 CommonQueryParams(q=..., skip=..., limit=...)
    # 然后把实例赋值给 commons
    commons: Annotated[CommonQueryParams, Depends()]
):
    return commons.limit
```

> Q1: 为什么可以只写`Depends()`不放入参数？
> A: 当 `Depends()` 里面是空的时候，FastAPI 会非常聪明地回头看一眼**左边的类型注解**。
> 它发现左边是 `CommonQueryParams`，就会自动认为：“哦，你没给我传参数，但我看你想要 `CommonQueryParams` 类型，那我帮你把 `CommonQueryParams` 填进 `Depends()` 里吧。”
> 
> Q2: 什么时候**不能**偷懒？
> A: **接口（Interface）与实现分离**，或者依赖返回的是一个 Pydantic 模型，但逻辑是一个函数。
> 例如：`user: Annotated[User, Depends(get_current_user)]`



### 缓存机制 (`use_cache`)
默认情况下，在一个请求的生命周期内，如果同一个依赖被多次调用，FastAPI **只会执行一次**，然后把结果缓存起来复用。如果你不需要这个特性（比如你需要依赖每次都返回新的随机数），可以关闭：
```python
Depends(get_value, use_cache=False)
```

### 依赖覆盖 (`dependency_overrides`)
这是依赖注入带来的最大好处之一：**极易测试**。
在写单元测试时，你不想真的连数据库，也不想真的发短信。你可以直接在运行时把依赖替换掉。

```python
# 业务代码中的依赖
def get_db():
    return RealDatabase()

# --- 测试代码 ---
from fastapi.testclient import TestClient

# 1. 定义假的依赖
def override_get_db():
    return MockDatabase() # 假数据库

# 2. 覆盖它
app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)
# 3. 发请求 -> 此时路由里拿到的是 MockDatabase！
client.get("/items/")

# 4. 测完记得清理
app.dependency_overrides = {}
```


## 生命周期管理
**版本要求**: FastAPI >= 0.93.0
**注意**：如果不使用 `lifespan`，旧版本使用的是 `on_startup` 和 `on_shutdown` 列表参数，但现在**强烈建议**使用 `lifespan`，因为它能更好处理异常和上下文。

生命周期管理函数用于管理那些在启动时加载一次，并在关闭时释放的资源，例如某个机器学习模块，或者数据库初始化连接池

通常会使用`from contextlib import asynccontextmanager` 这个异步上下文管理器来装饰生命周期函数

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, APIRouter

# 定义 lifespan 上下文管理器
@asynccontextmanager
async def router_lifespan(app: FastAPI):
	# ------------------------------------------------ 
	# 🟢 1. 启动逻辑 (Startup) 
	# ------------------------------------------------
    print("🤖 模型模块：开始加载模型...")
    # model = load_heavy_model()
    
    yield  # ⏸️ 这里的 yield 代表应用正在运行中 (处理请求)
    
    # ------------------------------------------------ 
    # 🔴 2. 关闭逻辑 (Shutdown) 
    # ------------------------------------------------
    print("🤖 模型模块：清理资源...")

# 在 Router 中使用
router = APIRouter(lifespan=router_lifespan)
# 或者在app中使用
app = FastAPI(lifespan=lifespan)

@router.get("/predict")
def predict():
    return {"result": "prediction"}
```

但是当你生命周期管理函数中做了多件事情，尽量还是将每个函数要做的事情划分清楚，并使用以下方式将其合并起来

###### 方案一：手动嵌套
既然 lifespan 是上下文管理器（Context Manager），它们是可以互相嵌套的（就像 `with open(...)` 里面还可以再写 `with open(...)`）。
你可以定义一个 **主 Lifespan**，然后在里面调用其他的 **子 Lifespan**。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

# --- 子 Lifespan 1: 负责数据库 ---
@asynccontextmanager
async def db_lifespan(app: FastAPI):
    print("🔋 [DB] 连接数据库...")
    yield
    print("🪫 [DB] 断开数据库...")

# --- 子 Lifespan 2: 负责 Redis ---
@asynccontextmanager
async def redis_lifespan(app: FastAPI):
    print("🔌 [Redis] 连接 Redis...")
    yield
    print("🔌 [Redis] 断开 Redis...")

# --- 👑 主 Lifespan: 负责调度 ---
@asynccontextmanager
async def main_lifespan(app: FastAPI):
    # 使用 async with 语法嵌套
    # 注意顺序：先进后出 (FILO)
    async with db_lifespan(app):
        async with redis_lifespan(app):
            print("🚀 系统启动完毕，开始接收请求")
            yield
            print("🌙 系统准备关闭")

app = FastAPI(lifespan=main_lifespan)
```

###### 方案二：使用 `APIRouter` 分散管理（最模块化）
这是 FastAPI 0.100.0+ 引入 Router lifespan 的主要原因。如果你的逻辑属于不同的业务模块，**不要把它们全塞在 `main.py` 里**。

让每个 Router 管理自己的 lifespan，FastAPI 会自动帮你组合。

```python
# users.py
@asynccontextmanager
async def users_lifespan(app: FastAPI):
    print("用户模块初始化...")
    yield
    print("用户模块清理...")

router = APIRouter(lifespan=users_lifespan)

# main.py
@asynccontextmanager
async def main_lifespan(app: FastAPI):
    print("主应用初始化...")
    yield
    print("主应用清理...")

app = FastAPI(lifespan=main_lifespan)
app.include_router(router)
```

FastAPI 会自动先执行 App 的 lifespan，再执行 Router 的 lifespan。


###### 方案三：使用 `AsyncExitStack`（高级用法）
如果你有 **很多** 个独立的 lifespan，写嵌套（`async with ... async with ...`）会写出“回调地狱”般的缩进。

这时可以使用 Python 标准库的 `AsyncExitStack` 来扁平化代码。

```python
from contextlib import asynccontextmanager, AsyncExitStack
from fastapi import FastAPI

@asynccontextmanager
async def lifespan_a(app):
    print("A start")
    yield
	print("A end")

@asynccontextmanager
async def lifespan_b(app):
    print("B start")
	yield
	print("B end")

@asynccontextmanager
async def lifespan_c(app):
    print("C start")
	yield
	print("C end")

# --- 组合器 ---
@asynccontextmanager
async def app_lifespan(app: FastAPI):
    async with AsyncExitStack() as stack:
        # 这里的 enter_async_context 会自动处理 enter 和 exit
        await stack.enter_async_context(lifespan_a(app))
        await stack.enter_async_context(lifespan_b(app))
        await stack.enter_async_context(lifespan_c(app))
        
        yield # 所有的 exit 逻辑会在这里之后，由 stack 自动逆序执行

app = FastAPI(lifespan=app_lifespan)
```


### Lifespan 和 Dependency 如何配合？
通常有两种方式：**全局变量** 或 **`request.app.state`**。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request

# 假装这是数据库驱动
class DatabasePool:
    def connect(self): print("✅ 连接池已建立")
    def close(self): print("❌ 连接池已销毁")
    def get_connection(self): return "Active Connection"

# 1. 定义全局变量 (或者放在 app.state)
db_pool = DatabasePool()

# 2. 定义 Lifespan (管理 Pool)
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 🟢 Startup: 建立池子
    db_pool.connect()
    yield
    # 🔴 Shutdown: 销毁池子
    db_pool.close()

app = FastAPI(lifespan=lifespan)

# 3. 定义 Dependency (管理 Connection)
# 这里的逻辑是：从全局的 Pool 里，取出一个 Connection 给当前请求用
async def get_db_conn():
    conn = db_pool.get_connection()
    try:
        yield conn
    finally:
        print("♻️ 连接已归还给池子")

# 4. 路由使用
@app.get("/users/")
async def get_users(conn: str = Depends(get_db_conn)):
    return {"db_status": conn}
```

### lifespan在APIRouter和在FastAPI中指定的区别
`APIRouter` 支持 `lifespan` 是 **FastAPI 0.100.0+** 才引入的新特性。在这之前，所有的启动/关闭逻辑都必须写在最顶层的 `FastAPI` app 里。
两者的区别核心在于：**作用域（Scope）** 和 **模块化（Modularity）**。

| **特性**   | **FastAPI (Global Lifespan)** | **APIRouter (Modular Lifespan)**              |
| -------- | ----------------------------- | --------------------------------------------- |
| **作用域**  | **全局**。影响整个应用。                | **局部**。只服务于该 Router 负责的业务模块。                  |
| **典型用途** | 数据库连接池、Redis 连接、全局日志配置。       | 加载特定业务的 ML 模型、连接该模块专用的第三方服务（如邮件服务器）。          |
| **代码位置** | 通常在 `main.py`。                | 通常在 `users.py`, `items.py` 等模块文件中。            |
| **触发条件** | App 启动即触发。                    | App 启动 **且** 该 Router 被 `include_router` 时触发。 |
| **设计哲学** | 集中管理基础设施。                     | **高内聚**：谁产生的业务需求，谁自己负责初始化。                    |



### 共享 State 的注意事项

无论是 `FastAPI` 还是 `APIRouter` 的 lifespan，它们接收到的参数 `app` 指向的都是**同一个** 最终运行的 FastAPI 实例（在 Starlette 层面）。这意味着你可以在 Router 的 lifespan 里往 `app.state` 塞东西，主 App 也能用。
```python
# router.py
@asynccontextmanager
async def sub_lifespan(app: FastAPI):
    # 这里设置的 state，全局都能访问
    app.state.sub_module_ready = True
    yield
```



# 环境管理 (Environment)
在FastAPI中常用`pydantic-settings`进行环境管理。
```bash
pip install pydantic-settings
```

多环境管理主要涉及三个方面：多个配置文件，配置类，启动时控制。

例如有以下环境变量文件
```text
my_project/
├── .env.dev            # 开发环境配置
├── .env.test           # 测试环境配置
├── .env.prod           # 生产环境配置
├── .gitignore          # ⚠️ 重要：把 .env.* 加入忽略列表
├── app/
│   ├── main.py
│   └── core/
│       └── config.py   # 配置加载逻辑
```

编写配置文件
```toml
# .env.dev
APP_NAME="FastAPI Dev"
DEBUG=True
DATABASE_URL="sqlite:///./dev.db"
SECRET_KEY="dev-secret-key"

# .env.prod
APP_NAME="FastAPI Prod"
DEBUG=False
DATABASE_URL="postgresql://user:pass@db-host:5432/dbname"
SECRET_KEY="super-secure-production-key-xxx"
```

在配置逻辑中：
```python
import os
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # 1. 定义配置项 (变量名大小写不敏感，自动匹配 .env 中的大写)
    APP_NAME: str = "FastAPI App"
    DEBUG: bool = False
    DATABASE_URL: str
    SECRET_KEY: str

    # 2. 配置 Pydantic 行为
    model_config = SettingsConfigDict(
        # 这里只是默认值，实际会被下面的 get_settings 覆盖
        env_file=".env", 
        env_file_encoding="utf-8",
        # 如果 .env 文件里没有某些变量，尝试从系统环境变量读取
        extra="ignore",
    )

@lru_cache
def get_settings():
    """
    根据环境变量 APP_ENV 加载对应的 .env 文件
    """
    # 获取启动时的环境变量，默认为 'dev'
    env = os.getenv("APP_ENV", "dev")
    
    # 拼接文件名，例如: .env.prod
    env_file = f".env.{env}"
    
    print(f"Loading config from: {env_file}")  # 调试用
    
    # 实例化 Settings，传入动态计算的文件路径
    return Settings(_env_file=env_file)

# 创建一个全局单例
settings = get_settings()
```

使用时只需导入`settings`对象即可
```python
from fastapi import FastAPI
from app.core.config import settings  # 导入配置单例

# 这里的 settings.DEBUG 会根据加载的 .env 自动变化
app = FastAPI(
    title=settings.APP_NAME,
    debug=settings.DEBUG
)

@app.get("/info")
def get_info():
    return {
        "app_name": settings.APP_NAME,
        "mode": "Debug Mode" if settings.DEBUG else "Production Mode",
        "db": settings.DATABASE_URL
    }
```

# 数据库集成 (Database Integration)
## 连接数据库 (SQLAlchemy)
首先是创建引擎，由于FastAPI是异步框架，数据库和连接池一般也使用异步数据库和异步连接池，因此最常见是使用`create_async_engine`：
但是在此之前需要先安装数据库的异步操作支持
```bash
pip install asyncpg  # 如果用 Postgres
pip install aiomysql # 如果用 MySQL，选择非常多
```

然后是创建引擎
```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
	settings.DATABASE_URL, # 数据库连接url
	echo=True  # 是否将操作的sql语句输出到控制台或日志
)
```


有了引擎之后，就需要获取会话，从引擎中获取连接进行操作。在FastAPI中，常用Depends获取数据库会话
```python
from typing import Annotated 
from fastapi import Depends, FastAPI 
from sqlmodel import select 
from sqlalchemy.ext.asyncio import create_async_engine 
from sqlmodel.ext.asyncio.session import AsyncSession 
from sqlalchemy.orm import sessionmaker

# 核心：定义异步依赖函数
async def get_async_session():
    # 创建一个 AsyncSession
    async_session = sessionmaker(
        bind=async_engine,  # 刚刚创建的那个引擎
        class_=AsyncSession,  # 使用异步Session以支持异步操作
        expire_on_commit=False  # 提交之后是否将会话失效，即交还线程回到线程池
    )
    
    async with async_session() as session:
        yield session


# 类型注解变成了 AsyncSession
AsyncSessionDep = Annotated[AsyncSession, Depends(get_async_session)]

@app.get("/books/")
async def read_books(session: AsyncSessionDep):
    # 注意：SQLModel 的异步查询语法略有不同，需要用 await session.exec
    result = await session.exec(select(Book))
    # 增删改需要commit，记得手动添加
    await session.commit()
    return result.all()
```

## 数据库迁移工具(Alembic)




## SQLModel
SQLModel是FastAPI作者写的另一个库，主要解决需要写两遍代码（一遍 SQLAlchemy Model 建表，一遍 Pydantic Schema 验证）的问题。它的核心理念就是：**同一个类，既是数据库模型（SQLAlchemy），又是数据验证模型（Pydantic）。**
```python
from typing import Optional
from sqlmodel import Field, SQLModel, create_engine, Session, select
from fastapi import FastAPI

# ==========================================
# 🌟 核心：一个类，身兼数职
# table=True  -> 告诉它这是一个数据库表 (SQLAlchemy)
# 继承 SQLModel -> 告诉它这是一个验证模型 (Pydantic)
# ==========================================
class Hero(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    secret_name: str
    age: Optional[int] = None

# ==========================================
# FastAPI 路由
# ==========================================
app = FastAPI()
engine = create_engine("sqlite:///database.db")

@app.post("/heroes/", response_model=Hero)
def create_hero(hero: Hero):  # 这里 hero 既做验证，又直接存库
    with Session(engine) as session:
        session.add(hero)
        session.commit()
        session.refresh(hero)
        return hero
```


## CRUD 操作实战


# 安全与认证 (Security & Auth)




# 中间件 (Middleware) 与 CORS

在请求到达你的路由函数（`def read_items`）之前，和响应离开你的 API 返回给用户之前，中间件都有机会拦截并修改它们。
当您使用`@app.middleware()`装饰器或`app.add_middleware()`方法添加多个中间件时，每个新的中间件都会包装应用程序，形成一个堆栈。最后添加的中间件位于最外层，第一个添加的中间件位于最内层。
在请求路径中，最外层的中间件首先运行。在响应路径中，它最后运行。（洋葱模型）


添加中间件有两种方式：
一：将`@app.middleware("http")`装饰在函数顶部
```python
import time

from fastapi import FastAPI, Request

app = FastAPI()


@app.middleware("http")
async def add_process_time_header(
	request: Request,  # 请求本身
	call_next  # 一个接收`request`参数的函数。
):
    start_time = time.perf_counter()
    response = await call_next(request)  # 调用这个函数走向下一个中间件
    process_time = time.perf_counter() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

二：在`main.py`中使用`app.add_middleware(middlewareClass)`
例如下面的`CORSMiddleware`

虽然它们都能实现“拦截请求和响应”，但在 **底层原理**、**适用场景** 和 **灵活性** 上有本质区别。
- **`@app.middleware("http")`**(装饰器模式): 
	适合逻辑简单的**函数式**中间件（如耗时统计）。这是 FastAPI 为了方便开发者，封装的一个语法糖。
	- **本质**: 它在底层其实是创建了一个 `BaseHTTPMiddleware` 类，把你写的函数塞进去。
	- **形式**: **函数 (Function)**。
	- **特点**: 写法简单，直接就写，不需要定义类。
	- **适合**：快速实现一些简单的逻辑。
	- 只需要访问 `request` 和 `call_next`，不需要复杂的初始化参数。
	- **典型例子**: 统计接口耗时、简单的全局日志。
	
	由于它底层基于 Starlette 的 `BaseHTTPMiddleware`，它有一些**已知的局限性**：
	1. **BackgroundTasks 失效风险**: 在某些情况下，如果你的中间件逻辑处理不当，可能会导致 `BackgroundTasks` 在响应发送前就执行，或者被意外阻断。
	2. **StreamingResponse 问题**: 对于流式响应，这种中间件处理起来比较笨重，因为它试图缓冲响应。
	3. **复用性差**: 很难把这个函数打包给其他项目用，因为它依赖于具体的 `app` 实例装饰器。


- **`app.add_middleware()`**(注册模式): 
	这是 ASGI 标准的中间件注册方式。
	- **本质**: 将一个 **类 (Class)** 添加到 ASGI 管道中。
	- **形式**: **类 (Class)**。
	- **特点**: 稍微繁琐一点（需要写类），但功能最强，支持传参。
	- 适合：**加载现成的库**: 比如 `CORSMiddleware`, `GZipMiddleware`, `SessionMiddleware`。这些库都是写好的类。
	- **需要传参**: 比如你需要传入一个 `secret_key` 或者配置项给中间件。
	- **纯 ASGI 中间件**: 如果你想要极致性能，或者要避开 `BaseHTTPMiddleware` 的坑，你需要自己写纯 ASGI 中间件类。




**`CORSMiddleware`**
**CORS (跨域资源共享)** 是中间件最常见、也最让前后端开发者头疼的应用场景。
FastAPI 内置了标准的解决方案。你不需要自己写 Header，直接配置这个中间件即可。
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 定义允许的来源 (白名单)
origins = [
    "http://localhost:3000",    # React 前端
    "http://localhost:8080",    # Vue 前端
    "https://my-frontend.com",  # 生产环境域名
]

app.add_middleware(
    CORSMiddleware,
    # 1. 允许哪些源？(安全起见，不要用 ["*"])
    allow_origins=origins,
    
    # 2. 是否允许携带 Cookie/凭证？(非常重要)
    # 如果设为 True，allow_origins 不能是 ["*"]
    allow_credentials=True,
    
    # 3. 允许哪些 HTTP 方法？(GET, POST, PUT...)
    allow_methods=["*"],
    
    # 4. 允许哪些 HTTP 头？(Authorization, Content-Type...)
    allow_headers=["*"],
)
```

**`GZipMiddleware` (压缩响应)**
如果你的 API 返回大量 JSON 数据，开启这个可以显著减少带宽，提升前端加载速度。
```python
from fastapi.middleware.gzip import GZipMiddleware

# minimum_size=1000: 只有超过 1KB 的响应才压缩
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

**`TrustedHostMiddleware` (防止 HTTP Host 头攻击)**
强制要求请求的 Host 头必须在白名单内。
```python
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(
    TrustedHostMiddleware, 
    allowed_hosts=["example.com", "*.example.com"]
)
```

# 工程化与架构 (Project Architecture)

## 项目目录结构最佳实践

```text
my_project/
├── alembic/                # 数据库迁移脚本 (由 Alembic 自动生成)
├── app/                    # 🐍 应用源代码主目录
│   ├── __init__.py
│   ├── main.py             # 🚀 程序入口 (App 初始化, 中间件, Lifespan)
│   │
│   ├── api/                # 🌐 接口层 (路由)
│   │   ├── __init__.py
│   │   ├── deps.py         # 依赖注入 (get_current_user, get_db)
│   │   └── v1/             # API 版本控制
│   │       ├── __init__.py
│   │       ├── router.py   # 负责把 `endpoints` 里的 router 收集起来，统一暴露给 `main.py`。
│   │       └── endpoints/  # 具体的业务接口
│   │           ├── login.py
│   │           ├── users.py
│   │           └── items.py
│   │
│   ├── core/               # ⚙️ 核心配置 (不包含业务逻辑)
│   │   ├── __init__.py
│   │   ├── config.py       # Pydantic Settings (环境变量加载)
│   │   ├── security.py     # JWT 加密/解密工具
│   │   ├── exceptions.py   # 自定义异常
│   │   └── logger.py       # 日志配置 (Loguru)
│   │
│   ├── crud/               # 💾 数据操作层 (只负责数据库 CRUD)
│   │   ├── __init__.py
│   │   ├── base.py         # 通用 CRUD 泛型类
│   │   ├── crud_user.py
│   │   └── crud_item.py
│   │
│   ├── models/             # 🗄️ 数据库模型 (ORM / SQLAlchemy / SQLModel)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── item.py
│   │
│   ├── schemas/            # 📝 数据模型 (Pydantic / 序列化与校验)
│   │   ├── __init__.py
│   │   ├── common.py       # 通用响应结构 (Msg, Data)
│   │   ├── user.py         # UserCreate, UserUpdate, UserResponse
│   │   └── token.py
│   │
│   ├── services/           # 🧠 (可选) 复杂业务逻辑层
│   │   └── ...             # 如果 CRUD 无法满足复杂逻辑，写在这里
│   │
│   └── db/                 # 🔌 数据库连接与会话
│       ├── __init__.py
│       ├── session.py      # Engine 和 SessionLocal
│       └── base.py         # 用于 Alembic 导入所有 Model
│
├── tests/                  # 🧪 测试用例
│   ├── __init__.py
│   ├── conftest.py         # Pytest 配置 (Fixtures)
│   └── api/
│       └── v1/
│           └── test_users.py
│
├── .env                    # 🔒 环境变量 (不要提交到 Git)
├── .gitignore
├── alembic.ini             # Alembic 配置文件
├── docker-compose.yml      # Docker 编排
├── Dockerfile              # 镜像构建
├── pyproject.toml          # 📦 依赖管理 (Poetry/PDM) 或 requirements.txt
└── README.md
```

## 统一返回格式
假设目录层级如下：
```text
app/
├── .env                 # 配置文件
├── main.py              # 入口文件
└── core/                # 核心组件包
    ├── __init__.py
    ├── config.py        # 1. 配置加载 (pydantic-settings)
    ├── context.py       # 2. 上下文管理 (contextvars)
    ├── schemas.py       # 3. 统一响应模型 (Generic Model)
    ├── middleware.py    # 4. 链路追踪中间件
    ├── responses.py     # 5. 返回工具类
    └── exceptions.py    # 6. 全局异常处理
```


### 1.配置层 (`.env` & `core/config.py`)
`.env`
```toml
# 修改 Header Key 以测试配置生效
TRACE_ID_HEADER_KEY=X-Request-ID
APP_NAME=FastAPI-Modern-Template
```

`app/core/config.py`
```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    # 配置项 (带默认值)
    APP_NAME: str = "My App"
    TRACE_ID_HEADER_KEY: str = "X-Trace-ID"

    # 配置加载规则
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True
    )

settings = Settings()
```

### 2.上下文层 (`app/core/context.py`)
新建一个 `core/context.py`，专门管理全局上下文。
```python
# core/context.py
from contextvars import ContextVar
from .config import settings  # 👈 引入实例化好的配置，假如你使用了pydantic_settings，并配置好了全局设置 TRACE_ID_HEADER_KEY: str = "X-Trace-ID"

# =========================================================
# 1. 直接使用配置
# =========================================================
# 以后如果想改 Key，改 .env 或环境变量即可，代码不用动
# 优先级为：系统环境变量 (常用生产/Docker环境) -> `.env` 文件 (常用开发环境) -> 只有代码 (默认值"X-Trace-ID")
TRACE_ID_HEADER_KEY = settings.TRACE_ID_HEADER_KEY

# =========================================================
# 2. 上下文变量定义
# =========================================================
TRACE_ID_CTX: ContextVar[str | None] = ContextVar("trace_id", default=None)

def get_trace_id() -> str | None:
    return TRACE_ID_CTX.get()

def set_trace_id(trace_id: str):
    return TRACE_ID_CTX.set(trace_id)
```

### 3.数据模型层 (`app/core/schemas.py`)
利用 Python 的 `Generic` (泛型)，让 `data` 字段可以是任何类型（User, List\[Item\] 等），同时保持外层结构不变。
```python
from typing import Annotated, Generic, TypeVar
from pydantic import BaseModel, Field, ConfigDict

# 定义泛型
T = TypeVar("T")

class UniformResponse(BaseModel, Generic[T]):
    code: Annotated[int, Field(description="业务状态码")] = 200
    message: Annotated[str, Field(description="响应信息")] = "Success"
    status: Annotated[bool, Field(description="业务状态")] = True
    data: Annotated[T | None, Field(description="业务数据")] = None

    trace_id: Annotated[str | None, Field(description="请求追踪ID")] = None

    model_config = ConfigDict(
        populate_by_name=True,  # 允许使用字段别名赋值
        from_attributes=True    # 允许从 ORM 对象读取数据 (备用)
    )
```

### 4.中间件层 (`app/core/middleware.py`)
我们需要一个中间件，在请求刚进来时就获取原有的或生成一个 `trace_id`，并把它挂载到 `request` 对象上，这样后续的路由函数和异常处理器都能拿到它。
```python
# core/middleware.py
import uuid
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
# 👇 导入常量和上下文方法
from .context import set_trace_id, TRACE_ID_CTX, TRACE_ID_HEADER_KEY

class TraceIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # 1. 尝试从 Header 获取 (接入上游)
        trace_id = request.headers.get(TRACE_ID_HEADER_KEY)
        
        # 2. 如果没有，生成新的 (链路起点)
        if not trace_id:
            trace_id = uuid.uuid4().hex
        
        # 3. 设置上下文
        token = set_trace_id(trace_id)
        
        try:
            response = await call_next(request)
            
            # 4. 写入响应 Header (透传给下游/前端)
            response.headers[TRACE_ID_HEADER_KEY] = trace_id
            return response
        finally:
	        # 5. 清理上下文
            TRACE_ID_CTX.reset(token)
```

### 5.返回工具类 (`app/core/responses.py`)
```python
# core/responses.py
from typing import Any
from .schemas import UniformResponse
from .context import get_trace_id

class Resp:
    """
    静态响应工厂
    使用方法: return Resp.ok(data=...)
    """
    
    @staticmethod
    def ok(data: Any = None, message: str = "Success") -> UniformResponse:
        return UniformResponse(
            status=True,
            code=200,
            message=message,
            data=data,
            trace_id=get_trace_id()
        )

    @staticmethod
    def fail(code: int = 400, message: str = "Fail", data: Any = None) -> UniformResponse:
        return UniformResponse(
            status=False,
            code=code,
            message=message,
            data=data,
            trace_id=get_trace_id()
        )
```


### 6.异常处理层 (`app/core/exceptions.py`)
这是最关键的一步。无论代码哪里崩了，或者校验失败了，都要拦截下来，把默认的错误转成你的统一格式。
```python
from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException
from .schemas import UniformResponse
from .context import get_trace_id

def register_exception_handlers(app: FastAPI):
    
    # 辅助函数：统一构建
    def make_response(code: int, message: str, status: bool, data: any = None):
        content = UniformResponse(
            code=code,
            message=message,
            status=status,
            trace_id=get_trace_id(),  # 统一从 ContextVars 获取 ID
            data=data
        )
        return JSONResponse(
            status_code=code if code < 600 else 500,
            content=content.model_dump(mode='json')
        )

    @app.exception_handler(StarletteHTTPException)
    async def http_exception_handler(request: Request, exc: StarletteHTTPException):
        return make_response(
            code=exc.status_code,
            message=exc.detail,
            status=False
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        first_error = exc.errors()[0].get("msg") if exc.errors() else "Args Error"
        return make_response(
            code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            message=f"校验错误: {first_error}",
            status=False,
            data=exc.errors()
        )

    @app.exception_handler(Exception)
    async def global_exception_handler(request: Request, exc: Exception):
        print(f"🔥 System Error: {exc}") 
        return make_response(
            code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            message="系统繁忙",
            status=False
        )
```


### 7.最终入口 (`app/main.py`)
```python
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict
from core.config import settings
from core.middleware import TraceIDMiddleware
from core.exceptions import register_exception_handlers
from core.responses import Resp
from core.schemas import UniformResponse

app = FastAPI(title=settings.APP_NAME)

# 1. 注册中间件
app.add_middleware(TraceIDMiddleware)

# 2. 注册异常处理器
register_exception_handlers(app)

# --- 业务演示 ---
class UserOut(BaseModel):
    id: int
    name: str
    model_config = ConfigDict(from_attributes=True)

@app.get("/users/{uid}", response_model=UniformResponse[UserOut])
async def get_user(uid: int):
    # 模拟业务
    if uid == 0:
        # 业务逻辑错误
        return Resp.fail(code=40001, message="用户不存在")
    
    user = UserOut(id=uid, name="Static User")
    
    # 正常返回
    return Resp.ok(data=user)

@app.get("/error")
def test_error():
    # 测试异常拦截
    return 1 / 0
```


### 总结：现在的架构流向
1. **Request 进来** $\to$ **Middleware** (读取/生成 ID，存入 `ContextVars`)
2. **路由处理** $\to$ **Resp** (从 `ContextVars` 取 ID，构建 Response)
3. **异常发生** $\to$ **Handler** (从 `ContextVars` 取 ID，构建 Error Response)
4. **Response 返回** $\to$ **Middleware** (从 Header 读 ID，写回 Response Header，清理 `ContextVars`)


## 日志记录(Logging)
### 方案一：使用 Python 标准库 `logging`
这是最通用的做法。核心思路是使用 **`logging.Filter`**。虽然它叫 Filter（过滤器），但它其实有“修改日志记录”的能力。
```python
import logging
from core.context import get_trace_id

# 1. 定义过滤器：自动注入 trace_id
class TraceIDFilter(logging.Filter):
    def filter(self, record):
        # 从 ContextVars 获取当前 ID
        trace_id = get_trace_id()
        # 注入到 record 对象中
        # 如果没有 trace_id (比如系统启动阶段)，给个默认值 "SYSTEM" 或 "N/A"
        record.trace_id = trace_id or "SYSTEM"
        return True

# 2. 定义日志格式
# 注意：这里多了一个 %(trace_id)s
LOG_FORMAT = "%(asctime)s | %(levelname)s | [%(trace_id)s] | %(message)s"

def setup_logging():
    # 获取根日志记录器
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)

    # 创建 Handler (输出到控制台)
    handler = logging.StreamHandler()
    
    # 设置格式
    formatter = logging.Formatter(LOG_FORMAT)
    handler.setFormatter(formatter)
    
    # 🌟 关键：添加过滤器
    handler.addFilter(TraceIDFilter())
    
    # 挂载 Handler
    # (先清除旧的，防止重复打印)
    if logger.hasHandlers():
        logger.handlers.clear()
    logger.addHandler(handler)
```
在main中初始化
```python
from fastapi import FastAPI
from core.log import setup_logging
import logging

# 1. 初始化日志配置
setup_logging()

app = FastAPI()
logger = logging.getLogger(__name__)

@app.get("/")
def root():
    # 2. 直接打印，trace_id 会自动出现
    logger.info("这是一个测试日志")
    return {"msg": "ok"}
```


### 方案二：使用 `Loguru` (推荐 ⭐)
Loguru 只有一个全局对象 `logger`。我们通过 `configure` 来修改它的行为。

```python
import sys
from loguru import logger
from core.context import get_trace_id

def setup_logging():
    # 1. 定义 Patcher (补丁函数)
    # Loguru 允许我们在日志记录之前，修改 record 字典
    def trace_id_patcher(record):
        trace_id = get_trace_id()
        record["extra"]["trace_id"] = trace_id or "SYSTEM"

    # 2. 配置 Loguru
    logger.configure(
        handlers=[
            {
                "sink": sys.stdout,
                # 🌟 在 format 中使用 {extra[trace_id]}
                "format": "<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}:{line}</cyan> | <cyan>[{extra[trace_id]}]</cyan> | <level>{message}</level>",
            }
        ],
        # 🌟 挂载补丁
        patcher=trace_id_patcher
    )
```

在main中使用
```python
from fastapi import FastAPI
from loguru import logger  # 👈 直接导入全局 logger
from core.log import setup_logging

# 初始化配置
setup_logging()

app = FastAPI()

@app.get("/")
def root():
    # 像 print 一样好用
    logger.info("Loguru 真的很爽")
    logger.warning("这是一个警告")
    return {"msg": "ok"}
```

### 方案三：拦截 Uvicorn/FastAPI 的原生日志 (进阶)
FastAPI 启动时（Uvicorn）自己会打印日志，那些日志默认没有 Trace ID。如果你想把 Uvicorn 的日志也统一接管，需要做更深度的配置。

这通常在 `logging.dictConfig` 中完成。这是一个比较完整的**生产级配置**（基于标准库）：

```python
# core/log.py
from core.context import get_trace_id
import logging

class TraceIDFilter(logging.Filter):
    def filter(self, record):
        record.trace_id = get_trace_id() or "SYSTEM"
        return True

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {
        "trace_id_filter": {
            "()": TraceIDFilter, # 使用我们定义的类
        }
    },
    "formatters": {
        "standard": {
            "format": "%(asctime)s - %(levelname)s - [%(trace_id)s] - %(name)s - %(message)s"
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "standard",
            "filters": ["trace_id_filter"], # 🌟 关键：挂载过滤器
        },
    },
    "loggers": {
        # 拦截 FastAPI/Uvicorn 的日志
        "uvicorn": {"handlers": ["console"], "level": "INFO"},
        "uvicorn.access": {"handlers": ["console"], "level": "INFO"},
        # 拦截应用自己的日志
        "": {"handlers": ["console"], "level": "INFO"}, 
    },
}

def setup_logging():
    import logging.config
    logging.config.dictConfig(LOGGING_CONFIG)
```



# 进阶特性与部署 (Advanced & Deployment)
## 后台任务 (Background Tasks)

## WebSockets
WebSockets主要涉及三个类：
- `WebSocket`(对象)：主要用它来发送接收消息。
- `WebSocketDisconnect`(被动异常)：接收消息时，对方结束连接，则会抛出这个异常。需要捕获它来结束本次连接。
- `WebSocketException`(主动异常)：主动结束连接，比如对方认证失败，直接抛出这个异常，FastAPI会帮忙结束连接并告知原因。

| **类名**                    | **角色**    | **动作方向**                | **你的代码怎么写？**                                      | **典型场景**              |
| ------------------------- | --------- | ----------------------- | ------------------------------------------------- | --------------------- |
| **`WebSocket`**           | **操作句柄**  | 双向                      | `await ws.accept()`<br><br>`await ws.send_text()` | 建立连接、收发数据。            |
| **`WebSocketDisconnect`** | **信号/通知** | **Client** $\to$ Server | **`try...except` 捕获它**                            | 处理用户关闭浏览器、断网、离开页面的情况。 |
| **`WebSocketException`**  | **控制/拒绝** | **Server** $\to$ Client | **`raise` 抛出它**                                   | 权限验证失败、数据格式错误、踢人下线。   |
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, WebSocketException, status

app = FastAPI()

# 模拟一个数据库验证
async def validate_token(token: str):
    if token != "secret-token":
        # 场景 C: 主动拒绝
        # 这里的 WebSocketException 就像 HTTP 里的 HTTPException
        # 我们主动 raise 它，FastAPI 会自动关闭连接并发送 code=1008
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int, token: str):
    # --- 1. WebSocket 类：建立连接 ---
    # 我们首先要 accept，才能开始对话
    # 如果上面的 validate_token 抛出了异常，这里就不会执行
    await websocket.accept() 
    
    try:
        while True:
            # --- 1. WebSocket 类：接收消息 ---
            # data = await websocket.receive_text()
            # 如果此时客户端直接关闭了浏览器，这一行会报错！
            data = await websocket.receive_text()
            
            # --- 1. WebSocket 类：发送消息 ---
            await websocket.send_text(f"Message text was: {data}")

    except WebSocketDisconnect:
        # --- 2. WebSocketDisconnect 类：处理断开 ---
        # 这是一个被动异常。我们必须捕获它，否则控制台会报一堆红色的 Error
        # 这里通常用来做“清理工作”，比如把用户从在线名单里移除
        print(f"Client #{client_id} left the chat")
        
    # 注意：我们通常不需要 catch WebSocketException
    # 因为它是给我们自己 raise 用的，FastAPI 会在顶层帮我们处理好
```


## 测试 (Testing)

## 部署 (Deployment)



