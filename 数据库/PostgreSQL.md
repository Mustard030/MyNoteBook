# 安装
使用包管理工具安装
```bash
# Debian/Ubuntu
sudo apt install postgresql postgresql-contrib postgresql-client
# docker
docker run --name postgres -e POSTGRES_PASSWORD=mysecretpassword -d -p 5432:5432 postgres

# 启动/停止服务
sudo systemctl start  postgresql   # 启动
sudo systemctl enable postgresql   # 开机自启
sudo systemctl status postgresql   # 查看状态
```

使用包管理工具安装后会自动创建一个用户`postgres`  

查看所有用户  
```bash
cat /etc/passwd
#或者
lastlog
```

切换至`postgres` 用户：`sudo -i -u postgres`

进入psql命令行：`psql`

设置密码：`\password postgres` （这是用户名，密码在输入命令后设置）

配置远程登录：
```sql
SHOW config_file;
SHOW hba_file;

-- 会显示这两个配置文件的地址
```

退出psql命令行 ：`exit`

编辑第一个文件`config_file`，找到`listen_addresses`，改为`*`

编辑第二个文件`hba_file`，找到`IPv4`，将允许连接的地址设置为`0.0.0.0/0`

重启`postgresql`服务：`systemctl restart postgresql`




psql命令行的参数：

| 参数                | 说明                             | 示例                                           |
| ----------------- | ------------------------------ | -------------------------------------------- |
| `-h HOST`         | 数据库主机地址                        | `psql -h 127.0.0.1`                          |
| `-p PORT`         | 端口号（默认 5432）                   | `psql -p 5433`                               |
| `-U USER`         | 用户名                            | `psql -U myuser`                             |
| `-d DBNAME`       | 数据库名                           | `psql -d mydb`                               |
| `-W`              | 强制提示密码（不自动取 .pgpass）           | `psql -W`                                    |
| `-w`              | 从不提示密码（无密码则失败）                 | `psql -w`                                    |
| `-c COMMAND`      | 执行单行 SQL 后退出                   | `psql -c "SELECT version();"`                |
| `-f FILE`         | 执行文件后退出                        | `psql -f script.sql`                         |
| `-l`              | 列出所有数据库后退出                     | `psql -l`                                    |
| `-v VAR=VALUE`    | 设置 psql 变量（可在 SQL 里 `:VAR` 引用） | `psql -v myvar=123`                          |
| `-t`              | 只输出数据行（无表头、无尾部计数）              | `psql -t -c "SELECT 1"`                      |
| `-A`              | 取消对齐输出（制表符分隔）                  | `psql -A -c "SELECT * FROM tbl"`             |
| `-F SEP`          | 自定义分隔符（配合 -A）                  | `psql -A -F ',' -c "SELECT * FROM tbl"`      |
| `-P VAR=VALUE`    | 设置输出格式变量（如 csv、aligned）        | `psql -P format=csv -c "SELECT * FROM tbl"`  |
| `-E`              | 显示 `\d` 等命令生成的实际 SQL           | `psql -E`                                    |
| `-q`              | 静默模式（不打印欢迎信息）                  | `psql -q`                                    |
| `-X`              | 不读取 ~/.psqlrc 启动文件             | `psql -X`                                    |
| `sslmode=require` | 强制 SSL 连接（作为 conninfo 参数）      | `psql "sslmode=require host=db.example.com"` |


| 操作            | 命令示例            |
| ------------- | --------------- |
| 列出所有数据库       | \l              |
| 查看当前连接的数据库    | \c              |
| 切换/连接到指定数据库   | \c dbname       |
| 列出当前库的所有表     | \dt             |
| 查看表结构         | \d tablename    |
| 详细表结构（含索引、约束） | \d+ tablename   |
| 查看当前使用者       | \du             |
| 执行SQL脚本       | \i 脚本路径         |
| 查看 SQL 帮助     | \h CREATE TABLE |
| 退出 psql       | \q              |

**忘记密码**
第一步：  
修改身份验证文件`pg_hba.conf`，将身份认证方式设置为`trust` （通常位于 `/var/lib/pgsql/<版本>/data/pg_hba.conf` 或者 `/etc/postgresql/<版本>/main/pg_hba.conf`）
  
