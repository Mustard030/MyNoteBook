## docker下安装
### 1.拉取镜像
```bash
docker pull jenkins/jenkins:lts
```
### 2.创建数据目录映射
```bash
mkdir -p /mydata/jenkins_home
```
```bash
chown -R 1000 /mydata/jenkins_home/
```
### 3.运行容器
```bash
docker run -di --name=jenkins -p 8080:8080 -v /mydata/jenkins_home/:/var/jenkins_home jenkins/jenkins:lts
```

初始密码会放在`/mydata/jenkins_home/secrets/initialAdminPassword`中
```bash
cat /mydata/jenkins_home/secrets/initialAdminPassword
```
### 4.安装插件
在 系统管理->插件管理 中搜索SSH安装
### 5.初始化
- 浏览器访问`http://yourIPAddress:8080`输入上述密码进入管理界面创建初始管理员账户
- 在 系统管理->全局工具配置 配置jdk和maven
	- 安装jdk时提示使用Orcale账号 2696671285@qq.com Oracle123
- 在 配置->凭据->全局->添加凭据 中[创建SSH](#4.安装插件)用的账号和密码
- 系统管理->系统配置->SSH remote hosts
	- Hostname: ip地址
	- Port: 22
	- Credentials: 刚刚添加的SSH账号
	- 点击Check connection
	- 保存

**可修改maven镜像源**
```bash
cd /mydata/jenkins_home/tools/hudson.task.Maven_MavenInstallation/mavenx.x.x/conf
vim settings.xml
```
```xml
<mirrors>
	<mirror>
		<id>nexus-aliyun</id>
		<mirrorOf>central</mirrorOf>
		<name>Nexus aliyun</name>
		<url>https://maven.aliyun.com/nexus/content/groups/public</url>
	</mirror>
</mirrors>
```
### 6.部署应用
#### 6.1编写构建脚本
##### 创建脚本存放目录
```bash
mkdir -p /usr/local/jenkins/
```
##### 创建jenkins.sh脚本
脚本存放目录
```bash
cd /usr/local/jenkins/
```
创建脚本
```bash
vim jenkins.sh
```
脚本示例
```shell
#!/usr/bin/env bash
app_name='jenkinsdemo'
docker stop ${app_name}
echo '----stop container----'
docker rm ${app_name}
echo '----rm container----'
docker run -di --name=${app_name} -p 7070:7070 test/${app_name}:1.0-SNAPSHOT
echo '----start container----'
```
添加执行权限
```bash
chmod +x ./jenkins.sh
```
#### 6.2创建一个任务
任务名自定
##### 源码管理
填写仓库URL和添加用户
##### 构建
- 添加构建步骤 调用顶层Maven目标 高级
	- 目标: clean package
	- POM: pom.xml
- 增加构建步骤 Execute shell script on remote host using ssh
	- SSH site: root@IPAddress:22
	- Command: /usr/local/jenkins/jenkins.sh
##### 保存&运行
