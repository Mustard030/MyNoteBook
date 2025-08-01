# kubectl
## 命令格式
```
kubectl [command] [type] [name] [flags]
```
- command：指定要对资源操作的类型，例如create、get、delete等
- type：指定资源类型，比如deployment、pod、service，可以指定单数、复数、缩写形式
- name：指定资源的名称，大小写敏感
- flags：指定额外的可选参数

[命令文档](http://docs.kubernetes.org.cn/683.html)
## 容器管理
```bash
# logs 查看pod的容器日志，如果pod中仅有一个容器，可省略容器名字
kubectl logs -f pod名 [-c container名]

# 进入容器
kubectl exec -it pod名 [-c container名] -- /bin/bash

# cp用于pod和外部文件交换，该命令依赖于容器镜像内的tar命令
yum update && yum install tar -y
tar --version
# pod中容器的文件复制到本地(tar默认相对路径而不是绝对路径，自动创建不存在的目录)
kubectl cp pod名:source local [-c container名]
# 本地的文件复制到pod中容器
交换pod名:source和local的位置即可
```
其中source不支持绝对路径，只允许使用相对路径，并且当前目录即工作目录

# K8S的几大部件

## Namespace
作用就是资源隔离

创建命名空间
```bash
kubectl create namespace xxx
```
或者写一个`namespace-xxx.yaml`
```txt
appVersion: v1
kind: Namespace
metadata:
	name: xxx
```
执行`kubectl apply -f namespace-xxx.yaml`

删除命名空间
```bash
kubectl delete namespace xxx
#或
kubectl delete -f namespace-xxx.yaml
```

## ResourceQuota和LimitRange
- **ResourceQuota​**​ 是集群资源的“总量阀门”，防止命名空间过度占用资源。
- ​**​LimitRange​**​ 是容器资源的“个体规范”，确保每个容器合理使用资源。

**ResourceQuota（资源配额）​**​
- ​**​作用范围​**​：命名空间（Namespace）级别。
- ​**​核心功能​**​：限制整个命名空间内资源的总使用量，包括：
    - 计算资源（如 CPU、内存的 `requests` 和 `limits` 总和）。
    - 存储资源（如存储卷大小）。
    - 对象数量（如 Pod、Service、ConfigMap 的数量）。
- ​**​目的​**​：防止单个命名空间消耗过多资源，确保集群资源的公平分配。
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example-quota
  namespace: my-namespace
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
    services: "5"
```

**LimitRange（资源限制范围）​**​
- ​**​作用范围​**​：命名空间内的单个 Pod 或容器。
- ​**​核心功能​**​：
    - 为容器或 Pod 设置默认的 `requests` 和 `limits`（如果用户未显式指定）。
    - 约束单个容器或 Pod 的资源使用范围（如最小/最大 CPU、内存）。
- ​**​目的​**​：避免资源浪费或争抢，确保每个容器都有合理的资源限制。
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "0.1"
      memory: "100Mi"
    default:
      cpu: "0.5"
      memory: "200Mi"
    max:
      cpu: "1"
      memory: "1Gi"
    min:
      cpu: "0.05"
      memory: "50Mi"
```


1. ​**​LimitRange 先应用​**​：
    - 当创建 Pod 时，LimitRange 自动为容器注入默认的 `requests/limits`，并校验是否超出单个容器的资源范围（如 `max`/`min`），如果小于最小资源要求，也不允许启动。
2. ​**​ResourceQuota 后校验​**​：
    - 检查命名空间的资源使用总量是否在配额范围内（如总 CPU、内存、Pod 数量等）。
    - 如果 Pod 的资源需求导致命名空间超出配额，则创建请求会被拒绝。

创建资源时要加上
```yaml
spec:
	containers:
		- name: 容器名
		  image: 镜像
		  # 当前容器的资源配额
		  resources:
		    requests:
		      cpu: "200m"
		      memory: "200Mi"
		    limits:
		      cpu: "400m"
		      memory: "400Mi"
```

**注意：**
LimitRange的限制优先级低于ResourceQuota限制，如出现冲突，以ResourceQuota为准。
绑定资源配额之前创建的资源不受限制。
如果不配置Limit，则默认与Quote保持一致。
当绑定了资源配额之后，创建的资源就必须写上申请的资源量，否则直接报Forbidden。
## Pod
Pod是k8s架构中最基础的部件，是可管理的最小计算单元，Pod是一组（一个或多个）容器，这些容器共享存储，网络，以及怎样运行这些容器的声明。

> 这些命令其实是通用的，比如把pod改成deploy就是对应资源的操作命令
> 最常用的命令是 kubectl apply -f 文件名

创建pod
```bash
kubectl run mynginx --image=nginx:1.14.2 -n xxxx

mynginx是pod名字
--image 指定镜像
-n 指定命名空间
```
获取pod基本信息
```bash
kubectl get pod [-o wide] [-n 命名空间 | -A] #owide看详情 -n指定命名空间，-A显示所有命名空间下的
```
查看资源详情
```bash
kubectl describe xxx(pod的名字) -n 命名空间
```
看日志
```bash
kubectl logs <pod-name>
```
删除pod
```bash
kubectl delete pod <pod-name>
```
进容器内部操作
```bash
kubectl exec -it <pod-name> -c <img-name: 如tomcat> --sh

exit #退出
```

**自定义命令**
某些容器没有指定Entrypoint，导致没有一个可持续运行的前台程序，这样的容器启动后就会立即退出
command用于指定进入容器后执行的自定义命令，可代替默认的容器命令。
```yaml
spec:
	containers:
		- name: 容器名
		  image: 容器镜像
		  # 自定义运行的命令
		  command: ["/bin/sh", "-c", "while true; do sleep 1; done;"]
```


## 容器探针
当pod处于运行态时，可使用**探针**(Probe)对容器的健康状态进行检查。
### 探针类型

| 探针类型                     | 探测作用         | 探测次数          | 探测失败后的措施                 |
| ------------------------ | ------------ | ------------- | ------------------------ |
| livenessProbe(存活探针或活性探针) | 容器是否运行正常     | 伴随容器的生命周期定期探测 | 据restartPolicy的策略执行重启    |
| readinessProbe(就绪探针)     | 容器是否进入就绪状态   | 伴随容器的生命周期定期探测 | 把Pod从Service的Endpoint中删除 |
| startupProbe(启动探针)       | 容器内的应用是否成功启动 | 仅在容器启动时探测一次   | 据restartPolicy的策略执行重启    |
注意1：Pod启动成功后(活性探针成功)，若Pod中的服务进程还未初始化完成，则来自Service的外部流量就会请求失败。就绪探针就是解决这个问题的
注意2：startupProbe探测成功之前，其他探针被禁用。

每种探针的探测结果包含三种：
- Success：探测成功
- Failure：探测失败
- Unknown：无法探测
- 注意：若容器不提供任何探针，则三种探针默认结果都是成功
三种探针的使用场景：
- 存活探针：如果容器遇到问题时无法自行崩溃，例如：容器内部应用发生死锁，而容器的状态仍然为正常，这时无法提供服务，可以用存活探针并指定restartPolicy不是Never
- 就绪探针：如果希望仅在探测成功后，才向Pod发送流量，可以使用就绪探针，否则，Pod将在启动阶段就开始接收流量。
- 启动探针：如果容器中的应用初始化时间较长，为避免存活探针将容器错误的kill掉，可以使用启动探针。

### 探测方式
- exec：通过在容器内执行的linux命令来进行探测，如果命令返回码为0，则认为探测成功，返回码非0则认为探测失败。`echo $?`  *\*$?为上一条命令的返回值*
- httpGet：通过向容器的指定端口和路径发起HTTP GET请求，如果HTTP返回码为>=200且<400，则认为探测成功，返回码为4xx，5xx则探测失败
- tcpSocket：通过向容器中的指定端口发送tcp三次握手连接，如果端口正确且tcp连接成功则探测成功，连接失败则探测失败

#### exec探测
```
apiVersion: v1
kind: Pod
```


## Deployment
Deployment负责创建和更新应用程序的实例，在生产环境中，基本不会直接对pod进行操作，都是再封一层Deployment，使其拥有多副本、自愈、扩缩容的能力。创建Deployment后，Master节点将应用程序实例调度到集群的各个节点上，如果托管的实例节点被关闭或删除，Deployment控制器会将该实例替换为集群另一个节点上的实例。

```bash
#创建
kubectl create deployment <dep-name> --image=xxx:version [--replicas=3]

#查看deployment信息
kubectl get deployment

#删除deployment
kubectl delete deployment <dep-name>

#修改
kubectl edit deploy <dep-name>
```
### 滚动升级与回滚
```bash
#以tomcat为例
kubectl set image deployment <dep-name> tomcat=tomcat:10.1.11 --record

#回滚
#查看历史回滚版本
kubectl rollout history deploy <dep-name>
#回滚命令
kubectl rollout undo deployment/my-dep [--to-revision=2] #不加默认到上一个，加了可以指定到某一个版本
```


## Service
将一组pod暴露给外部访问，但Service一般是东西方向流量沟通用的，比如会员服务与购物服务沟通，要暴露给真正的公网访问即南北方向流量，一般用Ingress
Service Spec标记type的方式暴露，type类型如下：
- ClusterIP（默认）：在集群的内部IP上公开Service，这种类型使得Service只能从集群内访问
- NodePort：使用NAT在集群中每个选定Node的相同端口上公开Service，使用\<NodeIP\>:\<NodePort\>从集群外部访问Service，是ClusterIP的超集
- LoadBalancer：在当前云中创建一个外部负载均衡器（如果支持），并为Service分配一个固定的外部IP，是NodePort的超集
- ExternalName：通过返回带有该名称的CNAME记录，使用任意名称（由spec中的externalName指定）公开Service，不使用代理

```bash
kubectl expose deployment <dep-name> --name=<svc-name> --port=8080 --type=NodePort

#查看Service信息，port信息里冒号后面的端口号就是对集群外暴露的访问接口
kubectl get svc
```
```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: my-tomcat  
  namespace: your-ns  
  labels:  
    app: my-tomcat
spec:  
  type: NodePort  
  ports:  
    - port: 9000  #srvice的虚拟ip对应的端口，在集群内网机器可以访问用Service的虚拟ip加该端口号访问服务
      targetPort: 9000  #pod暴露的端口，一般与pod内部容器暴露的端口一致
      protocol: TCP  
      name: http  
      nodePort: 30777  #Service在宿主机上映射的外网访问的端口，端口范围必须在30000-32767之间
  selector:  
    app: my-tomcat  
```


## Volume
存储卷，包含可被pod中容器访问的数据目录，容器中的文件在磁盘中是临时存放的，当容器崩溃时文件会丢失，同时无法在多个pod上共享文件，使用Volume解决这个问题
常用的卷类型有：**configMap**、**emptyDir**、**local**、**nfs**、**secret**等

- ConfigMap：可以将配置文件以键值对的形式保存到ConfigMap中，并且可以在Pod中以文件或者环境变量的形式使用。ConfigMap可以用来存储不敏感的配置信息，如应用程序的配置文件。
- EmptyDir：是一个空目录，可以在Pod中用来存储临时数据，当Pod被删除时一并删除
- Local：将本地文件系统的目录或文件映射到Pod中的一个Volume中，可以用来在Pod中共享文件或数据。
- NFS：将网络上的一个或多个NFS共享目录挂载到Pod的Volume中，可以用来在多个Pod中共享数据
- Secret：将敏感信息以密文的形式保存到Secret中，并且可以在Pod中以文件或环境变量的形式使用。Secret可以用来保存敏感信息，如用户名密码、证书等。

```yaml
spec:
	volumes: #存储卷
	- name: 存储卷的名字
	  hostPath: #存储卷的类型。hostPath类似Docker的绑定挂载
		  path: 节点路径(物理机内路径)
		  type: 配置hostPath卷的字段。如：Directory、File、DirectoryOrCreate、FileOrCreate
	containers:
	- name: 容器名
	  image: 容器镜像
	  volumeMounts: #使用存储卷
		  - name: 引用volumes.name
		    mountPath: 容器路径
```

### 使用方式
在`.spec.volumes`字段中设置为Pod提供的卷，并在`.spec.containers[*].volumeMounts`字段中声明卷在容器中挂载的位置。容器中的进程看到的文件系统视图是由他们的容器镜像的初始内容以及挂载在容器中的卷（如果定义了）所组成。其中根文件系统间容器镜像的内容相吻合。任何在该文件系统下的写入操作，如果被允许的话，都会影响接下来容器中进程访问文件系统时所看到的内容。

#### 搭建nfs文件系统

```shell
sudo yum install nfs-utils -y
```

启动NFS服务,并设置NFS服务开机自动启动

```shell
sudo systemctl start rpcbind 
sudo systemctl enable rpcbind 
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```

创建共享目录并修改权限（以`/var/nfsshare`为例）：/var/nfsshare是你存放数据的目录

```shell
sudo mkdir /var/nfsshare
sudo chown nfsnobody:nfsnobody /var/nfsshare
sudo chmod 755 /var/nfsshare
```

配置NFS导出的配置文件（编辑`/etc/exports`文件）

```shell
sudo vi /etc/exports
```

添加

```text
/var/nfsshare  <kubernetes-worker-node-ip>(rw,sync,no_root_squash,no_all_squash)
```

\<kubernetes-worker-node-ip\> 是你的k8s节点ip，例如： 代表哪些k8s工作节点有权限访问nfs的
查询k8s节点ip，命令：

```shell
kubectl get nodes -owide
```

拿到INTERNAL-IP列所有IP
例如：

```text
/var/nfsshare  172.16.177.23(rw,sync,no_root_squash,no_all_squash)
/var/nfsshare  172.16.177.24(rw,sync,no_root_squash,no_all_squash)
```

导出共享目录并重启NFS服务

```shell
sudo exportfs -rav
sudo systemctl restart nfs-server
```

==或者更直接的，使用下面这个脚本==

```shell
sudo vim nfs_setup.sh
```

粘贴下面的命令

```bash
#!/bin/bash

# 定义共享目录的变量
NFS_SHARE_DIR=$1
if [ -z "$NFS_SHARE_DIR" ]; then
  echo "Usage: $0 <NFS_SHARE_DIR>"
  exit 1
fi

# 确保 nfsnobody 用户和组存在
if ! id "nfsnobody" &>/dev/null; then
  echo "User nfsnobody does not exist. Creating user and group..."
  sudo groupadd nfsnobody
  sudo useradd -g nfsnobody nfsnobody
fi

# 安装 nfs-utils
echo "Installing nfs-utils..."
sudo yum install nfs-utils -y
# 设置开机启动
sudo systemctl start rpcbind 
sudo systemctl enable rpcbind 
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
echo "rpcbind nfs-server have started"

# 创建共享目录
sudo mkdir -p $NFS_SHARE_DIR

# 修改目录的权限
sudo chown nfsnobody:nfsnobody $NFS_SHARE_DIR
sudo chmod 755 $NFS_SHARE_DIR

# 获取 INTERNAL-IP 地址并生成 exports 配置
for ip in $(kubectl get nodes -o=jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'); do
  echo "$NFS_SHARE_DIR  $ip(rw,sync,no_root_squash,no_all_squash)" | sudo tee -a /etc/exports
done
# 将所有的IP都加入可访问地址还可以使用这条命令
# echo "<你的共享目录地址> *(insecure,rw,sync,no_root_squash)" > /etc/exports
# 如果你不想把所有的节点IP都加入到nfs的可访问地址中，可以自己添加条目到/etc/exports
#<你的共享目录地址如/var/nfsshare>  x.x.x.x(rw,sync,no_root_squash,no_all_squash)

# 重新导出共享目录
sudo exportfs -rav

# 重启 NFS 服务
sudo systemctl restart nfs-server

echo "NFS server setup completed with directory $NFS_SHARE_DIR"
```

赋予脚本执行权限

```shell
chmod +x nfs_setup.sh
```

运行并指定共享目录

```shell
./nfs_setup.sh /var/nfsshare
```



创建持久化卷声明（PVC）和持久化卷（PV）资源以使用NFS共享。以下是一个简单的PV和PVC的YAML

```yaml
# PersistentVolume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
 name: nfs-pv
spec:
 capacity:
   storage: 1Gi
 accessModes:
   - ReadWriteMany
 nfs:
   server: <nfs-server-ip>
   path: /var/nfsshare   #你创建的nfs目录

---
# PersistentVolumeClaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: nfs-pvc
spec:
 accessModes:
 - ReadWriteMany
 resources:
   requests:
     storage: 1Gi
```

将`<nfs-server-ip>`替换为NFS服务器的IP地址。

使用nfs配置示例：

```yaml
apiVersion: v1
kind: Deployment
metadata:
	labels:
		app: nginx-pv-demo
	name: nginx-pv-demo
spec:
	replicas: 2
	selector:
		matchLabels:
			app: nginx-pv-demo
	template:
		metadata:
			labels:
				app: nginx-pv-demo
		spec:
			containers:
			- name: nginx
			  image: nginx
			  volumeMounts:
				  - name: html
					mountPath: /usr/share/nginx/html #容器内部
			volumes:
				- name: html
				  nfs:
					  server: x.x.x.x(masterIPAddress)
					  path: /nfs/data/nginx-pv #nfs服务器中的地址必须先存在，否则pod会创建失败
```
缺陷：

1. nfs服务器中的地址必须先存在，否则pod会创建失败
2. 必须知道master的地址



#### PV&PVC

主要在于声明一个pv，使用时为pvc，实现解耦，不需要知道master的ip，也不需要创建文件夹，直接使用声明的pv即可。但是这两种方式都是静态的，即管理员需要提前分配好以上的区域，才可以使用。缺点是资源不能充分利用。



#### 动态供应

即如果没有满足pvc条件的pv，会动态创建pv。相比静态供应，动态供应有明显的优势：不需要提前创建pv，减少了管理员的工作量，效率高。动态供应是通过StorageClass实现的，StorageClass定义了如何创建pv，但需要注意的是每个StorageClass都有一个制备器（Provisioner）用来决定使用哪个卷插件制备pv，该字段必须指定才能实现动态创建

配置动态供应的默认存储类

```yaml
# 创建了一个存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
      storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
    archiveOnDelete: "true" #删除pv的时候，pv内容是否需要备份

---
apiVersion: apps/v1
kind: Deployment
metadata:
    name: nfs-client-provisioner
    labels:
        app: nfs-client-provisioner
    namespace: default
spec:
    replicas: 1
    strategy:
        type: Recreate
    selector:
        matchLabels:
            app: nfs-client-provisioner
    template:
        metadata:
            labels:
                app: nfs-client-provisioner
        spec:
            serviceAccountName: nfs-client-provisioner
            containers:
                - name: nfs-client-provisioner
                  image: registry.cn-hangzhou.aliyuncs.com/k8s_images/nfs-subdir-external-provisioner:v4.0.2
                  volumeMounts:
                      - name: nfs-client-root
                        mountPath: /persistentvolumes
                  env:
                      - name: PROVISIONER_NAME
                        value: k8s-sigs.io/nfs-subdir-external-provisioner
                      - name: NFS_SERVER
                        value: 192.168.1.100 #指定自己nfs服务器地址
                      - name: NFS_PATH
                        value: /nfs/data #指定nfs服务器上的共享目录

            volumes:
                - name: nfs-client-root
                  nfs:
                      server: 192.168.1.100 #指定自己nfs服务器地址
                      path: /nfs/data
```



## ConfigMap

是一种存储非敏感信息的Kubernetes对象，用于存储配置数据如键值对，整个配置文件，或者JSON数据等，通常用于容器镜像中的配置文件，命令行参数和环境变量等。ConfigMap通常用于容器镜像中的配置文件、命令行、环境变量等

1. 环境变量注入：将配置数据注入到Pod的容器环境变量中
2. 配置文件注入：将配置数据注入到Pod的容器文件系统中，容器可以读取这些文件
3. 命令行参数注入：将配置数据注入到容器的命令行参数中

优点：

1. 避免硬编码，将配置数据与应用代码分离（复杂配置和实时更新最好还是Nacos）
2. 便于维护和更新，可以单独修改ConfigMap而不需要重新构建镜像
3. 可以通过多种方式注入配置参数，更加灵活
4. 可以通过Kubernetes的自动化机制对ConfigMap进行版本控制和回滚
5. ConfigMap可以被多个Pod共享，减少配置数据的重复存储

操作：

```yaml
#查看configmap (cm是缩写，只打cm就行)
kubectl get configmap/cm

#查看详情
kubectl describe configmap/cm my-config

#删除cm
kubectl delete cm my-config
```

通过配置文件创建：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: my-config
data:
    key1: value1
    key2: value2

---
apiVersion: v1
kind: ConfigMap
metadata:
    name: my-config
data:
    application.yml: |
        name: my-app
        spring:
            datasource:
                ...
        
```

通过文件创建：

```bash
echo -n admin > ./username
echo -n 123456 > ./password
kubectl create cm myconfigmap --from-file=./username --from-file=./password
```

使用范例：

```bash
#docker安装redis
docker run -v /data/redis.conf:/etc/redis/redis.conf \
-v /data/redis/data:/data \
-d --name myredis \
-p 6379:6379 \
redis:latest redis-server /etc/redis/redis.conf
```

创建configmap：

```bash
#创建redis.conf
daemonize yes
requirepass root

#创建配置，redis保存到k8s的etcd
kubectl create cm redis-conf --from-file=redis.conf

#查看资源清单
kubectl get cm redis-conf -oyaml
```



## Ingress

> Ingress is frozen. New features are being added to the [Gateway API](https://kubernetes.io/zh-cn/docs/concepts/services-networking/gateway/).

[Ingress](https://kubernetes.github.io/ingress-nginx/)是一种Kubernetes资源类型，它允许在Kubernetes集群中暴露HTTP和HTTPS服务，通过Ingress可以将流量路由到不同的服务和端点，无需使用不同的负载均衡，Ingress通常使用Ingress Controller实现，他是一个运行在Kubernetes集群中的负载均衡器，他根据Ingress规则配置路由并将流量转发到对应服务。

客户端 ----Ingress管理的负载均衡器----> Ingress ----路由规则----> Service ----> pod

所有的Service使用ClusterIP，使其只能内部访问

```yaml
apiVersion: v1
kind: Ingress
metadata:
    name: web-ingress
    annotations:
        nginx.ingress.kuberneters.io/limit-rps: "1" #限制每个请求的RPS为1
spec:
    rules:
        - host: tomcat.example.com  #转发域名
          http:
              paths:
                  - pathType: Prefix
                    path: /
                    backend:
                        service:
                            name: tomcat-svc #tomcat的Service名字
                            port:
                                number: 8080 #Service的端口
```





# 安装教程（建议看网上的）

## 1. 安装: 1.20.9版本
[参考文档](https://blog.csdn.net/Enchanter06/article/details/131326507)
### 1.1 安装docker
参考Docker笔记
### 1.2 关闭防火墙
```bash
#关闭防火墙且开机不启动
systemctl stop firewalld && systemctl disable firewalld

#关闭selinux，将SElinux设置为permissive模式（相当于禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#关闭swap
swapoff -a  #如果有其他软件影响，可能会失败，可以先systemctl reboot再执行这个
sed -ri 's/.*swap.*/#&/' /etc/fstab

systemctl reboot #重启生效
free -m #查看swap交换区是否为0，都为0则成功

#允许IP层数据包转发
cat >> /etc/sysctl.conf <<-'EOF'
net.ipv4.ip_forward=1
EOF
sysctl -p


#分别给机器加上主机名
hostnamectl set-hostname xxx #如k8s-master

#添加hosts，
cat >> /etc/hosts << EOF
x.x.x.x k8s-master
x.x.x.x k8s-node1
x.x.x.x k8s-node2
EOF

#允许iptables检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

sudo sysctl --system

#设置时间同步
systemctl status chronyd
chronyc tracking
# 如果没安装chrony
yum -y install chrony
systemctl start chronyd
systemctl enable chronyd

#安装ipset工具
yum install ipset ipvsadm -y
cat << EOF > /etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack
```

### 1.3 安装kubelet、kubeadm、kubectl
```bash
#配置k8s的yum源地址
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#如果之前安装过k8s，先卸载旧的
yum remove -y kubelet kubeadm kubectl

#查看安装的版本
yum list kubelet --showduplicates | sort -r

#安装 kubelet，kubeadm，kubectl指定版本，我们使用kubeadm方式安装k8s集群
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes

#配置kubelet
vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd

#开机启动kubelet
sudo systemctl enable kubelet --now
```

### 1.4 初始化master节点
#### 1.4.1 下载各机器需要的镜像
```bash
sudo tee ./images.sh <<-'EOF
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]}; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF

