# sonarqube docker方式安装部署

### Sonarqube介绍

[SonarQube](http://www.sonarqube.org/) ® 是一种自动代码审查工具，用于检测代码中的错误、漏洞和代码异味。它可以与您现有的工作流程集成，以支持跨项目分支和拉取请求的持续代码检查。详细文档请参阅[此处](https://docs.sonarqube.org/latest/)。

**sonarqube支持三种数据库：sql server、oracle、postgresql。**本文介绍的使用postgresql作为数据库引擎。

### 创建挂载文件夹

```shell
cd /opt
mkdir sonar 
cd sonar
mkdir postgres sonarqube
cd sonarqube
mkdir data extensions logs
```

### 创建自定义网络

```shell
docker network create sonar
```

### 安装postgresql

```shell
docker run -d  \
	--name=postgresql \
	-p 5432:5432 \
	--net=sonar \
	-v /opt/sonar/postgres:/var/lib/postgresql/data \
	-e POSTGRES_USER=sonar \
	-e POSTGRES_PASSWORD=sonar \
	postgres:10
```

### 安装sonarqube

#### 挂载参数

- `/opt/sonarqube/data`：数据文件，例如嵌入式 H2 数据库和 Elasticsearch 索引。
- `/opt/sonarqube/extensions`：将包含您安装的任何插件和必要时的 Oracle JDBC 驱动程序。
- `/opt/sonarqube/logs`：有关访问、Web 进程、CE 进程和 Elasticsearch 的 SonarQube 日志。

#### docker安装命令

```shell
docker run -d
	--name sonarqube \
	--link postgresql:postgres \
	--net=sonar \
	-e SONARQUBE_JDBC_URL=jdbc:postgresql://postgres:5432/sonar \        
	-e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \        
	-e SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql:5432/sonar \        
	-e SONARQUBE_JDBC_USERNAME=sonar \        
	-e SONARQUBE_JDBC_PASSWORD=sonar \        
	-p 9000:9000  \       
	-v /opt/sonar/sonarqube/data:/opt/sonarqube/data \        
	-v /opt/sonar/sonarqube/extensions:/opt/sonarqube/extensions \        
	-v /opt/sonar/sonarqube/logs:/opt/sonarqube/logs \        
	sonarqube:8.6.0-community
```

### 安装时可能遇到的问题

#### 启动时可能会出现es的错误

错误提示：max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144] ， 

解决方法：

编辑 /etc/sysctl.conf，追加以下内容：

```go
vm.max_map_count=262144
```

保存后，执行：

sysctl  -p

重启docker

```
systemctl restart docker
```

### 开放端口

```bash
firewall-cmd --zone=public --add-port=9000/tcp --permanent
systemctl reload firewalld
```

### 登录

http://localhost:9000初始化用户名密码为`admin/admin`。

### 中文在线切换

点击 Administration-> Marketplace

在插件搜索框输入chinese搜索 点击插件右侧install 开始安装。

安装完成后会看到重启提示。

重启后跳转到登陆页面。

登陆即可。

### 参考资料

https://docs.sonarqube.org/latest/setup/install-server/

https://www.cnblogs.com/xiaobotester/p/13906142.html





