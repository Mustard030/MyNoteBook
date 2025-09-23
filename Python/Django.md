# Django
## 模型（Models）

## 视图（Views）

### FBV
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

### CBV

View -> TemplateView, RedirectView -> ListView, DetailView -> CreateView, UpdateView, DeleteView

#### View

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

#### TemplateView

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

#### RedirectView

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

#### Django 的 CRUD 集成视图

| 视图类          | 核心功能   | 主要配置项                        |
| ------------ | ------ | ---------------------------- |
| `ListView`   | 对象列表展示 | `model`, `queryset`          |
| `DetailView` | 单个对象详情 | `slug_field`, `pk_url_kwarg` |
| `CreateView` | 新建对象   | `form_class`, `fields`       |
| `UpdateView` | 更新对象   | `template_name_suffix`       |
| `DeleteView` | 删除对象   | `success_url`                |



## 中间件

## 路由

## 安全
django settings里面安全相关的配置分为四大类`ALLOWED_HOSTS`、`CSRF_*`、`CORS_*`、`SESSION_*`
CSRF保护的是表单提交等操作，而CORS保护的是前端资源请求，CSRF 关注的是请求的 ​**​来源（Origin）是否可信​**​（用于表单提交等改变状态的操作），CORS 关注的是浏览器是否允许前端代码 ​**​读取响应​**​。

### 常见请求头
#### Host
就是url中去掉协议头，去掉路径的主要部分，例如
`http://localhost:8000/api/users`最终提取的请求头为`Host: localhost:8000`，然后经过DNS服务器解析后发送到对应IP的主机的反向代理服务器中，例如Nginx。此时将会比对配置的`server_name`和`listen`并路由到对应的后端服务器中。

#### Origin
标识了发起请求的**源（协议 + 域名 + 端口）**。它的出现主要是为了解决跨域资源共享（CORS）的安全性问题。Origin**只在跨域请求 (Cross-Origin Request) 时发送**。同源请求不发送此头。
例如：`https://www.google.com`

#### Referer
发起当前这个请求的**上一个页面**的完整 URL 是什么。包含完整的 URL，包括路径和查询参数。
例如：`https://www.google.com/search?q=http+headers`

### 关键总结与联系​​

| 设置类别              | 核心目的                | 相互关系与注意事项                                                       |
| ----------------- | ------------------- | --------------------------------------------------------------- |
| `ALLOWED_HOSTS`   | ​**​基础安全​**​，过滤非法主机 | 必须正确配置生产环境域名                                                    |
| `CSRF_*`          | ​**​防跨站提交保护​**​     | `CSRF_TRUSTED_ORIGINS` 允许非安全来源; `CSRF_COOKIE_DOMAIN` 与跨子域相关     |
| `CORS_*`          | ​**​控制跨域AJAX访问​**​  | 依赖 `django-cors-headers`；`CORS_ALLOW_CREDENTIALS=True` 时不能使用通配符 |
| `*_COOKIE_DOMAIN` | ​**​跨子域共享认证状态​**​   | `SESSION_COOKIE_DOMAIN` 和 `CSRF_COOKIE_DOMAIN` 通常同步设置           |

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


### 基础安全设置

#### ALLOWED_HOSTS

**作用​**​: ​**​安全基础​**​，防止 HTTP Host 头攻击。Django 只响应请求头中 `Host` 或 `X-Forwarded-Host` 与此列表匹配的请求。Django 在验证 HTTP Host 头时，会**忽略**端口部分，只比较域名或 IP 地址。允许一个域名意味着​**​允许该域名的所有端口请求​**

```python
ALLOWED_HOSTS = [
    'example.com',        # 精确域名
    '.example.com',       # 匹配 *.example.com 的所有子域
    'localhost',
    '127.0.0.1',
    '[::1]',              # IPv6
]
```

**注意​**​:

- ​**​开发环境​**​: 可设为 `['*']`（​**​不安全，仅限测试!​**​）
- ​**​生产环境​**​: ​**​必须​**​配置为你的正式域名/IP。
- **代理/负载均衡器​**​：当使用 Nginx/Apache 时，确保它们正确传递 `Host` 头（不含端口）

### CSRF
CSRF校验主要通过**Django**自带的中间件`"django.middleware.csrf.CsrfViewMiddleware"`进行。  

在 **Django** 里，`csrftoken`（CSRF Token）是一个**会话级别的随机值**，它通常在用户第一次访问站点时由中间件生成并下发到浏览器的 `csrftoken` **Cookie** 中（不是 sessionStorage），默认有效期是 **1 年**。  

**表单 / AJAX 请求** 时，需要把这个 token 一起带回（通常放在请求头 `X-CSRFToken` 或表单隐藏字段）。只要这个 Cookie 不过期或不被清除，Django 就会一直复用这个 Token。  

默认情况下，**Django 只会在视图中用到 `{% csrf_token %}` 标签时，才下发 CSRF Cookie**。  

每个 POST（或 PUT/DELETE 等 “修改数据” 的请求）必须带上正确的 `csrftoken`。Django 会从 **Cookie** 和 **请求头/表单隐藏字段** 中分别取出 token 值做比对，一致才放行。

当
- 用户清除浏览器 Cookie。
- 服务器端调用 `rotate_token(request)` 主动刷新。
- 用户过了一年，Cookie 过期。
的时候就会变更

这就会带来一个问题：
如果这是 **前后端分离**的应用，前端第一次访问 API 时，Django 并不会自动给它下发 `csrftoken` cookie，因为模板里根本没有 `{% csrf_token %}`。这样前端发起的 **第二次 POST 请求** 就会因为缺少 CSRF token 而被拒绝。

**CSRF 保护机制**相关的几个装饰器：
`@csrf_exempt`
**作用**：让某个视图函数 **不进行 CSRF 验证**。
```python
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def my_view(request):
    return HttpResponse("CSRF not checked")
```
但更推荐使用 `@csrf_exempt` + JWT/TokenAuth，而不是完全关闭保护

`@csrf_protect`
**作用**：强制某个视图函数进行 CSRF 验证。
```python
from django.views.decorators.csrf import csrf_protect

@csrf_protect
def my_view(request):
    return HttpResponse("CSRF checked")
```

通常用于全局配置中关闭了 CSRF（例如 API 项目默认禁用），但某个视图仍然希望强制保护，特别是表单提交的视图。

`@requires_csrf_token`
**作用**：用于**错误处理视图**或**模板渲染时**，保证模板上下文里一定有 CSRF token。  
**区别**：不会强制检查请求的 CSRF token，只是确保在 `context` 中有 `csrf_token` 变量。  
**典型用法**：
```python
from django.views.decorators.csrf import requires_csrf_token
from django.shortcuts import render

@requires_csrf_token
def custom_403_view(request, exception=None):
    # 即使这里没有经过CSRF验证，模板里也能安全使用 {{ csrf_token }}
    return render(request, "403.html")
```
**应用场景**：
- 自定义 `403 Forbidden` 页面。
- 需要在模板中放置 `csrf_token`（例如表单）但又不是正常视图时。

`@ensure_csrf_cookie`
**作用**：保证 **响应里会设置 CSRF cookie**（即使模板没有调用 `{{ csrf_token }}` 标签）。只确保下发，不负责校验。
**典型用法**：
```python
from django.views.decorators.csrf import ensure_csrf_cookie

@ensure_csrf_cookie
def my_view(request):
    return render(request, "index.html")
```

**CSRF 保护机制**相关的几个函数：
`get_token(request)`
**作用：** 获取当前请求的 CSRF token。如果请求里已经有 CSRF token（比如 cookie 里的 `csrftoken`），就直接返回。如果没有，就会生成一个新的 token 并附加到 `request.META['CSRF_COOKIE']` 上。

`rotate_token(request)`
**作用：** 强制生成一个新的 CSRF token，并替换掉旧的。通常用于 **用户登录、权限变更** 等**安全边界发生变化**的关键操作后，防止 token 固定攻击（session fixation）。Django 在 `login()` 的时候会调用它。
例如用户登录后刷新 CSRF Token、提升权限时刷新 CSRF Token、重要操作后主动调用（比如绑定银行卡、修改邮箱）

**相关配置：**

