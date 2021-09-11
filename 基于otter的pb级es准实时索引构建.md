# 基于otter的pb级es准实时索引构建

[what is otter?](https://github.com/alibaba/otter/wiki)

> 本文基于docker构建,并且默认读者掌握基本docker命令



#### 首先要构建一个基于docker的mysql容器(mysql复制模式需要row类型)

获取mysql镜像

`docker pull mysql:5.6`

启动容器

`docker run -it --name ottermaster -e MYSQL_ROOT_PASSWORD=123 -d mysql:5.6 --log-bin=mysql-bin --server-id=1 --binlog_format=ROW`

在mysql容器中的mysql客户端执行以下链接的sql文件

```
https://raw.github.com/alibaba/otter/master/manager/deployer/src/main/resources/sql/otter-manager-schema.sql 
```

#### zookeeper集群

> otter node依赖于zookeeper进行分布式调度，需要安装一个zookeeper节点或者集群.

下载镜像

`docker pull zookeeper:latest`

此处使用docker compose搭建zookeeper集群

参考链接[docker compose 搭建zookeeper集群](https://zhuanlan.zhihu.com/p/121728783)

首先创建docker-compose.yml配置文件

```yaml
version: '3.7'

# 给zk集群配置一个网络，网络名为zk-net
networks:
  otter-net:
    name: otter-net

# 配置zk集群的
# container services下的每一个子配置都对应一个zk节点的docker container
services:
  zk1:
    # docker container所使用的docker image
    image: zookeeper
    hostname: zk1
    container_name: zk1
    # 配置docker container和宿主机的端口映射
    ports:
      - 2181:2181
      - 8081:8080
    # 配置docker container的环境变量
    environment:
      # 当前zk实例的id
      ZOO_MY_ID: 1
      # 整个zk集群的机器、端口列表
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181
    # 当前docker container加入名为zk-net的隔离网络
    networks:
      - otter-net

  zk2:
    image: zookeeper
    hostname: zk2
    container_name: zk2
    ports:
      - 2182:2181
      - 8082:8080
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zk3:2888:3888;2181
    networks:
      - otter-net

  zk3:
    image: zookeeper
    hostname: zk3
    container_name: zk3
    ports:
      - 2183:2181
      - 8083:8080
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    networks:
      - otter-net
```

启动容器

`docker-compose -f 相应目录\docker-compose.yml up -d`

我们可以在宿主机上通过浏览器访问这三个zk实例内嵌的web控制台

>http://localhost:8081/commands
>
>http://localhost:8082/commands
>
>http://localhost:8083/commands

将前面的mysql也链接到otter-net网络

`docker network connect otter-net ottermaster`

可以通过命令`docker network inspect otter-net`查看网络otter-net的状态

```
[
    {
        "Name": "otter-net",
        "Id": "4fe880dff2efc25eb65a9be5bdeed94791c3e740fd403dcfcceb90ded2831d39",
        "Created": "2021-09-08T13:52:06.2676962Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "1e80b09767c624175e52051d014f5cbb9a61a2a0a9c4094cba46d3edd170881a": {
                "Name": "zk3",
                "EndpointID": "32cf349c31309a40a98bd543b8f3f027ec9a4ae7e55e4f9f5084ff9f44ea3397",
                "MacAddress": "02:42:ac:14:00:02",
                "IPv4Address": "172.20.0.2/16",
                "IPv6Address": ""
            },
            "2c59e582a1cacf1c5ee27a860aeacda2ba46660ee150c0b1ab2fa1af53155071": {
                "Name": "zk2",
                "EndpointID": "682c7768fec329f1bdd8d02d2c96cab840594d98ab0c3938ccc0167a426c4b51",
                "MacAddress": "02:42:ac:14:00:04",
                "IPv4Address": "172.20.0.4/16",
                "IPv6Address": ""
            },
            "35fa1e1071322af90f9d89b264a1afc2d4e7f39b6dec31615d15ec4ad4cd6aea": {
                "Name": "ottermaster",
                "EndpointID": "dd118ea6a32b7c76ae57456ef2fdbe36ce2ad5d1e290b18a16eeb856b13db818",
                "MacAddress": "02:42:ac:14:00:05",
                "IPv4Address": "172.20.0.5/16",
                "IPv6Address": ""
            },
            "8bf5057e35ed92dcee6f764cac0e1b8d868aed33ee5899222a707733d0eb7bad": {
                "Name": "zk1",
                "EndpointID": "380d80b4c31d1377e79dda70c932cff77f3517593162dd8b6fbb277884878625",
                "MacAddress": "02:42:ac:14:00:03",
                "IPv4Address": "172.20.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "otter-net",
            "com.docker.compose.project": "dockerspace",
            "com.docker.compose.version": "2.0.0"
        }
    }
]
```

可见zk1-3,ottermaster均连接到网络

## 下载   otter manager

我们使用一个centos7 来构建otter manager容器

`docker pull centos:7`

创建容器

docker run -it -d -p 8080:8080 --name ottermanager centos:7 

进入容器`docker exec -it ottermanager /bin/bash`

更新yum源 `yum update`

安装wget命令`yum install wget`

创建ottermanagert的安装路径`mkdir ottermanager`

进入ottermanagert目录`cd ottermanager`

下载相应的版本`wget https://github.com/alibaba/otter/releases/download/otter-4.2.18/manager.deployer-4.2.18.tar.gz` 我用的4.2.18

创建manager目录`mkdir manager`

解压`tar zxvf manager.deployer-4.2.18.tar.gz -C manager`

去conf中修改配置文件 otter.properties

```
## otter manager domain name
otter.domainName = 127.0.0.1
## otter manager http port
otter.port = 8080
## jetty web config xml
otter.jetty = jetty.xml

## otter manager database config
otter.database.driver.class.name = com.mysql.jdbc.Driver
otter.database.driver.url = jdbc:mysql://ottermaster:3306/otter
otter.database.driver.username = root
otter.database.driver.password = 123

## otter communication port
otter.communication.manager.port = 1099

## otter communication payload size (default = 8388608)
otter.communication.payload = 8388608

## otter communication pool size
otter.communication.pool.size = 10

## default zookeeper address
otter.zookeeper.cluster.default = zk1:2181
## default zookeeper sesstion timeout = 60s
otter.zookeeper.sessionTimeout = 60000

## otter arbitrate connect manager config
otter.manager.address = ${otter.domainName}:${otter.communication.manager.port}

## should run in product mode , true/false
otter.manager.productionMode = true

## self-monitor enable or disable
otter.manager.monitor.self.enable = true
## self-montir interval , default 120s
otter.manager.monitor.self.interval = 120
## auto-recovery paused enable or disable
otter.manager.monitor.recovery.paused = true
# manager email user config
otter.manager.monitor.email.host = smtp.gmail.com
otter.manager.monitor.email.username =
otter.manager.monitor.email.password =
otter.manager.monitor.email.stmp.port = 465
```

### 将manager加入otter-net网络(在运行docker的主机上执行)

`docker network connect otter-net ottermanager`

### 还需要构建jdk环境(读者自行百度解决)

`java -version` 查看java版本无误后启动

### 去manager的bin目录中启动otter manager

注意:执行启动命令后发现找不到java命令不知道为什么,我的解决方案是修改startup.sh文件为

> 在## set java path下面添加`JAVA="/jdk/jdk1.8.0_301/bin/java"`

进入bin目录中` ./startup.sh`

在/logs目录中 manager.log文件中 如果看到以下输出说明启动成功

````
2021-09-08 15:04:30.570 [] INFO  com.alibaba.otter.manager.deployer.JettyEmbedServer - ##Jetty Embed Server is startup!
2021-09-08 15:04:30.570 [] INFO  com.alibaba.otter.manager.deployer.OtterManagerLauncher - ## the manager server is running now ......
````

此时访问本地的8080端口可以看到otter控制台

在控制台登录账户(默认账户密码都是admin)

机器管理中添加zookeeper集群

集群的ip地址可以通过进入ottermanager容器中ping zk1-3来获取

我的集群地址为   172.18.0.4:2181, 172.18.0.2:2181, 172.18.0.3:2181

## 下载otter-node

我们使用一个centos7 来构建otter node容器

`docker pull centos:7`

创建容器

docker run -it -d  --name otternode1 centos:7 

进入容器`docker exec -it otternode1 /bin/bash`

更新yum源 `yum update`

安装wget命令`yum install wget`

创建ottermanagert的安装路径`mkdir otternode`

进入ottermanagert目录`cd otternode`

下载相应的版本`wget https://github.com/alibaba/otter/releases/download/otter-4.2.18/node.deployer-4.2.18.tar.gz` 我用的4.2.18

创建manager目录`mkdir node`

解压`tar -zxvf node.deployer-4.2.18.tar.gz -C node`

nid配置 (将环境准备中添加node节点后节点列表的序号，保存到conf目录下的nid文件，比如我添加的机器对应序号为1)

`echo 1 > conf/nid`

进入conf目录中修改otter.properties文件为

```
# otter node root dir
otter.nodeHome = ${user.dir}/../

## otter node dir
otter.htdocs.dir = ${otter.nodeHome}/htdocs
otter.download.dir = ${otter.nodeHome}/download
otter.extend.dir= ${otter.nodeHome}/extend

## default zookeeper sesstion timeout = 60s
otter.zookeeper.sessionTimeout = 60000

## otter communication payload size (default = 8388608)
otter.communication.payload = 8388608

## otter communication pool size
otter.communication.pool.size = 10

## otter arbitrate & node connect manager config
otter.manager.address = ottermanager:1099
```

### 将node加入网络(在docker运行的主机中运行)

`docker network connect otter-net otternode1`

### 还需要构建jdk环境(读者自行百度解决)

`java -version` 查看java版本无误后启动

### 去manager的bin目录中启动otter node

注意:执行启动命令后发现找不到java命令不知道为什么,我的解决方案是修改startup.sh文件为

> 在## set java path下面添加`JAVA="/jdk/jdk1.8.0_301/bin/java"`

进入bin目录中` ./startup.sh`

在/logs目录中 node.log文件中 如果看到以下输出说明启动成功

```
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
2021-09-09 09:55:12.779 [main] INFO  com.alibaba.otter.node.deployer.OtterLauncher - INFO ## the otter server is running now ......
```

此时在manager的管理界面可以看到node已经启动

![node启动](https://camo.githubusercontent.com/c9597b9237d0db6161360075b7ff8c084a92882d8b9a63499cf9e381c3c23eb5/687474703a2f2f646c322e69746579652e636f6d2f75706c6f61642f6174746163686d656e742f303038382f313930342f66616531343964362d383739302d336133622d616635382d3239383163383738346334652e706e67)

