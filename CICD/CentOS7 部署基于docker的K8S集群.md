# CentOS7 部署基于docker的K8S集群



> 开源地址：https://github.com/kubernetes/kubernetes
>
> 官方文档：https://kubernetes.io/zh-cn/docs/home/
>
> 容器运行时文档：https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/
>
> 参考文档①：https://www.cnblogs.com/Sunzz/p/15184167.html
>
> 参考文档②：https://www.cnblogs.com/fenghua001/p/16849875.html
>
> 参考文档③：https://blog.51cto.com/u_16213670/12152667

## 一、k8s介绍

### 1.简介

Kubernetes（简称 K8s）是一个开源的容器编排平台，用于自动化容器化应用的部署、扩展和管理。它最初由 Google 开发，并于 2014 年捐赠给了云原生计算基金会（CNCF）。Kubernetes 能够帮助开发者和运维团队高效管理大规模容器化应用，是当今主流的容器编排解决方案之一。

### 2.核心功能

1. **容器编排**：管理多个容器的部署和运行。
2. **负载均衡和服务发现**：自动分配流量到容器，支持服务的自动发现。
3. **自动扩缩容**：根据应用负载动态调整容器数量。
4. **自愈功能**：检测故障容器并重新启动，替换或重新调度失败的容器。
5. **滚动更新和回滚**：支持应用的无中断更新和版本回退。
6. **存储编排**：自动挂载本地存储、云存储或网络存储。
7. **配置管理和密钥管理**：集中管理应用的配置和敏感数据。

### 3.核心组件

1. **Master 节点：**
   - `API Server`：提供 RESTful API，作为用户和集群管理的入口。
   - `Scheduler`：负责将待部署的容器分配到合适的节点。
   - `Controller Manager`：处理集群的控制任务，如副本维护、节点监控等。
   - `etcd`：分布式键值存储，用于保存集群的元数据。
2. **Worker 节点：**
   - `Kubelet`：管理每个节点上的容器生命周期。
   - `Kube-proxy`：负责网络通信，支持服务负载均衡。
   - `Container Runtime`：容器运行时，如 Docker 或 containerd。

### 4.优势

- **跨平台支持**：支持多种云平台（如 AWS、Azure、GCP）和本地数据中心。
- **灵活性**：支持多种编排模式和插件。
- **社区活跃**：拥有强大的开源社区支持，更新和迭代迅速。
- **生态系统丰富**：与 Helm、Istio、Prometheus 等工具深度集成，形成完整的云原生生态。

### 5.应用场景

1. **微服务架构**：简化服务部署和管理。
2. **持续集成/持续部署（CI/CD）**：支持快速开发和发布流程。
3. **大数据处理**：在分布式环境中高效运行批处理任务。
4. **混合云和多云管理**：跨多种环境统一调度容器。



## 二、主机准备

本文档基于 Kubernetes 经典的三节点环境进行部署，三个节点分别是 Master 节点、Worker01 节点、Worker02 节点。

|     角色      |   IP地址    |   主机名称   |  配置   | 网卡名称 |
| :-----------: | :---------: | :----------: | :-----: | :------: |
|  Master 节点  | 10.22.51.71 |  k8s-master  | 4C4G40G |   eth0   |
| Worker01 节点 | 10.22.51.72 | k8s-worker01 | 4C4G40G |   eth0   |
| Worker02 节点 | 10.22.51.73 | k8s-worker02 | 4C4G40G |   eth0   |

**操作系统：**`CentOS Linux release 7.9.2009 (Core)` 。

**CPU要求：**`>= 2C` 。

**内存要求：**`>= 2G` 。

硬盘要求：`>= 30G` 。

**VMware网络模式：**NAT模式。



## 三、初始化环境

> 三台节点均需要进行初始化环境配置。

### 1.升级系统内核配置

> 在 CentOS 7.9 系统上部署 k8s，推荐使用内核版本 `4.x` 或更高版本，这里安装的版本为稳定版 `6.9.10` 。

#### 1.1 下载RPM包

```bash
cd /usr/local/src
wget http://dl.lamp.sh/kernel/el7/kernel-ml-6.9.10-1.el7.x86_64.rpm
wget http://dl.lamp.sh/kernel/el7/kernel-ml-devel-6.9.10-1.el7.x86_64.rpm
wget http://dl.lamp.sh/kernel/el7/kernel-ml-tools-6.9.10-1.el7.x86_64.rpm
wget http://dl.lamp.sh/kernel/el7/kernel-ml-tools-libs-6.9.10-1.el7.x86_64.rpm
wget http://dl.lamp.sh/kernel/el7/kernel-ml-tools-libs-devel-6.9.10-1.el7.x86_64.rpm
wget http://dl.lamp.sh/kernel/el7/kernel-ml-headers-6.9.10-1.el7.x86_64.rpm
```

> Sunline 内网下载：
>
> ```bash
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-6.9.10-1.el7.x86_64.rpm
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-devel-6.9.10-1.el7.x86_64.rpm
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-tools-6.9.10-1.el7.x86_64.rpm
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-tools-libs-6.9.10-1.el7.x86_64.rpm
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-tools-libs-devel-6.9.10-1.el7.x86_64.rpm
> wget http://mirrors.sunline.cn/kernel/el7/x86_64/RPMS/kernel-ml-headers-6.9.10-1.el7.x86_64.rpm
> ```

#### 1.2 安装createrepo工具

```bash
yum install -y createrepo
```

#### 1.3 生成仓库元数据

执行以下命令后，会在 `/usr/local/src` 目录下生成 `repodata` 目录：

```bash
createrepo /usr/local/src
```

#### 1.4 创建本地repo配置文件

```bash
cat > /etc/yum.repos.d/kernel-ml-local.repo <<'EOF'
[kernel-ml-local]
name=Kernel ML Local Repo
baseurl=file:///usr/local/src
enabled=1
gpgcheck=0
EOF
```

#### 1.5 刷新缓存

```bash
yum clean all && yum makecache
```

#### 1.6 使用yum安装

```bash
yum install -y kernel-ml
```

#### 1.7 查看内核启动序号

使用以下命令可查看 GRUB 菜单中的所有内核条目：

```bash
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

# 输出信息如下
0 : CentOS Linux (6.9.10-1.el7.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
2 : CentOS Linux (0-rescue-97177eede4d044cba92e1dadd8dd3086) 7 (Core)
```

#### 1.8 设置默认启动内核

```bash
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 1.9 重启系统

```bash
reboot
```

#### 1.10 验证内核版本

```bash
[root@localhost /root]# uname -r
6.9.10-1.el7.x86_64
[root@localhost /root]# hostnamectl
   Static hostname: localhost.localdomain
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 97177eede4d044cba92e1dadd8dd3086
           Boot ID: 88b01fee8a4b4f579af33b7d47066a99
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 6.9.10-1.el7.x86_64
      Architecture: x86-64
