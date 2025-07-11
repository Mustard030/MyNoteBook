
# CSRF、CORS、Session

django settings里面安全相关的配置分为四大类`ALLOWED_HOSTS`、`CSRF_*`、`CORS_*`、`SESSION_*`
CSRF保护的是表单提交等操作，而CORS保护的是前端资源请求，CSRF 关注的是请求的 ​**​来源（Origin）是否可信​**​（用于表单提交等改变状态的操作），CORS 关注的是浏览器是否允许前端代码 ​**​读取响应​**​。

### 关键总结与联系​​

| 设置类别              | 核心目的                | 相互关系与注意事项                                                       |
| ----------------- | ------------------- | --------------------------------------------------------------- |
| `ALLOWED_HOSTS`   | ​**​基础安全​**​，过滤非法主机 | 必须正确配置生产环境域名                                                    |
| `CSRF_*`          | ​**​防跨站提交保护​**​     | `CSRF_TRUSTED_ORIGINS` 允许非安全来源; `CSRF_COOKIE_DOMAIN` 与跨子域相关     |
| `CORS_*`          | ​**​控制跨域AJAX访问​**​  | 依赖 `django-cors-headers`；`CORS_ALLOW_CREDENTIALS=True` 时不能使用通配符 |
| `*_COOKIE_DOMAIN` | ​**​跨子域共享认证状态​**​   | `SESSION_COOKIE_DOMAIN` 和 `CSRF_COOKIE_DOMAIN` 通常同步设置           |
| ### 常见配置场景示例​​    |                     |                                                                 |

​**​场景：前后端分离开发 (localhost:3000 -> localhost:8000)​**​

```python
ALLOWED_HOSTS = ['localhost', '127.0.0.1']
CSRF_TRUSTED_ORIGINS = ['http://localhost:3000']  # 允许HTTP来源
CORS_ORIGIN_WHITELIST = ['http://localhost:3000']  # 允许跨域来源
CORS_ALLOW_CREDENTIALS = True  # 允许带Cookie
```

​**​场景：生产环境主站+API子域 (front.example.com -> api.example.com)​**​

```python
ALLOWED_HOSTS = ['.example.com']  # 所有子域
SESSION_COOKIE_DOMAIN = '.example.com'     # 登录Cookie共享
CSRF_COOKIE_DOMAIN = '.example.com'        # CSRF Cookie共享
CORS_ORIGIN_WHITELIST = ['https://www.example.com']  # 仅允许主站跨域
```

​**​重要安全原则​**​:

- ​**​永不​**​在生产环境设置 `DEBUG = True`
- ​**​永不​**​在生产环境使用 `CORS_ORIGIN_ALLOW_ALL = True`
- `ALLOWED_HOSTS` 是基础安全屏障，必须配置正确
- 使用HTTPS时，`CSRF_TRUSTED_ORIGINS` 应明确包含前端地址

## 基础安全设置

### ALLOWED_HOSTS

**作用​**​: ​**​安全基础​**​，防止 HTTP Host 头攻击。Django 只响应请求头中 `Host` 或 `X-Forwarded-Host` 与此列表匹配的请求。Django 在验证 HTTP Host 头时，会**忽略**端口部分，只比较域名或 IP 地址。允许一个域名意味着​**​允许该域名的所有端口请求​**

```python
ALLOWED_HOSTS = [
    'example.com',        # 精确域名
    '.example.com',       # 匹配 *.example.com 的所有子域
    'localhost',
    '127.0.0.1',
    '[::1]',               # IPv6
]
```

**注意​**​:

- ​**​开发环境​**​: 可设为 `['*']`（​**​不安全，仅限测试!​**​）
- ​**​生产环境​**​: ​**​必须​**​配置为你的正式域名/IP。
- **代理/负载均衡器​**​：当使用 Nginx/Apache 时，确保它们正确传递 `Host` 头（不含端口）

## CSRF 相关设置 (防止跨站请求伪造)​

### CSRF_TRUSTED_ORIGINS

**​作用​**​: 允许 ​**​不安全协议（如 HTTP）或非标准端口​**​ 的来源绕过 HTTPS 检查。定义哪些来源被信任，用于接收合法的 CSRF token。

**何时需要​**​:

- 前端从 `http://localhost:3000` 访问后端 `https://api.example.com`
- 后端运行在非标准端口（如 `https://example.com:8443`）

​

**​配置​**​:

```python
CSRF_TRUSTED_ORIGINS = [
    'https://*.example.com',  # 允许所有子域的 HTTPS
    'http://localhost:3000',   # 开发环境明确允许
]
```

### CSRF_COOKIE_DOMAIN

​**​作用​**​: 控制 `csrftoken` Cookie 的作用域（Domain）。

​**​典型场景​**​: 共享 Cookie 跨子域（如 `auth.example.com` 设置 Cookie 让 `shop.example.com` 使用）。

​**​配置​**​:

```python
 CSRF_COOKIE_DOMAIN = '.example.com'  # 设置顶级域名，所有子域共享 Cookie
```

​**​默认​**​: `None`（每个域名独享自己的 Cookie）

## CORS 相关设置

> 需 `django-cors-headers` 包

### CORS_ORIGIN_WHITELIST

**CORS_ORIGIN_WHITELIST 匹配的是请求头中的 `Origin` 头部**。当浏览器发起一个跨域请求时，会自动在请求头中添加一个 `Origin` 字段，**其值为请求发起页面的源（协议+域名+端口）**。例如，如果前端应用运行在 `https://www.example.com:8080`，那么请求头中的 `Origin` 就是 `https://www.example.com:8080`。

**​作用​**​: 精确指定 ​**​允许跨域请求的来源列表​**​（浏览器会阻止不在列表中的跨域 AJAX 请求）。

**配置​**​:

```python
CORS_ORIGIN_WHITELIST = [
    'https://frontend.example.com',
    'http://localhost:8080',
    'https://*.trusted-app.com',
]
```

**注意**：

- 协议（http 或 https）和端口（如果指定）都是匹配的一部分。例如，`http://example.com` 和 `https://example.com` 是不同的源。
- 如果端口没有明确指定，则默认端口（HTTP为80，HTTPS为443）会被省略。例如，`https://example.com` 和 `https://example.com:443` 是等价的。
- 当 `CORS_ALLOW_CREDENTIALS = True` 时，​**​禁止使用通配符​**（如 `https://*.example.com`），且必须明确列出所有允许的来源（包括端口，如果端口不是默认端口的话）。这是因为当需要发送凭证时，浏览器要求 Access-Control-Allow-Origin 响应头必须是具体的值，不能是通配符。
- 通配符 `*.` 仅匹配一级子域（`https://app.domain.com`），不匹配多级子域（`https://dev.app.domain.com`）

### CORS_ORIGIN_ALLOW_ALL

**作用​**​: ​**​禁用白名单​**​，允许 ​**​所有来源​**​ 的跨域请求（⚠️ ​**​高危！生产环境应避免使用​**​）。
​
**​配置​**​:

```python
CORS_ORIGIN_ALLOW_ALL = True  # 替代 `CORS_ORIGIN_WHITELIST`
```

### CORS_ALLOW_CREDENTIALS

**作用​**​: 控制浏览器是否在跨域请求中发送 ​**​凭证（Cookies、HTTP认证等）​**​。
​
**​必须与前端配合​**​: 前端需要设置 `withCredentials = true`（例如在 axios 或 fetch 中）。

**配置​**​:

```python
CORS_ALLOW_CREDENTIALS = True  # 默认为 False
```