| 配置项                  | 默认值                              | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CSRF_COOKIE_AGE      | `31449600` （约 1 年，以秒为单位）         | 将此设置改为 `None`，以使用基于会话的 CSRF cookie，它将 cookie 保存在内存中，而不是持久性存储中。                                                                                                                                                                                                                                                                                                                                                                                |
| CSRF_COOKIE_DOMAIN   | None                             | 设置 CSRF cookie 时要使用的域。它应该设置为一个字符串，如 `".example.com"`，以允许一个子域上的表单的 POST 请求被另一个子域的视图所接受。                                                                                                                                                                                                                                                                                                                                                        |
| CSRF_COOKIE_HTTPONLY | False                            | 是否对 CSRF cookie 使用 `HttpOnly` 标志。如果设置为 `True`，客户端的 JavaScript 将无法访问 CSRF cookie。将 CSRF cookie 指定为 `HttpOnly` 并不能提供任何实际的保护，因为 CSRF 只是为了防止跨域攻击。如果攻击者可以通过 JavaScript 读取 cookie，就浏览器所知，他们已经在同一个域上了，所以他们可以做任何他们喜欢的事情。（XSS 是一个比 CSRF 更大的漏洞）。                                                                                                                                                                                                        |
| CSRF_COOKIE_NAME     | 'csrftoken'                      | 用于 CSRF 认证令牌的 cookie 的名称。这可以是任何你想要的名字（只要它与你的应用程序中的其他 cookie 名字不同）。                                                                                                                                                                                                                                                                                                                                                                            |
| CSRF_COOKIE_PATH     | ’/‘                              | 在 CSRF cookie 上设置的路径。这个路径应该与你的 Django 安装的 URL 路径相匹配，或者是该路径的父路径。                                                                                                                                                                                                                                                                                                                                                                               |
| CSRF_COOKIE_SAMESITE | 'Lax'                            | CSRF cookie 上 [SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite) 标志的值。该标志可防止在跨站点请求中发送 cookie。                                                                                                                                                                                                                                                                                                          |
| CSRF_COOKIE_SECURE   | False                            | 是否为 CSRF cookie 使用安全 cookie。如果设置为 `True`，cookie 将被标记为 `安全`，这意味着浏览器可以确保 cookie 只在 HTTPS 连接下发送。                                                                                                                                                                                                                                                                                                                                                 |
| CSRF_USE_SESSIONS    | False                            | 是否将 CSRF 标记存储在用户的会话中，而不是 cookie 中。这需要使用 `django.contrib.session`。                                                                                                                                                                                                                                                                                                                                                                             |
| CSRF_FAILURE_VIEW    | 'django.views.csrf.csrf_failure' |                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| CSRF_HEADER_NAME     | 'HTTP_X_CSRFTOKEN'               | 用于 CSRF 认证的请求头的名称。与 `request.META` 中的其他 HTTP 头文件一样，从服务器接收到的头文件名通过将所有字符转换为大写字母，用下划线代替任何连字符，并在名称中添加 `'HTTP_'` 前缀进行规范化。例如，如果你的客户端发送了一个 `'X-XSRF-TOKEN'` 头，配置应该是 `'HTTP_X_XSRF_TOKEN'`。                                                                                                                                                                                                                                                           |
| CSRF_TRUSTED_ORIGINS | `[]` （空列表）                       | 不安全请求（例如 `POST`）的可信来源主机列表。对于 [`可靠`](https://docs.djangoproject.com/zh-hans/3.2/ref/request-response/#django.http.HttpRequest.is_secure "django.http.HttpRequest.is_secure") 的不安全请求，Django 的 CSRF 保护要求该请求的 `Referer` 头必须与 `Host` 头中的来源匹配。例如，这可以防止来自 `subdomain.example.com` 的 `POST` 请求对 `api.example.com` 成功。如果你需要通过 HTTPS 的跨源不安全请求，继续这个例子，在这个列表中添加 `"subdomain.example.com"`。该设置还支持子域，所以你可以添加 `".example.com"`，例如，允许从 `example.com` 的所有子域访问。 |
#### CSRF_TRUSTED_ORIGINS

**​作用​**​: 允许 ​**​不安全协议（如 HTTP）或非标准端口​**​ 的来源绕过 HTTPS 检查。定义哪些来源被信任，用于接收合法的 CSRF token。

Django 的 CSRF 保护会拿「浏览器发来的 Origin（或 Referer）」跟「请求的 Host」做对比，两者必须来自同一个「源」，否则就拒绝。一些旧版浏览器或简单 GET 请求可能不发送 `Origin`，但会发送 `Referer`。Django 退而求其次，拿 `Referer` 去跟 `Host` 比对。这个配置项就是告诉Django除了当前请求的Host以外，还有什么来源是可信的。`Origin`的值就是浏览器头上那一串。

**何时需要​**​:

- 前端从 `http://localhost:3000` 访问后端 `https://api.example.com`
- 后端运行在非标准端口（如 `https://example.com:8443`）

**​配置​**​:

```python
# django4.0+, 必须明确协议头，如果是非标端口也要明确
CSRF_TRUSTED_ORIGINS = [
    'https://*.example.com',  # 允许所有子域的 HTTPS
    'http://localhost:3000',   # 允许非标端口
]
# django3.2, 不要携带任何协议头和端口号
CSRF_TRUSTED_ORIGINS = [
    '*.example.com',  # 允许通配符
    'localhost',
]
```

#### CSRF_COOKIE_DOMAIN

​**​作用​**​: 控制 `csrftoken` Cookie 的作用域（Domain）。

​**​典型场景​**​: 共享 Cookie 跨子域（如 `auth.example.com` 设置 Cookie 让 `shop.example.com` 使用）。

​**​配置​**​:

```python
 CSRF_COOKIE_DOMAIN = '.example.com'  # 设置顶级域名，所有子域共享 Cookie
```

​**​默认​**​: `None`（每个域名独享自己的 Cookie）



### CORS

> 需 `django-cors-headers` 包

#### CORS_ALLOWED_ORIGINS

旧版本为：`CORS_ORIGIN_WHITELIST`，但该名称在新版本仍然可用

**CORS_ALLOWED_ORIGINS** 匹配的是请求头中的 `Origin` 头部。当浏览器发起一个跨域请求时，会自动在请求头中添加一个 `Origin` 字段，**其值为请求发起页面的源（协议+域名+端口）**。例如，如果前端应用运行在 `https://www.example.com:8080`，那么请求头中的 `Origin` 就是 `https://www.example.com:8080`。

这个配置与Django提供的[CSRF_TRUSTED_ORIGINS](#CSRF_TRUSTED_ORIGINS)不一样，CORS 和 CSRF 是分开的，Django 无法使用 CORS 配置来免除站点对安全请求所做的 Referer 检查。

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

#### CORS_ORIGIN_ALLOW_ALL

**作用​**​: ​**​禁用白名单​**​，允许 ​**​所有来源​**​ 的跨域请求（⚠️ ​**​高危！生产环境应避免使用​**​）。
​
**​配置​**​:

```python
CORS_ORIGIN_ALLOW_ALL = True  # 替代 `CORS_ORIGIN_WHITELIST`
```

#### CORS_ALLOW_CREDENTIALS

**作用​**​: 控制浏览器是否在跨域请求中发送 ​**​凭证（Cookies、HTTP认证等）​**​。
​
**​必须与前端配合​**​: 前端需要设置 `withCredentials = true`（例如在 axios 或 fetch 中）。

**配置​**​:

```python
CORS_ALLOW_CREDENTIALS = True  # 默认为 False
```

**限制​**​: 当启用时，​**​不能​**​同时使用 `CORS_ORIGIN_ALLOW_ALL = True`，且 `CORS_ORIGIN_WHITELIST` 必须明确指定域名（不能包含通配符如 `https://*.example.com`）。

#### CORS_ALLOW_HEADERS

**​作用​**​: 控制跨域请求中允许的 ​**​额外 HTTP 请求头​**​（超出浏览器默认安全集）。

**默认包含常用头​**​: `'Content-Type'`, `'Authorization'` 等已内置。

**扩展配置​**​:

```python
CORS_ALLOW_HEADERS = [
    'content-disposition',  # 允许前端接收下载头
    'x-custom-header',      # 自定义头
]

# 或者可以

from corsheaders.defaults import default_headers

CORS_ALLOW_HEADERS = (
    *default_headers,
    "my-custom-header",
)

```

#### CORS_ALLOW_METHODS

**作用​**​: 控制跨域请求中允许使用的 ​**​HTTP 方法​**​。

**默认包含**: `['DELETE', 'GET', 'OPTIONS', 'PATCH', 'POST', 'PUT']`

### Session

#### SESSION_COOKIE_DOMAIN

**作用​**​: 控制 `sessionid` Cookie 的作用域（Domain），实现跨子域共享登录状态

**配置​**​:

```python
SESSION_COOKIE_DOMAIN = '.example.com'  # 顶级域名，所有子域可访问此 Cookie
```

**默认​**​: `None`（每个域名独享 Session Cookie）


## 模板（Templates）


## 配置


## 认证与权限


## 后台管理（Admin）


## 信号（Signals）


## 缓存、异步与任务

## 测试与调试



## 高级技巧与优化
#### `@method_decorator`
`method_decorator` 是 Django 提供的一个非常有用的工具，它的核心功能是**将一个为普通函数设计的装饰器（Function-Based Decorator）转换为可以用于类方法（Class Method）的装饰器**。因为类方法中有一个隐式的`self`或`cls`参数，直接使用参数不对。

`method_decorator` 主要有两种使用方式：
1. **直接应用在类的某个方法上**
2. **应用在整个类上，并指定要装饰的方法名**

```python
from django.contrib.auth.decorators import login_required
from django.utils.decorators import method_decorator
from django.views import View
from django.http import HttpResponse

@method_decorator(csrf_protect, name='post')  # 多个装饰器可以堆叠，name指定为类方法名
@method_decorator(login_required, name='dispatch')  # 方法1
class ProtectedView(View):
    # 方法2
    @method_decorator(login_required)
    def get(self, request, *args, **kwargs):
        return HttpResponse("This is a protected GET response.")
```

> 对于 Django 的类视图，所有请求都会先经过 `dispatch()` 方法，由它来判断是 GET、POST 还是其他请求，并分发给对应的方法（如 `get()`、`post()`）。因此，**将装饰器应用到 `dispatch` 方法上，就等同于保护了这个视图处理的所有请求。** DRF的类视图，无论是基础的 `APIView`，还是更高级的 `GenericAPIView`、`ViewSet` 等，其底层都继承自 Django 自己的 `django.views.generic.View` 类。这意味着 DRF 的视图遵循与 Django 普通类视图相同的生命周期和方法分发机制（即 `dispatch()` 方法）。因此，设计用来桥接函数装饰器和 Django 类方法的 `method_decorator`，自然也适用于 DRF 的类方法。


## 数据库操作

### 读写分离

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

### migrate工具

#### makemigrations

`makemigrations`时，**Django 不会对数据库做任何改动**，也不会读写 `django_migrations` 表。`makemigrations` 每次生成迁移文件时，**只关心当前模型状态 vs 数据库最后一次已应用的迁移**（记录在 `django_migrations` 表中的那个）。迁移文件的编号是Django扫描`app/migrations/` 目录下已有的迁移文件，取最大的数字加 1生成的。

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

`--merge`的作用：假设我们都在0003之后各自拉分支，然后各自出现了0004，这时候拉代码就会有两个0004，如果`migrate`就会报错，因为Django不知道用哪个0004。就需要使用`makemigrations --merge <app_label>`。这时Django**不会**修改现有的迁移文件（两个0004），并新增一条“合并迁移”0005。这时再`migrate`就可以了。它解决的是同一级编号的并行冲突。

例：

```shell
python manage.py makemigrations --dry-run -v 3 <app_name>  # 只展示迁移，不生成实际文件
python manage.py makemigrations <app_name>
```

#### migrate

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
python manage.py migrate --verbosity 3      # 显示详细的迁移执行过程  - 可选级别: 0=静默, 1=正常, 2=详细, 3=非常详细
python manage.py migrate --database <db_name>  # 指定要迁移的数据库（在settings.DATABASES中配置）
python manage.py migrate --fake <app_name>  # 假迁移 标记迁移为完成状态但不实际执行SQL 当迁移已手动应用或不需要操作时使用
```

#### showmigrations

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

#### sqlmigrate

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

---

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

### 统一返回类（集成trace_id）
创建trace_utils工具集
```python title:trace_utils.py
import uuid  
from contextvars import ContextVar  
  
trace_id_var: ContextVar[str] = ContextVar("trace_id", default="")  
  
def get_trace_id() -> str:  
    return trace_id_var.get()  
  
def set_trace_id(trace_id: str) -> None:  
    trace_id_var.set(trace_id)  
  
def generate_trace_id() -> str:  
    return uuid.uuid4().hex
```

在统一返回类中添加`trace_id`字段
```python title:common_response.py hl:14,50,75,88-97,112,124,139,152-161,175,190
from __future__ import annotations  
  
from dataclasses import dataclass  
from enum import Enum  
from typing import Optional, Any, Union  
  
from django.conf import settings  
from django.http import JsonResponse  
from django.utils.functional import Promise  
from django.utils.translation import gettext_lazy as _  
from rest_framework import serializers, status  
from rest_framework.response import Response  
  
from utils.trace_utils import get_trace_id  
  
_StrOrPromise = Union[str, Promise]  
  
  
# ---------- 1. 枚举 ----------@dataclass(frozen=True)  
class _Info:  
    code: int  
    detail: _StrOrPromise  
  
  
class RespCode(Enum):  
    OK = _Info(200, _("success"))  
    CREATED = _Info(201, _("created"))  
  
    BAD = _Info(400, _("bad request"))  
    UNAUTHORIZED = _Info(401, _("unauthorized"))  
    FORBIDDEN = _Info(403, _("forbidden"))  
    NOT_FOUND = _Info(404, _("not found"))  
  
    SERVER_ERROR = _Info(500, _("server error"))  
    VALIDATION_ERROR = _Info(500, _("validation error"))  
  
    @property  
    def code(self) -> int:  
        return self.value.code  
  
    @property  
    def detail(self) -> str:  
        return str(self.value.detail)  # 自动触发 __str__  
  
# ---------- 2. 序列化器 ----------class CommonResponseSerializer(serializers.Serializer):  
    status = serializers.BooleanField()  
    code = serializers.IntegerField()  
    detail = serializers.CharField(allow_blank=True)  
    data = serializers.JSONField(required=False, allow_null=True)  
    trace_id = serializers.CharField(required=False, allow_blank=True)  
  
  
# ---------- 3. 统一响应 ----------class CommonResponse(Response):  
    """  
    统一返回对象  
    用法：  
        CommonResponse()       
        CommonResponse(code=RespCode.NOT_FOUND)       
        CommonResponse.ok(data={"id": 1})       
        CommonResponse.fail(code=RespCode.BAD_REQUEST, detail="missing name")  
    对于django原生视图  
       CommonResponse.django()       
       CommonResponse.django(code=RespCode.NOT_FOUND)       
       CommonResponse.django.ok(data={"id": 1})       
       CommonResponse.django.fail(code=RespCode.BAD_REQUEST, detail="missing name")    
    """  
    def __init__(  
            self,  
            status: bool = True,  
            code: RespCode = RespCode.OK,  
            detail: Optional[str] = None,  
            data: Any = None,  
            *,  
            http_code: int = status.HTTP_200_OK,  
            use_trace_id: Optional[bool] = None,  
    ):  
        # 未显式 detail 时取枚举默认值  
        if detail is None:  
            detail = code.detail  
  
        ser_data = {  
            "status": status,  
            "code": code.code,  
            "detail": detail,  
            "data": data,  
        }  
  
        # 决定是否添加 trace_id 的核心逻辑  
        if use_trace_id is None:  
            # 未传入参数，读取全局配置。getattr 第二个参数是默认值，防止 settings 中未定义而出错。  
            should_add_trace = getattr(settings, 'USE_TRACE_ID', False)  
        else:  
            # 传入了参数，以传入的为准  
            should_add_trace = use_trace_id  
  
        if should_add_trace:  
            ser_data['trace_id'] = get_trace_id()  
  
        ser = CommonResponseSerializer(data=ser_data)  
        ser.is_valid(raise_exception=True)  
        super().__init__(data=ser.data, status=http_code)  
  
    # 快捷方法  
    @classmethod  
    def ok(  
            cls,  
            data: Any = None,  
            code: RespCode = RespCode.OK,  
            *,  
            detail: Optional[str] = None,  
            http_code: int = status.HTTP_200_OK,  
            use_trace_id: Optional[bool] = None,  
    ) -> CommonResponse:  
        return cls(status=True, code=code, detail=detail, data=data, http_code=http_code, use_trace_id=use_trace_id)  
  
    @classmethod  
    def fail(  
            cls,  
            detail: Optional[str] = None,  
            code: RespCode = RespCode.BAD,  
            *,  
            data: Any = None,  
            http_code: int = status.HTTP_400_BAD_REQUEST,  
            use_trace_id: Optional[bool] = None,  
    ) -> CommonResponse:  
        return cls(status=False, code=code, detail=detail, data=data, http_code=http_code, use_trace_id=use_trace_id)  
  
    class DjangoHelper(JsonResponse):  
        """  
        一个继承自 JsonResponse 的辅助类，用于原生 Django 视图。  
        """        def __init__(  
                self,  
                status: bool = True,  
                code: RespCode = RespCode.OK,  
                detail: Optional[str] = None,  
                data: Any = None,  
                *,  
                http_code: int = 200, # JsonResponse 的 status 参数  
                use_trace_id: Optional[bool] = None,  
                **kwargs, # 接收 JsonResponse 的其他参数  
        ):  
            if detail is None:  
                detail = code.detail  
  
            response_data = {  
                "status": status,  
                "code": code.code,  
                "detail": detail,  
                "data": data,  
            }  
  
            # 决定是否添加 trace_id 的核心逻辑  
            if use_trace_id is None:  
                # 未传入参数，读取全局配置。getattr 第二个参数是默认值，防止 settings 中未定义而出错。  
                should_add_trace = getattr(settings, 'USE_TRACE_ID', False)  
            else:  
                # 传入了参数，以传入的为准  
                should_add_trace = use_trace_id  
  
            if should_add_trace:  
                response_data['trace_id'] = get_trace_id()  
  
            # 这里是关键：调用父类 JsonResponse 的 __init__            
            # JsonResponse 的第一个参数是 data，状态码是 status            
            super().__init__(data=response_data, status=http_code, **kwargs)  
  
        @classmethod  
        def ok(  
                cls,  
                data: Any = None,  
                code: RespCode = RespCode.OK,  
                *,  
                detail: Optional[str] = None,  
                http_code: int = status.HTTP_200_OK,  
                use_trace_id: Optional[bool] = None,  
        ) -> JsonResponse:  
            """  
            工厂方法：创建并返回一个成功的 DjangoHelper 实例。  
            """            
            return cls(status=True, data=data, code=code, detail=detail, use_trace_id=use_trace_id, http_code=http_code)  
  
        @classmethod  
        def fail(  
                cls,  
                detail: Optional[str] = None,  
                code: RespCode = RespCode.BAD,  
                *,  
                data: Any = None,  
                http_code: int = status.HTTP_400_BAD_REQUEST,  
                use_trace_id: Optional[bool] = None,  
        ) -> JsonResponse:  
            """  
            工厂方法：创建并返回一个失败的 DjangoHelper 实例。  
            """            
            return cls(status=False, code=code, detail=detail, data=data, use_trace_id=use_trace_id, http_code=http_code)  
  
    # 将嵌套类赋值给一个类属性，作为命名空间  
    django = DjangoHelper
```

添加中间件
```python title:middlewares.py
from django.http import HttpRequest, HttpResponse  
  
from .trace_utils import generate_trace_id, set_trace_id, get_trace_id  
  
class TraceIdMiddleware:  
    """  
    每个请求进来时：  
      1. 先从 X-Trace-Id 头部读取（方便链路串联），没有则生成  
      2. 保存到 contextvars  
	  3. 在响应里加回 X-Trace-Id 头部  
    """  
    def __init__(self, get_response) -> None:  
        self.get_response = get_response  
  
    def __call__(self, request: HttpRequest) -> HttpResponse:  
        # 1. 生成/提取 trace_id        
        trace_id = request.META.get("HTTP_X_TRACE_ID") or generate_trace_id()  
        set_trace_id(trace_id)  
        request.trace_id = trace_id  
  
        # 2. 执行原视图函数  
        response = self.get_response(request)  
  
        # 3. 写回响应头  
        response["X-Trace-Id"] = get_trace_id()  
        return response
```

添加logging_filters
```python title:logging_filters.py
import logging  
from .trace_utils import get_trace_id  
  
class TraceIdFilter(logging.Filter):  
    def filter(self, record: logging.LogRecord) -> bool:  
        record.trace_id = get_trace_id()  
        return True
```

在配置中注册中间件和logging的设置
```python title:setting.py hl:3,11,17
MIDDLEWARE = [  
	...
    'utils.middlewares.TraceIdMiddleware',  
]

LOGGING = {  
    "version": 1,  
    "disable_existing_loggers": False,  
    "formatters": {  
        "verbose": {  
            "format": "[{asctime}] {levelname} {name} trace_id={trace_id} {message}",  
            "style": "{",  
        },  
    },  
    "filters": {  
        "trace_id": {  
            "()": "utils.logging_filters.TraceIdFilter",  
        },  
    },  
    "handlers": {  
        "console": {  
            "class": "logging.StreamHandler",  
            "formatter": "verbose",  
            "filters": ["trace_id"],  
        },  
    },  
    "root": {  
        "handlers": ["console"],  
        "level": "INFO",  
    },  
}
```

在视图中使用`logging.info(xxx)`将自动带上`trace_id`

## 视图

视图大体上分为两类：`CBV`(基于类的视图)和`FBV`(基于方法的视图)

### FBV
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

APIView -> GenericAPIView（配合Mixins） -> 视图集（ViewSet, GenericViewSet, ModelViewSet） -> 自定义动作（@action）

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


##### GenericAPIView

`GenericAPIView`类扩展了 REST 框架的 `APIView` 类。

其主要作用是

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

**4. 鉴权** ：
`permission_classes` 用于指定视图的准入权限类列表，只要列表里**任意一个**权限类返回 `False`，DRF 就立即抛 `PermissionDenied` → 前端收到 **403 Forbidden**  
鉴权和认证容易混淆，认证只用于识别“你是谁”，鉴权才是判断你能不能执行这个操作的关键。详情见[权限](#权限)章节。

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

> 注意：如果通用视图中使用的 `serializer_class` 跨越 orm 关系，导致 n+1 问题，你可以使用 `select_related` 和 `prefetch_related` 在此方法中优化查询集。因为无论是 `serializer_class` 还是`filter_class`获取的都是这个方法返回的查询集。

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
您通常不需要覆盖以下方法，但如果您使用 `GenericViewSet` 编写自定义视图，则可能需要调用这些方法。

- `get_serializer_context(self)` 返回一个字典，其中包含应提供给序列化程序的任何额外上下文。默认包含 `'request'`、`'view'` 和 `'format'` 键。
- `get_serializer(self, instance=None, data=None, many=False, partial=False)` 返回一个序列化程序实例。
- `get_paginated_response(self, data)` 返回分页样式的`Response`对象。
- `paginate_queryset(self, queryset)` 如果需要，对 queryset 进行分页，返回一个页面对象，如果没有为此视图配置分页，则返回`None`。
- `filter_queryset(self, queryset)` 给定一个 queryset，使用正在使用的 filter 后端对其进行过滤，返回一个新的 queryset。

##### Mixin

###### ViewSetMixin
只要继承了ViewSetMixin，将不再提供类似`get()`、`post()`的方法，而是提供`list()`、`create()`等方法，并且提供以下提到的属性和方法，这将有助于模型的CURD操作进一步简化。本质上只干了一件事：把方法名映射成路由动作。

Mixin就是一个多继承的应用示例，通过多继承提供各类不同的通用方法给子类使用
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

##### GenericViewSet
`GenericViewSet`继承了`ViewSetMixin`，这个Mixin覆写了`as_view()`方法，这个方法将`get`、`post`等请求类型映射到了`list()`、`create()`等方法中。但是在这里还没有提供这些操作的标准实现模板，还需要自己手动编写每个请求类型对应的方法实现。如果需要标准实现，请参考下面的`ModelViewSet`。包中提供的每个具体通用视图都是通过将 `GenericViewSet` 与一个或多个 `Mixin` 类组合来构建的。

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

##### `@action` 装饰器
> 在旧版本（DRF≤3.7）中是`@list_route`和`@detail_route`，其实就是detail=False和True的情况，在之后的版本中使用`@action`装饰器和参数`detail`代替。

`url_path`：定义此操作对应的URL路径片段。默认为被修饰的方法的名称。
`url_name`：定义此操作使用反向查询（`reverse`）时名称。默认为被修饰的方法的名称（将其中的下划线替换为短横线）。
在视图集中添加自定义端点，支持独立配置权限、序列化器等。启用`detail`将会使生成的url后跟上查询具体资源的参数如`{pk}`，具体需要看所在的ViewSet定义的`lookup_url_kwarg`。`url_path`的值将会作为`url`的最后一部分。

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

当你定义了额外方法，可以在view.action中获取到这个被装饰的方法名，它可以使用在这样的地方

```python
class SomePermission(permissions.BasePermission):
    def has_permission(self, request, view):
        user = request.user
        if not user.is_authenticated:
            return False
        if view.action == 'create':  # 获取到的方法名用来做其他判断
            return user.is_guest or user.is_developer or user.is_superuser
        else:
            return user.is_developer or user.is_superuser
```

`view` 这个参数（视图实例）会在以下 **官方钩子方法** 中传递

**权限类（`BasePermission` 子类）**

```python
class MyPermission(permissions.BasePermission):
    def has_permission(self, request, view):
        # view 就是当前请求的视图实例
        return True

    def has_object_permission(self, request, view, obj):
        # view 同样可用
        return True
```

**限流类（`BaseThrottle` 子类）**

```python
class MyThrottle(throttling.BaseThrottle):
    def allow_request(self, request, view):
        # view 参数可用
        return True
```

**分页类（`BasePagination` 子类）**

```python
class MyPagination(pagination.PageNumberPagination):
    def paginate_queryset(self, queryset, request, view=None):
        # view 参数可用（可选，但 DRF 会传）
        return super().paginate_queryset(queryset, request, view)
```

**过滤器后端（`BaseFilterBackend` 子类）**

```python
class MyFilterBackend(filters.BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        # view 参数可用
        return queryset
```

**版本控制类（`BaseVersioning` 子类）**

```python
class MyVersioning(versioning.BaseVersioning):
    def determine_version(self, request, *args, **kwargs):
        view = kwargs.get('view')  # view 参数在 kwargs 中
        return 'v1'
```

**元数据类（`BaseMetadata` 子类）**

```python
class MyMetadata(metadata.BaseMetadata):
    def determine_metadata(self, request, view):
        # view 参数可用
        return {}
```

**视图集（ViewSet）中的钩子方法（非参数，但可访问 `self`）**

```python
class MyViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # 这里可以直接用 self.action / self.request 等
        return MyModel.objects.all()
```

## 路由

用于声明FBV或CBV的访问路径

以ViewSet为例

```python
from rest_framework import routers

router = routers.SimpleRouter()
router.register(r'users', UserViewSet)
router.register(r'accounts', AccountViewSet)

urlpatterns += router.urls
# 或者
urlpatterns = [
    path('forgot-password', ForgotPasswordFormView.as_view()),
    path('', include(router.urls)),
]

```

`register()`有两个强制参数：

- `prefix`：用于指定这组路由的URL前缀
- `viewset`：viewset类

可选：

- `basename`：创建URL名称的基础，如果未设置，则将根据视图集的 `queryset` 属性（如果有）获取模型名，并且根据模型名小写生成 `basename`。请注意，如果视图集不包含 `queryset` 属性，则必须在注册视图集时设置 `basename`。

以上示例会生成：

- URL pattern: `^users/$` Name: `'user-list'`
- URL pattern: `^users/{pk}/$` Name: `'user-detail'`
- URL pattern: `^accounts/$` Name: `'account-list'`
- URL pattern: `^accounts/{pk}/$` Name: `'account-detail'`

通常，您**不需要**指定 `basename` 参数，但如果您的视图集定义了自定义 `get_queryset` 方法，则该视图集可能没有 `.queryset` 属性集。如果您尝试注册该 viewset，您将看到如下错误：

```
'basename' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.
```

这意味着你需要在注册 viewset 时显式设置 `basename` 参数，因为它无法从模型名称中自动确定。

| URL 样式                        | HTTP 方法                                    | 行动                                       | URL 名称                |
| ----------------------------- | ------------------------------------------ | ---------------------------------------- | --------------------- |
| {prefix}/                     | GET                                        | list                                     | {basename}-list       |
| {prefix}/                     | POST                                       | create                                   | {basename}-list       |
| {prefix}/{url_path}/          | GET, or as specified by `methods` argument | `@action(detail=False)` decorated method | {basename}-{url_name} |
| {prefix}/{lookup}/            | GET                                        | retrieve                                 | {basename}-detail     |
| {prefix}/{lookup}/            | PUT                                        | update                                   | {basename}-detail     |
| {prefix}/{lookup}/            | PATCH                                      | partial_update                           | {basename}-detail     |
| {prefix}/{lookup}/            | DELETE                                     | destroy                                  | {basename}-detail     |
| {prefix}/{lookup}/{url_path}/ | GET, or as specified by `methods` argument | `@action(detail=True)` decorated method  | {basename}-{url_name} |

默认情况下，`SimpleRouter` 创建的 URL 附加一个尾部斜杠。在实例化路由器时，可以通过将 `trailing_slash` 参数设置为 `False` 来修改此行为。例如：

```python
router = SimpleRouter(trailing_slash=False)
```

## 序列化

序列化器可以说是DRF的核心，它的作用是把Json或者XML与原生 Python 数据类型互相转换（**序列化和反序列化**），并且**验证**字段的合法性
最主要使用的序列化器有两种，`ModelSerializer` 和 `Serializer`，它们的共同点都是：

1. 序列化：对象 → Python 基本类型 → 渲染器 → 最终响应
2. 反序列化：解析请求体 → Python 基本类型 → 调用 `.is_valid()` → 校验 → `.validated_data`
3. 字段声明语法一致：`CharField`、`IntegerField`、`ListField`…
4. 校验钩子一致：
   - 字段级 `validate_<field_name>`
   - 对象级 `validate`
5. 都可以嵌套（Nested Serializer），也支持 `source=`、`read_only=`、`write_only=`、`required=` 等通用参数。

差异：

| 维度                  | Serializer    | ModelSerializer                                                                                |
| ------------------- | ------------- | ---------------------------------------------------------------------------------------------- |
| **字段来源**            | 全部手写。         | 默认把模型字段映射成对应的 DRF 字段，可 `exclude`/`fields` 控制。                                                  |
| **校验器**             | 仅你自己写的。       | 额外追加模型层约束：`unique_together`、`max_length`、`blank=False` 等自动映射成 DRF 校验器。                         |
| **create / update** | 必须自己实现（除非只读）。 | 已帮你实现通用版本：直接把 `validated_data` 解包成 `Model.objects.create(**data)` 或 `instance.update(**data)`。 |
| **Meta 类**          | 不需要。          | 必须有 `class Meta:`，指定 `model = ...`。                                                            |
| **额外字段**            | 无限制，自由组合。     | 可以混写，比如再写 `custom_field = SerializerMethodField()`。                                            |
| **性能/可控性**          | 字段越少越轻量。      | 字段多时省事，但可能带出你不想要的 `id`、反向关联等。                                                                  |

签名：

```python
class BaseSerializer(Field):
	def __init__(self, instance=None, data=empty, **kwargs):
		self.instance = instance  # 可以拿到原始的对象
		if data is not empty:
			self.initial_data = data  # 可以拿到原始的data值
		self.partial = kwargs.pop('partial', False)
		self._context = kwargs.pop('context', {})
		kwargs.pop('many', None)
		super().__init__(**kwargs)
```

`instance`用来**序列化**（Python对象 → JSON）
`data`用来**反序列化**（把 JSON → Python 对象，并做校验/保存）
两个参数在 `Serializer` 和 `ModelSerializer` 里的含义、取值、使用阶段完全一样；区别只在于 `ModelSerializer` 的 `instance` 必须是 **Django 模型实例**，而普通 `Serializer` 的 `instance` 可以是任意 Python 对象（dict、自定义类都行），对于普通 `Serializer`，`.data` 会按照字段定义，通过 `instance[field]`（如果是 dict）或 `getattr(instance, field)`（如果是对象）提取值

使用示例：

```python title:usage

book = Book.objects.get(pk=1)
ser = BookSerializer(instance=book)   # 只有 instance
ser.data                              # → {'id':1,'title':'三体',...}


payload = {'title': '三体Ⅲ', 'price': 29.9}
ser = BookSerializer(data=payload)   # 只有 data
ser.is_valid(raise_exception=True)   # 校验


book = Book.objects.get(pk=1)        # 已存在的模型实例
payload = {'title': '三体全集', 'price': 59.9}
ser = BookSerializer(instance=book, data=payload, partial=True)  # `instance` 告诉 DRF“这是要被更新的对象”，`data` 是前端传来的新值。  `partial=True` 表示允许部分字段更新。
ser.is_valid(raise_exception=True)
ser.save()                           # 调用 update()


queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]

```

反序列化多个对象的默认行为是支持创建多个对象，但不支持多个对象更新。有关如何支持或自定义这两种情况的更多信息，请参阅下面的 [ListSerializer](#ListSerializer) 文档。

---
### Serializer

通常来说`Serializer`接收的是与数据库对象无关的数据，比如`CommonResponse`之类的

#### 字段级验证

在序列化器中，字段级验证是一个很有用的工具，他会在`serializer.is_valid()`时调用，校验错误时请抛出`serializers.ValidationError`，例如：

```python hl:13
from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        Check that the blog post is about Django.
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value  # 务必返回原始值
```

**注意：** 如果 `<field_name>` 在序列化程序上使用参数 `required=False` 声明，则如果未包含该字段，则不会执行此验证步骤。

或者也可以在字段上定义验证器

```python hl:6
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = serializers.IntegerField(validators=[multiple_of_ten])
    ...
```

#### 对象级验证

如果需要验证整个对象，需要添加一个固定签名的方法`validate()`到序列化器类中，这个方法只有一个参数`data`，是字段值的字典。如果验证错误，请抛出`serializers.ValidationError`

```python hl:14
from rest_framework import serializers

class EventSerializer(serializers.Serializer):
    description = serializers.CharField(max_length=100)
    start = serializers.DateTimeField()
    finish = serializers.DateTimeField()

    def validate(self, data):
        """
        Check that start is before finish.
        """
        if data['start'] > data['finish']:
            raise serializers.ValidationError("finish must occur after start")
        return data  # 务必返回原始值
```

序列化程序类还可以包括应用于整个字段数据集的可重用验证器。通过在内部 `Meta` 类中声明这些验证器来包含它们，如下所示：

```python hl:8-13
class EventSerializer(serializers.Serializer):
    name = serializers.CharField()
    room_number = serializers.ChoiceField(choices=[101, 102, 103, 201])
    date = serializers.DateField()

    class Meta:
        # Each room only has one event per day.
        validators = [
            UniqueTogetherValidator(
                queryset=Event.objects.all(),
                fields=['room_number', 'date']
            )
        ]
```

更多高级的验证器请查看[官方文档](https://www.django-rest-framework.org/api-guide/validators/)

#### 处理嵌套对象

`Serializer` 类本身就是一种 `Field` 类型，可用于表示一种对象类型嵌套在另一种对象类型中的关系。

```python hl:6
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()  #many=True
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

同样，如果嵌套表示应该是一个列表，你应该将 `many=True` 标志传递给嵌套序列化器。

由于嵌套创建和更新的行为可能不明确，并且可能需要相关模型之间的复杂依赖关系，因此DRF不会帮我们创建可用的嵌套支持。但是有第三方库可以做到[drf-writable-nested](https://github.com/beda-software/drf-writable-nested)

---
### ModelSerializer
**`ModelSerializer` 类与常规 `Serializer` 类相同，不同之处在于** ：
- 它将根据模型自动生成一组字段。
- 它将为序列化器自动生成验证器，例如 unique_together 验证器。
- 它包括 `.create()` 和 `.update()` 的简单默认实现。

默认情况下，类上的所有模型字段都将映射到相应的序列化器字段。

模型上的任何关系（如外键）都将映射到 `PrimaryKeyRelatedField`，即显示为主键。默认情况下，除非按照[序列化程序关系](https://www.django-rest-framework.org/api-guide/relations/)文档中的指定明确包含反向关系，否则不会包含反向关系。

#### 指定包含字段
例如：
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']  # 这里需要包含显式声明的序列化字段
        # 还可以将 `fields` 属性设置为特殊值 `'__all__'` 以指示应使用模型中的所有字段。例如：
        fields = '__all__'
        # 可以将 `exclude` 属性设置为要从序列化程序中排除的字段列表。例如：
        exclude = ['users']
		# 如果 `Account` 模型有 3 个字段 `account_name`、`users` 和 `created`，这将导致字段 `account_name` 和 `created` 被序列化。
```

`fields` 和 `exclude` 这两个字段是互斥的。

`fields` 和 `exclude` 属性中的名称通常会映射到模型类上的模型字段。

`fields` 选项中的名称还可以映射到 `@property` 装饰的方法 或 模型上的 **无参方法**，或者你在序列化器里手动写的字段名（只要它能通过 `source` 解析），即使这个字段不存在于数据库模型中也可以。例如：
```python
class Book(models.Model):
    title = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=6, decimal_places=2)

    @property
    def discounted_price(self):
        return float(self.price) * 0.9        # 不需要参数

    def full_title(self):
        return f"{self.title} (${self.price})"  # 无参方法

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ('id', 'title', 'price', 'discounted_price', 'full_title')
        # discounted_price 和 full_title 都不是数据库字段 
```
只要能通过 `getattr(instance, name)` 拿到值即可

> 从版本 3.3.0 开始， **必须**提供 `fields`或 `exclude`其中一个。

#### 指定只读字段

如果需要添加多个只读字段，可以使用`Meta.read_only_fields`，而不是在每个字段使用 `read_only=True` 属性。

`Meta.read_only_fields`应该是字段名称的列表或元组：

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'users', 'created']
        read_only_fields = ['account_name']
```

默认情况下，设置了 `editable=False` 和 `AutoField` 字段的模型字段将设置为只读，无需添加到 `read_only_fields` 选项中。
**特殊场景：**
如果只读字段同时需要参与`unique_together`约束，例如 `user` 字段对用户是只读的（不能自己改），且数据库要求`(user, some_other_id)`唯一，如果此时`user`是只读的，验证器拿不到`user`的值就会有问题。正确做法是同时给这个字段 `read_only=True`（前端不可改），`default=CurrentUserDefault()`（后端自动填当前登录用户）
```python
class RecordSerializer(serializers.ModelSerializer):
    user = serializers.PrimaryKeyRelatedField(
        read_only=True,
        default=serializers.CurrentUserDefault()
    )

    class Meta:
        model = Record
        fields = ['id', 'user', 'some_other_id', 'data']
        # 不需要再把 user 放进 read_only_fields，上面已显式声明
```

还有一个快捷方式，允许您使用 `extra_kwargs` 选项在字段上指定任意附加关键字参数。与 `read_only_fields` 的情况一样，这意味着您不需要在序列化程序上显式声明字段。

此选项是一个字典，用于将字段名称映射到关键字参数的字典。例如：

```python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['email', 'username', 'password']
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```

请记住，如果已在序列化器类上显式声明了该字段，则 `extra_kwargs` 选项将被忽略。


#### 创建和更新
`ModelSerializer`中自带了`update`和`create`方法的实现，它最终会调用`self.Meta.model`本身的`create`和`save`方法
签名：

```python title:ModelSerializer hl:5,10
class ModelSerializer(Serializer):
    def create(self, validated_data):
	    ...
	    ModelClass = self.Meta.model
	    instance = ModelClass._default_manager.create(**validated_data)
	    ...
	    
	def update(self, instance, validated_data):
		...
		instance.save()
		...
```

并且DRF会自动调用`serializer.save()`，由它来自动确定调用`create`还是`update`

```python title:DRFMixin hl:13,30,44,47
class CreateModelMixin:
    """
    Create a model instance.
    """
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()

        
class UpdateModelMixin:
    """
    Update a model instance.
    """
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)
        ...
        return Response(serializer.data)

    def perform_update(self, serializer):
        serializer.save()

    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(request, *args, **kwargs)


class BaseSerializer(Field):
	def save(self, **kwargs):
        ...

        validated_data = {**self.validated_data, **kwargs}

        if self.instance is not None:
            self.instance = self.update(self.instance, validated_data)
            ...
        else:
            self.instance = self.create(validated_data)
            ...

        return self.instance
```

可以看到`save`方法其实是支持传入更多参数以保存到数据库的，但是DRF的默认`perform_`方法里面没有添加额外的参数，我们可以重写以传入一些额外的内容例如`serializer.save(owner=request.user)`，并且`save`方法是会返回对应的实例的，但是DRF丢弃了它

#### ModelSerializer 的 Meta 选项

| 选项                | 说明                                                           |
| ----------------- | ------------------------------------------------------------ |
| model             | 必写，对应 ORM 模型                                                 |
| fields            | '\_\_all\_\_' 或列表 / 元组                                       |
| exclude           | 反向排除                                                         |
| depth             | 自动嵌套深度（0~10）                                                 |
| read_only_fields  | 仅序列化                                                         |
| write_only_fields | 仅反序列化（3.x 已弃用，用 extra_kwargs）                                |
| extra_kwargs      | 对单个字段再附加参数：extra_kwargs = {'password': {'write_only': True}} |
| validators        | 模型级验证器列表                                                     |
| field_classes     | 自定义映射，如 {'JSONField': MyJSONField}                           |

#### 高级字段默认值
有时可能会有字段需要用于验证器，但不由用户直接输入，比如说当前用户。此时就可以使用 `HiddenField`，这个字段将会出现在 `validated_data` 中，但不会用于序列化程序输出表示形式。使用了`read_only=True`的字段不生效`default=...`参数。
##### 当前用户默认值
可用于表示当前用户的默认类。为了使用它，在实例化序列化程序时，必须将“请求”作为上下文字典的一部分提供。

```python
owner = serializers.HiddenField(default=serializers.CurrentUserDefault())
```

##### 仅创建默认值
一个默认类，仅可用于在创建作期间设置默认参数。在更新期间，该字段被省略。它采用单个参数，该参数是创建作期间应使用的默认值或可调用对象。
```python
created_at = serializers.DateTimeField(default=serializers.CreateOnlyDefault(timezone.now))
```

---
### HyperlinkedModelSerializer
不同之处在于所有的外键关系都使用超链接表示，而不是默认的主键表示
默认情况下，序列化程序将包含一个 `url` 字段，而不是一个主键字段。

url 字段将使用 `HyperlinkedIdentityField` 序列化器字段表示，模型上的任何关系都将使用 `HyperlinkedRelatedField` 序列化器字段表示。如果不想用url这个字段可以在Django设置中修改
```python
REST_FRAMEWORK = {
    'URL_FIELD_NAME': 'api_url'   # 现在所有 HyperlinkedModelSerializer 的字段叫 api_url
}
```

您可以通过将主键添加到 `fields` 选项来显式包含主键，例如：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ['url', 'id', 'account_name', 'users', 'created']
```

实例化 `HyperlinkedModelSerializer` 时，必须包含当前的 `请求` ，例如：

```python
serializer = AccountSerializer(queryset, context={'request': request})
```

这样做将确保超链接可以包含适当的主机名，以便生成的表示形式使用完全限定的 URL，例如：

```
http://api.example.com/accounts/1/
```

而不是相对 URL，例如：

```
/accounts/1/
```

如果你**确实**想使用相对 URL，你应该显式地传递 `{'request'： None}` 在序列化程序上下文中。

**HyperlinkedModelSerializer 最终生成的“超链接”到底指向哪个 URL，完全由“视图名(view_name)” + “查找字段(lookup_field)”决定；你可以用 `extra_kwargs` 快捷改，也可以显式写字段，甚至全局改默认字段名。**

**默认规则（无需配置）：**
- **视图名**：`<model_name>-detail`  
    例：`account-detail`
- **查找字段**：`pk`  
    例：`/accounts/123/`

**快捷改写`Meta.extra_kwargs`：**
只想改某字段的视图名或查找字段，又不想在类里显式写字段，就用 `extra_kwargs`：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model   = Account
        fields  = ['url', 'account_name', 'users', 'created']
        extra_kwargs = {
            'url':   {'view_name': 'accounts', 'lookup_field': 'account_name'},
            'users': {'lookup_field': 'username'}   # 只改 users 的查找字段
        }
```

结果
- `url` → `/accounts/<account_name>/`
- `users` → `/users/<username>/`

**显式写字段：完全掌控**

当需要更多参数（`queryset`, `lookup_url_kwarg`, `format` 等）或一次性声明多个关系字段时，直接在类里定义：

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    url   = serializers.HyperlinkedIdentityField(
        view_name='accounts', lookup_field='slug'
    )
    users = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        lookup_field='username',
        many=True,
        read_only=True
    )

    class Meta:
        model  = Account
        fields = ['url', 'account_name', 'users', 'created']
```

`view_name`就是url的name属性，`lookup_field`是模型里的字段名，`lookup_url_kwarg`是url里的`<...:xxx>`捕获组名

---
### ListSerializer

`ListSerializer` 类提供了一次序列化和验证多个对象的行为。 通常不需要直接使用 `ListSerializer`，而是应该在实例化序列化器时简单地传递 `many=True`。

当实例化序列化器并传递 `many=True` 时，将创建一个 `ListSerializer` 实例。然后，序列化器类成为父 `ListSerializer` 的子类。

以下参数也可以传递给 `ListSerializer` 字段或传递 `many=True` 的序列化器：

`allow_empty`
默认情况下，这是 `True`，但如果要不允许将空列表作为有效输入，则可以将其设置为 `False`。

`max_length`
默认情况下，这是 `None` ，但如果要验证列表包含的元素数是否不超过此数量的元素，则可以将其设置为正整数。

`min_length`
默认情况下，这是 `None` ，但如果要验证列表包含的元素数是否少于此数量的元素，则可以将其设置为正整数。

#### 多个创建
创建多个对象的默认实现是简单地为列表中的每个项目调用 `.create()` 。如果要自定义此行为，则需要自定义 `ListSerializer` 类上的 `.create()` 方法，该方法在传递 `many=True` 时使用。
```python hl:2-4,9
class BookListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        books = [Book(**item) for item in validated_data]
        return Book.objects.bulk_create(books)

class BookSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```

#### 多个更新
默认情况下，`ListSerializer` 类不支持多次更新。这是因为插入和删除应该预期的行为是不明确的。如果需要支持多个更新，只能重写并指定`list_serializer_class`，例如：
```python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # We need to identify elements in the list using their primary key,
    # so use a writable field here, rather than the default which would be read-only.
    id = serializers.IntegerField()
    ...

    class Meta:
        list_serializer_class = BookListSerializer
```

---
### 序列化器字段

#### 布尔字段
##### **`BooleanField`**
**签名：**`BooleanField()`
省略值将会被视为`False`，即使指定了`default=True`也不行

#### 字符串字段
##### **`CharField`**
**签名：** `CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)`
- `max_length` - 验证输入包含的字符数是否不超过此数量。
- `min_length` - 验证输入包含的字符数是否不少于此数量。
- `allow_blank` - 如果设置为 `True`，则空字符串应被视为有效值。如果设置为 `False`，则空字符串被视为无效，并将引发验证错误。默认为 `False`。
- `trim_whitespace` - 如果设置为 `True`，则修剪前导和尾随空格。默认为 `True`。

##### **`EmailField`**
**签名：** `EmailField(max_length=None, min_length=None, allow_blank=False)`
文本表示形式，验证文本是否为有效的电子邮件地址。

##### **`RegexField`**
**签名：** `RegexField(regex, max_length=None, min_length=None, allow_blank=False)`
- `regex` - 可以是字符串，也可以是编译的 python 正则表达式对象。
一种文本表示形式，用于验证给定值与特定正则表达式的匹配。
使用 Django 的 `django.core.validators.RegexValidator` 进行验证。

##### **`SlugField`**
**签名：** `SlugField(max_length=50, min_length=None, allow_blank=False)`
等价于`RegexField(r'^[a-zA-Z0-9_-]+$')`

##### **`URLField`**
**签名：** `URLField(max_length=200, min_length=None, allow_blank=False)`
也是一个特殊的`RegexField`，需要格式为 `<scheme>://<host>/<path>` 的完全限定 URL。
使用 Django 的 `django.core.validators.URLValidator` 进行验证。

##### **`UUIDField`**
**签名：** `UUIDField(format='hex_verbose')`
- `format`：确定 uuid 值的表示格式
    - `'hex_verbose'` - 规范十六进制表示，包括连字符： `"5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`
    - `'hex'` - UUID 的紧凑十六进制表示，不包括连字符： `"5ce0e9a55ffa654bcee01238041fb31a"`
    - `'int'` - UUID 的 128 位整数表示： `"123456789012312313134124512351145145114"`
    - `'urn'` - RFC 4122 UUID 的 URN 表示： `"urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a"` 

**format 参数只决定 `UUIDField` 在JSON 输出时用什么字符串样式，无论哪种样式，反序列化（用户传进来的数据）都能被正确解析成 UUID 对象。**

##### **`FilePathField`**
**签名：** `FilePathField(path, match=None, recursive=False, allow_files=True, allow_folders=False, required=None, **kwargs)`
- `path` - 从指定的文件系统绝对路径中选择。
- `match` - 一个正则表达式，作为字符串，FilePathField 将用于过滤文件名。
- `recursive` - 指定是否应包含路径的所有子目录。默认值为 `False`。
- `allow_files` - 指定是否应包括指定位置中的文件。默认值为 `True`。
- `allow_folders` - 指定是否应包括指定位置中的文件夹。默认值为 `False`。

`allow_files`或`allow_folders` 必须有一个为`True`，否则就没东西可以选了。且`FilePathField`仅“选文件名”，不传文件内容。

例：
```python
serializers.FilePathField(
    path="/absolute/path/to/dir",   # 必须绝对路径
	match=r".*\.html$",             # 正则过滤文件名，如只允许 .html
    recursive=False,                # 是否递归子目录
    allow_files=True,               # 是否包含文件
    allow_folders=False,            # 是否包含文件夹
    allow_empty=False,              # 空字符串是否合法
)
```
`FilePathField` = **“目录枚举器”** → 把服务器某个目录下的文件名变成 **下拉选项**，只能选，不能传。

##### **`IPAddressField`**
**签名** ： `IPAddressField(protocol='both', unpack_ipv4=False, **options)`
- `protocol`将有效输入限制为指定协议。接受的值为“both”（默认值）、“IPv4”或“IPv6”。匹配不区分大小写。
- `unpack_ipv4` 解压缩 IPv4 映射地址，例如`::ffff:192.0.2.1`。如果启用此选项，则该地址将解压缩为 `192.0.2.1`。默认值为`False`。仅当`protocol`设置为`'both'`时才能使用。
例如，某些网络设备或系统可能会将 IPv4 地址以 IPv6 的形式发送，但你希望在内部处理时使用普通的 IPv4 格式。
该参数主要用于 **反序列化**（用户提交数据时），**序列化输出** 时不会受影响。


#### 数值字段
##### **`IntegerField`**
**签名** ： `IntegerField(max_value=None, min_value=None)`
- `max_value` 验证提供的数字是否不大于此值。
- `min_value` 验证提供的数字是否不小于此值。

##### **`FloatField`**
**签名** ： `FloatField(max_value=None, min_value=None)`
- `max_value` 验证提供的数字是否不大于此值。
- `min_value` 验证提供的数字是否不小于此值。

##### **`DecimalField`**
**签名** ： `DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)`
- `max_digits` 数字中允许的最大位数（包括小数点前后）。它必须是 `None` 或大于或等于 `decimal_places` 的整数。
- `decimal_places` 小数点后的位数。
- `coerce_to_string` 如果应为表示返回字符串值，则设置为 `True`，如果应返回 `Decimal` 对象，则设置为 `False`。默认值与 `COERCE_DECIMAL_TO_STRING` 设置键相同的值，除非覆盖，否则该值将为 `True`。如果序列化程序返回 `Decimal` 对象，则最终输出格式将由呈现器确定。请注意，设置 `localize` 将强制该值为 `True`。
- `max_value` 验证提供的数字是否不大于此值。应该是整数或`Decimal`对象。
- `min_value` 验证提供的数字是否不小于此值。应该是整数或`Decimal`对象。
- `localize`设置为 `True` 可根据当前区域设置启用输入和输出的本地化。这也会强制 `coerce_to_string` 为 `True`。默认为 `false`。请注意，如果您在设置文件中设置了 `USE_L10N=True`，则启用数据格式。
- `rounding`设置量化到配置精度时使用的舍入模式。有效值为`Decimal`模块的[舍入模式](https://docs.python.org/3/library/decimal.html#rounding-modes) 。默认为 `None`。
- `normalize_output` 序列化时将规范化十进制值。这将剥离所有尾随零，并将值的精度更改为所需的最小精度，以便能够在不丢失数据的情况下表示值。默认为 `false`。

#### 日期和时间字段
##### **`DateTimeField`**
**签名：** `DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None, default_timezone=None)`
- `format` - 输出格式。默认值由 `DATETIME_FORMAT` 设置决定，通常是 `iso-8601`。可以设置为 `None`，此时返回 `datetime` 对象，由渲染器决定最终格式。
- `input_formats` - 输入格式列表。默认值由 `DATETIME_INPUT_FORMATS` 设置决定，通常是 `['iso-8601']`。
- `default_timezone` - 默认时区。如果 `USE_TZ=True`，则默认为当前时区。接受表示时区的 `tzinfo` 子类（`zoneinfo` 或 `pytz`）。

`format`支持传入[Python strftime格式](https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior)，也可以是特殊值 `'iso-8601'`。

当使用 `ModelSerializer` 或 `HyperlinkedModelSerializer` 时，如果模型字段设置了 `auto_now=True` 或 `auto_now_add=True`，那么在生成序列化器字段时，DRF 会自动将这些字段设置为 **`read_only=True`**。如果希望改变这种默认行为（例如允许用户更新这些字段），则需要在序列化器中 **显式声明** 这些字段。
```python
class Comment(models.Model):
    created = models.DateTimeField(auto_now_add=True)  # 创建时自动设置
    updated = models.DateTimeField(auto_now=True)     # 每次保存时自动设置

from rest_framework import serializers
from .models import Comment

class CommentSerializer(serializers.ModelSerializer):
    created = serializers.DateTimeField(read_only=False)  # 显式声明，允许更新

    class Meta:
        model = Comment
        fields = ['id', 'created', 'updated']
```

##### **`DateField`**
**签名：** `DateField(format=api_settings.DATE_FORMAT, input_formats=None)`
- `format` - 输出格式。默认值由 `DATE_FORMAT` 设置决定，通常是 `iso-8601`。可以设置为 `None`，此时返回 `datetime` 对象，由渲染器决定最终格式。
- `input_formats` - 输入格式列表。默认值由 `DATE_INPUT_FORMATS` 设置决定，通常是 `['iso-8601']`。
同上。

##### **`TimeField`**
**签名：** `TimeField(format=api_settings.TIME_FORMAT, input_formats=None)`
- `format` - 输出格式。默认值由 `TIME_FORMAT` 设置决定，通常是 `iso-8601`。可以设置为 `None`，此时返回 `datetime` 对象，由渲染器决定最终格式。
- `input_formats` - 输入格式列表。默认值由 `TIME_INPUT_FORMATS` 设置决定，通常是 `['iso-8601']`。
同上。

##### **`DurationField`**
**签名：** `DurationField(max_value=None, min_value=None)`
- `max_value` 验证提供的持续时间是否不大于此值，接收`datetime.timedelta`对象。
- `min_value` 验证提供的持续时间是否不小于此值，接收`datetime.timedelta`对象。

`DurationField` 接受字符串格式的时间间隔，例如 `'[DD] [HH:[MM:]]ss[.uuuuuu]'`。例如：`'1 day, 2:30:45'` 表示 1 天 2 小时 30 分 45 秒。如果用户提交的值是 `'1 day, 2:30:45'`，则会解析为 `datetime.timedelta(days=1, hours=2, minutes=30, seconds=45)`。

#### 选择选项字段
##### **`ChoiceField`**
如果相应的模型字段包含 `choices=...` 参数，则由`ModelSerializer` 用于自动生成字段。
**签名：**`ChoiceField(choices)`
- `choices` - 有效值的列表，或 `(key, display_name)` 元组的列表。
- `allow_blank` - 如果设置为 `True`，则空字符串应被视为有效值。如果设置为 `False`，则空字符串被视为无效，并将引发验证错误。默认为 `false`。
- `html_cutoff` - 如果设置，这将是 HTML 选择下拉列表显示的最大选项数。可用于确保自动生成的具有非常大的可能选择的 ChoiceFields 不会阻止模板呈现。默认为 `None`。
- `html_cutoff_text` - 如果设置，如果 HTML 选择下拉列表中已截断最大项目数，则将显示文本指示器。默认为 `“超过 {count} 项...”`

`allow_blank` 和 `allow_null` 都是 `ChoiceField` 上的有效选项，但强烈建议您只使用一个选项，而不是同时使用两个选项。文本选择应首选 `allow_blank`，数字或其他非文本选择应首选 `allow_null`。

##### **`MultipleChoiceField`**
同上。在验证器中收到的值是一个包含了用户所选项的列表，验证通过返回的也是这个列表。前端需要传入数组。

#### 文件上传字段
使用这两个字段时，前端应该使用`'Content-Type': 'multipart/form-data'`

##### **`FileField`**
**签名：**`FileField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`
- `max_length` - 文件名的最大长度
- `allow_empty_file` - 是否允许上传空文件
- `use_url` - 如果为 `True`，则输出文件的 URL；如果为 `False`，则输出文件名。默认为`True`。

##### **`ImageField`**
**签名：**`ImageField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`
同上。使用`ImageField`需要`Pillow`配合工作。用于验证上传的文件是否为有效的图像格式，并提供图像处理功能。

#### 关联关系字段
##### `RelatedField`
所有关联关系字段的基类
```python
class RelatedField(Field):
    def __init__(self, read_only=False, write_only=False,
                 required=None, allow_null=False, default=empty,
                 source=None, validators=None, error_messages=None,
                 label=None, help_text=None, style=None,
                 # 关系字段专用
                 queryset=None, many=None,
                 allow_empty=True, html_cutoff=None, html_cutoff_text=None):
        super().__init__(...)   # 基类 Field 的其它通用参数
```

`queryset`
当字段需要 **写入**（POST/PUT/PATCH）时，DRF 会用这个 `queryset` 做两件事：1. 校验提交的 pk 或 URL 是否在范围内；2. 通过 `queryset.get()` 拿到真正的对象实例。
如果不给 `queryset`，DRF 会把字段自动设为 `read_only=True`，从而变成只读。

`many`
当模型字段本身是 `ForeignKey`（单对象）时，`many` 默认为 `False`。当模型字段是 `ManyToManyField` 或反向 `ForeignKey` 时，需要 `many=True`，否则序列化器会抛异常。
- `many=False` → 单个值（`{"artist": 3}`）
- `many=True` → 列表（`{"tracks": [1, 2, 3]}`）

##### `StringRelatedField`
**签名：**`StringRelatedField()`
这个字段类型定死`read_only=True`（只读），其余参数跟随`RelatedField`，序列化时直接使用`str(value)`来获取显示内容，因此可以自定义模型字段的`__str__()`来决定显示什么内容。

##### `PrimaryKeyRelatedField`
**签名：** `PrimaryKeyRelatedField(pk_field=None)`
读/写。用主键（id）表示关联对象。可以自定义主键字段是哪个。

##### `SlugRelatedField`
**签名：** `SlugRelatedField(slug_field)`
读/写。必须传入`slug_field`，用 model 上某个 `slug` 字段的值表示关联对象。

##### `HyperlinkedRelatedField`
**签名：** `HyperlinkedRelatedField(view_name, lookup_field='pk', lookup_url_kwarg='pk')`
渲染成指向关联对象的超链接（URL）。当你的序列化类继承自`HyperlinkedModelSerializer`的时候自动使用这个字段，可以在`Meta.fields`中添加`url`字段以显示。`lookup_url_kwarg`不设置时默认与`lookup_field`相同。
序列化时使用`url = self.reverse(view_name, kwargs=kwargs, request=request)`（就是django的`reverse`函数，如果给了request上下文，则会再使用`request.build_absolute_uri(url)`构建完整的uri）

##### `HyperlinkedIdentityField`
**签名：** `HyperlinkedRelatedField(view_name, lookup_field='pk', lookup_url_kwarg='pk')`
`HyperlinkedIdentityField`稍微有点特殊，它继承自`HyperlinkedRelatedField`，在语义上表达**指向模型自身**的超链接，而`HyperlinkedRelatedField`可以表达其他模型字段的超链接。并且`HyperlinkedIdentityField`是**只读**的，而`HyperlinkedRelatedField`不是。


#### 复合字段
##### **`ListField`**
**签名：**`ListField(child=<A_FIELD_INSTANCE>, allow_empty=True, min_length=None, max_length=None)`
- `child` - 指定列表中每个元素的字段类型，必须是一个字段实例。
- `allow_empty` - 是否允许空列表。默认为 `True`。
- `min_length` - 列表的最小长度。如果设置，列表长度不能小于该值。
- `max_length` - 列表的最大长度。如果设置，列表长度不能大于该值。

例如：
```python
tags = serializers.ListField(child=serializers.CharField(max_length=100))
```

##### **`DictField`**
**签名：**`DictField(child=<A_FIELD_INSTANCE>, allow_empty=True)`
- `child` - 指定字典中每个值的字段类型，必须是一个字段实例。
- `allow_empty` - 是否允许空字典。默认为 `True`。

##### **`JSONField`**
**签名：**`JSONField(binary=False, encoder=None)`
- `binary` - 如果为 `True`，则输入和输出都以 JSON 字符串形式处理。默认为 `False`。
- `encoder` - 自定义 JSON 编码器，用于序列化输出。默认为 `None`。

##### **`HStoreField`**
**签名：**`HStoreField(child=<A_FIELD_INSTANCE>, allow_empty=True)`
- **`child`**：指定字典中每个值的字段类型，必须是一个字段实例。默认情况下，`HStoreField` 使用 `CharField` 作为子字段，允许值为字符串类型。
- **`allow_empty`**：是否允许空字典。默认为 `True`。
`HStoreField` 是 DRF 提供的一个字段类型，用于处理 PostgreSQL 的 `HStoreField` 数据类型。`HStoreField` 是一个键值对字典，其中键是字符串，值也是字符串。它非常适合存储非结构化的键值对数据。

#### 特殊字段

##### **`ReadOnlyField`**
**签名** ：`ReadOnlyField()`
用于表示只读字段，这些字段在序列化时会输出，但在反序列化时不会被处理。**专用于模型属性/方法**（如 `@property`、无参方法），这些字段不需要用户输入，但需要在输出时显示。
例如：
```python
expiry_date = ReadOnlyField(source='get_expiry_date')
```
通常来说不需要手动指定这个类型，例如，如果 `has_expired` 是 `Account` 模型上的属性（`@property`），则以下序列化程序会自动将其生成为 `ReadOnlyField`：
```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ['id', 'account_name', 'has_expired']
```
- **模型字段** → 用普通字段 + `read_only=True`
- **模型属性/方法** → 用 `ReadOnlyField` + 指定 `source`

##### **`HiddenField`**
**签名：**`HiddenField(default=...)`
用于隐藏字段，这些字段在序列化时不会输出，但在反序列化时会使用默认值。通常用于需要在反序列化时自动填充的字段，但不需要用户输入。
例如：
```python
created_by = serializers.HiddenField(default=serializers.CurrentUserDefault())
```
**注意：**`HiddenField()` 不会出现在 `partial=True` 序列化器中（当发出 `PATCH` 请求时）。

##### **`ModelField`**
**签名：** `ModelField(model_field=<Django ModelField instance>)`
用于将任意 Django 模型字段桥接到序列化器字段。通常用于自定义模型字段，这些字段需要特殊的验证逻辑。
`ModelField` 类通常供内部使用，但如果需要，您的 API 也可以使用该类。为了正确实例化 `ModelField`，必须向它传递一个附加到实例化模型的字段。例如： `ModelField(model_field=MyModel()._meta.get_field('custom_field'))`

##### **`SerializerMethodField`**
**签名** ： `SerializerMethodField(method_name=None)`
- `method_name` - 指定方法名，默认为 `get_<field_name>`。
用于表示只读字段，这些字段的值通过调用序列化器上的方法动态生成。通常用于需要动态计算的字段，这些字段不需要用户输入，但需要在输出时显示。
方法名必须以 `get_` 开头，接收一个参数 `obj`，返回字段的值。例如：
```python
class ProfileSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()

    class Meta:
        model = Profile
        fields = ['id', 'first_name', 'last_name', 'full_name']

    def get_full_name(self, obj):
        return f"{obj.first_name} {obj.last_name}"
```

#### 自定义字段
如果需要自定义字段，则需要继承`serializers.Field`，然后实现两个关键的钩子函数，`to_representation(self, value)`和`to_internal_value(self, data)`，例如：
```python
from rest_framework import serializers
from django.core.exceptions import ValidationError

class ColorField(serializers.Field):
    def to_representation(self, value: 'Color'):
        # 模型 -> JSON
        return f"rgb({value.r},{value.g},{value.b})"

    def to_internal_value(self, data):
        # JSON -> 模型
        if not isinstance(data, str) or not data.startswith("rgb("):
            raise ValidationError("格式必须为 rgb(r,g,b)")
        try:
            r, g, b = map(int, data[4:-1].split(","))
        except ValueError:
            raise ValidationError("rgb 值必须为整数")
        return Color(r, g, b)
```



#### 序列化字段与模型字段对照表

| 序列化字段          | 模型字段                                                                          |
| -------------- | ----------------------------------------------------------------------------- |
| BooleanField   | BooleanField                                                                  |
| CharField      | CharField、TextField                                                           |
| EmailField     | EmailField                                                                    |
| RegexField     | RegexField                                                                    |
| SlugField      | SlugField                                                                     |
| URLField       | URLField                                                                      |
| FilePathField  | FilePathField                                                                 |
| IPAddressField | IPAddressField、GenericIPAddressField                                          |
| IntegerField   | IntegerField、SmallIntegerField、PositiveIntegerField、PositiveSmallIntegerField |
| FloatField     | FloatField                                                                    |
| DecimalField   | DecimalField                                                                  |
| DateTimeField  | DateTimeField                                                                 |
| DateField      | DateField                                                                     |
| TimeField      | TimeField                                                                     |
| DurationField  | DurationField                                                                 |
| FileField      | FileField                                                                     |
| ImageField     | ImageField                                                                    |




#### 字段构造器公共参数

| 参数                | 类型   | 说明          | 示例                               |
| ----------------- | ---- | ----------- | -------------------------------- |
| read_only         | bool | 仅序列化输出      | read_only=True                   |
| write_only        | bool | 仅反序列化输入     | write_only=True                  |
| required          | bool | 反序列化时是否必填   | required=False                   |
| allow_null        | bool | 是否接受 None   | allow_null=True                  |
| default           | 任意   | 缺失时的默认值     | default=''                       |
| source            | str  | 指定属性路径      | source='user.profile.avatar.url' |
| validators        | list | 自定义验证器      | validators=[my_func]             |
| error_messages    | dict | 自定义错误文案     | error_messages={'blank': '不能为空'} |
| style             | dict | HTML 表单渲染提示 | style={'input_type': 'password'} |
| label / help_text | str  | 元信息         | label='标题'                       |

#### `source`参数详解

```python
class UserSerializer(serializers.Serializer):
    # 前端看到的是 nickName，实际取模型字段 username
    nickName = serializers.CharField(source='username')
    
	author_name = serializers.CharField(source='author.username')  # 跨关联对象（点语法）

```

**字典/列表/方法**

- 字典：`source='dict_key'`
- 方法：`source='get_full_name'`（DRF 会自动调用无参方法）
- 列表索引：`source='tags.0.name'`（DRF 3.12+）

**只读计算字段**

```python
class OrderSerializer(serializers.Serializer):
    total_price = serializers.DecimalField(
        max_digits=10, decimal_places=2,
        source='calc_total', read_only=True
    )
```

模型里：

```python
class Order(models.Model):
    ...
    def calc_total(self):
        return sum(item.price for item in self.items.all())
```

- 对外暴露 `total_price`，内部实际调用 `order.calc_total()`。

与`SerializerMethodField`的区别是：`source` 只能**静态映射**某个已有属性/方法，而 `SerializerMethodField` 让你**完全自定义**一段 Python 代码来计算值，自由度更高，但永远是只读

| 维度         | `source='xxx'`            | `SerializerMethodField`         |
| ---------- | ------------------------- | ------------------------------- |
| **数据来源**   | 必须能写成 **属性链** 或 **无参方法名** | 任意 Python 逻辑（ORM、调用服务、聚合计算……）   |
| **可否反向写入** | ✅ 如果映射到可写字段               | ❌ 永远只读                          |
| **代码位置**   | 声明字段时一次性写完                | 单独写 `get_<field>(self, obj)` 方法 |
| **可读性**    | 简单场景直观                    | 复杂逻辑更清晰                         |
| **性能**     | ORM 会连带 select            | 自己控制 queryset 优化                |

### 常见踩坑提示

1. DateTimeField 带微秒的字符串反序列化失败 → 设置 `format='%Y-%m-%d %H:%M:%S.%f'`。
2. 外键字段默认是 PrimaryKeyRelatedField，前端需要传 ID；若想传 slug，改用 SlugRelatedField(slug_field='slug')。
3. 在 ModelSerializer 里给字段加 validators 会 **覆盖** 模型层的 validators，除非显式 extend。
4. Serializer.save() 只有在调用 is_valid() 后才能用，否则抛 ValueError。
5. 上传文件时，确保视图 parser_classes 包含 MultiPartParser / FormParser。

## 认证
这部分的作用仅仅是标识当前请求的发出者，并不负责允许或不允许传入请求，即权限校验部分。

身份验证方案是定定义为类列表，REST 框架将尝试对列表中的每个类进行身份验证，并将使用成功进行身份验证的**第一个类**的返回值设置 `request.user` 和 `request.auth`。

如果没有类进行身份验证， 则`request.user` 将设置为 `django.contrib.auth.models.AnonymousUser` 的实例，并且 `request.auth` 将设置为 `None`。

对于未经身份验证的请求，可以使用 `UNAUTHENTICATED_USER` 和 `UNAUTHENTICATED_TOKEN` 设置修改 `request.user` 和 `request.auth` 的值。

### 设置身份验证方案
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',  # 通常来说都不用这个
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # 例如需要jwt登录方案
    ]
}
```

在层级比较低的APIView中即可使用
```python
class ExampleView(APIView):
    authentication_classes = [SessionAuthentication, BasicAuthentication]
    permission_classes = [IsAuthenticated]