```

#### 1.11 升级Kernel工具包

```bash
rpm -e kernel-headers kernel-tools kernel-tools-libs --nodeps
yum install -y kernel-ml*
```

#### 1.12 删除repo文件

```bash
rm -rf /etc/yum.repos.d/kernel-ml-local.repo
```

### 2.启用cgroups v2（CentOS7系统可忽略）

#### 2.1 前言

默认使用 `cgroups v1` ，在初始化 Master 节点时会弹出以下警告：

```bash
[WARNING SystemVerification]: cgroups v1 support is in maintenance mode, please migrate to cgroups v2
```

说明当前节点运行在 `cgroups v1` ，而 Kubernetes `1.28+` 开始推荐 `cgroups v2` ，到 `1.34.x` 已经明确提示 v1 处于维护模式。虽然不影响集群能起来，但长期来看建议迁移到 `cgroups v2` 。

**为什么要迁移到 cgroups v2？**

- `cgroups v1`：功能分散在不同的子系统目录，复杂而且有些功能不一致。
- `cgroups v2`：统一的层级管理，功能更强大（比如更好的内存控制、I/O 限速），是 Linux 未来的方向。
- 新版 containerd / CRI-O 等容器运行时对 v2 支持更完整。

#### 2.2 检查内核是否支持

```bash
grep cgroup2 /proc/filesystems

# 输出信息如下
nodev   cgroup2
```

#### 2.3 修改grub配置

编辑 `/etc/default/grub` 配置文件，找到 `GRUB_CMDLINE_LINUX` 行，修改以下内容，启用 `unified hierarchy` ：

```bash
sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1 systemd.legacy_systemd_cgroup_controller=false /' /etc/default/grub
```

> **参数解释：**
>
> - `systemd.unified_cgroup_hierarchy=1`：启用 `cgroups v2` 的统一层级（unified hierarchy），让 systemd 管理的所有服务使用 `cgroups v2` 。
> - `systemd.legacy_systemd_cgroup_controller=false`：禁用 systemd 对 `cgroups v1` 控制器的兼容模式。
>   - 加了之后，systemd 只使用 `cgroups v2` ，不再在后台同时维护 v1 控制器。
>   - 不加的话，systemd 仍然保留部分 v1 控制器兼容，K8s 能继续运行，但可能出现“部分资源无法完全受控”的情况（主要是对 CPU/内存限制的精细控制）。

#### 2.4 更新grub并重启

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

#### 2.5 验证生效

执行以下命令，验证是否启用了 `cgroups v2` ：

```bash
mount | grep cgroup2
```

### 3.基础配置

#### 3.1 关闭selinux

```bash
# 三个节点均配置
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

#### 3.2 关闭防火墙

```bash
# 三个节点均配置
systemctl stop firewalld
systemctl disable firewalld
```

#### 3.3 加载内核模块

`modprobe` 是一个用于加载和卸载内核模块的命令行工具。在 Linux 系统中，内核模块是可加载的代码片段，可以扩展内核的功能，比如驱动程序、文件系统等。

`br_netfilter` 模块允许 iptables 对桥接接口上的流量进行过滤。具体来说，它提供了一种机制，使得在 Linux 桥接网络中，iptables 可以应用于通过桥接接口传输的数据包。

```bash
# 三个节点均配置
modprobe br_netfilter
```

`ip_vs`（IP Virtual Server）是 Linux 内核中用于实现负载均衡的一个模块，它是 Linux Virtual Server（LVS）架构的一部分。`ip_vs` 模块通过内核实现高性能的 IP 层负载均衡，广泛用于构建高可用和高性能的负载均衡集群。

```bash
# 核心模块，处理请求调度
modprobe ip_vs
# 轮询调度算法模块
modprobe ip_vs_rr
# 加权轮询调度算法模块
modprobe ip_vs_wrr
# Source Hashing（源地址哈希）调度算法模块
modprobe ip_vs_sh
# 连接跟踪模块，用于记录请求的连接状态
modprobe nf_conntrack
```

使用以下命令检查操作系统是否加载了上方各个内核模块：

```bash
lsmod | grep -E 'br_netfilter|ip_vs|nf_conntrack'

# 输出信息
ip_vs_sh               12288  0 
ip_vs_wrr              12288  0 
ip_vs_rr               12288  0 
ip_vs                 200704  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          184320  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         12288  1 nf_conntrack
br_netfilter           32768  0 
libcrc32c              12288  3 nf_conntrack,xfs,ip_vs
```

#### 3.4 配置sysctl.conf

在 `/etc/sysctl.d` 目录下创建 `k8s.conf` 文件，配置内核参数，开启网桥流量过滤和内核转发功能：

```bash
# 三个节点均配置
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

> **解释说明：**
>
> - `net.bridge.bridge-nf-call-ip6tables`：用 IPv6 数据包通过桥接网络时，经过 `ip6tables` 的规则处理，保证 K8s 的 Service、NetworkPolicy、CNI 插件规则能生效。
> - `net.bridge.bridge-nf-call-iptables`：启用 IPv4 数据包通过桥接网络时，经过 `iptables` 的规则处理，保证 K8s 的 Service、NetworkPolicy、CNI 插件规则能生效。
> - `net.ipv4.ip_forward`：启用 IPv4 数据包的路由功能，使 Linux 主机能够充当路由器。让节点具备路由能力，保证 Pod/Service 的跨主机通信。

执行生效：

```bash
sysctl --system
```

#### 3.5 关闭swap分区

```bash
swapoff -a                            # 临时关闭
sed -i 's/.*swap.*/#&/' /etc/fstab    # 永久关闭
```

#### 3.6 配置hosts解析

这里只需要在 Master 节点配置 hosts 解析：

```bash
cat >> /etc/hosts <<'EOF'
10.22.51.71 k8s-master
10.22.51.72 k8s-worker01
10.22.51.73 k8s-worker02
EOF
```

然后复制 hosts 文件到所有 Worker 节点：

```bash
scp -rp /etc/hosts root@k8s-worker01:/etc/hosts
scp -rp /etc/hosts root@k8s-worker02:/etc/hosts
```

#### 3.7 修改主机名称

```bash
# 修改 Master 节点主机名称
hostnamectl set-hostname k8s-master

# 修改 Worker01 节点主机名称
hostnamectl set-hostname k8s-worker01

# 修改 Worker02 节点主机名称
hostnamectl set-hostname k8s-worker02
```

### 4.时间同步配置

#### 4.1 安装chrony

```bash
# 三个节点均安装
yum install -y chrony
```

#### 4.2 配置chrony.conf

在 Master 节点上配置 `chrony.conf` 文件，使用阿里云的 NTP 服务器作为时间源：

```bash
cp /etc/chrony.conf{,.bak}
cat > /etc/chrony.conf <<EOF
server time1.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 0.0.0.0/0
logdir /var/log/chrony
EOF
```

> **解释说明：**
>
> - `iburst`：表示如果初始时间同步失败，会快速发送多个数据包。
> - `driftfile`：记录系统时钟的漂移率。
> - `makestep 1.0 3`：如果时间偏差超过1秒，前3次校正会直接"跳步"而非渐进调整。
> - `rtcsync`：将系统时间同步到硬件时钟。
> - `allow 0.0.0.0/0`：允许所有客户端访问此NTP服务器（如果是作为NTP服务器使用）。
> - `logdir`：指定日志目录。

在两台 Worker 节点配置 `chrony.conf` 文件，使用 Master 节点的 NTP 服务器作为时间源：

```bash
cp /etc/chrony.conf{,.bak}
cat > /etc/chrony.conf <<EOF
server k8s-master iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
log measurements statistics tracking
EOF
```

> **日志参数说明：**
>
> - `measurements`：记录 NTP 服务器的测量信息。
> - `statistics`：记录统计数据（有助于调试）。
> - `tracking`：记录时间追踪信息（对系统时间的校正历史）。
>
> **注：不添加具体日志类型时，`/var/log/chrony` 下没有数据。**

#### 4.3 启动服务

```bash
# 三个节点均启动服务并开机自启
systemctl enable --now chronyd
```

#### 4.4 查看同步源

```bash
chronyc sources -v
```



## 四、安装Docker

> 三台节点都需要安装，这里安装的版本为 `28.4.0` 。

### 1.下载并解压docker源码包

下载 docker 源码包，并指定解压到 `/data/docker` 目录下，并重命名为 `bin` 目录：

```bash
mkdir -p /data/docker
cd /usr/local/src
wget https://download.docker.com/linux/static/stable/x86_64/docker-28.4.0.tgz
tar -xzf docker-28.4.0.tgz -C /data/docker
mv /data/docker/docker /data/docker/bin
```

### 2.下载Docker Compose二进制文件并授权

先创建 Docker Compose 文件所存放的目录：

```bash
mkdir -p /usr/local/lib/docker/cli-plugins
```

下载 Docker Compose 文件到该安装目录上，并重命名为 `docker-compose` ：

```bash
curl -SL -o /usr/local/lib/docker/cli-plugins/docker-compose https://github.com/docker/compose/releases/download/v2.39.3/docker-compose-linux-x86_64
```

赋予 `docker-compose` 文件可执行权限：

```bash
chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

