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
- `status`: 各种HTTP请求状态码

- `FastAPI`: 主服务类
- `APIRouter`: 路由分组

- `BackgroundTasks`: 后台任务
- `UploadFile`: 接收文件上传
- `HTTPException`: Starlette的异常类
- `Body`: 表示这个参数应该从请求体获取
- `Cookie`: 表示这个参数应该从Cookie获取
- `Header`: 表示这个参数应该从Header获取
- `Path`: 表示这个参数应该从路径参数中获取
- `Query`: 表示这个参数应该从查询参数中获取
- `Depends`: 表示这个参数是通过依赖函数获取的
- `File`: 
- `Form`: 
- `Security`: 
- `Request`:
- `Response`:
- `WebSocket`:
- `WebSocketDisconnect`
- `WebSocketException`: Starlette的WebSocket异常类


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




## 路径参数与查询参数 (Parameters)
### 路径参数(Path)
正常情况下，如果参数名与路径中的标签名相同，是不需要主动声明参数默认值为`Path`的。
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


## 数据验证与 Pydantic (Data Validation)

### Request Body (请求体)

### Response Model (响应模型)

### 表单数据与文件上传



# 依赖注入与生命周期管理 (Dependency Injection & Lifespan)

## 依赖注入

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
    print("🤖 模型模块：开始加载模型...")
    # model = load_heavy_model()
    yield
    print("🤖 模型模块：清理资源...")

# 在 Router 中使用
router = APIRouter(lifespan=router_lifespan)

@router.get("/predict")
def predict():
    return {"result": "prediction"}
```


# 数据库集成 (Database Integration)
## 连接数据库 (SQLAlchemy)

## CRUD 操作实战

## 异步数据库 (Async SQL)

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


# 安全与认证 (Security & Auth)




# 中间件 (Middleware) 与 CORS




# 工程化与架构 (Project Architecture)
## 路由分组 (APIRouter)
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

### 1. 路由控制类参数 (Path & Routing)
#### `prefix` (最常用)
- **类型**: `str`
- **作用**: 给该 Router 下定义的所有路径加上统一的前缀。
- **⚠️ 注意**: 前缀**不要**以 `/` 结尾（FastAPI 会自动处理拼接，写了容易造成双斜杠 `//`）。

#### `redirect_slashes`
- **类型**: `bool`
- **默认值**: `True`
- **作用**: 控制是否自动重定向斜杠。
- **详解**:
    - 如果你定义了接口 `@router.get("/items/")` (带尾部斜杠)。
    - 用户访问 `/items` (不带斜杠)。
    - **默认 (True)**: FastAPI 会返回 `307 Temporary Redirect` 跳转到 `/items/`。
    - **设为 False**: FastAPI 会直接返回 `404 Not Found`。



### 2.文档与元数据类参数 (Docs & Metadata)
这些参数主要影响 Swagger UI (`/docs`) 和 ReDoc 的显示效果。

#### `tags`
- **类型**: `List[str]` 或 `List[Enum]`
- **作用**: 在文档中对接口进行**分组**。如果不写，接口会混在一起；写了之后，Swagger UI 会把它们折叠在一个大标题下。

#### `deprecated`
- **类型**: `bool` (默认 `False`)
- **作用**: 将该 Router 下的**所有**接口标记为“已过时”。文档中会显示一条删除线，但接口依然可用。适合版本迁移时使用。

#### `include_in_schema`
- **类型**: `bool` (默认 `True`)
- **作用**: 是否在文档中显示该 Router 的接口。如果设为 `False`，这些接口就像“隐形”了一样（通常用于内部管理接口）。

#### `generate_unique_id_function`
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
#### `callbacks`
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


### 3.逻辑行为类参数 (Logic & Behavior)
这是 `APIRouter` 最强大的地方，可以实现**批量配置**。

#### `dependencies` (核心)
- **类型**: `List[Depends]`
- **作用**: 给该 Router 下的**所有**接口强制加上依赖。
- **场景**: 比如 `/admin` 模块下的所有接口都需要管理员权限，你不需要在每个 `@router.get` 里写 `Depends(check_admin)`，直接在 `APIRouter` 里定义一次即可。

#### `responses`
- **类型**: `dict`
- **作用**: 定义该 Router 下所有接口可能返回的**通用错误响应**。这会让文档更准确。

#### `default_response_class`
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

## 项目目录结构最佳实践

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