**限制​**​: 当启用时，​**​不能​**​同时使用 `CORS_ORIGIN_ALLOW_ALL = True`，且 `CORS_ORIGIN_WHITELIST` 必须明确指定域名（不能包含通配符如 `https://*.example.com`）。

### CORS_ALLOW_HEADERS

**​作用​**​: 控制跨域请求中允许的 ​**​额外 HTTP 请求头​**​（超出浏览器默认安全集）。

**默认包含常用头​**​: `'Content-Type'`, `'Authorization'` 等已内置。

**扩展配置​**​:

```python
CORS_ALLOW_HEADERS = [
    'content-disposition',  # 允许前端接收下载头
    'x-custom-header',      # 自定义头
]
```

### CORS_ALLOW_METHODS

**作用​**​: 控制跨域请求中允许使用的 ​**​HTTP 方法​**​。

**默认包含**: `['DELETE', 'GET', 'OPTIONS', 'PATCH', 'POST', 'PUT']`

## Session相关设置

### SESSION_COOKIE_DOMAIN

**作用​**​: 控制 `sessionid` Cookie 的作用域（Domain），实现跨子域共享登录状态

**配置​**​:

```python
SESSION_COOKIE_DOMAIN = '.example.com'  # 顶级域名，所有子域可访问此 Cookie
```

**默认​**​: `None`（每个域名独享 Session Cookie）

# 数据库操作

## 读写分离

在 Django 中，**同一张表的读写分离（写操作走主库，读操作走从库）可以通过自定义数据库路由实现**

> setting.py

```python
DATABASES = {
    'default': {  # 主库
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'master_db',
        'USER': 'root',
        'PASSWORD': 'password',
        'HOST': 'master_host',
        'PORT': 3306,
    },
    'slave': {  # 从库
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'slave_db',
        'USER': 'slave_user',
        'PASSWORD': 'slave_pass',
        'HOST': 'slave_host',
        'PORT': 3306,
    }
}

DATABASE_ROUTERS = ['path.to.routers.AutoReadWriteRouter']
```

> routers.py

```python
class AutoReadWriteRouter:
    def db_for_read(self, model, **hints):
        """读操作走从库"""
        return 'slave'

    def db_for_write(self, model, **hints):
        """写操作走主库"""
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        """允许跨库关联（根据需求调整）"""
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """迁移时仅操作主库"""
        return db == 'default'
```

使用：

```
# 写入主库
User.objects.create(name="Alice")  # 使用 default 数据库
# 读取从库
users = User.objects.all()  # 使用 slave 数据库
# 强制读主库（例如需要实时数据时）
user = User.objects.using('default').get(id=1)
```

多从库负载均衡

```python
import random

class LoadBalancedRouter:
    SLAVE_DATABASES = ['slave1', 'slave2']

    def db_for_read(self, model, **hints):
        return random.choice(self.SLAVE_DATABASES)
```

按条件路由（如特定Model强制读取主库）

```python
class CriticalModelRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'payment':  # 关键业务模型
            return 'default'
        return 'slave'
```

Django 默认事务使用主库，跨库操作需手动管理：

```python
from django.db import transaction

with transaction.atomic(using='default'):
    # 主库写入
    user = User.objects.using('default').create(name="Bob")
    # 主库读取（确保一致性）
    user = User.objects.using('default').get(id=user.id)
```

迁移操作必须指定主库

```shell
python manage.py makemigrations
python manage.py migrate --database=default
```

迁移前建议先看看迁移计划

```shell
python manage.py makemigrations --dry-run <app_name>
```

## migrate工具

### makemigrations

```shell
usage: manage.py makemigrations [-h] [--dry-run] [--merge] [--empty] [--noinput] [-n NAME] [--no-header] [--check] [--version] [-v {0,1,2,3}] [--settings SETTINGS] [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color] [--skip-checks] [app_label [app_label ...]]

Creates new migration(s) for apps.

positional arguments:
  app_label             Specify the app label(s) to create migrations for.

optional arguments:
  -h, --help            show this help message and exit
  --dry-run             显示在不向磁盘写入任何迁移文件的情况下进行的迁移。使用这个选项和 --verbosity 3 也会显示将被写入的完整迁移文件。
  --merge               可以解决迁移冲突。
  --empty               为指定的应用程序输出一个空的迁移，用于手动编辑。这是为高级用户准备的，除非你熟悉迁移格式、迁移操作和迁移之间的依赖关系，否则不应使用。
  --noinput, --no-input 压制所有用户提示。如果不能自动解决被抑制的提示，命令将以 error code 3 退出
  -n NAME, --name NAME  允许对生成的迁移进行命名，而不是使用生成的名称。名称必须是有效的 Python 标识符
  --no-header           生成没有 Django 版本和时间戳头的迁移文件。
  --check               在检测到模型更改而没有迁移时，使 `makemigrations` 以非零状态退出。意味着 --dry-run。
  --version             show program's version number and exit
  -v {0,1,2,3}, --verbosity {0,1,2,3}
                        Verbosity level; 0=minimal output, 1=normal output, 2=verbose output, 3=very verbose output
  --settings SETTINGS   The Python path to a settings module, e.g. "myproject.settings.main". If this isn't provided, the DJANGO_SETTINGS_MODULE environment variable will be used.
  --pythonpath PYTHONPATH
                        A directory to add to the Python path, e.g. "/home/djangoprojects/myproject".
  --traceback           Raise on CommandError exceptions
  --no-color            Don't colorize the command output.
  --force-color         Force colorization of the command output.
  --skip-checks         Skip system checks.
```

通常建议先使用`--dry-run`来看看会执行什么操作，并且指定`<app_name>`来限制只变更自己修改的那个app。
一般来说不太会用到`--merge`，因为很少会改动数据库结构

例：

```shell
python manage.py makemigrations --dry-run <app_name>  # 只展示迁移，不生成实际文件
python manage.py makemigrations <app_name>
```

### migrate

当取消应用迁移时，所有依赖的迁移也将被取消应用，无论 `<app_label>`。你可以使用 `--plan` 来检查哪些迁移将被取消应用。

```shell
usage: manage.py migrate [-h] [--noinput] [--database DATABASE] [--fake] [--fake-initial] [--plan] [--run-syncdb] [--check] [--version] [-v {0,1,2,3}] [--settings SETTINGS] [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color] [--skip-checks] [app_label] [migration_name]

Updates database schema. Manages both apps with migrations and those without.

positional arguments:
  app_label             App label of an application to synchronize the state.
  migration_name        Database state will be brought to the state after that migration. Use the name "zero" to unapply all migrations.

optional arguments:
  -h, --help            show this help message and exit
  --noinput, --no-input
                        Tells Django to NOT prompt the user for input of any kind.
  --database DATABASE   Nominates a database to synchronize. Defaults to the "default" database.
  --fake                Mark migrations as run without actually running them.
  --fake-initial        Detect if tables already exist and fake-apply initial migrations if so. Make sure that the current database schema matches your initial migration before using this flag. Django will only check for an existing table name.
  --plan                Shows a list of the migration actions that will be performed.
  --run-syncdb          Creates tables for apps without migrations.
  --check               Exits with a non-zero status if unapplied migrations exist.
  --version             show program's version number and exit
  -v {0,1,2,3}, --verbosity {0,1,2,3}
                        Verbosity level; 0=minimal output, 1=normal output, 2=verbose output, 3=very verbose output
  --settings SETTINGS   The Python path to a settings module, e.g. "myproject.settings.main". If this isn't provided, the DJANGO_SETTINGS_MODULE environment variable will be used.
  --pythonpath PYTHONPATH
                        A directory to add to the Python path, e.g. "/home/djangoprojects/myproject".
  --traceback           Raise on CommandError exceptions
  --no-color            Don't colorize the command output.
  --force-color         Force colorization of the command output.
  --skip-checks         Skip system checks.
```

