`SQLAlchemy`是现在最流行的ORM。其中最重要的就是四个概念：
- Engine (引擎): 它负责维护连接池（Connection Pool），管理底层的 TCP 连接。
- Declarative Base (声明式基类) / Model: 你需要定义一个基类，然后让你所有的数据库模型（User, Item）都继承它。这样 SQLAlchemy 才知道哪些类对应哪些表。 _在 SQLModel 中，`SQLModel` 类本身就是这个 Base。_
- Session (会话): 暂存区或者说一个事务，只有手动commit之后数据才会被真正写入。
- Alembic (迁移工具): SQLAlchemy 只负责运行时的操作，不负责建表和改表结构。`Alembic` 负责生成迁移脚本，告诉数据库如何修改表结构。

安装：
```bash
pip install sqlalchemy
```

# 引擎
首先是创建引擎：
```python
from sqlalchemy import create_engine

# 格式: dialect+driver://username:password@host:port/database

DATABASE_URL = "mysql+pymysql://root:123456@localhost/test_db"

# ⚠️ 注意：这行代码执行时，并没有真正连接数据库！
# 它是"惰性"的，只有当你真正发起查询时，才会建立连接。
engine = create_engine(
    DATABASE_URL, 
    echo=True,          # echo=True: 会在控制台打印生成的 SQL，开发调试神器
    pool_size=10,       # 池子里平时保持 10 个连接
    max_overflow=20     # 忙不过来时，最多再临时创建 20 个
)
```

通常来说，整个应用程序只需要创建一个引擎就够了。



# Declarative Base / Model (模型)
它是 Python 类与数据库表之间的**映射协议**。

作用：
- **翻译字典**：告诉 SQLAlchemy，Python 里的 `class User` 对应数据库里的 `users` 表；属性 `name` 对应字段 `varchar(50)`。
- **元数据中心 (Metadata)**：`Base` 类有一个 `metadata` 属性，它像一个笔记本，记录了所有继承自它的子类。当你运行 `Base.metadata.create_all(engine)` 时，它就是翻看这个笔记本，去数据库把缺少的表建出来。

现在的写法使用了 Python 的 Type Hints (`Mapped`)，非常直观。

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String
from typing import Optional

# 1. 定义基类 (所有模型的祖宗)
class Base(DeclarativeBase):
    pass

# 2. 定义业务模型
class User(Base):
    __tablename__ = "users"  # 必填：数据库表名

    # Mapped[int] -> Python 类型
    # mapped_column(...) -> 数据库约束
    
    # 主键，自动增长
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # String(50) -> VARCHAR(50)
    name: Mapped[str] = mapped_column(String(50))
    
    # Optional -> 允许为 NULL
    age: Mapped[Optional[int]] = mapped_column()

    def __repr__(self):
        return f"<User {self.name}>"
```



# Session
**请求级生命周期**：**这是铁律**。一个 HTTP 请求进来 -> 创建一个 Session -> 处理业务 -> 提交/回滚 -> 关闭 Session。
千万不要创建一个全局的 `session` 变量给所有请求共用，那是线程不安全的，会导致数据乱套。

最传统的Session使用是：
```python
from sqlalchemy.orm import sessionmaker

# 1. 造一个工厂 (Factory)
# 因为每个请求都需要一个新的 Session，所以我们需要一个工厂来生产它
SessionLocal = sessionmaker(bind=engine)

# 2. 生产 Session (通常在依赖注入里做)
session = SessionLocal()

try:
    # --- 开始干活 ---
    user = User(name="Jerry")
    session.add(user)  # 放入沙箱 (State: Pending)
    
    # 此时 user.id 是 None，因为还没存库
    
    session.commit()   # 提交事务 (State: Persistent)
    # 此时数据库里有数据了，SQLAlchemy 会自动把生成的 ID 填回 user.id
    
    # --- 继续干活 ---
    user.name = "Tom"  # 修改属性 (State: Dirty)
    session.commit()   # 再次提交自动生成 UPDATE
    
except Exception:
    session.rollback() # 出错了？把沙箱倒空，撤销所有未提交的操作
    raise
finally:
    session.close()    # 关门送客，把连接还给 Engine 的连接池
```