```

或者基于FBV的使用
```python
@api_view(['GET'])
@authentication_classes([SessionAuthentication, BasicAuthentication])
@permission_classes([IsAuthenticated])
def example_view(request, format=None):
    content = {
        'user': str(request.user),  # `django.contrib.auth.User` instance.
        'auth': str(request.auth),  # None
    }
    return Response(content)
```

当未经身份验证的请求被拒绝权限时，可能会返回两种不同的错误代码。
- HTTP 401 未授权
- HTTP 403 权限被拒绝

当返回401时，响应必须包含 `WWW-Authenticate`请求头，它的作用是告诉客户端如何进行身份验证。但403不应该包含 `WWW-Authenticate`请求头。

虽然一次请求里可以配置多个验证类，但最终只会按照第一个成功返回或抛出错误的类响应。例如现在配置了`[JWTAuthentication, SessionAuthentication]`，`JWTAuthentication`验证失败，如果此时直接抛出`AuthenticationFailed`并自带 `WWW-Authenticate: Bearer...`，就会**立即返回 401 + Bearer 头**，**不会继续尝试** `SessionAuthentication`，也不会把错误类型改成 403。但是如果`JWTAuthentication`返回 `None`（不抛异常），就会执行`SessionAuthentication`的校验逻辑，只要中间任一成功，就立即成功，后续类不再执行。

请注意，当请求可以成功进行身份验证，但仍被拒绝执行请求的权限时，在这种情况下，将始终使用 `403 Permission Denied` 响应，而不管身份验证方案如何。


## 权限
权限与认证一起配合，决定这个请求是放行还是拒绝。**权限检查始终在视图的前面运行**，然后才允许任何其他代码继续执行。权限检查通常会使用 `request.user` 和 `request.auth` 属性中的身份验证信息来确定是否应允许传入请求。权限用于授予或拒绝不同类别的用户对 API 不同部分的访问权限。

最简单的权限样式是允许对任何经过身份验证的用户进行访问，并拒绝对任何未经身份验证的用户进行访问。这对应于 REST 框架中的 `IsAuthenticated` 类。

稍微不那么严格的权限样式是允许对经过身份验证的用户进行完全访问，但允许对未经身份验证的用户进行只读访问。这对应于 REST 框架中的 `IsAuthenticatedOrReadOnly` 类。

在运行具体的视图函数之前，会检查权限类列表里的每个权限，如果有任何一个权限类检查失败（返回False或抛出错误），则直接拦截并返回错误信息，后面的权限类不再执行。

所有权限类的基类签名如下：
```python title:BasePermission
class BasePermission(metaclass=BasePermissionMetaclass):
    """
    A base class from which all permission classes should inherit.
    """

    def has_permission(self, request, view):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True

    def has_object_permission(self, request, view, obj):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True