### 3.配置环境变量

```bash
echo 'export PATH=/data/docker/bin:/usr/local/lib/docker/cli-plugins:$PATH' > /etc/profile.d/docker.sh
source /etc/profile
```

### 4.验证版本

```bash
[root@k8s-master /root]# docker -v
Docker version 28.4.0, build d8eb465
[root@k8s-master /root]# docker-compose -v
Docker Compose version v2.39.3
[root@k8s-master /root]# docker compose version
Docker Compose version v2.39.3
```

### 5.创建守护进程配置文件

创建 `/etc/docker` 目录，并在目录下创建 `daemon.json` 守护进程配置文件，添加以下内容：

```bash
mkdir -p /etc/docker
cat >/etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://ghcr.nju.edu.cn",
    "https://k8s.nju.edu.cn",
    "https://quay.nju.edu.cn",
    "https://gcr.nju.edu.cn",
    "https://nvcr.nju.edu.cn"
  ],
  "insecure-registries": [
    "docker.1ms.run",
    "ghcr.nju.edu.cn",
    "k8s.nju.edu.cn",
    "quay.nju.edu.cn",
    "gcr.nju.edu.cn",
    "nvcr.nju.edu.cn"
  ],
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "data-root": "/data/docker",
  "bip": "192.168.112.1/24",
  "default-address-pools": [
    {
      "base": "192.168.112.0/20",
      "size": 24
    }
  ]
}
EOF
```

> 如果用 Sunline 内网镜像源，只需要配置 `docker.sunline.cn` ：
>
> ```bash
> cat > /etc/docker/daemon.json <<'EOF'
> {
>   "registry-mirrors": [
>     "http://docker.sunline.cn"
>   ],
>   "insecure-registries": [
>     "docker.sunline.cn"
>   ],
>   "max-concurrent-downloads": 3,
>   "max-concurrent-uploads": 5,
>   "log-driver": "json-file",
>   "log-opts": {
>     "max-size": "100m",
>     "max-file": "3"
>   },
>   "data-root": "/data/docker",
>   "bip": "192.168.112.1/24",
>   "default-address-pools": [
>     {
>       "base": "192.168.112.0/20",
>       "size": 24
>     }
>   ]
> }
> EOF

### 6.配置启动服务

创建一个名为 `docker.service` 的 systemd 服务单元文件，用于管理 docker 服务，存放于 `/usr/lib/systemd/system` 目录下，并添加以下内容：

```bash
cat > /usr/lib/systemd/system/docker.service <<'EOF'
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
Environment="PATH=/data/docker/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/data/docker/bin/dockerd --default-ulimit nofile=65535:65535
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

### 7.启动docker服务

```bash
systemctl enable --now docker
```



## 五、安装cri-docker

> 安装文档：https://mirantis.github.io/cri-dockerd/usage/install/
>
> 下载地址：https://github.com/Mirantis/cri-dockerd/releases
>
> 三台节点都需要安装，这里安装的版本为 `0.3.20` 。

### 1.简介

Kubernetes 从 v1.6 开始引入了 CRI，允许以标准化的接口接入不同的容器运行时（如 containerd、CRI-O 等）。

Kubernetes 在 v1.20 宣布弃用 `Dockershim`，并在 v1.24 移除该模块。这意味着 Kubernetes 不再直接支持 Docker 作为容器运行时。

尽管 Docker 使用了 containerd 作为其底层运行时，但原生的 Docker 并未直接实现 CRI 接口。为了继续使用 Docker，社区开发了 CRI-Dockerd，作为 Kubernetes 与 Docker 的桥梁。

CRI-Dockerd 是一个插件，旨在为 Docker 提供 Kubernetes 的 CRI（Container Runtime Interface）兼容支持，使 Docker 可以继续用作 Kubernetes 的容器运行时。**但要注意的是， Docker 安装版本需要在 `27.0.2` 以上。**

### 2.下载cri-docker二进制文件

下载并解压 cri-docker 二进制文件包，并将该二进制文件移动到 `/usr/local/bin` 目录下：

```bash
cd /usr/local/src
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.20/cri-dockerd-0.3.20.amd64.tgz
tar -xzf cri-dockerd-0.3.20.amd64.tgz
cp cri-dockerd/cri-dockerd /usr/local/bin
```

### 3.验证版本

```bash
[root@k8s-master /root]# cri-dockerd --version
cri-dockerd 0.3.20 (b11203a)
```

### 4.配置启动服务

创建一个名为 `cri-docker.service` 的 systemd 服务单元文件，用于管理 cri-docker 服务，存放于 `/usr/lib/systemd/system` 目录下，并添加以下内容：

```bash
cat > /usr/lib/systemd/system/cri-docker.service <<'EOF'
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.10.1 --container-runtime-endpoint fd://
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

> **解释说明：**
>
> - `Requires=cri-docker.socket`：表示服务需要 `cri-docker.socket` 文件存在，否则服务启动将失败。
> - `ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://`：定义启动服务时执行的命令。
>   - `/usr/bin/cri-dockerd`：CRI-Dockerd 可执行文件的路径。
>   - `--network-plugin=cni`：指定网络插件为 CNI（Container Network Interface），Kubernetes 默认使用 CNI 插件进行容器网络管理。
>   - `--pod-infra-container-image`：指定 Pod 的基础容器镜像（Pause 容器），**这里指定的版本必须与 K8S 拉取的 `pause` 镜像版本保持一致**。
>   - `--container-runtime-endpoint fd://`：指定与 Docker 引擎通信的文件描述符（FD）套接字。
> - `ExecReload=/bin/kill -s HUP $MAINPID`：当服务接收到 `systemctl reload` 时，发送 `SIGHUP` 信号给主进程，触发服务重新加载配置。
> - `TimeoutSec=0`：禁用超时时间，服务不会因超时未响应而被杀死。
> - `RestartSec=2`：在服务崩溃或停止后，等待 2 秒再重新启动。
> - `Restart=always`：无论退出原因如何，总是重启服务。
> - `StartLimitBurst=3`：定义服务在 `StartLimitInterval` 时间内最多允许启动失败的次数，超过限制后服务将进入失败状态。
> - `StartLimitInterval=60s`：指定服务启动失败次数统计的时间窗口为 60 秒。