例：

```shell
python manage.py migrate --plan <app_name>  # 显示将要执行的迁移操作列表（不实际执行）
python manage.py migrate <app_name>         # 应用所有未应用的迁移
python manage.py migrate <app_name> <migration_name:如0002> # 应用特定迁移版本

python manage.py migrate <app_name> zero    # 完全回滚应用（到零状态）
python manage.py migrate --verbosity 2      # 显示详细的迁移执行过程  - 可选级别: 0=静默, 1=正常, 2=详细, 3=非常详细
python manage.py migrate --database <db_name>  # 指定要迁移的数据库（在settings.DATABASES中配置）
python manage.py migrate --fake <app_name>  # 假迁移 标记迁移为完成状态但不实际执行SQL 当迁移已手动应用或不需要操作时使用
```

### showmigrations

查看migration有哪些已经执行了，哪些还没执行，已执行的会显示`[X]  <app_name>.<编号>_xxx`，未执行的是`[ ]`

```shell
usage: manage.py showmigrations [-h] [--database DATABASE] [--list | --plan] [--version] [-v {0,1,2,3}] [--settings SETTINGS] [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color] [--skip-checks] [app_label [app_label ...]]

Shows all available migrations for the current project

positional arguments:
  app_label             App labels of applications to limit the output to.

optional arguments:
  -h, --help            show this help message and exit
  --database DATABASE   Nominates a database to synchronize. Defaults to the "default" database.
  --list, -l            列出 Django 知道的所有应用，每个应用的可用迁移，以及每个迁移是否被应用（在迁移名称旁边用 `[X]` 标记）。对于 `--verbosity` 2 及以上的应用，也会显示应用的日期时间。没有迁移的应用程序也在列表中，但下面印有 `(no migrations)`。这是默认的输出格式。
  --plan, -p            显示 Django 将遵循的迁移计划。和 `--list` 一样，应用的迁移也用 `[X]` 标记。对于 `--verbosity` 2 以上，也会显示迁移的所有依赖关系。
  --version             show program's version number and exit
  -v {0,1,2,3}, --verbosity {0,1,2,3}
                        Verbosity level; 0=minimal output, 1=normal output, 2=verbose output, 3=very verbose output
  --settings SETTINGS   The Python path to a settings module, e.g. "myproject.settings.main". If this isn't provided, the DJANGO_SETTINGS_MODULE environment variable will be used.
  --pythonpath PYTHONPATH
                        A directory to add to the Python path, e.g. "/home/djangoprojects/myproject".
  --traceback           Raise on CommandError exceptions
  --no-color            Don't colorize the command output.
  --force-color         Force colorization of the command output.
  --skip-checks         Skip system checks.
```

例：

```shell
python manage.py showmigrations <app_name>
```

### sqlmigrate

```shell
usage: manage.py sqlmigrate [-h] [--database DATABASE] [--backwards] [--version] [-v {0,1,2,3}] [--settings SETTINGS] [--pythonpath PYTHONPATH] [--traceback] [--no-color] [--force-color] [--skip-checks] app_label migration_name

Prints the SQL statements for the named migration.

positional arguments:
  app_label             App label of the application containing the migration.
  migration_name        Migration name to print the SQL for.

optional arguments:
  -h, --help            show this help message and exit
  --database DATABASE   Nominates a database to create SQL for. Defaults to the "default" database.
  --backwards           生成用于解除应用迁移的 SQL。默认情况下，所创建的 SQL 是用于向前运行迁移。
  --version             show program's version number and exit
  -v {0,1,2,3}, --verbosity {0,1,2,3}
                        Verbosity level; 0=minimal output, 1=normal output, 2=verbose output, 3=very verbose output
  --settings SETTINGS   The Python path to a settings module, e.g. "myproject.settings.main". If this isn't provided, the DJANGO_SETTINGS_MODULE environment variable will be used.
  --pythonpath PYTHONPATH
                        A directory to add to the Python path, e.g. "/home/djangoprojects/myproject".
  --traceback           Raise on CommandError exceptions
  --no-color            Don't colorize the command output.
  --force-color         Force colorization of the command output.
  --skip-checks         Skip system checks.
```

例：

```shell
python manage.py sqlmigrate <app_label> <migration_name>  # 展示前向的sql变更具体语句
python manage.py sqlmigrate --backwards <app_label> <migration_name>  # 展示反向的sql变更具体语句（比如取消这个变更）
```

# DRF

## 请求

REST框架的Request类扩展了标准的HttpRequest。DRF 的 Request 是 Django HttpRequest 的​**​包装器​**​，添加了 REST API 特有的功能。

### Django HttpRequest 详解

| 属性                     | 类型               | 描述                                                                                            |
| ---------------------- | ---------------- | --------------------------------------------------------------------------------------------- |
| `request.method`       | 字符串              | HTTP 方法: 'GET','POST','PUT'等                                                                  |
| `request.GET`          | `QueryDict`      | ​**​查询参数​**​（URL 中的 ?key=value）                                                               |
| `request.POST`         | `QueryDict`      | ​**​表单数据​**​（仅限 POST）                                                                         |
| `request.FILES`        | `MultiValueDict` | 上传的文件                                                                                         |
| `request.path`         | 字符串              | 完整路径（不含域名）                                                                                    |
| `request.path_info`    | 字符串              | URL 的路径部分                                                                                     |
| `request.content_type` | 字符串              | 内容类型（如 'application/json'）                                                                    |
| `request.META`         | 字典               | 原始 HTTP 头（全大写，前缀 HTTP_）                                                                       |
| `request.COOKIES`      | 字典               | 请求携带的 Cookie                                                                                  |
| `request.session`      | `SessionStore`   | 会话对象                                                                                          |
| `request.user`         | `User`           | 认证用户对象（需中间件支持）如果请求未经身份验证， `则 request.user` 的默认值为 `django.contrib.auth.models.AnonymousUser` 。 |

#### 常用方法

| 方法                             | 描述                                     |
| ------------------------------ | -------------------------------------- |
| `request.is_ajax()`            | 判断是否为 AJAX 请求（检查 `X-Requested-With` 头） |
| `request.is_secure()`          | 判断是否通过 HTTPS 访问                        |
| `request.get_full_path()`      | 获取完整路径（含查询字符串）                         |
| `request.build_absolute_uri()` | 构建完整绝对 URL                             |
| `request.get_host()`           | 获取主机名（含端口）                             |

### DRF Request 详解

| 属性                            | 类型                   | 描述                                                                                                   |
| ----------------------------- | -------------------- | ---------------------------------------------------------------------------------------------------- |
| `request.data`                | `dict` 或 `QueryDict` | ​**​核心特性​**​：  <br>请求体自动解析（支持 JSON/表单等），它支持解析 `POST` 以外的 HTTP 方法的内容，这意味着您可以访问 `PUT` 和 `PATCH` 请求的内容。 |
| `request.query_params`        | `QueryDict`          | 查询参数（`request.GET` 的别名）                                                                              |
| `request.parser_context`      | 字典                   | 解析上下文（包含视图、请求方法等）                                                                                    |
| `request.accepted_renderer`   | 渲染器                  | 当前使用的渲染器对象                                                                                           |
| `request.accepted_media_type` | 字符串                  | 接受的媒体类型                                                                                              |
| `request.version`             | 字符串                  | API 版本标识符                                                                                            |
| `request.user`                | 增强实现                 | ​**​包含认证用户信息​**​（比 Django 更强大）                                                                       |
| `request.auth`                | 任意                   | 认证凭据（如 Token、JWT 等），如果请求未经身份验证，或者不存在其他上下文， `则 request.auth` 的默认值为 `None`。                            |