```

其中最关键的就是这两个抽象方法。
视图级检查`has_permission`会在请求进入时自动调用，`has_object_permission`会在调用`get_object()`时调用，通常用于详情 retrieve / 更新 update / 删除 destroy。

注意：获取列表的场景不会调用`has_object_permission`，因为会性能爆炸，列表场景的权限应该靠`get_queryset`的过滤实现。创建场景也不会调用。

如果想要限制谁可以创建有两个常规做法，在Serialzer里做校验（字段级或者validate()），或者重写`ViewSet.perform_create()`

权限类只要是继承自`rest_framework.permissions.BasePermission`的，就可以使用位运算符进行权限组合，例如`permission_classes = [IsAuthenticated|ReadOnly]`，它支持 `&`（与），`|`（或）和`~`（非）。

## 缓存
Django 提供了一个 `method_decorator`，可以将装饰器与CBV一起使用。这可以与其他缓存装饰器一起使用，例如 `cache_page`、 `vary_on_cookie` 和 `vary_on_headers`。FBV只需要直接把装饰器打在方法上就行了。
例如：
```python

from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie, vary_on_headers

from rest_framework.response import Response
from rest_framework import viewsets


class UserViewSet(viewsets.ViewSet):
    # With cookie: cache requested url for each user for 2 hours
    @method_decorator(cache_page(60 * 60 * 2))
    @method_decorator(vary_on_cookie)
	@method_decorator(vary_on_headers("Authorization"))
    def list(self, request, format=None):
        content = {
            "user_feed": request.user.get_user_feed(),
        }
        return Response(content)

