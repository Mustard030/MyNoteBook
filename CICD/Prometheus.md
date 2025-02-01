# 安装
## 本地安装

[下载地址](https://prometheus.io/download/)

\#1.1 下载安装包

```bash
cd /opt # 安装包放置在/opt目录下
wget {}.tar.gz  # 自行去下载地址复制
tar -zxvf {}.tar.gz  # 解压
mv prometheus-xxx-xxx-xxxx/ prometheus  # 重命名

mkdir -p /opt/prometheus/{data,config,rules} 
chmod -R 777 /opt/prometheus/data 
chmod -R 777 /opt/prometheus/config 
chmod -R 777 /opt/prometheus/rules
```




\#1.2 编写服务文件
```bash
sudo vim /etc/systemd/system/prometheus.service
```
服务文件内容：
```txt
[Unit] 
Description=Prometheus Server 
Documentation=https://prometheus.io/docs/introduction/overview/ 
After=network-online.target 
Wants=network-online.target 

[Service] 
Type=simple 
User=prometheus 
Group=prometheus 
Restart=on-failure 

# 指定 Prometheus 的启动命令和配置 
ExecStart=/opt/prometheus/prometheus \
--config.file=/opt/prometheus/prometheus.yml \
--storage.tsdb.path=/opt/prometheus/data \
--storage.tsdb.retention.time=60d \
--web.enable-lifecycle

[Install] 
WantedBy=multi-user.target
```
配置完服务文件之后重新加载
```bash
sudo systemctl daemon-reload
```
如果提示时间不对要重置时间
```bash
sudo apt-get update 
sudo apt-get install ntp 
timedatectl set-timezone Asia/Shanghai 

sudo apt-get update 
sudo apt-get install ntpdate 
# 同步时间 
ntpdate ntp1.aliyun.com
```

默认端口为`9090`



## 在docker安装

```bash
docker run --name prometheus -d \
-p 9090:9090 \
-v /etc/localtime:/etc/localtime:ro \
-v /opt/prometheus/data:/prometheus/data \
-v /opt/prometheus/config:/prometheus/config \
-v /opt/prometheus/rules:/prometheus/rules \
-v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
prom/prometheus
```



## k8s安装

在k8s集群的master节点上创建数据存储目录

```yaml
mkdir /data_prometheus
chmod 777 /data_prometheus
```

创建deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prometheus
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
  selector:
    app: prometheus
---

apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    # 加载告警规则文件
    rule_files:
      - /etc/prometheus/alerts.yml

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'node_exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: node-exporter
          - source_labels: [__meta_kubernetes_pod_ip]
            action: replace
            target_label: __address__
            replacement: ${1}:9100

  alerts.yml: |
    groups:
    - name: shutdown-alerts
      rules:
      # 1. 实例关机告警
      - alert: 实例关机
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 已关机"
          description: "实例 {{ $labels.instance }} 无法访问，可能已关机或网络断开。"

      # 2. 节点意外宕机
      - alert: 节点意外宕机
        expr: node_uname_info == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "节点 {{ $labels.instance }} 意外宕机"
          description: "节点 {{ $labels.instance }} 无法报告系统信息，可能已宕机。"

      # 3. 高 CPU 使用率
      - alert: 高CPU使用率
        expr: (100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "实例 {{ $labels.instance }} 高 CPU 使用率"
          description: "实例 {{ $labels.instance }} 的 CPU 使用率超过 80% 持续 5 分钟。"

      # 4. 高内存使用率
      - alert: 高内存使用率
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "实例 {{ $labels.instance }} 高内存使用率"
          description: "实例 {{ $labels.instance }} 的内存使用率超过 80% 持续 5 分钟。"

      # 5. 磁盘空间不足
      - alert: 磁盘空间不足
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 磁盘空间不足"
          description: "实例 {{ $labels.instance }} 的磁盘空间小于 20% 持续 10 分钟。"

      # 6. 高网络流量
      - alert: 高网络流量
        expr: rate(node_network_receive_bytes_total[5m]) > 1e8 or rate(node_network_transmit_bytes_total[5m]) > 1e8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "实例 {{ $labels.instance }} 高网络流量"
          description: "实例 {{ $labels.instance }} 的网络接收或发送流量高于 100MB/s 持续 5 分钟。"          
```

配置权限

```bash
vim prometheus-rbac.yaml
```



```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
    - nodes
    - nodes/proxy
    - services
    - endpoints
    - pods
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: prometheus
```





# 相关插件

> node_exporter: 用于监控服务器现在的状态，如CPU、内存、磁盘信息。
>
> nginx-prometheus-exporter: nginx监控
>
> rabbitmq_exporter: RabbitMQ
>
> mongodb_exporter: mongodb数据库
>
> mysql_exporter: mysql
>
> Redis_exporter: redis
>
> BlackBox_erporter: 网络协议、http(get、post)、dns、tcp、icmp、ssl证书过期时间
>
> domain_exporter: 域名过期时间
>
> cAdvisor: docker容器
>
> process_exporter: 进程





## node_exporter

该组件默认端口为`9100`

下载：

```bash
cd /opt
wget {}.tar.gz  # 自行去下载地址复制
tar -zxvf {}.tar.gz  # 解压
mv node_exporter-xxx-xxx-xxxx/ node_exporter  # 重命名
cd node_exporter
nohub ./node_exporter &  # 后台运行
```

或者使用docker启动

```bash
docker run --restart=always --name exporter -p 9100:9100 -d prom/node-exporter
```

又或者用k8s启动，配置文件如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet  #可以保证k8s集群的每个节点都运行完全一样的pod
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
     name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
# hostNetwork、hostIPC、hostPID都为True时，表示这个Pod里的所有容器，会直接使用宿主机的网络，直接与宿主机进行IPC（进程间通信）通信，可以看到宿主机里正在运行的所有进程。加入了hostNetwork:true会直接将我们的宿主机的9100端口映射出来，从而不需要创建service 在我们的宿主机上就会有一个9100的端口
      containers:
      - name: node-exporter
        image: node-exporter
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15  #这个容器运行至少需要0.15核cpu
        securityContext:
          privileged: true  #开启特权模式
        args:
        - --path.procfs  #配置挂载宿主机（node节点）的路径
        - /host/proc
        - --path.sysfs  #配置挂载宿主机（node节点）的路径
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
#通过正则表达式忽略某些文件系统挂载点的信息收集
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
#将主机/dev、/proc、/sys这些目录挂在到容器中，这是因为我们采集的很多节点数据都是通过这些文件来获取系统信息的。
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
```



然后到prometheus中修改配置文件以检测到`node_exporter`

```bash
cd /opt/prometheus
vim prometheus.yml
# 将配置项复制一份并改名为node_exporter，监听url为localhost:9100
systemctl restart prometheus
#或者
docker restart prometheus
```



## Grafana

[Grafana下载地址](https://grafana.com/grafana/download)

```bash
cd /opt
wget {}  # 自行替换
tar -zxvf {}.tar.gz  # 解压
mv grafana-xxx-xxx-xxxx/ grafana  # 重命名
sudo chown -R prometheus:prometheus /opt/grafana
```

编辑服务文件

```bash
vim /etc/systemd/system/grafana.service
```

服务文件：

```txt
[Unit]
Description=Grafana server
Documentation=http://docs.grafana.org

[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/grafana/bin/grafana-server \
  --config=/opt/grafana/conf/defaults.ini \
  --homepath=/opt/grafana

[Install]
WantedBy=multi-user.target
```

docker启动方式：

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana-enterprise
```



重新加载 

```bash
sudo systemctl daemon-reload
# 启动
systemctl start grafana
```

默认密码 admin admin 	默认端口：3000

在Grafana的管理-插件和数据-插件里面搜索并安装Prometheus插件，要填写Prometheus的地址：`http://localhost:9090`，保存并退出

点击仪表盘，创建仪表盘，导入，会有一个超链接，点击使用Grafana官方提供的仪表盘，复制ID到仪表盘中即可

选择Prometheus数据源，点击导入



## redis

假设在docker安装了redis

```bash
docker run --name my-redis -d -p 6379:6379 redis redis-server --appendonly yes --requirepass "123456"
```

安装redis监控组件模块

该组件默认端口为：`9121`

```bash
cat >docker-compose.yaml <<EOF
version: '3.3'
services:
  redis_exporter:
    image: oliver886/redis_exporter
    container_name: redis_exporter
    restart: always
    environment:
	  REDIS_ADDR: "x.x.x.x:6379"
	  EDIS_PASSWORD: 123456
	ports:
	  - "9121:9121"
EOF
```

启动docker-compose

```bash
docker-compose up -d
```

添加prometheus配置文件监控redis

```yaml
- job_name: 'redis_eporter'
  static_configs:
    - targets: ['x.x.x.x:9121']
      labels:
      	instance: redis服务器
```

重启prometheus

```bash
systemctl restart prometheus
```



# 告警

安装`alertmanager`组件，在prometheus的下载链接中能找到

```bash
cd /opt
wget {}  # 自行复制
tar -zxvf alertmanager-xx.tar.gz
mv alertmanager-xx/ alertmanager
```

或者使用docker启动alertmanager

```bash
docker run -d -p 9093:9093 \
    -v /opt/alertmanager/config:/etc/alertmanager \
    -v /opt/alertmanager/data:/data \
    prom/alertmanager
```



`alertmanager`组件的默认端口为`9093`

配置告警信息

```bash
cd /opt/alertmanager
vim alertmanager.yml
```

编辑

```yaml
global:

route:
  # 根据告警名称分组
  group_by: ['alertname']
  # 新告警来临等待10秒，合并同类告警
  group_wait: 10s
  # 同一组告警发送间隔10秒
  group_interval: 10s
  # 告警重复发送间隔10分钟
  repeat_interval: 10m
  # 指定接收者为dingtalk
  receiver: dingtalk

receiver:
- name: "dingtalk"
  webhook_configs:
  - url: "http://x.x.x.x:8011/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=钉钉机器人回调地址"
    send_resolved: true

inhibit_rules:
  # 根据需要保留抑制规则
  - source_match: 
      serverity: 'critical'
    target_match: 
      serverity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```



在Prometheus添加告警规则

在`/opt/prometheus`中创建一个`redis.yml`

复制进去

```yaml
groups:
- name: redis-alerts
  rules:
    - alert: Redis内存使用过高
      expr: |
        redis_memory_used_bytes / (redis_memory_max_bytes > 0 or vector(1)) * 100 > 85
      for: 5m
      labels:
      	serverity: critical
      annotations:
        summary: "Redis内存使用过高"
        description: "实例 {{ $labels.instance }} 的Redis内存使用率超过了85%， 当前使用率为 {{ $value }}%。"

```

再打开`prometheus.yml`

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093 # alertmanager插件的地址

rule_files:  # 配置告警规则文件
  - "redis.yml"
```

重启

```bash
systemctl restart prometheus
```