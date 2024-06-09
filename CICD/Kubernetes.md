
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
kubectl get pod [-owide] [-n 命名空间 | -A] #owide看详情 -n指定命名空间，-A显示所有命名空间下的
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




# 安装教程（建议看网上的）
## 1. 安装: 1.20.1版本
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

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

#设置时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

### 1.3 安装kubelet、kubeadm、kubectl
```bash
#配置k8s的yum源地址
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-e17-x86_64
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
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9

#开机启动kubelet
sudo systemctl enable --now kubelet
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
#### 1.4.2 初始化主节点
```bash
#在k8s-master机器上执行初始化操作(里面的第一个ip地址就是k8s-master机器的ip，改成你自己机器的，第二个改成自己的主机名，后面两个ip网段不用动)#所有网络范围不重叠
kubeadm init \
--apiserver-advertise-address=x.x.x.x \
--control-plane-endpoint=k8s-master \
--image-repository registry.cn-hangzhou.aliyuncs.com/xxxxx \
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

#如果初始化失败，重置kubeadmkubeadm reset
rm -rf /etc/cni/net.d $HOME/.kube/config
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
wget https://github.com/Mirantis/cri-dockerd/releases/download/ve.3.1/cri-dockerd-0.3.1-3.e17.x86_64.rpm
rpm ivh cri-dockerd-0.3.1-3.e17.x86_64.rpm
#修改/usr/lib/systemd/system/cri-docker.service文件中的ExecStart配置
vim /usr/lib/systemd/system/cri-docker.service

Execstart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containe

systemctl daemon-reload
systemctl enable --now cri-docker
```

