# 安装
使用包管理工具安装
```bash
# Debian/Ubuntu
sudo apt install postgresql postgresql-client

# 启动/停止服务
sudo systemctl start  postgresql   # 启动
sudo systemctl enable postgresql   # 开机自启
sudo systemctl status postgresql   # 查看状态
```

使用包管理工具安装后会自动创建一个用户`postgres`
进入psql命令行
```bash
sudo -u postgres psql               # 本地超级用户登录
psql -h 127.0.0.1 -U <user name> -d <dbname> -W   # 远程/指定用户
```


| 操作            | 命令示例            |
| ------------- | --------------- |
| 列出所有数据库       | \l              |
| 切换/连接数据库      | \c dbname       |
| 列出当前库的所有表     | \dt             |
| 查看表结构         | \d tablename    |
| 详细表结构（含索引、约束） | \d+ tablename   |
| 查看 SQL 帮助     | \h CREATE TABLE |
| 退出 psql       | \q              |