打开`pg_hba.conf`，找到类似这样的行，将`md5`修改为`trust`
```txt
host    all             all             127.0.0.1/32            md5
-->
host    all             all             127.0.0.1/32            trust
```

第二步：  
重启服务
```bash
sudo systemctl restart postgresql
```

第三步：  
直接连接
```bash
psql -U postgres
```

第四步：  
执行修改密码指令（下面有）

第五步：  
改回`pg_hba.conf`的配置 + 重启服务


# SQL语句

## 数据库

```sql
-- 创建数据库
CREATE DATABASE <数据库名>;

-- 删除数据库
 DROP DATABASE <数据库名>;
```


## 用户和权限

```sql
-- 创建账户
CREATE USER <账户> WITH PASSWORD '<密码>';

-- 修改密码
ALTER USER <账户> PASSWORD '<新密码>';

-- 授权
GRANT ALL PRIVILEGES ON DATABASE <数据库名> TO <账户>;
```


### 使用.pgpass文件登录
在用户主目录下创建`.pgpass`文件，内容为
```txt
hostname:port:database:username:password 
# 如
localhost:5432:mydb:myuser:mypassword
```
并设置权限为只读
```bash
chmod 600 ~/.pgpass
```

在开发环境下还可以根据忘记密码的步骤将`local`的登录方式直接改成`trust`，但是生产就不要这样搞了


# 索引
`Postgresql`的所有索引都是二级索引，没有聚簇索引，索引中只保存主键值，叶子节点保存的是指向数据的存储位置：**堆表** 的指针。索引和数据是分离的，插入数据的时候可以直接添加到**堆表**的末尾，有效避免了随机写入带来的数据页分裂。
这种堆表+独立索引的架构使得索引更方便扩展。

## 索引类型
### GIN索引（通用倒排索引）
全称是`Generalized Inverted Index`
传统索引（B-树）是从行→值进行查找，而倒排索引是值→行的反向映射。

### GiST（通用搜索树）
全称是`Generalized Search Tree`
这不是一个具体的索引，而是一个框架，只要数据类型实现了`GiST`要求的接口，就可以利用`GiST`框架创建索引，`GiST`索引非常适合对于非线性数据，比如几何图形、地理位置、IP地址、时间范围等是否有重叠或者距离远近的查询，`GiST`索引内部也是平衡树，但是跟B树不同，节点里存的不是具体的值而是一个范围。

### 部分索引
有时候我们表里有逻辑删除字段和一个唯一字段，比如电话号码，但是由于逻辑删除的存在，相同电话号码的新数据无法插入，这时就可以使用部分索引
```sql
CREATE UNIQUE INDEX unique_phone_idx ON users (phone) WHERE (active_ind = 'Y')
```

这样删除的数据不再纳入索引的管理。部分索引还有一个优势就是它只管理满足条件的行，这样确保了索引树在物理层面上就比较小，可以加快索引速度，减少磁盘占用。


### 表达式索引
支持把列的计算结果进行索引
比如可以把用户表的邮箱先转成小写，然后对这个转成小写的结果进行索引。
```sql
CREATE INDEX users_lower ON users ((lower(email)));
SELECT * FROM users WHERE LOWER(email) = 'xxx@example.com';
```


# 数据一致性
## 事务
PG支持把DDL即表修改语句也纳入到事务中管理 。PG的默认隔离级别是RC（读已提交），只能读到其他事务提交后的数据。

## 可延迟约束
指这个约束只有在提交的时候才会对约束进行检查


# Django中使用PG
安装依赖
```bash
# 安装 psycopg2
pip install psycopg2

# 或安装 psycopg3（支持异步）
pip install "psycopg[binary]"
```

django中配置
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DATABASE_NAME'),
        'USER': os.environ.get('DATABASE_USER'),
        'PASSWORD': secrets.get("STAGING_DATABASE_PASSWORD"),
        'HOST': os.environ.get('DATABASE_HOST'),
        'PORT': os.environ.get('DATABASE_PORT', '5432'),
        'TEST': {  # 测试时新建一个数据库保证环境独立，其他参数不配置则与上面的相同
            'NAME': 'xxx_{0}'.format(
                datetime.utcnow().strftime('%Y%m%d%H%M%S%f'),
            ),
        },
    },
}
```