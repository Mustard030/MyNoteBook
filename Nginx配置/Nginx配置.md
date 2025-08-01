# Nginx下载和安装

## windows

 [nginx下载](https://nginx.org/en/download.html)

## Linux

自行查找对应系统安装教程

以下默认安装路径为 `/usr/local/nginx`





# 配置部分

配置文件中的默认路径是你的安装路径，因此第一步建议创建配置文件用的文件夹。尽量避免堆放在nginx.conf文件中，难以管理

```bash
cd /usr/local/nginx
mkdir cert  #ssl证书存放文件夹
cd conf
mkdir conf.d  #配置文件夹
chmod 777 conf.d
```

接下来配置主配置文件nginx.conf，直接把其中的内容替换为以下内容

nginx.conf

```nginx
# Nginx运行的用户和用户组
# user nobody;
# 工作进程：数目。根据硬件调整，通常等于CPU数量或者2倍于CPU。
worker_processes  1;

# #全局错误日志定义类型[ debug | info | notice | warn | error | crit ]
error_log  logs/error.log;

# 进程pid文件
pid        logs/nginx.pid;

events {
	# 每个工作进程的最大连接数量。
    worker_connections  1024;
}

# 设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
	# 设定mime类型,类型由mime.type文件定义
    include       mime.types;
    # 默认文件类型
    default_type  application/octet-stream;
    
	# 日志格式设置。
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
	
	# 日志文件的存放路径
    access_log  logs/access.log  main;

	# #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
    sendfile on;
    
	gzip on; # 默认off，是否开启gzip
	gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    # keepalive超时时间
    keepalive_timeout 65;

	# 包含和关联各个域名配置文件
    include conf.d/*.conf;
}
```



配置好主配置文件后就可以在conf.d里添加域名的配置文件了，建议一个域名一个文件，方便管理

例如 xxx.com.conf

```nginx
#负载均衡用的
upstream server_list{
	server localhost:8080;
	server localhost:8090;
}

#如果不需要转发到443
server {
        listen       8090;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
             root /root/dataShareService/dev;
             index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
}


# 用于把80的请求都转到443 https
server{

	# 监听端口
	listen 80;
	# 如果要同时开放某个端口的http和https配置，可以都写上，并配上default_server，下面记得有ssl相关的配置
    # listen 80 ssl default_server; #这样http和https的80都能访问了
	
	# 访问域名，多个可以用空格分隔
    server_name *.xxx.com;
    
    return 301 https://$http_host$request_uri;
	
	# 日志文件的存放路径
    access_log  logs/xxx.com.access.log;
	error_log   logs/xxx.com.error.log;
	
	error_page  403 /403.html;
	error_page  404 /404.html;
	error_page  500 502 503 504  /500.html;
	
	# 系统临时维护请打开下面这行注释，并重启nginx,维护完毕后请注释下面这行,并重启nginx,自行创建maintainace.html 
	# rewrite ^(.*)$ /maintainace.html break;
}

server {
    listen 443 ssl http2;

    server_name *.xxx.com;

    # 跨域配置
    add_header 'Access-Control-Allow-Origin' '*';
    add_header 'Access-Control-Allow-Headers' '*';
    add_header 'Access-Control-Allow-Methods' '*';
    add_header 'Access-Control-Allow-Credentials' 'true';

    # 防盗链设置
    valid_referers *.xxx.com;
    if ($invalid_referer) {
        return 404;
    }

    # ssl配置
    ssl_certificate     /usr/local/nginx/cert/x.xxx/x.xxx.com.cer;  # 还有可能是.pem文件
    ssl_certificate_key /usr/local/nginx/cert/x.xxx/x.xxx.com.key;  # 这里建议写绝对路径，保证不出错
    
    # 如果你有中间证书，可以使用 ssl_certificate_chain 指令
    ssl_certificate_chain /path/to/chain_of_certificates.pem; # 中间证书

    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 5m;
    ssl_protocols TLSV1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_prefer_server_ciphers on;
    
    access_log logs/xxx.com.access.log;
    error_log  logs/xxx.com.error.log;

	#动静分离
	location / {
		root /path/to/web; #前端部署目录
		index index.html index.htm;
	}
	
	location /images/ {
		alias /path/to/file/;
		autoindex off;
	}
    
    location /api {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
        
	    # try_files $uri $uri/ /index.html last; # 当用户刷新页面时，Nginx会先检查当前URL是否存在，如果不存在，就会尝试访问index.html，从而可以正常显示页面。
	    
        proxy_pass http://server_list;  # 转发到后端的服务器处理
    }

    # 文件白名单
    location /upload {
        # 设置允许上传的文件大小限制
        client_max_body_size 10M;
 
        # 检查文件后缀
        if ($http_upgrade ~* "multipart/form-data"){
            set $is_upload 0;
            # 设置白名单的文件后缀
            if ($request_uri ~* "\.jpg$|\.jpeg$|\.png$|\.gif$"){
                set $is_upload 1;
            }
            # 如果不在白名单内，返回403禁止访问
            if ($is_upload = 0) {
                return 403;
            }
        }
 
        # 上传文件的处理逻辑（例如传递给后端应用）
        # proxy_pass http://backend/upload;
    }
    
    location ~* /.svn/ {
         deny all;
    }

    location ~* \.(tar|gz|zip|tgz|sh)$ {
        deny all;
    }
}

```

### proxy_pass的转发规则
大体上我们可以把转发分成proxy_pass后面有`/`，无`/`，有子路径三种情况
1. 无`/`：
```nginx
location /get {
	proxy_pass http://localhost:8080;
}
```
此时访问`http://localhost:8080/get/test`，则后端接收到的路径为`/get/test`
而如果配置变成
```nginx
location /get/ {
	proxy_pass http://localhost:8080;
}
```
行为与上相同，即在proxy_pass后面没有`/`时，无论location后面有没有`/`，真正的请求地址=proxy_pass地址+原始请求转发路径
即`http://localhost:8080 + /get/test = http://localhost:8080/get/test`

2. 有`/`
```nginx
location /get {
	proxy_pass http://localhost:8080/;
}
```
这种情况直接报错404
如果在location加上后`/`，
```nginx
location /get/ {
	proxy_pass http://localhost:8080/;
}
```
访问`http://localhost:8080/get/test`，则后端接收到的路径为`/test`
真正的请求地址=proxy_pass地址+原始请求转发路径-location地址
即在第一种情况中真正的请求地址为`http://localhost:8080 + /get/test - /get = http:localhost:8080//test`
而第二种则是`http://localhost:8080 + /get/test - /get/ = http:localhost:8080/test`

3. 后面有子路径
```nginx
location /get {
	proxy_pass http://localhost:8080/abc;
}
```
访问`http://localhost:8080/get/test`，则后端接收到的路径为`/abc/test`
如果get后面也加上`/`
```nginx
location /get/ {
	proxy_pass http://localhost:8080/abc;
}
```
访问`http://localhost:8080/get/test`，则后端接收到的路径为`/abctest`
即它的行为与有`/`完全相同，真正的请求地址=proxy_pass地址+原始请求转发路径-location地址


| proxy_pass格式        | location规则     | 示例请求      | 代理结果                    | 注意事项          |
| ------------------- | -------------- | --------- | ----------------------- | ------------- |
| http://backend      | location /get  | /get/test | http://backend/get/test | 完整保留原始路径      |
| http://backend/     | location /get/ | /get/test | http://backend/test     | 需确保location带/ |
| http://backend/abc  | location /get  | /get/test | http://backend/abc/test | 可能产生非预期路径     |
| http://backend/abc/ | location /get/ | /get/test | http://backend/abc/test | 推荐的安全写法       |


## 其他配置项

**负载均衡配置状态**

| 状态           | 概述                                                       |
| ------------ | -------------------------------------------------------- |
| down         | 当前的server暂不参与负载均衡                                        |
| backup       | 预留的备份服务器，当其他服务器都挂掉时启用                                    |
| max_fails    | 允许请求失败次数，如果请求失败次数超过限制，则经过fail_timeout时间后从虚拟服务池中kill掉该服务器 |
| fail_timeout | 经过max_fails失败后，服务器暂停时间，max_fails设置后必须设置此值                |
| max_conns    | 限制最大的连接数，用于服务器硬件配置不同的情况下                                 |

```nginx
upstream node {
	server IP地址:8081 down;
	server IP地址:8082 backup;
	server IP地址:8081 max_fails=1 fail_timeout=10s;
}
```



**负载均衡调度策略**

| 调度算法       | 概述                                    |
| ---------- | ------------------------------------- |
| 轮询         | 逐一轮询，默认方式                             |
| weight     | 加权轮询，weight越大，分配几率越高                  |
| ip_hash    | 按照访问IP的hash结果分配，会导致来自同一IP的请求固定访问一台服务器 |
| url_hash   | 按照访问URL的hash结果分配                      |
| least_conn | 最少链接数，哪个服务器链接数少分配给谁                   |
| hash关键数值   | hash自定义的key                           |

```nginx
upstream node {
	server IP地址:8081 weight=10;
	server IP地址:8082 weight=20;
	server IP地址:8083 weight=30;
}
```

```nginx
upstream node {
	hash $request_uri;
	server IP地址:8081;
	server IP地址:8082;
	server IP地址:8083;
}
```

```nginx
upstream node {
	ip_hash;
	server IP地址:8081;
	server IP地址:8082;
	server IP地址:8083;
}
```

```nginx
upstream node {
	least_conn;
	server IP地址:8081;
	server IP地址:8082;
	server IP地址:8083;
}
```


**location匹配**

```nginx
location [ = | ~ | ~* | ^~ ] uri{}
```

- `=`：用于不含正则表达式的uri前，要求请求字符串与uri严格匹配，匹配成功就停止向下搜索并立即处理该请求
- `~`：用于表示uri包含正则表达式，并且区分大小写
- `~*`：用于表示uri包含正则表达式，并且**不**区分大小写
- `^~`：用于不含正则表达式的uri前，要求Nginx服务器找到标识uri和请求，字符串匹配度最高的location后，立即使用此location处理请求，而不再使用location块中的正则uri和请求字符串做匹配

loation块中，alias和root的区别是
- alias，将请求的uri替换为`alias`指定的路径。例如：请求为`/images/logo.png`，则nginx会查找`/path/to/file/logo.png`。
- root，将请求的uri直接附加到root指定的路径后面，还是那个请求，nginx会查找`/path/to/web/images/logo.png`
需要注意的是，如果使用alias的时候，location的路径是以`/`结尾的话，alias配置的路径也必须以`/`结尾，否则会路径错误