```

**注意：**`cache_page` 装饰器仅缓存 状态为 200 的 `GET` 和 `HEAD` 响应。

### cache_page
**作用**：把视图（或 URLConf）的**完整响应**缓存起来，下一次命中直接返回 HTML，**不执行视图代码**。

### vary_on_cookie
**作用**：告诉缓存系统：**“不同 Cookie 值视为不同页面”**，防止「A 用户登录后看到 B 用户缓存页面」。

### vary_on_headers
**作用**：按任意 **HTTP 请求头** 区分缓存，比 `Cookie` 更通用。

Django 的 `vary_on_cookie` / `vary_on_headers` 本质上就是在响应里加一条
```
Vary: Cookie
# 或
Vary: User-Agent, Accept-Language
```

缓存中间件 **只按 `Vary` 头里列出的字段** 生成缓存键；  
浏览器、CDN、反向代理（如 Nginx、Varnish、Cloudflare）也都遵循 **HTTP 规范**，用 **同样的 `Vary` 字段** 判断是否命中缓存。

## 限流
限流这个功能比较微妙，很多时候其实用不上限流

首先需要说明自定义基类的属性和方法，再说明官方三个限流类的作用，再看用法才能看懂

首先，官方的三个类都继承自`SimpleRateThrottle`，我们自定义限流类也基本是继承这个。

### SimpleRateThrottle
```python
class SimpleRateThrottle(BaseThrottle):
    """
    A simple cache implementation, that only requires `.get_cache_key()`
    to be overridden.

    The rate (requests / seconds) is set by a `rate` attribute on the Throttle
    class.  The attribute is a string of the form 'number_of_requests/period'.

    Period should be one of: ('s', 'sec', 'm', 'min', 'h', 'hour', 'd', 'day')

    Previous request information used for throttling is stored in the cache.
    """
    cache = default_cache
    timer = time.time
    cache_format = 'throttle_%(scope)s_%(ident)s'
    scope = None
    THROTTLE_RATES = api_settings.DEFAULT_THROTTLE_RATES

    def __init__(self):
        if not getattr(self, 'rate', None):
            self.rate = self.get_rate()
        self.num_requests, self.duration = self.parse_rate(self.rate)
