# docker环境下借助mycat2搭建mysql集群

### 本文主要参考参考文档

[mycat2官方教程](https://www.yuque.com/books/share/6606b3b6-3365-4187-94c4-e51116894695/fb2285b811138a442eb850f0127d7ea3)

### 环境如下

jdk1.8

docker version:20.10.7

### 使用docker 搭建mysql主从

注:本文默认读者有一定的计算机基础并且已经掌握了基础的docker语法,并且拥有docker环境

> 本文使用的mysql镜像是从docker hub获取的mysql官方镜像

docker 环境下执行

  `docker pull mysql:5.6` 

获取5.6版本的mysql镜像文件

镜像文件下载完成之后启动容器,启动参数可以参考镜像说明文件

> 主从复制原来大致如下

![mysql主从复制原理图](https://images2015.cnblogs.com/blog/1043616/201612/1043616-20161213151808011-1732852037.jpg)

[mysql镜像说明文档](https://hub.docker.com/_/mysql)

> 启动主mysql的容器

`docker run -it --name master -e MYSQL_ROOT_PASSWORD=123 -d mysql:5.6 --log-bin=mysql-bin --server-id=1`

> 启动从mysql的容器

`docker run -it --name slave -e MYSQL_ROOT_PASSWORD=123 -d --link master:master  mysql:5.6 --log-bin=mysql-bin --server-id=2 --relay-log=relay-log --read_only=1 --log_slave_update=1` 

> 进入master容器(此处的id替换为读者自己的容器id)

`docker exec -it f2d3c9613b21 /bin/bash`

> 通过以下命令可以进入容器中mysql命令行

`mysql -u -root -p`

> 接下来在主从两个容器中创建用于同步数据的账号    

` GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO repl@'%' IDENTIFIED BY '123456';`

> 进入主容器中查看logbin日志当前位置(mysql 命令行执行以下命令)

`show master status\G;`

`*************************** 1. row ***************************
             File: mysql-bin.000004
         Position: 338
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)`

> 进入salve容器中创建于master的链接

`CHANGE MASTER TO MASTER_HOST='master',MASTER_PORT=3306,MASTER_USER='repl',MASTER_PASSWORD='123456',MASTER_LOG_FILE='mysql-bin.000004',MASTER_LOG_POS=338;`

> 此时通过`show slave status \G`命令可以看到刚才配置的master链接,如下

`mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 338
               Relay_Log_File: relay-log.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 338
              Relay_Log_Space: 120
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 0
                  Master_UUID:
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)`

> 检查无误后执行`start slave`命令开始复制,此时再次执行`show slave status\G`命令查看slave状态

发现

`Slave_IO_Running: Yes
Slave_SQL_Running: Yes`

io线程和sql线程已经开启

` Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it`

说明slave已经处于就绪状态

### 测试主从复制

此时在主库执行`create database testcopy;`创建一个测试库

再进入到slave库中执行 `show databases;`命令显示如下

`mysql> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| testcopy           |
+--------------------+
4 rows in set (0.00 sec)`

> 至此mysql主从搭建完成

### 搭建mycat2实现mysql读写分离

#### 制作mycat2镜像

鉴于docker hub中并没有mycat2的镜像,我决定自己制作一个mycat2镜像

我们使用容器commit的方式创建镜像文件

首先获取一个centos7镜像文件命令为` docker pull centos:8`

启动容器` docker run --name mycat2 -it -d  -p 8066:8066 --link master:master --link slave:slave centos:8`

进入容器`docker exec -it 7c4bf54d1e08 /bin/bash`

安装wget命令`yum install wget`

######  jdk安装

安装jdk `wget https://download.oracle.com/otn/java/jdk/8u301-b09/d3c52aa6bfa54d3ca74e617f18309292/jdk-8u301-linux-x64.tar.gz` 这个命令可能会过期,可自行百度centos安装jdk并配置环境变量

解压jdk

`tar -zxvf jdk-8u301-linux-x64.tar.gz`

配置环境变量

`#set java environment
JAVA_HOME=/jdk1.8.0_301
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH`
配置生效

`source /etc/profile`

验证jdk

`java -version`

###### mycat2

官网说:mycat2当前版本不提供安装包,只提供核心JAR包,JAR包可以独立运行,安装包是使用Java Service Wrapper做壳的,安装包请自己制作

所以我们需要自己下载java service wrapper做壳安装java服务

文档地址:[tar包制作](https://www.yuque.com/ccazhw/ml3nkf/gnqwyv)

我下载了1.19版本`wget http://dl.mycat.org.cn/2.0/install-template/mycat2-install-template-1.19.zip`

还需要安装zip工具`yum install zip` 安装完成后执行 `unzip mycat2-install-template-1.19.zip`后当前目录下多了一个mycat文件夹

进入mycat文件夹下的lib目录

下载1.19版本的mycat2  jar包` wget http://dl.mycat.org.cn/2.0/1.19-release/mycat2-1.19-jar-with-dependencies.jar`

现在开始修改mycat2的配置文件参考以下文档:[mycat2配置指导](https://www.yuque.com/ccazhw/ml3nkf/068fbb201c1b187203cf3dbd05f60358)

安装vim编辑器来编辑配置文件`yum install vim`

##### 修改集群配置文件

进入conf文件夹下的`vim clusters/prototype.cluster.json`修改为如下配置注(谨记,Mycat2内置的所有集群名字或者数据源名字必须有一个是prototype.该服务器用来处理非select,update,delete,insert.当Mycat2完全实现系统表的时候,该prototype服务能被去掉.)

`{
        "clusterType":"MASTER_SLAVE",
        "heartbeat":{
                "heartbeatTimeout":1000,
                "maxRetry":3,
                "minSwitchTimeInterval":300,
                "slaveThreshold":0
        },
        "masters":["master"],
        "replicas":["slave"],
        "maxCon":200,
        "name":"prototype",
        "readBalanceType":"BALANCE_ALL",
        "switchType":"SWITCH"
}`

###### 配置数据源

`cd datasources/`

`vim prototypeDs.datasource.json`

配置如下

`{
        "dbType":"mysql",
        "idleTimeout":60000,
        "initSqls":[],
        "initSqlsGetConnection":true,
        "instanceType":"READ_WRITE",
        "maxCon":1000,
        "maxConnectTimeout":3000,
        "maxRetryCount":5,
        "minCon":1,
        "name":"master",
        "password":"123",
        "type":"JDBC",
        "url":"jdbc:mysql://master:3306/mysql?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8",
        "user":"root",
        "weight":0
}`

重命名配置文件:

`mv prototypeDs.datasource.json ./master.datasource.json`

添加slave数据源

`cp master.datasource.json slave.datasource.json`

`vim slave.datasource.json`

配置为

`{
        "dbType":"mysql",
        "idleTimeout":60000,
        "initSqls":[],
        "initSqlsGetConnection":true,
        "instanceType":"READ",
        "maxCon":1000,
        "maxConnectTimeout":3000,
        "maxRetryCount":5,
        "minCon":1,
        "name":"slave",
        "password":"123",
        "type":"JDBC",
        "url":"jdbc:mysql://slave:3306/mysql?useUnicode=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8",
        "user":"root",
        "weight":0
}`

server配置

注意"serverVersion": "5.6.51-mycat-2.0",配置项中5.x.xx是mysql的版本 通过在mysql命令行中` select version(); `查看

`{
  "loadBalance":{
    "defaultLoadBalance":"BalanceRandom",
    "loadBalances":[]
  },
  "serverVersion": "5.6.51-mycat-2.0",
  "mode":"local",
  "properties":{},
  "server":{
    "bufferPool":{

    },
    "idleTimer":{
      "initialDelay":3,
      "period":60000,
      "timeUnit":"SECONDS"
    },
    "ip":"0.0.0.0",
    "mycatId":1,
    "port":8066,
    "reactorNumber":1,
    "tempDirectory":null,
    "timeWorkerPool":{
      "corePoolSize":0,
      "keepAliveTime":1,
      "maxPendingLimit":65535,
      "maxPoolSize":2,
      "taskTimeout":5,
      "timeUnit":"MINUTES"
    },
    "workerPool":{
      "corePoolSize":1,
      "keepAliveTime":1,
      "maxPendingLimit":65535,
      "maxPoolSize":1024,
      "taskTimeout":5,
      "timeUnit":"MINUTES"
    }
  }
}`

根据自己的需要配置用户

`vi users/root.user.json`

我的配置为

`{
        "dialect":"mysql",
        "ip":".",
        "password":"123",
        "transactionType":"xa",
        "username":"root"
}`

进入mycat/bin目录发现没有执行权限

`[root@bb56312f1909 bin]# ls -l
total 2588
-rw-r--r-- 1 root root  15666 Mar  5 00:33 mycat
-rw-r--r-- 1 root root   3916 Mar  5 00:33 mycat.bat
-rw-r--r-- 1 root root 281540 Mar  5 00:33 wrapper-aix-ppc-32
-rw-r--r-- 1 root root 319397 Mar  5 00:33 wrapper-aix-ppc-64
-rw-r--r-- 1 root root 253808 Mar  5 00:33 wrapper-hpux-parisc-64
-rw-r--r-- 1 root root 140198 Mar  5 00:33 wrapper-linux-ppc-64
-rw-r--r-- 1 root root  99401 Mar  5 00:33 wrapper-linux-x86-32
-rw-r--r-- 1 root root 111027 Mar  5 00:33 wrapper-linux-x86-64
-rw-r--r-- 1 root root 114052 Mar  5 00:33 wrapper-macosx-ppc-32
-rw-r--r-- 1 root root 233604 Mar  5 00:33 wrapper-macosx-universal-32
-rw-r--r-- 1 root root 253432 Mar  5 00:33 wrapper-macosx-universal-64
-rw-r--r-- 1 root root 112536 Mar  5 00:33 wrapper-solaris-sparc-32
-rw-r--r-- 1 root root 148512 Mar  5 00:33 wrapper-solaris-sparc-64
-rw-r--r-- 1 root root 110992 Mar  5 00:33 wrapper-solaris-x86-32
-rw-r--r-- 1 root root 204800 Mar  5 00:33 wrapper-windows-x86-32.exe
-rw-r--r-- 1 root root 220672 Mar  5 00:33 wrapper-windows-x86-64.exe`

通过

` chmod +x mycat` 

`chmod +x wrapper-linux-x86-64`

命令赋权



执行` ./mycat start`即可启动mycat2

执行` ./mycat status`查看mycat状态

显示`mycat2 is running (436).`

> 此时 mycat2已经配置完成

使用navicat链接本地8066端口可以跟访问普通mysql一样访问mycat