### 关键方法

| 方法                                       | 描述              |
| ---------------------------------------- | --------------- |
| `request.query_params.get(key, default)` | 获取查询参数          |
| `request.data.get(key, default)`         | 获取解析后的请求体数据     |
| `request.stream`                         | 原始请求数据流（用于流式处理） |
| `request.successful_authenticator`       | 返回成功认证器实例       |

## 响应

> REST 框架通过提供 `Response` 类来支持 HTTP 内容协商，该类允许您返回可呈现为多种内容类型的内容，具体取决于客户端请求。
> `Response` 类是 Django 的 `SimpleTemplateResponse` 的子类。 `响应`对象使用数据进行初始化，数据应由本机 Python 基元组成。然后，REST 框架使用标准 HTTP 内容协商来确定它应该如何呈现最终的响应内容。
> 你不是必须使用 `Response` 类，还可以从视图中返回常规的 `HttpResponse` 或 `StreamingHttpResponse` 对象。使用 `Response` 类只是提供了一个更好的接口，用于返回内容协商的 Web API 响应，该响应可以呈现为多种格式。
> 除非出于某种原因想要对 REST 框架进行大量自定义，否则应始终对返回 `Response` 对象的视图使用 `APIView` 类或 `@api_view` 函数。这样做可以确保视图可以执行内容协商，并在从视图中返回响应之前为响应选择适当的渲染器。

`from rest_framework.response import Response`
签名：`Response(data, status=None, template_name=None, headers=None, content_type=None)`
参数：