```

**属性**

| 属性               | 类型         | 作用                                                                                |
| ---------------- | ---------- | --------------------------------------------------------------------------------- |
| `cache`          | Django缓存实例 | 保存请求历史时间戳的存储后端。默认=`default_cache`（即`CACHES['default']`）。想换 Redis 直接换 `CACHES` 即可。 |
| `timer`          | 可调用对象      | 取当前时间的函数。默认=`time.time`（秒级浮点）。测试时可 MonkeyPatch 成 `lambda: 1234567890` 做时间冻结。      |
| `cache_format`   | str        | 生成缓存 key 的模板。`%(scope)s` 和 `%(ident)s` 会被替换，例如 `throttle_user_42`。                |
| `scope`          | str/None   | 当前限流器的“命名空间”。**必须**在子类里设置，或显式传 `rate=`。                                           |
| `THROTTLE_RATES` | dict       | 从 `api_settings.DEFAULT_THROTTLE_RATES` 读取配置，例如 `{'upload': '10/min'}`。           |

实际上，`scope`就是用来获取这个限流类真正限制速率（`rate`）的key，所以要么直接设置`rate`，要么通过`scope`获得`rate`。


**方法**

| 名称                             | 作用                                                                                         |
| ------------------------------ | ------------------------------------------------------------------------------------------ |
| `__init__()`                   | 把 `rate` 字符串解析成 `(num_requests, duration)` 两个整数，供后面做比较。                                    |
| `get_cache_key(request, view)` | **必须**被子类覆盖；返回一个唯一字符串作为缓存键，或返回 `None` 表示“此请求不限流”。                                          |
| `get_rate()`                   | 若子类没写 `rate = '5/m'`，则按 `self.scope` 去 `THROTTLE_RATES` 里查配置。找不到就抛 `ImproperlyConfigured`。 |
| `parse_rate(rate)`             | 把 `'5/min'` 拆成 `(5, 60)`；把 `'100/d'` 拆成 `(100, 86400)`。                                    |
| `allow_request(request, view)` | **真正判断是否放行**：<br>1. 取 key → 2. 读历史 → 3. 清过期 → 4. 计数 → 5. 成功/失败。                            |
| `throttle_success()`           | 把当前时间戳插入缓存列表，并重新写回缓存（带过期时间）。                                                               |
| `throttle_failure()`           | 直接返回 `False`，给 DRF 抛 `Throttled` 异常。                                                       |
| `wait()`                       | 返回客户端需要等待的秒数（用于 `Retry-After` 响应头）。                                                        |

**配置**
```python title:setting.py
# 全局生效
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',  # 指定限流类
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',  # 指定限流类所限制的速率，其中的key来自于限流类的类属性scope
        'user': '1000/day'
    }
}