### 5.配置socket套接字文件

创建一个名为 `cri-docker.socket` 的 systemd 套接字单元文件，用于定义 CRI-Dockerd 服务的套接字（socket）监听配置，存放于 `/usr/lib/systemd/system` 目录下，并添加以下内容：

```bash
cat > /usr/lib/systemd/system/cri-docker.socket <<'EOF'
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=root

[Install]
WantedBy=sockets.target
EOF
```

> **解释说明：**
>
> - `ListenStream`：指定系统监听的套接字文件路径。`%t` 是一个占位符，在大多数现代系统中，`%t` 被解析为 `/run`，而不是 `/tmp`，因此该配置会监听 `/run/cri-dockerd.sock` 套接字文件。
> - `SocketMode`：设置套接字文件的权限为 `0660`，这意味着文件的所有者和所属组可以读写该套接字文件，其他用户没有访问权限。
> - `SocketUser`：指定该套接字文件的用户。`root` 表示套接字文件的所有者是 `root` 用户。
> - `SocketGroup`：指定该套接字文件的用户组。`root` 表示套接字文件的用户组是 `root`。

### 6.启动cri-docker服务

```bash
# 重载服务
systemctl daemon-reload

# 启用 cri-docker 服务并设置开机自启
systemctl enable --now cri-docker
```



## 六、安装Kubernetes组件

> `kubeadm` 、`kubelet` 和 `kubectl` 都是 Kubernetes 集群中常用的软件工具，三台都需要安装，这里安装的版本为 `1.34.1` 。
>
> - `kubeadm`：用来快速初始化和配置 Kubernetes 集群。只做“集群初始化”，不管理集群运行。
> - `kubelet`：节点级别的代理，负责在每个节点上启动和管理 Pod。保持节点状态与 API Server 同步。
> - `kubectl`：命令行工具，用来和 Kubernetes API 交互。用户直接操作集群，部署应用、查看状态。

### 1.添加k8s源

在 `/etc/yum.repos.d` 仓库目录下创建名为 `kubernetes.repo` 的仓库配置文件，并添加以下内容：

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes-new/core/stable/v1.34/rpm/
enabled=1
gpgcheck=0
EOF
```

### 2.刷新缓存

```bash
yum clean all && yum makecache
```

### 3.使用yum安装

安装前可先查看可安装的版本，这里显示的最新版为 `1.34.1` ：

```bash
yum list kubeadm --showduplicates
yum list kubelet --showduplicates
yum list kubectl --showduplicates
```

> `--showduplicates` 表示显示所有可用版本，包括已经安装的版本和远程仓库中的版本。

开始安装 Kubernetes 集群软件工具：

```bash
yum install -y kubelet kubeadm kubectl
```

> `kubectl` 组件一般只需要装在 Master 节点上就可以。

### 4.验证版本

```bash
[root@k8s-master /root]# kubelet --version
Kubernetes v1.34.1
```

### 5.配置kubelet

为了实现 docker 使用的 Cgroup Driver 与 kubelet 使用的 cgroup 的一致性，需编辑 `/etc/sysconfig/kubelet` 配置文件，修改以下内容：

```bash
# 配置 /etc/sysconfig/kubelet
sed -i 's/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"/g' /etc/sysconfig/kubelet

# 查看 Docker 的 cgroup 驱动
docker info | grep Cgroup

# 设置 kubelet 开机自启
systemctl enable kubelet

# 重载服务
systemctl daemon-reload
```

### 6.配置crictl.yaml

`crictl.yaml` 是 crictl 的配置文件，主要用来告诉 crictl 如何连接容器运行时以及一些调试选项。创建 `/etc/crictl.yaml` 配置文件，并添加以下内容：

```bash
cat > /etc/crictl.yaml <<'EOF'
runtime-endpoint: unix:///run/cri-dockerd.sock
image-endpoint: unix:///run/cri-dockerd.sock
timeout: 10
debug: false
EOF
```

> **解释说明：**
>
> - `runtime-endpoint`：指定 CRI runtime 的 socket 地址。这里 `crictl` 会通过 `cri-dockerd` 去管理 Pod、容器（运行、停止、删除等操作）。
> - `image-endpoint`：指定 CRI image service 的 socket 地址。这里意味着镜像拉取、删除、列出等操作也走同一个 cri-dockerd runtime。
> - `timeout`：与 CRI 建立连接时的超时时间（秒），防止网络或 socket 卡住导致命令 hang。
> - `debug`：关闭调试模式，不会打印详细日志。



## 七、配置Master节点

### 1.拉取K8S初始化镜像

先查看 Kubernetes 初始化所需的所有镜像：

```bash
kubeadm config images list