chmod +x ./images.sh && ./images.sh
```
或者使用这种方式
```bash
kubeadm config images pull --images-repository registry.aliyuncs.com/google_containers --cri-socket=unix:///var/run/cri-dockerd.sock
```
#### 1.4.2 初始化主节点
```bash
#在k8s-master机器上执行初始化操作(里面的第一个ip地址就是k8s-master机器的ip，改成你自己机器的，第二个改成自己的主机名，后面两个ip网段不用动)#所有网络范围不重叠
kubeadm init \
--apiserver-advertise-address=192.168.31.22 \
--control-plane-endpoint=master \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.244.0.0/16

#可以查看kubelet日志
journalctl -xefu kubelet

#安装完成后他会提示你要执行两条命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf

#如果初始化失败，重置kubeadm
kubeadm reset
rm -rf /etc/cni/net.d $HOME/.kube
sysctl -w net.ipv4.ip_forward=1
#清理 iptables 规则
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```
#### 1.4.3 安装Calico网络插件
```bash
curl https://docs.projectcalico.org/archive/v3.20/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```
到这一步时有可能会出现calico镜像下载失败的问题，原因在于yaml中规定了
下载源为官方镜像，解决方法如下：
1. 在官方的[Github Releases](https://github.com/projectcalico/calico/releases/tag/v3.20.6)中下载，再传到服务器中
2. 进到文件所在目录执行以下脚本
```bash
# !/bin/bash

# 解压
tar -zxvf release-v3.20.6.tgz

cd release-v3.20.6
cd images

# 循环加载 image
sudo tee load-images.sh <<-'EOF'
#!/bin/bash
tars=(
calico-kube-controllers.tar
calico-node.tar
calico-typha.tar
calico-cni.tar
calico-pod2daemon-flexvol.tar
)
for tar in ${tars[@]} ; do
docker load < $tar
done
EOF

chmod +x load-images.sh && ./load-images.sh

# 替换 docker.io 前缀
cp calico.yaml calico.cp.yaml
sed -i 's/image: docker.io\//image: /g' calico.yaml

# 如果你的 VPC 网段和 pod-network 不冲突，那么可以解开下面注释
# kubectl apply -f ../k8s-manifests/calico.yaml
```

#### 1.4.4 加入node节点
```bash
#重新生成node令牌，用刚装好时弹出来那个也可以，最好用这个新生成的
kubeadm token create --print-join-command

#在node节点执行生成出来的那个命令
```
## 2. 安装1.27.1版本
CentOS安装高版本时，经常会出问题，可能是内核版本导致的
```bash
uname -a #查看内核版本
#大部分情况下会是3.10.0版本，虽然满足了Docker的要求，但是不满足k8s 1.24以上的要求了

yum list kernel --showduplicates
#如果list中有需要版本的内核可以直接update升级，但大多数情况是没有的

#导入ELRepo仓库公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

#Centos7系统安装ELRepo
yum install https://www.elrepo.org/elrepo-release-7.e17.elrepo.noarch.rpm
#Centos8系统安装ELRepo
yum install https://www.elrepo.org/elrepo-release-8.e18.elrepo.noarch.rpm

#查看ELRepo提供的内核版本
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

#安装主线内核(32位安装kernek-m1)
yum --enablerepo=elrepo-kernel install kernel-ml.x86_64

grub2-mkconfig -o /boot/grub2/grub.cfg
sudo awk -F\' '$1=="menuentry " {print i++" : " $2}' /etc/grub2.cfg

#指定开机启动内核版本
cp /etc/default/grub /etc/default/grub-bak #备份
grub2-set-default 0 #设置默认内核版本，选刚刚下载那个6.4.3(可能)的

#更新软件包并重启
yum makecache
reboot

#验证内核
uname -a
```

其他的步骤基本和上述一致，主要是在高版本下要使用docker容器，还要额外安装一个东西cri-dockerd (所有节点)
```bash
#https://pithub.com/Mirantis/cri-dockerd/releases
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1-3.el7.x86_64.rpm
rpm ivh cri-dockerd-0.3.1-3.el7.x86_64.rpm
#修改/usr/lib/systemd/system/cri-docker.service文件中的ExecStart配置
vim /usr/lib/systemd/system/cri-docker.service

Execstart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9 --container-runtime-endpoint fd://

systemctl daemon-reload
systemctl start cri-docker && systemctl enable --now cri-docker
```


## 便捷操作
### ✨bash-completion
安装插件`bash-completion`
```bash
# centos
sudo yum install bash-completion -y
# ubuntu
sudo apt install bash-completion -y

# 写入bashrc文件
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```

### kube-shell
提供一个交互式的shell环境，用户可以直接在其中使用kubectl命令，它依赖于python和pip
```bash
# centos
sudo yum install python3-pip -y
# ubuntu
sudo apt install python3-pip -y

pip3 install kube-shell
kube-shell
```

### helm
```bash
# github地址
https://github.com/helm/helm/releases
# 下载二进制包
curl -o https://get.helm.sh/helm-v3.13.0-linux-amd64.tar.gz
tar -zxvf helm-v3.13.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
# 查看版本
helm version
```

### KubeSphere
一个k8s集群的可视化管理面板
```bash
# 安装ks-core(Kubesphere core)核心组件，即可访问 Kubesphere 的 web 控制台
helm upgrade --instal1 -n kubesphere-system --create-namespace ks-core https://charts.kubesphere.io/main/ks-core-1.1.3.tgz --debug --wait
# 访问 Docker Hub 受限，请在命令后添加如下配置，修改默认的镜像拉取地址(华为云)
--set global.imageRegistry=swr.cn-southwest-2.myhuaweicloud.com/ks
--set extension.imageRegistry=swr.cn-southwest-2.myhuaweicloud.com/ks

# 当pod为running时，表示安装成功
kubectl get pods -n kubesphere-system
```
安装完成后它会提示登陆面板的url和账号密码，通常是主机ip+30880端口，初次登陆会要求修改密码

### Chart仓库
类似于DockerHub
```bash
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami && \
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

# 查看所有仓库
helm repo list

# 更新仓库和搜索chart
helm repo update [仓库]
helm search repo [仓库]

# 删除仓库
helm repo remove aliyun
```

使用
```bash
helm install my-nginx bitnami/nginx --version 18.2.4
```

### YAML模板
使用VSCode安装插件：1、`RedHat-YAML` 2、`kubernetes-templates`。