# 如
class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

```


### AnonRateThrottle
`AnonRateThrottle`只会限制未经身份验证的用户。传入请求的 IP 地址用于生成要限制的唯一密钥。

允许的请求速率由以下选项之一确定（按优先顺序）。
- 类上的 `rate` 属性，可以通过重写 `AnonRateThrottle` 并设置属性来提供。
- `DEFAULT_THROTTLE_RATES['anon']` 设置。其中`anon`是`AnonRateThrottle`默认的`scope`

`AnonRateThrottle` 适用于要限制来自未知来源的请求速率。

### UserRateThrottle
`UserRateThrottle` 会将用户限制为跨 API 的给定请求速率。用户 ID 用于生成要限制的唯一密钥。未经身份验证的请求将回退为使用传入请求的 IP 地址来生成要限制的唯一密钥。

允许的请求速率由以下选项之一确定（按优先顺序）。
- 类上的 `rate` 属性，可以通过重写 `UserRateThrottle` 并设置属性来提供。
- `DEFAULT_THROTTLE_RATES['user']` 设置。其中`user`是`UserRateThrottle`默认的`scope`

### ScopedRateThrottle
`ScopedRateThrottle`用于限制所有配置了同一个`scope`的API。仅当正在访问的视图包含 `.throttle_scope` 属性时，才会应用此限制。然后，通过将请求的`scope`与唯一的用户 ID 或 IP 地址连接起来，形成唯一的节流键。

```python
# 对视图应用限流类
class ReportView(APIView):
    throttle_classes = [UserRateThrottle]   # 只对这个视图生效
    throttle_scope   = "report"             # 与 ScopedRateThrottle 配合，同样的，也需要在DEFAULT_THROTTLE_RATES配置report: xxx/day或其他