- `data`：响应的序列化数据。
- `status`：响应的状态代码。默认值为 200。另请参阅 [状态代码](https://www.django-rest-framework.org/api-guide/status-codes/) 。
- `template_name`：选择 `HTMLRenderer` 时要使用的模板名称。
- `headers`：要在响应中使用的 HTTP 标头的字典。
- `content_type`：响应的内容类型。通常，这将由渲染器自动设置，由内容协商确定，但在某些情况下，可能需要显式指定内容类型。

通常来说只需要设置`data`和`status`就行

```python
serializer = self.get_serializer(data=request_data)
serializer.is_valid(raise_exception=True)
return Response(serializer.data)
```

注意：data不能直接传入`Serializer`对象，必须调用`serializer.data`

## 视图

视图大体上分为两类：`CBV`(基于类的视图)和`FBV`(基于方法的视图)

### FBV

#### Django原生

是Django中最简单的视图实现方式，通过 Python 函数处理 HTTP 请求并返回响应。
需手动判断请求方法（如 GET/POST），通过 `request.method` 分支处理不同逻辑

```python
def contact_view(request):
    if request.method == 'GET':
        return render(request, 'contact.html')
    elif request.method == 'POST':
        name = request.POST.get('name')
        return HttpResponse(f"Thanks, {name}!")
        
def download_file(request):
    file = open('report.pdf', 'rb')
    return FileResponse(file, as_attachment=True)
```

适合如返回静态页面、API 接口等一次性功能。
配置映射：

```python
from django.urls import path
from . import views

urlpatterns = [
    path('hello/', views.hello_world),
    path('contact/', views.contact_view),
]
```

---

#### DRF

DRF中也有FBV的增强方法。DRF提供了一组简单的装饰器，这些装饰器包装了基于函数的视图，以确保它们接收 `Request` 的实例（而不是通常的 Django `HttpRequest`），并允许它们返回 `Response`（而不是 Django `HttpResponse`），并允许你配置请求的处理方式。

##### @api_view()

**签名：** `@api_view(http_method_names=['GET'])`
此功能的核心是 `api_view` 装饰器，它接受视图应响应的 HTTP 方法列表。例如，这是编写一个非常简单的视图的方式，该视图只是手动返回一些数据：

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view()
def hello_world(request):
    return Response({"message": "Hello, world!"})
```

此视图将使用[设置](https://www.django-rest-framework.org/api-guide/settings/)中指定的默认渲染器、解析器、身份验证类等。

默认情况下，仅接受 `GET` 方法。其他方法将响应 “405 Method Not Allowed”。要更改此行为，请指定视图允许的方法，如下所示：

```python
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Got some data!", "data": request.data})
    return Response({"message": "Hello, world!"})
```

###### API 策略装饰器

为了覆盖默认设置，REST 框架提供了一组额外的装饰器，这些装饰器可以添加到您的视图中。这些必须位于 `@api_view` 装饰器之后（下面）。例如，要创建一个使用[限流](https://www.django-rest-framework.org/api-guide/throttling/)的视图，以确保特定用户每天只能调用一次，请使用 `@throttle_classes` 装饰器，并传递限流类列表：

```python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
    rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

这些装饰器对应于 `APIView` 子类上设置的属性，如上所述。

可用的装饰器有：

- `@renderer_classes(...)`
- `@parser_classes(...)`
- `@authentication_classes(...)`
- `@throttle_classes(...)`
- `@permission_classes(...)`

这些装饰器中的每一个都接受一个参数，该参数必须是类的列表或元组。

###### 视图架构装饰器

要覆盖FBV的默认生成文档方式，您可以使用 `@schema` 装饰器。这必须在 `@api_view` _之后（下面）_ 装饰。例如：

```python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

此装饰器采用单个 `AutoSchema` 实例、`AutoSchema` 子类实例或 `ManualSchema` 实例，如 [Schemas 文档](https://www.django-rest-framework.org/api-guide/schemas/)中所述。您可以传递 `None` 以便从文档中排除视图。

```python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```

### CBV

CBV分为两部分，分别是**Django原生**的类和**DRF**的类

**Django原生**：\
View -> TemplateView, RedirectView -> ListView, DetailView -> CreateView, UpdateView, DeleteView

**DRF**：\
APIView -> GenericAPIView（配合Mixins） -> 组合的通用视图（如ListAPIView） -> 视图集（ViewSet, ModelViewSet） -> 自定义动作（@action）

#### Django原生CBV

##### View

**抽象程度最低​**​，需要完全手动处理请求。允许的方法名为`http_method_names = ['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace']`
**核心机制​**​：

- 通过 `dispatch()` 方法自动路由请求到对应的 HTTP 方法处理器
- 必须覆盖实现 `get()`/`post()` 等方法

**适用场景​**​：需要绝对控制权的特殊需求

```python
from django.views import View

class ManualView(View):
    def get(self, request, *args, **kwargs):
        return HttpResponse("GET handled manually")

    def post(self, request, *args, **kwargs):
        return HttpResponse("POST handled manually")
```

##### TemplateView

预配置模板渲染
签名：`class TemplateView(TemplateResponseMixin, ContextMixin, View)`

```python
class AboutView(TemplateView):
    template_name = "about.html"
    extra_context = {"company": "Tech Inc"}
```

可配置项：

```python
class TemplateResponseMixin:
    """A mixin that can be used to render a template."""
    template_name = None  # 模板名称
    template_engine = None
    response_class = TemplateResponse
    content_type = None
    
class ContextMixin:
    """
    A default context mixin that passes the keyword arguments received by
    get_context_data() as the template context.
    """
    extra_context = None  # 模板需要传递的参数
```

##### RedirectView

预设重定向行为
可配置项：

```python
class RedirectView(View):
    """Provide a redirect on any GET request."""
    permanent = False  # False=临时重定向 (302)  True=​​永久重定向 (301)
    url = None  # 直接指定目标 URL 字符串（例如 `url="/new-path/"`）
    pattern_name = None  # 通过 Django URL 名称反向生成 URL（例如 `pattern_name="new_view_name"`）
    query_string = False  # 设置 `query_string=True` 时，原 URL 的查询参数（如 `?key=value`）会附加到目标 URL

	def get_redirect_url(self, *args, **kwargs):
	"""
	Return the URL redirect to. Keyword arguments from the URL pattern
	match generating the redirect request are provided as kwargs to this
	method.
	"""
	...
```

临时重定向（302） vs 永久重定向（301）：核心区别

| ​**​特性​**​        | ​**​临时重定向 (302)​**​                    | ​**​永久重定向 (301)​**​                                  |
| ----------------- | -------------------------------------- | ---------------------------------------------------- |
| ​**​HTTP 状态码​**​  | 302 Found                              | 301 Moved Permanently                                |
| ​**​设计目的​**​      | 临时资源位置变更                               | 永久性资源位置变更                                            |
| ​**​浏览器行为​**​     | 每次访问都向原始 URL 发送请求                      | 自动缓存重定向，后续直接跳新 URL                                   |
| ​**​SEO 影响​**​    | 保持原 URL 权重，不传递权重                       | 原 URL 权重完全转移到新 URL                                   |
| ​**​典型场景​**​      | • 临时维护页面  <br>• A/B 测试  <br>• 登录后返回原页面 | • 网站改版/域名更换  <br>• URL 结构永久变更  <br>• HTTP → HTTPS 迁移 |
| ​**​性能影响​**​      | 每次访问都需服务端响应                            | 后续访问浏览器直接跳转（减少请求）                                    |
| ​**​缓存行为​**​      | 默认不缓存                                  | 被浏览器和代理服务器长期缓存                                       |
| ​**​用户影响​**​      | 每次访问都经历重定向                             | 后续访问无感直达新页面                                          |
| ​**​Django 实现​**​ | `RedirectView(permanent=False)`        | `RedirectView(permanent=True)`                       |

1. **缓存机制区别​**​
   - 301：浏览器和 CDN 会永久缓存重定向关系\
     （除非强制清除缓存）
   - 302：不缓存，每次请求都需要服务器确认
2. ​**​SEO 后果（关键差异）​**​
   - ​**​301​**​：搜索引擎会将旧 URL 的排名信号、外链权重完全转移到新 URL，旧 URL 会逐渐从索引中移除
   - ​**​302​**​：搜索引擎继续索引旧 URL，不会传递任何权重到新地址（可能被判定为试图操纵排名）

##### Django 的 CRUD 集成视图

| 视图类          | 核心功能   | 主要配置项                        |
| ------------ | ------ | ---------------------------- |
| `ListView`   | 对象列表展示 | `model`, `queryset`          |
| `DetailView` | 单个对象详情 | `slug_field`, `pk_url_kwarg` |
| `CreateView` | 新建对象   | `form_class`, `fields`       |
| `UpdateView` | 更新对象   | `template_name_suffix`       |
| `DeleteView` | 删除对象   | `success_url`                |

---

#### DRF CBV

##### APIView

**DRF 体系的底层基础**，继承自 Django 的 `View` 类，是 DRF 所有 CBV 的基类，提供 HTTP 方法分发的底层逻辑（如 `get()` 处理 GET 请求），从这里开始`request`对象被[[#DRF Request 详解|DRF增强]]了

​**核心增强​**​：

- 增强`request`对象
- `dispatch()` 方法解析请求方法调用对应的处理器（`get()`/`post()`）
- 自动解析 content-type（JSON/表单等）
- 集成认证、权限、限流系统
- 自动生成 `OPTIONS` 方法响应

```python
from rest_framework.views import APIView

# 可配置项
class APIView(View):
    # The following policies may be set at either globally, or per-view.
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES  # 控制响应数据的格式（如 JSON）如：renderer_classes = [JSONRenderer]
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES  # 控制请求数据的格式（如 JSON）如：parser_classes = [JSONParser]
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES  # 认证
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES  # 限流
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES  # 权限
    content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
    metadata_class = api_settings.DEFAULT_METADATA_CLASS
    versioning_class = api_settings.DEFAULT_VERSIONING_CLASS

# 示例
class BasicAPIView(APIView):
    def get(self, request, format=None):
        return Response({"status": "basic API response"})
```

**注册方式：**

```python
# myapp/urls.py
from django.urls import path, include
from .views import UserAPIView

urlpatterns = [
    # 最基础形式
    path('users/', UserAPIView.as_view(), name='user-api'),
    # 带参数的路径 (users/5/) 
    path('users/<int:pk>/', UserAPIView.as_view(), name='user-detail'),
    # as_view可以指定对应请求调用的方法名
    path('users/', UserAPIView.as_view({'get': 'list'})),
    path('users/<int:pk>/', UserAPIView.as_view({'get': 'retrieve'})),
]
```

> **注意​**​：`APIView` 不能直接使用 `router.register()`，这是给 `ViewSet` 和 `ModelViewSet` 专用的

---

##### GenericAPIView和Mixin

从这个层级开始，将不再提供类似`get()`、`post()`的方法，而是提供`list()`、`create()`等方法，并且提供以下提到的属性和方法，这将有助于模型的CURD操作进一步简化。

###### ✨GenericAPIView

`GenericAPIView`类扩展了 REST 框架的 `APIView` 类。包中提供的每个具体通用视图都是通过将 `GenericAPIView` 与一个或多个 `Mixin` 类组合来构建的。

`GenericAPIView`继承了`ViewSetMixin`，这个Mixin覆写了`as_view()`方法，这个方法将`get`、`post`等请求类型映射到了`list()`、`create()`等方法中。但是在这里还没有提供这些操作的标准实现模板，还需要自己手动编写每个请求类型对应的方法实现。如果需要标准实现，请参考下面的`ModelViewSet`。

**属性：**

**1. 基本设置** ：以下属性控制基本视图行为。

- （必需）`queryset`: 用于从此视图返回对象的 `queryset`。通常，您必须设置此属性或重写 `get_queryset()` 方法。如果你正在覆盖一个视图方法，那么你应该调用 `get_queryset()` 而不是直接访问这个属性，因为 `queryset` 会被评估一次，这些结果将被缓存用于所有后续请求。
- （必需）`serializer_class`: 用于验证和反序列化输入以及序列化输出的序列化程序类。通常，您必须设置此属性或覆盖 `get_serializer_class()` 方法。
- `lookup_field`: 用于执行单个模型实例的对象查找的 `model` 字段。默认为 `'pk'`。请注意，在使用超链接 API 时，如果需要使用自定义值，则需要确保 API 视图和序列化程序类都设置查找字段。
- `lookup_url_kwarg`: 用于对象查找的`URL关键字`参数。URL conf 应包含与此值对应的 keyword 参数。如果未设置，则默认使用与 `lookup_field` 相同的值。

例：`path('user/<int:xxx>/', ..., ...)`将`lookup_url_kwarg = 'xxx'`，`lookup_field = 'name'`即代表url中的`xxx`指向模型中的`name`属性，用于确定详情视图中查找的对象。

**2. 分页** ：
`pagination_class` 属性用于控制与列表视图一起使用时的分页。

对列表结果进行分页时应使用的分页类。默认为与 `DEFAULT_PAGINATION_CLASS` 设置相同的值，即 `'rest_framework.pagination.PageNumberPagination'` 基于页码的分页。

设置 `pagination_class=None` 将禁用此视图的分页。

详细请看[分页](#分页)章节

**3. 过滤** ：
`filter_backends` 用于过滤查询集的过滤器后端类的列表。默认为与 `DEFAULT_FILTER_BACKENDS` 设置相同的值。
并且如果使用`django-filter`，`filter_backends` 务必指定为`from django_filters.rest_framework import DjangoFilterBackend`，这个backend提供了`filterset_class`和`filterset_fields`字段的处理。
`filterset_class`用于指定要在视图上使用的过滤器类。详情请查看[过滤](#过滤)章节。

**方法：**

**1. 基本方法：**

1. `get_queryset(self)`
   返回应该用于列表视图的 queryset，该 queryset 应该用作详细视图中查找的基础。默认返回由 `queryset 属性指定的 queryset`。
   应该始终使用此方法，而不是直接访问 `self.queryset`，因为 `self.queryset` 只被评估一次，并且这些结果会被缓存用于所有后续请求。
   可以重写以提供特定于发出请求的用户的动态行为，例如返回查询集。

```python
def get_queryset(self):
	user = self.request.user
	return user.accounts.all()
```

> 注意：如果通用视图中使用的 `serializer_class` 跨越 orm 关系，导致 n+1 问题，你可以使用 `select_related` 和 `prefetch_related` 在此方法中优化查询集。

2. `get_object(self)`
   返回应该用于局部视图的对象实例。默认使用 `lookup_field` 参数来过滤基本查询集。
   可以重写以提供更复杂的行为，例如基于多个 URL kwarg 的对象查找。总之只要能返回一个确定的对象即可。如果您的 API 不包含任何对象级权限，您可以选择排除 `self.check_object_permissions`，而只是从 `get_object_or_404` 查找中返回对象
   例如：

```python
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)  # 若不包含对象级权限可移除
    return obj