# 输出信息如下
registry.k8s.io/kube-apiserver:v1.34.1
registry.k8s.io/kube-controller-manager:v1.34.1
registry.k8s.io/kube-scheduler:v1.34.1
registry.k8s.io/kube-proxy:v1.34.1
registry.k8s.io/coredns/coredns:v1.12.1
registry.k8s.io/pause:3.10.1
registry.k8s.io/etcd:3.6.4-0
```

使用指定镜像仓库预拉 Kubernetes 控制面所需的所有镜像，并打开详细日志（便于排查拉取问题）：

```bash
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --v=5
```

> **参数解释：**
>
> - `--image-repository`：把所有控制面镜像指向国内阿里云仓库，提高拉取速度并避免公网访问失败。
> - `--v=5`：输出拉取每个镜像的过程、progress、错误信息，方便调试网络问题。

拉取完成后，可通过 `docker` 命令查看镜像：

```bash
docker images
```

也可以通过 `crictl ` 命令查看镜像：

```bash
crictl images
```

>  `docker images` 和 `crictl images`，虽然都是查看镜像，但原理、用途和范围有区别：
>
> - `docker images`：查看 Docker（或者 dockerd）本地管理的镜像。
> - `crictl images`：查看 CRI（Container Runtime Interface）管理的镜像，支持的 CRI有 `containerd` 、`cri-o` 、`cri-dockerd` 等。

### 2.初始化K8S集群

在  Master 节点上执行以下命令， 初始化 Kubernetes 集群的第一个控制平面节点（**注：安装的版本必须小于等于前面安装的集群软件工具版本**），主要目的是写入配置文件（如 `/etc/kubernetes/manifests/*.yaml` ），启动控制平面组件（通过 `kubelet` 以 Pod 形式运行）：

```bash
kubeadm init \
--kubernetes-version=v1.34.1 \
--apiserver-advertise-address=10.22.51.71 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.245.0.0/16 \
--cri-socket=unix:///run/cri-dockerd.sock \
--image-repository=registry.aliyuncs.com/google_containers
```

> **解释说明：**
>
> - `kubeadm init`：初始化 Kubernetes 集群的控制平面。这个命令会启动 Kubernetes 控制平面组件（如 API Server、Controller Manager、Scheduler 等）并配置集群环境。
> - `--kubernetes-version`：指定要安装的 Kubernetes 版本，`kubeadm` 会自动下载对应版本的控制平面组件镜像。
> - `--apiserver-advertise-address`：指定 API Server 对外服务的监听地址（通常是 Master 节点的 IP 地址）。其他节点通过此地址与 API Server 通信。如果服务器有多个网卡，建议明确指定地址，否则可能会绑定错误的网卡。
> - `--service-cidr`：指定 Kubernetes 集群中 Service 的虚拟 IP 地址范围（Service CIDR）。这些 IP 地址不会被直接分配给实际的网络接口，而是用于内部通信。默认值为 `10.96.0.0/16`，可以根据需求调整，但应避免与实际网络冲突。
> - `--pod-network-cidr`：指定 Pod 网络的地址范围（Pod CIDR），每个 Pod 都会从此范围内分配一个独立的 IP 地址。此地址范围应与 Service CIDR 和主机网络无冲突。不同的 CNI 插件（如 Flannel、Calico）可能要求特定的 CIDR 地址。
> - `--cri-socket`：指定 Kubernetes 使用的容器运行时接口（CRI，Container Runtime Interface）的通信套接字路径。`unix://` 是一种通信协议，表示使用 UNIX 套接字（Unix Domain Socket）进行本地进程间通信。
> - `--image-repository`：指定 Kubernetes 所需镜像的仓库地址，这里使用国内镜像源。

等待镜像拉取成功后，会继续初始化集群，等到初始化完成后，会看到类似如下信息，保留最后两行的输出后边会用到：

```bash
[init] Using Kubernetes version: v1.34.1
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: cgroups v1 support is in maintenance mode, please migrate to cgroups v2
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'

......

[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.22.51.71:6443 --token u845z0.29om4zuxg07ra8sr \
        --discovery-token-ca-cert-hash sha256:7639915f6b970f248b3fdd35babe808ce7e25718a4a4d5a19bfb85a36c18cfc5 
```

### 3.配置kubectl

初始化 Kubernetes 集群后，设置 Kubernetes 客户端（kubectl）的访问权限和配置文件路径（也就是执行初始化成功后输出的那三条命令）：

1. 创建 `.kube` 目录：

   ```bash
   mkdir -p ~/.kube
   ```

   - `$HOME/.kube` 是 kubectl 默认查找配置文件的位置。

2. 复制 `admin.conf` 配置文件：

   ```bash
   cp /etc/kubernetes/admin.conf ~/.kube/config
   ```

   - `/etc/kubernetes/admin.conf` 是 kubeadm 初始化生成的管理员 `kubeconfig` 文件。
   - 复制后，kubectl 默认就会使用这个配置文件去连接 Kubernetes 集群。

3. 修改配置文件权限：

   ```bash
   chown $(id -u):$(id -g) ~/.kube/config
   ```

   - 把配置文件的所有者改为当前用户（通常是 root 或普通用户）。
   - `$(id -u)`：当前用户 UID。
   - `$(id -g)`：当前用户 GID。
   - 防止 kubectl 因权限不足而无法读取配置文件。

**为什么需要这些步骤？**

- Kubernetes 使用 `kubectl` 与集群交互。
- 默认情况下，`admin.conf` 是由 `kubeadm init` 生成的，只有 root 用户能访问。
- 通过这些命令，可以将管理员的访问权限配置为当前用户，以便普通用户运行 `kubectl` 命令，同时就可以直接在 master 节点使用 kubectl 操作集群。

### 4.验证配置是否成功

执行以下命令，验证是否能够成功连接到集群：

```bash
kubectl cluster-info

# 输出信息如下
Kubernetes control plane is running at https://10.22.51.71:6443
CoreDNS is running at https://10.22.51.71:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 5.查看集群节点信息

执行以下命令，查看当前 Kubernetes 集群中的节点状态，它会显示集群中所有节点的基本信息：

```bash
kubectl get nodes

# 输出信息如下
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   16m   v1.34.1
```

这里节点状态为 `NotReady` ，是由于 CNI 网络插件未安装（如 Flannel、Calico），后续操作会安装。

> **字段解释：**
>
> - `NAME`：节点的名称（主机名）。
> - `STATUS`：节点的当前状态。
>   - `Ready`：节点正常，可接受调度。
>   - `NotReady`：节点不可用，可能是 kubelet 未运行或网络问题。
>   - `SchedulingDisabled`：节点被标记为不可调度，通常用于维护（如执行 `kubectl cordon`）。
> - `ROLES`：节点的角色。
>   - `control-plane`：控制平面节点（Master 节点）。
>   - `<none>`：Worker 节点，无特定角色。
> - `AGE`：节点加入集群的时间长度。
> - `VERSION`：节点运行的 Kubernetes 版本。

### 6.查看集群Pods状态

执行以下命令，查看当前 Kubernetes 集群中所有命名空间下的 Pod 状态：

```bash
kubectl get pods -A

# 输出信息如下
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-7cc97dffdd-clz69             0/1     Pending   0          18m
kube-system   coredns-7cc97dffdd-mcrkf             0/1     Pending   0          18m
kube-system   etcd-k8s-master                      1/1     Running   0          18m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          18m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          18m
kube-system   kube-proxy-v49nt                     1/1     Running   0          18m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          18m
```

这里 CoreDNS 状态为 `Pending` ，也是由于 CNI 网络插件未安装（如 Flannel、Calico），后续操作会安装。



## 八、配置Worker节点

### 1.加入K8S集群

在两台 Worker 节点服务器上分别执行以下命令，加入到主节点集群上： 

```bash
kubeadm join k8s-master:6443 --token u845z0.29om4zuxg07ra8sr --discovery-token-ca-cert-hash sha256:7639915f6b970f248b3fdd35babe808ce7e25718a4a4d5a19bfb85a36c18cfc5
```

输出信息如下：

```bash
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: cgroups v1 support is in maintenance mode, please migrate to cgroups v2
[preflight] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[preflight] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.503447506s
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### 2.查看集群节点信息

在 Master 节点服务器上执行以下命令，查看集群节点：

```bash
kubectl get nodes

# 输出信息如下
NAME           STATUS     ROLES           AGE   VERSION
k8s-master     NotReady   control-plane   22m   v1.34.1
k8s-worker01   NotReady   <none>          47s   v1.34.1
k8s-worker02   NotReady   <none>          44s   v1.34.1
```

可以看到：

- `STATUS` 状态都是 `NotReady` , 这是因为确实网络插件导致的，等安装好网络插件就好了。

- `k8s-master` 节点被标记为 `control-plane`（也就是 Master 节点），负责调度、API 访问、控制等核心功能。
- `k8s-worker01` 和 `k8s-worker02` 没有分配角色，即 `<none>` 。

### 3.分配节点角色

在 Master 节点服务器上执行以下命令，给 Wroker 节点打上标签，用于 Kubernetes 调度和管理：

```bash
kubectl label node k8s-worker01 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker02 node-role.kubernetes.io/worker=worker
```

> **参数解释：**
>
> - `node k8s-worker01`：指定要打标签的节点。
> - `node-role.kubernetes.io/worker=worker`：标签键值对。
>   - `node-role.kubernetes.io/worker`：约定俗成的 worker 节点角色标签键。
>   - `worker`：自定义的值，也可以不写值，只用键。

再次查看集群节点信息：

```bash
kubectl get nodes

# 输出信息如下
NAME           STATUS     ROLES           AGE     VERSION
k8s-master     NotReady   control-plane   25m     v1.34.1
k8s-worker01   NotReady   worker          3m35s   v1.34.1
k8s-worker02   NotReady   worker          3m32s   v1.34.1
```

也可执行 `kubectl get nodes --show-labels` 查看详细的标签信息。



## 九、安装网络插件

> Calico 官方地址：https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

### 1.前言

在使用容器的时候，只是提供一个 CNI（Container Network Interface）标准的通用的接口，容器网络解决方案 fiannel、calico、Canal、weave，使用这些解决方案可以满足该协议的所有容器平台提供网络功能。

- Calico 开源地址：https://github.com/projectcalico/calico
- Flannel 开源地址：https://github.com/coreos/flannel/
- Weave 开源地址：https://github.com/weaveworks/weave
- Canal 开源地址：https://github.com/projectcalico/canal

这里使用 `Calico` ，因为支特网络策略、支持服务网格 istio 集成。

### 2.Calico插件介绍

Calico 是一个流行的 Kubernetes 网络插件，用于为 Pod 提供高性能的网络通信和网络安全（网络策略）。它支持多种底层网络实现方式，例如基于 BGP 的路由、IP-in-IP、VXLAN 等，适合中小规模以及大规模的 Kubernetes 集群。

**主要功能：**

- **Pod 网络互联：**提供 Pod 到 Pod 的网络通信；支持基于多种技术（BGP、VXLAN）的网络转发。
- **网络策略：**实现细粒度的网络访问控制（基于 Pod 的访问规则）。
- **灵活的网络实现方式：**支持多种方式，例如纯三层网络、IP-in-IP 隧道、VXLAN、WireGuard 加密。
- **高性能：**纯三层网络（无 Overlay）方式在性能和资源占用方面优于 Overlay 网络（如 Flannel 的 VXLAN 模式）。
- **跨集群连接：**支持使用 BGP 将多个 Kubernetes 集群的网络互联。

**Calico 架构：**Calico 主要由两部分组成：

|           组件            |                       作用                        |           部署方式            |
| :-----------------------: | :-----------------------------------------------: | :---------------------------: |
| `calico-kube-controllers` |      控制器，用于管理 IP 地址池、网络策略等       | 仅在控制平面节点部署 1 个即可 |
|       `calico-node`       | 节点代理，负责每个节点的路由、网络策略和 Pod 网络 |   **每个节点都有一个 Pod**    |

### 3.下载calico.yaml文件

Calico 提供了一键式安装方式，可以使用 YAML 文件快速部署。在 master 主节点服务器上，下载完整版的地址文件：

```bash
cd /usr/local/src
wget https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

### 4.修改calico.yaml文件

下载后，编辑 `calico.yaml` 配置文件，修改 Pod 网络池参数和相应的镜像源地址，使用国内的毫秒镜像：

```bash
# 取消注释并修改 CALICO_IPV4POOL_CIDR 选项的值
sed -i "s@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g" calico.yaml
sed -i 's@#   value: "192.168.0.0/16"@  value: "10.245.0.0/16"@g' calico.yaml

# 修改镜像源地址
sed -i "s/docker.io/docker.1ms.run/g" calico.yaml
```

> `CALICO_IPV4POOL_CIDR` 是 Calico 分配 Pod IP 的网段，必须与初始化时的 `--pod-network-cidr` 选项指定的 Pod 网络网段配置保持一致，否则 Pod 无法分配 IP 或节点一直 `NotReady` 。
>
> 由于该配置文件显式地定义了仓库地址，因此才需要替换，但如果使用的容器进行时为 Containerd，可以不修改，会自动替换为配置好的镜像加速仓库地址，而 Docker 无法替换，所以这里替换主要针对 Docker 容器进行时。

### 5.部署Calico Pod

执行以下命令，开始安装 Calico Pod：

```bash
kubectl apply -f calico.yaml
```

> 卸载 Calico 执行命令：`kubectl delete -f calico.yaml` 。
>
> 查看删除情况执行命令：`kubectl get pods -n kube-system | grep calico` 。

### 6.查看Pod状态

执行以下命令查看 Calico Pod 状态，检查 Calico 和 coredns 是否安装成功：

```bash
kubectl get pods -n kube-system
kubectl get pods -A
```

这里会看到镜像正在创建中，等待创建完成后，再查看状态变为 `Running` ，输出信息如下：

```bash
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-8d95b6db8-h9f9t   1/1     Running   0          16m
calico-node-2zlhn                         1/1     Running   0          16m
calico-node-c2qzx                         1/1     Running   0          16m
calico-node-sjgnm                         1/1     Running   0          16m
coredns-66f779496c-k7sqq                  1/1     Running   0          112m
coredns-66f779496c-rvl6n                  1/1     Running   0          112m
etcd-k8s-master                           1/1     Running   0          112m
kube-apiserver-k8s-master                 1/1     Running   0          112m
kube-controller-manager-k8s-master        1/1     Running   0          112m
kube-proxy-8m779                          1/1     Running   0          112m
kube-proxy-q66vh                          1/1     Running   0          73m
kube-proxy-znx6g                          1/1     Running   0          73m
kube-scheduler-k8s-master                 1/1     Running   0          112m
```

这里显示 3 个 `calico-node` ，是因为集群是三节点部署的，每个 `calico-node` 作为节点代理，负责每个节点的路由、网络策略和 Pod 网络，即每个节点对应一个 calico-node Pod。可执行以下命令，查看每个 Pod 对应的节点：

```bash
kubectl get pods -n kube-system -o wide
```

### 7.查看集群节点信息

```bash
kubectl get nodes

# 输出信息如下
NAME           STATUS   ROLES           AGE    VERSION
k8s-master     Ready    control-plane   153m   v1.34.1
k8s-worker01   Ready    worker          131m   v1.34.1
k8s-worker02   Ready    worker          131m   v1.34.1
```

可以看到，`STATUS` 状态变成了 `Ready` , 表示网络已经组成。

### 8.查看Calico节点组件日志

```bash
# 先列出所有 kube-system 命名空间下的 Pod
kubectl get pods -n kube-system

# 查看 Pod（calico-node-8mbwk）的状态
kubectl logs -n kube-system calico-node-52dmf
```



## 十、创建测试Pods

### 1.部署并运行Nginx镜像

#### 1.1 启动并运行Pod

在 Kubernetes 集群里创建并启动一个单 Pod 来运行 Nginx：

```bash
kubectl run nginx --image=nginx --restart=Never
```

> **参数解释：**
>
> - `kubectl run`：创建并运行一个 Pod 或 Deployment（取决于 `--restart` 参数）。
> - `nginx`：Pod 的名字。
> - `--image=nginx`：指定要运行的镜像，这里是官方 Nginx 镜像。
> - `--restart=Never`：告诉 Kubernetes 只创建一个 Pod，而不是 Deployment。

#### 1.2 查看Pod状态

```bash
# 显示当前命名空间下的 Pod 简要信息
kubectl get pods

# 显示更多详细信息
kubectl get pods -o wide

# 查看 Pod 详细状态、事件、节点分配等
kubectl describe pod nginx

# 查看容器日志
kubectl logs nginx
```

#### 1.3 暴露Service

如果想从外部访问 Pod，需要暴露服务，执行以下命令，把运行的 nginx Pod 暴露成一个 Service，把 Pod 的 80 端口映射到 NodePort，通过 NodePort 类型让集群外部也能访问：

```bash
kubectl expose pod nginx --type=NodePort --port=80
```

> **参数解释：**
>
> - `kubectl expose`：给 Pod / Deployment / ReplicaSet 创建一个 Service。
> - `pod nginx`：表示暴露名为 `nginx` 的 Pod。
> - `--type=NodePort`：Service 类型是 NodePort，会在集群中每个节点开放一个高位端口（默认范围 30000–32767）。
> - `--port=80`：Service 对外暴露的端口（即 Service 的端口）。

#### 1.4 查看Service信息

创建后，默认在当前 `default` 命名空间，查看 Service 的端口信息：

```bash
kubectl get svc   # 或：kubectl get service

# 输出信息如下
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        3h8m
nginx        NodePort    10.96.151.182   <none>        80:31820/TCP   9s


# 查看指定Service信息
kubectl get svc nginx
```

> **解释说明：**
>
> - `NAME`：Service 名字。
> - `TYPE`：类型，这里是 `ClusterIP`（只能集群内访问）。
> - `CLUSTER-IP`：K8s 自动分配的虚拟 IP。
> - `EXTERNAL-IP`：外部 IP（ClusterIP 类型没有，所以是 `<none>`）。
> - `PORT(S)`：端口映射信息。
>   - `80`：Service 内部访问端口（ClusterIP 内部用）。
>   - `31567`：随机分配的 NodePort。
> - `AGE`：Service 存活时间，

#### 1.5 访问Nginx

1. **在集群节点外访问：**

   用任意一个 Kubernetes 节点的 IP（Master 或 Worker 都行）+ NodePort：

   ```bash
   http://<NodeIP>:31820
   ```

   <img src="https://raw.githubusercontent.com/zyx3721/Picbed/main/blog-images/2025/09/15/91680e4f31b0dfdf0389a585322fce04-image-20250915203151062-f98b6c.png" alt="image-20250915203151062" style="zoom:67%;" />

2. **在节点内部访问：**

   可以直接用 ClusterIP，这个 IP 只能在集群内部访问（Pod、节点内部都能访问），外部主机访问不到：

   ```bash
   curl http://10.96.151.182
   ```

#### 1.6 进入Pod容器

执行以下命令，可进入 nginx 容器：

```bash
kubectl exec -it nginx -- /bin/bash
```

还可以在 nginx 容器里直接用 `curl` 测试服务：

```bash
kubectl exec -it nginx -- curl http://127.0.0.1 
```

### 2.使用Netshoot镜像

在 Kubernetes 集群里临时启动一个 Pod，镜像是 `nicolaka/netshoot`，然后进入 bash 交互模式：

```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

> **参数解释：**
>
> - `--rm`：Pod 退出后自动删除，不会残留在集群里。
> - `-it`：可以交互式地使用终端。
> - `--image=nicolaka/netshoot`：指定使用的镜像。`netshoot` 镜像包含各种网络调试工具，如 `curl`、`tcpdump`、`dig`、`traceroute` 等。
> - `-- bash`：在容器里启动 bash 终端。

进入后，执行 `ping` 网络测试：

```bash
netshoot:~# ping -c 2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=11.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=16.6 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 11.075/13.831/16.587/2.756 ms
```

### 3.定位Pod所在节点及已拉取镜像

在上方拉取了 nginx 镜像并启动 Pod 后，如果要查找该镜像所在节点和查看镜像信息，需按以下操作执行。

先查看 Pod 所在节点，可以看到，该 Pod 所在的节点位于 `k8s-worker01` 节点上：

```bash
kubectl get pods nginx -o wide

# 输出信息如下
NAME    READY   STATUS    RESTARTS   AGE    IP             NODE           NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          3h7m   10.245.79.65   k8s-worker01   <none>           <none>
```

在 `k8s-worker01` 节点上，执行 `docker` 命令即可查看已拉取的镜像信息：

```bash
docker images
docker ps
```



## 十一、部署busybox来测试集群各网络情况

### 1.配置busybox.yaml文件

编写 `busybox.yaml` 配置文件，添加以下内容：

```yaml
cat > busybox.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        imagePullPolicy: IfNotPresent
        args:
        - /bin/sh
        - -c
        - sleep 1; touch /tmp/healthy; sleep 30000
        readinessProbe:   
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 1
EOF
```

> **解释说明：**这是一份 Kubernetes 的 Deployment 配置文件，用来部署一个名为 `busybox` 的应用。Deployment 是 Kubernetes 中用来管理无状态应用的控制器。
>
> - `apiVersion`：定义了资源使用的 API 版本，`apps/v1` 是 Deployment 资源的稳定版本。
> - `kind`：资源类型为 Deployment，用于管理 Pod 的创建和升级。
> - `metadata.name`：指定 Deployment 的名称为 `busybox`。
> - `spec`：Deployment 的具体配置。
>   - `replicas: 2`：定义了该 Deployment 需要运行的 Pod 副本数量为 2。
>   - `selector.matchLabels`：指定 Pod 的选择器，Deployment 会管理所有 `app=busybox` 的 Pod。
>   - `template`：描述 Pod 模板的配置，所有被创建的 Pod 都会以此为模板。
>     - `metadata.labels`：定义 Pod 的标签，需与 `selector.matchLabels` 对应。
>     - `spec`：定义 Pod 的内容，包括容器等。
>       - `name`：容器的名称。
>       - `image`：使用 `busybox` 镜像。
>       - `imagePullPolicy`：拉取策略：如果本地已有镜像则不重新拉取。
>       - `args`：容器启动时执行的命令和参数：
>         - `/bin/sh`：使用 shell。
>         - `-c`：执行后续的命令。
>         - `sleep 1; touch /tmp/healthy; sleep 30000`:
>           - `sleep 1`：等待 1 秒。
>           - `touch /tmp/healthy`：创建一个名为 `/tmp/healthy` 的文件。
>           - `sleep 30000`：挂起容器 30000 秒。
>         - `readinessProbe`：是 Kubernetes 用来判断容器是否已经准备好处理流量的探针。
>           - `exec`：使用命令检查容器状态：
>             - `command: [cat, /tmp/healthy]`：运行 `cat /tmp/healthy`，如果文件存在且命令成功运行，探针认为容器已就绪。
>           - `initialDelaySeconds: 1`：在容器启动后等待 1 秒再开始探测。

### 2.部署busybox Pod

由于没有指定命名空间，因此会默认把资源放在 `default` 命名空间：

```bash
kubectl apply -f busybox.yaml
```

### 3.查看Pod状态

默认在当前 `default` 命名空间，通过标签选择器只列出 BusyBox Pod 的状态，并显示详细信息：

```bash
kubectl get pods -l app=busybox -o wide

# 输出信息如下
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
busybox-7f67b44fd7-5nbcb   1/1     Running   0          12m   10.245.69.199   k8s-worker02   <none>           <none>
busybox-7f67b44fd7-8cjbp   1/1     Running   0          12m   10.245.79.71    k8s-worker01   <none>           <none>
```

> 标签选择器对应开头定义的 `busybox.yaml` 文件中的 `labels` 选项。

### 4.查看Service网络

查看所有命名空间下的 Service 的网络信息：

```bash
kubectl get service -A

# 输出信息如下
NAMESPACE     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP                  4h13m
default       nginx        NodePort    10.96.151.182   <none>        80:31820/TCP             65m
kube-system   kube-dns     ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   4h13m
```

> `kube-dns` 是 Kubernetes 集群内默认的 DNS 解析服务，负责 Pod 内部的域名解析，让 Pod 可以通过 Service 名称互相访问，而不需要记 IP。

### 5.测试跨节点pod互通性

进入名为 `busybox-7f67b44fd7-5nbcb` 的 BusyBox Pod 容器：

```bash
kubectl exec -it busybox-7f67b44fd7-5nbcb -- /bin/sh
```

执行 `ping` 命令，去 `ping` 另一个 IP 地址为 `10.245.79.71` 的 BusyBox Pod 容器，测试互通性：

```bash
/ # ping -c2 10.245.79.71
PING 10.245.79.71 (10.245.79.71): 56 data bytes
64 bytes from 10.245.79.71: seq=0 ttl=62 time=0.986 ms
64 bytes from 10.245.79.71: seq=1 ttl=62 time=0.925 ms

--- 10.245.79.71 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.925/0.955/0.986 ms
```

### 6.测试pod和各节点互通性

进入名为 `busybox-7f67b44fd7-5nbcb` 的 BusyBox Pod 容器：

```bash
kubectl exec -it busybox-7f67b44fd7-5nbcb -- /bin/sh
```

执行 `ping` 命令，分别去 `ping` 各节点的 IP 地址，测试互通性：

```bash
/ # ping -c2 10.22.51.71
PING 10.22.51.71 (10.22.51.71): 56 data bytes
64 bytes from 10.22.51.71: seq=0 ttl=63 time=1.015 ms
64 bytes from 10.22.51.71: seq=1 ttl=63 time=0.589 ms

--- 10.22.51.71 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.589/0.802/1.015 ms

/ # ping -c2 10.22.51.72
PING 10.22.51.72 (10.22.51.72): 56 data bytes
64 bytes from 10.22.51.72: seq=0 ttl=63 time=0.943 ms
64 bytes from 10.22.51.72: seq=1 ttl=63 time=0.670 ms

--- 10.22.51.72 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.670/0.806/0.943 ms

/ # ping -c2 10.22.51.73
PING 10.22.51.73 (10.22.51.73): 56 data bytes
64 bytes from 10.22.51.73: seq=0 ttl=64 time=0.382 ms
64 bytes from 10.22.51.73: seq=1 ttl=64 time=0.238 ms

--- 10.22.51.73 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.238/0.310/0.382 ms
```

### 7.测试pod和Service网络连通性

进入名为 `busybox-7f67b44fd7-5nbcb` 的 BusyBox Pod 容器：

```bash
kubectl exec -it busybox-7f67b44fd7-5nbcb -- /bin/sh
```

执行 `nc` 命令，分别测试不同 Service 的端口连通性：

- 测试 kubernetes Service 的 `443` 端口：

  ```bash
  / # nc -vz kubernetes 443
  kubernetes (10.96.0.1:443) open
  ```

  - `nc`（netcat）命令是一个 TCP/UDP 网络调试工具，常用来测试端口连通性。
  - `-v`：表示 verbose，显示连接过程信息。
  - `-z`：表示 zero-I/O mode，只扫描端口，不发送数据

- 测试 nginx Service 的 `80` 端口：

  ```bash
  / # nc -vz nginx 80
  nginx (10.96.151.182:80) open
  ```

- 测试 kube-dns Service 的 `53` 端口：

  ```bash
  / # nc -vz kube-dns.kube-system 53
  kube-dns.kube-system (10.96.0.10:53) open
  ```

  > **注：**Kubernetes 默认 DNS 服务 `kube-dns` 是在 `kube-system` 命名空间，和 pod 所在的命名空间不同，因此需要指定。

在 Kubernetes 中，默认情况下不能通过 `ping` (ICMP) 访问 Service 的 ClusterIP，这是设计使然：

- ClusterIP 是一个虚拟 IP，由 kube-proxy 转发 TCP/UDP 流量到后端 Pod。
- kube-proxy 不处理 ICMP 流量，所以 `ping <ClusterIP>` 会失败，即使网络正常。



## 十二、更新K8S证书

### 1.检查证书

执行以下命令，检查 Kubernetes 集群中由 kubeadm 管理的证书（主要是控制平面组件的证书）的过期情况：

```bash
kubeadm certs check-expiration
```

也可以通过 `openssl` 命令查看某个单独的组件证书过期情况：

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### 2.导出kubeadm配置文件

执行以下命令，导出 kubeadm 的配置文件：

```bash
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm-config.yaml
```

### 3.修改kubeadm配置文件

编辑 `kubeadm-config.yaml` 配置文件，修改 `certificateValidityPeriod` 选项参数，即证书的有效时间，这里修改为 10 年，相当于 87600 小时：

```bash
sed -i 's/certificateValidityPeriod.*/certificateValidityPeriod: 87600h/g' kubeadm-config.yaml
```

### 4.更新证书

使用 kubeadm 重新生成证书：

```bash
# 更新所有证书
kubeadm certs renew all --config kubeadm-config.yaml