```

### 自定义限流类
可以继承`BaseThrottle`，并实现`.allow_request(self, request, view)`方法，如果通过，则应该返回`True`，否则返回`False`。
还可以覆盖`wait()`方法，如果实现了这个方法，应该返回建议的秒数，以便于在尝试下个请求之前等待。仅当`.allow_request()`返回`False`时，才会调用`wait()`，如果实现了`wait()`且方法受到了流控，则响应中会包含`Retry-After`请求头。

## 过滤

通常来说过滤会使用`django-filter`库，它支持 REST 框架的高度可定制的字段过滤。
REST 框架的通用列表视图的默认行为是返回模型管理器的整个查询集。通常，你会希望你的 API 限制 queryset 返回的项目。
过滤继承了 `GenericAPIView` 的任何视图的 queryset 的最简单方法是覆盖 `.get_queryset()` 方法。
覆盖此方法允许您以多种不同的方式自定义视图返回的 queryset。

安装：

```shell
pip install django-filter
```

```python
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
#### 可重命名的排序字段
由于官方提供的排序过滤器支持的名称是模型字段名称，如果此时发生了连表，就会暴露中间的连表字段，比如`ordering_fields = ['user__org__org_type']`，也不方便前端传入，因此开发了一个可以支持别名的排序过滤器。

```python title:AliasedOrderingFilter
class AliasedOrderingFilter(filters.OrderingFilter):  # drf的filters.OrderingFilter
    """
    支持：
    1) ordering_fields = ['id', 'name']          # 传统
    2) ordering_fields = {'alias': 'real', ...}  # 别名
    3) ordering_fields = '__all__'               # 父类原生
    """
	def get_ordering_fields(self, view):
        return getattr(view, 'ordering_fields', self.ordering_fields)

	def get_valid_fields(self, queryset, view, context={}):
        valid_fields = self.get_ordering_fields(view)

        if valid_fields is None:
            # Default to allowing filtering on serializer fields
            return self.get_default_valid_fields(queryset, view, context)

        elif valid_fields == '__all__':
            # View explicitly allows filtering on any model field
            valid_fields = [
                (field.name, field.verbose_name) for field in queryset.model._meta.fields
            ]
            valid_fields += [
                (key, key.title().split('__'))
                for key in queryset.query.annotations
            ]
        # 新增对字典类型的支持
        elif isinstance(valid_fields, dict):
            valid_fields = [
                (real, alias) for alias, real in valid_fields
            ]
        else:
            valid_fields = [
                (item, item) if isinstance(item, str) else item
                for item in valid_fields
            ]

        return valid_fields

    def get_ordering(self, request, queryset, view):
		params = request.query_params.get(self.ordering_param)
        if not params:
            return self.get_default_ordering(view)

        # 统一成映射表
        fields = self.get_ordering_fields(view)
        alias_map = fields if isinstance(fields, dict) else {}

        ordering = []
        for field in (f.strip() for f in params.split(',')):
            descending = field.startswith('-')
            alias = field.lstrip('-')

            # 先翻译别名
            real = alias_map.get(alias, alias)
            ordering.append(('-' if descending else '') + real)

        # 让父类剔除无效字段
		return self.remove_invalid_fields(queryset, ordering, view, request)


class XXXViewSet(viewsets.ModelViewSet):
    filter_backends = [filters.DjangoFilterBackend, AliasedOrderingFilter, SearchFilter]  # 这里使用自定义的AliasedOrderingFilter

    # 在这里指定前端传入应该映射到哪个真实字段
    ordering_fields = {
        'status': 'status',  # 相同的映射也要明确写出
        'username': 'organisation__name',
        'org_name': 'organisation__billing_company_name',
        'software': 'software__name',
        'license': 'data',
        'applied_at': 'applied_at',
    }
```

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

`Meta.fields`的列表写法将为`fields`中每个字段创建**所有可用的默认过滤器**（包括 `exact`、`lt`、`gt`、`in` 等），具体取决于字段类型。字典语法将为每个字段创建一个过滤器表达式相符的过滤器。且务必要注意，列表和字段的key是**模型的字段名**，而不是显式定义的**过滤器名称**，如果错误使用了过滤器名称，将会引发`TypeError`。并且无论是列表还是字典写法，都不需要再包含显式声明的过滤器。
`Meta.fields` 的目的仅是基于模型字段声明可用的过滤器。
`Meta.fields` 是必须的，哪怕显式定义了过滤器字段也必须把这些名字全都写进去，更推荐全部使用显式声明过滤器。
`Meta.fields` 使用`fields = '__all__'`这种写法包含模型的全部字段。
`Meta.exclude`将排除声明的字段，将模型中其他字段都加入过滤器中。不可以同时指定`exclude`和`fields`。

**小坑**
如果使用filters.UUIDFilter，就只能使用精确匹配（`exact`），前端必须传入 **完整且格式正确的 UUID 字符串**。任何试图改成 `icontains`、`contains`、`iexact` 都会触发 PostgreSQL 的`function upper(uuid) does not exist`。想模糊匹配：改用 `CharFilter` 并承担性能损耗，或在前端限制用户输入完整 UUID。

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
    page_size_query_param = 'size'  # 前端传入page size的参数名称
    max_page_size = 100             # 自定义数量的上限
    
    page_query_param = 'page'       # 前端传入page的参数名称

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