```

3. `filter_queryset(self, queryset)`
   给定一个 queryset，使用正在使用的 filter 后端对其进行过滤，返回一个新的 queryset。通常不需要直接调用这个方法，除非你需要根据不同条件动态地进行过滤。
   例如：

```python
def filter_queryset(self, queryset):
    filter_backends = [CategoryFilter]

    if 'geo_route' in self.request.query_params:
        filter_backends = [GeoRouteFilter, CategoryFilter]
    elif 'geo_point' in self.request.query_params:
        filter_backends = [GeoPointFilter, CategoryFilter]

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
```

4. `get_serializer_class(self)`
   返回应该用于序列化程序的类。默认返回 `serializer_class` 属性。
   可以重写以提供动态行为，例如使用不同的序列化程序进行读取和写入，或为不同类型的用户提供不同的序列化程序。
   例如：

```python
def get_serializer_class(self):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
```

5. **保存和删除钩子：**
   以下方法由 mixin 类提供，用于轻松覆盖对象保存或删除行为。
   - `perform_create(self, serializer)` - 在保存新对象实例时由 `CreateModelMixin` 调用。
   - `perform_update(self, serializer)` - 在保存现有对象实例时由 `UpdateModelMixin` 调用。
   - `perform_destroy(self, instance)` - 在删除对象实例时由 `DestroyModelMixin` 调用。
   这些钩子对于设置请求中隐含但不属于请求数据的属性特别有用。例如，您可以根据请求用户或 URL 关键字参数在对象上设置属性。

```python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```

这些覆盖点对于添加在保存对象之前或之后发生的行为也特别有用，例如通过电子邮件发送确认或记录更新。

```python
def perform_update(self, serializer):
    instance = serializer.save()
    send_email_confirmation(user=self.request.user, modified=instance)
```

你也可以通过引发 `ValidationError()` 来使用这些钩子来提供额外的验证。如果您需要在数据库保存时应用一些验证逻辑，这可能很有用。例如：

```python
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```

**其他方法** ：
您通常不需要覆盖以下方法，但如果您使用 `GenericAPIView` 编写自定义视图，则可能需要调用这些方法。

- `get_serializer_context(self)` 返回一个字典，其中包含应提供给序列化程序的任何额外上下文。默认包含 `'request'`、`'view'` 和 `'format'` 键。
- `get_serializer(self, instance=None, data=None, many=False, partial=False)` 返回一个序列化程序实例。
- `get_paginated_response(self, data)` 返回分页样式的`Response`对象。
- `paginate_queryset(self, queryset)` 如果需要，对 queryset 进行分页，返回一个页面对象，如果没有为此视图配置分页，则返回`None`。
- `filter_queryset(self, queryset)` 给定一个 queryset，使用正在使用的 filter 后端对其进行过滤，返回一个新的 queryset。

###### Mixin

Mixin本质上就是一个多继承的应用示例，通过多继承提供各类不同的通用方法给子类使用
**核心作用​**
提供单一操作（如创建、列表查询）的复用逻辑，需组合使用

- 示例 Mixin：
  - `CreateModelMixin`：处理对象创建（POST）。
  - `ListModelMixin`：处理列表查询（GET）。

**主要配置​**​

1. ​**​组合 Mixin 与 `GenericAPIView`**
2. **必需属性​**​（由 `GenericAPIView` 提供）

```python
from rest_framework import mixins
class UserCreateView(mixins.CreateModelMixin, GenericAPIView):
    def post(self, request):
        return self.create(request)  # 调用 CreateModelMixin 的 create()

	# CreateModelMixin 的 create()
	def create(self, request):
	    serializer = self.get_serializer(data=request.data)  # 从 GenericAPIView 获取
	    serializer.is_valid(raise_exception=True)
	    serializer.save()
	    return Response(serializer.data)
```

---

##### ModelViewSet

ModelViewSet就是一个继承了CURD Mixin的综合类：

```python
class ModelViewSet(mixins.CreateModelMixin, mixins.RetrieveModelMixin, mixins.UpdateModelMixin, mixins.DestroyModelMixin, mixins.ListModelMixin, GenericViewSet):
```

这个层级提供了CURD的标准流程，如果需要自定义某个流程，只需要覆写对应Mixin提供的方法。
要查看具体的参数和方法请查看[GenericAPIView](#GenericAPIView)章节。

**注册方式：**
**使用 DRF Routers (适用于 ViewSets)​：**
对于 `APIView` 通常不直接使用路由器，但如果是视图集 (`ViewSet`)

```python
# urls.py
from rest_framework.routers import DefaultRouter
from .views import UserViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')

urlpatterns = [...]
urlpatterns += router.urls
```

---

##### DRF 扩展操作：`@action` 装饰器

`url_path`：定义此操作对应的URL路径片段。默认为被修饰的方法的名称。
`url_name`：定义此操作使用反向查询（`reverse`）时名称。默认为被修饰的方法的名称（将其中的下划线替换为短横线）。
在视图集中添加自定义端点，支持独立配置权限、序列化器等。启用`detail`将会使生成的url后跟上查询具体资源的参数如`{pk}`，具体需要看所在的ViewSet定义。`url_path`的值将会作为`url`的最后一部分。

```python
class UserViewSet(ModelViewSet):
    @action(
        methods=['post'], 
        detail=True,  # 作用于单个模型实例​
        permission_classes=[IsAdmin]
    )
    def activate(self, request, pk=None):
        user = self.get_object()
        user.activate()
        return Response({"status": "activated"})