# 或者单独更新某个证书，例如 apiserver
kubeadm certs renew apiserver --config kubeadm-config.yaml
```

> 如果不修改配置文件，默认情况下只有效期只更新一年。

### 5.重启kubelet服务

配置完成后，为了加载到新证书，需要重启 kubelet 服务，kubelet 会重新检测 `/etc/kubernetes/manifests/` 目录下的文件，自动删除并重建控制平面 Pod，加载新证书：

```bash
systemctl restart kubelet
```

### 6.更新kubeconfig文件（可选）

重新复制 `/etc/kubernetes` 目录下的 `admin.conf` 配置文件到 `~/.kube/config` 文件中：

```bash
cp -f /etc/kubernetes/admin.conf ~/.kube/config
```

> 这个只是更新本地 kubectl 访问控制平面的凭证，不会影响集群中已经运行的节点或 Pod。

### 7.验证

确认新证书已生效：

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
kubeadm certs check-expiration
kubectl get nodes
kubectl get pods -n kube-system
```



## 十三、其他说明

### 1.K8S集群命名空间

执行 `kubectl get namespaces` 命令后，默认的集群命名空间有：

```bash
NAME              STATUS   AGE
default           Active   139m
kube-node-lease   Active   139m
kube-public       Active   139m
kube-system       Active   139m
```

解释如下：

|     命名空间      |  状态  |                             说明                             |
| :---------------: | :----: | :----------------------------------------------------------: |
|     `default`     | Active | 默认命名空间，如果创建 Pod/Service 没指定 namespace，就在这里 |
|   `kube-system`   | Active | Kubernetes 系统组件所在的命名空间，比如 kube-apiserver、kube-controller-manager、kube-proxy、coredns 等 |
|   `kube-public`   | Active | 公共命名空间，所有用户都可以读取，通常用于公开信息或配置共享 |
| `kube-node-lease` | Active | 用于节点心跳（Node Lease），kubelet 会定期更新，用来判断节点是否存活 |

补充说明：

- `kubectl get pods -A` 会显示**所有命名空间**的 Pod，包括这些系统命名空间。
- 系统命名空间不建议随意删除或修改，里面的组件对集群稳定性很关键。

### 2.配置YAML文件报错说明

执行 `kubectl apply -f xxx.yaml` 后，如果报错了，不需要卸载再安装，Kubernetes 资源可以直接通过修改 YAML 并重新 `apply` 来更新，即无需先执行 `kubectl delete -f xxx.yaml` 命令后再安装。