```

例如上面的设置自动生成的路由为：`/path_to_view/{pk}/activate/`

签名：

```python
def action(methods=None, detail=None, url_path=None, url_name=None, **kwargs):
    """
    Mark a ViewSet method as a routable action.

    `@action`-decorated functions will be endowed with a `mapping` property,
    a `MethodMapper` that can be used to add additional method-based behaviors
    on the routed action.

    :param methods: A list of HTTP method names this action responds to.
                    Defaults to GET only.
    :param detail: Required. Determines whether this action applies to
                   instance/detail requests or collection/list requests.
    :param url_path: Define the URL segment for this action. Defaults to the
                     name of the method decorated.
    :param url_name: Define the internal (`reverse`) URL name for this action.
                     Defaults to the name of the method decorated with underscores
                     replaced with dashes.
    :param kwargs: Additional properties to set on the view.  This can be used
                   to override viewset-level *_classes settings, equivalent to
                   how the `@renderer_classes` etc. decorators work for function-
                   based API views.
    """
```

## 路由

## 序列化

## 认证

## 权限

## 缓存

## 限流

## 过滤

通常来说过滤会使用`django-filter`库，它支持 REST 框架的高度可定制的字段过滤。
REST 框架的通用列表视图的默认行为是返回模型管理器的整个查询集。通常，你会希望你的 API 限制 queryset 返回的项目。
过滤继承了 `GenericAPIView` 的任何视图的 queryset 的最简单方法是覆盖 `.get_queryset()` 方法。
覆盖此方法允许您以多种不同的方式自定义视图返回的 queryset。

安装：

```
pip install django-filter


INSTALLED_APPS = [
    ...
    'django_filters',
    ...
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend']
}

```

如果你在`REST_FRAMEWORK.DEFAULT_FILTER_BACKENDS`中指定了`DjangoFilterBackend`，则在ModelViewSet中就不需要显示书写`filter_backends`了

比较常见的情况有以下几种：
**针对当前用户进行筛选：**
比如你只希望查询集返回与当前登录用户相关的结果

```python
from myapp.models import Purchase
from myapp.serializers import PurchaseSerializer
from rest_framework import generics

class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases
        for the currently authenticated user.
        """
        user = self.request.user
        return Purchase.objects.filter(purchaser=user)
```

**根据URL的参数限制查询集：**
比如url中包含以下条目

```python
re_path('^purchases/(?P<username>.+)/$', PurchaseList.as_view()),
```

然后可以按照url中的`username`部分过滤

```python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases for
        the user as determined by the username portion of the URL.
        """
        username = self.kwargs['username']
        return Purchase.objects.filter(purchaser__username=username)
```

**根据查询参数过滤：**
我们可以覆盖 `.get_queryset()` 来处理诸如 `http://example.com/api/purchases?username=denvercoder9`的 URL，并且仅当 URL 中包含 `username` 参数时才过滤查询集

```python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned purchases to a given user,
        by filtering against a `username` query parameter in the URL.
        """
        queryset = Purchase.objects.all()
        username = self.request.query_params.get('username')
        if username is not None:
            queryset = queryset.filter(purchaser__username=username)
        return queryset
```

#### 常用过滤后端​​

| 过滤后端                  | 功能     | 是否需要额外包            |
| --------------------- | ------ | ------------------ |
| `DjangoFilterBackend` | 字段精确过滤 | 需要 `django-filter` |
| `SearchFilter`        | 全文搜索   | 内置                 |
| `OrderingFilter`      | 结果排序   | 内置                 |

##### DjangoFilterBackend（字段精确匹配）

如果你只需要简单的基于相等的筛选，你可以在视图或视图集上设置 `filterset_fields` 属性，列出你想要筛选的字段集。
DjangoFilterBackend会处理两个字段：`filterset_class`和`filterset_fields`

```python
# views.py
from django_filters.rest_framework import DjangoFilterBackend

class ProductView(GenericAPIView):
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['category', 'in_stock']
    
# 请求示例： /products/?category=electronics&in_stock=true
```

---

##### SearchFilter（全文搜索）

仅当视图设置了 `search_fields` 属性时，才会应用 `SearchFilter` 类。`search_fields` 属性应该是模型上文本类型字段的名称列表，例如 `CharField` 或 `TextField`。

```python
from rest_framework import filters

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['username', 'email']
```

这将允许客户端通过进行如下查询来筛选列表中的项目：

```
http://example.com/api/users?search=russell
```

您还可以使用查找 API 双下划线表示法对 ForeignKey 或 ManyToManyField 执行相关查找：

```python
search_fields = ['username', 'email', 'profile__profession']
```

对于 [JSONField](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/fields/#jsonfield) 和 [HStoreField](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/fields/#hstorefield) 字段，您可以使用相同的双下划线表示法根据数据结构中的嵌套值进行筛选：

```python
search_fields = ['data__breed', 'data__owner__other_pets__0__name']
```

默认情况下，搜索将使用不区分大小写的部分匹配。search 参数可以包含多个搜索词，这些搜索词应以空格和/或逗号分隔。如果使用多个搜索词，则仅当提供的所有词都匹配时，才会在列表中返回对象。搜索可能包含带_引号的短语_ ，每个短语都被视为一个搜索词。

可以通过在 `search_fields` 中的字段名称前加上以下字符之一来指定搜索行为（这相当于将 `__<lookup>` 添加到字段中）：

| 前缀  | 查找            | 说明                                                                                                            |
| --- | ------------- | ------------------------------------------------------------------------------------------------------------- |
| `^` | `istartswith` | Starts-with 搜索。                                                                                               |
| `=` | `iexact`      | 精确匹配。                                                                                                         |
| `$` | `iregex`      | 正则表达式搜索。                                                                                                      |
| `@` | `search`      | 全文搜索（目前仅支持 Django 的 [PostgreSQL 后端](https://docs.djangoproject.com/en/stable/ref/contrib/postgres/search/) ）。 |
| 没有  | `icontains`   | 包含搜索 （默认）。                                                                                                    |

例如：

```python
search_fields = ['^username', '=email']
```

默认情况下，search 参数名为 `'search'`，但可能会被 `search_param` 属性设置覆盖。

要根据请求内容动态更改搜索字段，可以将 `SearchFilter` 子类化并覆盖 `get_search_fields()` 函数。例如，如果请求中包含查询参数 `title_only`，则以下子类将仅搜索 `title`：

```python
from rest_framework import filters

class CustomSearchFilter(filters.SearchFilter):
    def get_search_fields(self, view, request):
        if request.query_params.get('title_only'):
            return ['title']
        return super().get_search_fields(view, request)
```

---

##### OrderingFilter（排序过滤器）

支持简单的查询参数控制的结果排序。默认情况下，查询参数名为 `'ordering'`，但这可能会被 `ordering_param` 设置覆盖。
例如，要按用户名对用户进行排序：

```
http://example.com/api/users?ordering=username
```

客户端还可以通过在字段名称前加上`'-'` 来指定反向排序，如下所示：

```
http://example.com/api/users?ordering=-username
```

还可以指定多个排序：

```
http://example.com/api/users?ordering=account,username
```

**指定可以针对哪些字段进行排序**

建议您明确指定 API 应在排序筛选条件中允许哪些字段。您可以通过在视图上设置 `ordering_fields` 属性来执行此作，如下所示：

```python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['username', 'email']
```

这有助于防止意外的数据泄露，例如允许用户根据密码哈希字段或其他敏感数据进行排序。
如果未在视图上指定 `ordering_fields` 属性，则 filter 类将默认允许用户对 `serializer_class` 属性指定的序列化程序上的任何可读字段进行筛选。
如果你确信视图使用的查询集不包含任何敏感数据，你也可以通过使用特殊值 `'__all__'` 显式指定视图应该允许对任何模型字段或查询集聚合进行排序。

```python
class BookingsListView(generics.ListAPIView):
    queryset = Booking.objects.all()
    serializer_class = BookingSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = '__all__'
```

**指定默认顺序**

如果在视图上设置了 `ordering` 属性，则此属性将用作默认 ordering。
通常，你会通过在初始查询集上设置 `order_by` 来控制这一点，但是在视图上使用 `ordering` 参数可以让你指定排序，然后它可以作为上下文自动传递给渲染的模板。这样，如果列标题用于对结果进行排序，则可以自动以不同方式呈现列标题。

```python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['username', 'email']
    ordering = ['username']
```

`ordering` 属性可以是字符串或字符串列表/元组。

---

#### 自定义FilterSet（使用django-filters）

支持的过滤器请查看[官方文档](https://django-filter.readthedocs.io/en/latest/ref/filters.html#filters)
`Filter`父类的签名：

```python
class Filter:
    creation_counter = 0
    field_class = forms.Field

    def __init__(
        self,
        field_name=None,
        lookup_expr=None,
        *,
        label=None,
        method=None,  # 指定使用的过滤器方法
        distinct=False,  # 是否在查询集上使用distinct
        exclude=False,  # 指定这个过滤器使用filter方法还是exclude方法
        required=False,  # 这个过滤器是否必须传入
        **kwargs
    ):
    ...
```

要使用 `FilterSet` 启用过滤，请将其添加到视图类上的 `filterset_class` 参数中。

```python
from rest_framework import generics
from django_filters import rest_framework as filters
from myapp import Product

class ProductFilter(filters.FilterSet):
    min_price = filters.NumberFilter(field_name="price", lookup_expr='gte')
    max_price = filters.NumberFilter(field_name="price", lookup_expr='lte')

    class Meta:
        model = Product
        fields = ['category', 'in_stock']

class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (filters.DjangoFilterBackend,)  # backends需要添加这个，还可以添加其他的backends
    filterset_class = ProductFilter  # 通过filterset_class设置这个Filter
```

`min_price = filters.NumberFilter(field_name="price", lookup_expr='gte')`这种写法将会生成一个过滤方式。提供一个请求参数`?min_price=xxx`，将其转换为`.filter(price__gte='xxx')`。`field_name`中的字段可以使用`__`ORM查找分隔符。`lookup_expr`默认为`exact`，如果表达式部分由ORM查找分隔符连接，则可以包含转换，比如说按`datetime`的年份筛选`year__gt`。

**使用`method`参数指定这个过滤器的处理方法**
`method`参数可以指定一个方法告诉过滤器如何处理queryset。它可以接受`Callable`或者`FilterSet`类中定义的方法的方法名字符串。
这个方法应该有以下签名：

```python
def method_name(queryset, name, value):
    # 参数说明：
    #   queryset: 当前的查询集
    #   name:    过滤器的字段名（注意：是过滤器字段名，不是模型字段名）
    #   value:   用户传入的过滤值
    # 返回：更新后的查询集
```

**使用 `filterset_fields` 快捷键**
你可以通过向 view 类添加 `filterset_fields` 来绕过创建 `FilterSet`。这相当于创建一个只包含 `Meta.fields` 的 `FilterSet`。

```python
from rest_framework import generics
from django_filters import rest_framework as filters
from myapp import Product

class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    filter_backends = (filters.DjangoFilterBackend,)
    filterset_fields = ('category', 'in_stock')

# 等价于:
class ProductFilter(filters.FilterSet):
    class Meta:
        model = Product
        fields = ('category', 'in_stock')
```

请注意，不支持同时使用 `filterset_fields` 和 `filterset_class`。

**声明可过滤字段**
`fields` 选项或者`exclude`选项可以与 `model` 结合使用以自动生成过滤器。请注意，生成的过滤器不会覆盖在 `FilterSet` 上声明的过滤器。`fields` 选项接受两种语法：

```python
class UserFilter(django_filters.FilterSet):
    class Meta:
        model = User
        fields = ['username', 'last_login']
		# or
        fields = {
            'username': ['exact', 'contains'],
            'last_login': ['exact', 'year__gt'],
        }
        # or
        exclude = ['password']
```

`Meta.fields`的列表写法将为`fields`中每个字段创建一个`exact`查找过滤器。字典语法将为每个字段创建一个过滤器表达式相符的过滤器。且务必要注意，列表和字段的key是**模型的字段名**，而不是显式定义的**过滤器名称**，如果错误使用了过滤器名称，将会引发`TypeError`。并且无论是列表还是字典写法，都不需要再包含显式声明的过滤器。
`Meta.fields` 的目的仅是基于模型字段声明可用的过滤器。
`Meta.fields` 并不是必须的，其实更推荐全部使用显式声明过滤器。
`Meta.fields` 可以使用`fields = '__all__'`这种写法包含模型的全部字段。
`Meta.exclude`将排除声明的字段，将模型中其他字段都加入过滤器中。不可以同时指定`exclude`和`fields`。

## 分页

**DRF 提供的分页类​**​

| 分页类型                    | 特点                                         | 适用场景            |
| ----------------------- | ------------------------------------------ | --------------- |
| `PageNumberPagination`  | 经典页码分页 (`?page=2&size=20`)                 | 通用分页需求          |
| `LimitOffsetPagination` | 使用 limit/offset 参数 (`?limit=20&offset=40`) | 需要自定义截取范围时      |
| `CursorPagination`      | 基于游标的加密分页 (`?cursor=abc123`)               | 实时更新数据、避免页码跳跃问题 |

**PageNumberPagination** 的定义

```python
class PageNumberPagination(BasePagination):
    """
    A simple page number based style that supports page numbers as
    query parameters. For example:

    http://api.example.org/accounts/?page=4
    http://api.example.org/accounts/?page=4&page_size=100
    """
    # The default page size.
    # Defaults to `None`, meaning pagination is disabled.
    page_size = api_settings.PAGE_SIZE

    django_paginator_class = DjangoPaginator

    # Client can control the page using this query parameter.
    page_query_param = 'page'
    page_query_description = _('A page number within the paginated result set.')

    # Client can control the page size using this query parameter.
    # Default is 'None'. Set to eg 'page_size' to enable usage.
    page_size_query_param = None
    page_size_query_description = _('Number of results to return per page.')

    # Set to an integer to limit the maximum page size the client may request.
    # Only relevant if 'page_size_query_param' has also been set.
    max_page_size = None

    last_page_strings = ('last',)

    template = 'rest_framework/pagination/numbers.html'

    invalid_page_message = _('Invalid page.')
```

### 配置方式​​

​**​全局配置 (settings.py):​**​

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',  # 默认使用的分页类
    'PAGE_SIZE': 20  # 默认每页数量
}
```

​**​视图级配置 (views.py):​**​

```python
from rest_framework.pagination import PageNumberPagination

class CustomPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'size'  # 允许客户端自定义每页数量
    max_page_size = 100             # 自定义数量的上限

class UserListView(APIView):
    pagination_class = CustomPagination
    queryset = User.objects.all()
    
    def get(self, request):
        page = self.paginate_queryset(self.queryset)
        serializer = UserSerializer(page, many=True)
        return self.get_paginated_response(serializer.data)
```

### ​​响应结构​​

```json
{
    "count": 1023,          // 总记录数
    "next": "http://api.example.com/users/?page=3",
    "previous": "http://api.example.com/users/?page=1",
    "results": [             // 当前页数据
        { "id": 21, "name": "User21" },
        { "id": 22, "name": "User22" },
        ...
    ]
}
```

### ​​最佳实践​​

1. ​**​数据量 > 100条​**​ 建议启用分页
2. ​**​移动端API​**​ 优先使用 `CursorPagination`
3. 提供 `page_size` 参数允许客户端控制
4. 设置合理的 `max_page_size` 防止DoS攻击
5. 配合`ordering`参数保证分页稳定性
