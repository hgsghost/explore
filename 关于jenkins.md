# 关于jenkins学习记录(初级)

## 1. 理论基础

### 1.1  敏捷开发模型

通过将瀑布开发模型模型中大开发周期拆分成多个小周期的开发来达到敏捷开发的目的,其核心是迭代开发和增量开发

#### 1.1 .1迭代开发

迭代开发的思路是将一个大项目拆分成多次迭代,例如先完成一个实现大体功能的小项目,然后发现问题,慢慢迭代成一个成熟的产品,侧重于通过迭代来完成目标

#### 1.1.2 增量开发

增量开发的侧重点在于按照功能拆分,将一个大模块拆分成多个小模块之后,通过逐个小模块的开发来完成,这样可以保证用户的感知度,其中的每个模块都有完整的软件开发流程(需求分析->设计->开发->测试->进化)

#### 1.1.3 个人理解 

感觉迭代开发更倾向去项目层面的多次迭代(横向拆分),增量开发倾向去按照功能的拆分(纵向拆分)

#### 1.1.4 敏捷开发模型优点

1. 早期交付,尽早部分回款
2. 降低风险,完成部分模块遇到的问题可以给其他模块借鉴解决

### 1.2 持续集成(continuous integration)

![](resource/continuousintegration.png)

持续集成是指频繁的将代码集成的主干分支上

目的是让产品可以在保证质量的情况下让产品快速迭代

为达到以上目的,每次代码提交到主干分支之前,都会进行自动化测试,来保证代码质量

**持续集成是敏捷开发得以落实的基础**

#### 1.2.1 持续集成的流程

提交->一轮测试(自动化测试)->构建->二轮测试(对第一轮测试的补充完善,也可以把第一轮测试并到第二轮)->部署->问题回滚

#### 1.2.2 持续集成组成要素

1. 一个自动化的构建过程完成以下流程

   检出代码->编译构建->运行测试->结果记录->测试统计

2. 一个代码仓库(现在使用codeup)

3. 一个持续集成服务器(jenkins)

## 2.  jenkins

持续集成服务器

![](resource/jenkinsDeploy.jpg)

鉴于实际情况,上图中gitlib对应codeup,我们也不需要tomcat,直接打成jar包即可

### 2.1 jenkins的安装(centos 环境)

所有环境均基于docker 模拟操作

**安装jdk**

1. 构建centos环境

   ` docker pull centos:7`

   `docker run -it -d -p 8888:8080 --privileged=true --name centos  centos:7 /usr/sbin/init `

2. 进入容器

   `docker exec -it centos /bin/bash`

3. 安装 jenkins

   1. 更新yum源  `yum update`

   2. `yum install wget`

   3. `yum install fontconfig java-11-openjdk` **注意 使用oracle解压的方式安装jdk会使jenkins启动报错,如果想用oracle-jdk 需要使用rpm方式安装**

   4. `wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo`

   5. `rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key`

   6. `yum install jenkins`

   7. `vi /etc/sysconfig/jenkins`

      `JENKINS_USER="root"`

   8. `systemctl daemon-reload`  加载配置文件
   9. `systemctl start jenkins`![](resource/jenkinsInit.jpg)
   10. 跳过插件安装(选择插件安装->选择无插件)
   11. 创建管理员账户

4. 安装jenkins插件

   例:汉化插件

   ![](resource/jenkinsChinesePlugin.png)

   ![](resource/jenkinsChinesePlugin2.png)

   ![](resource/jenkinsChinesePluginResult.png)

5. 权限管理

   通过安装插件实现权限管理的功能

   1. 安装插件![](resource/jenkinsRoleBasePlugin.png)

   2. 系统管理->全局安全配置->选择 role-Based Strategy->应用保存

      ![](resource/jenkinsRoleBased.png)

   3. 系统管理->Manage and Assign Roles->manage Roles

      根据情况添加角色,分为三种角色

      + global roles 全局角色
      + item roles 项目角色
      + node roles 节点角色

      ![](resource/jenkinsRoles.png)

   4. 创建两个用户

      ![](resource/jenkinsAddUser.png)

   5. 分配权限

      ![](resource/jenkinsAllocateRole.png)

   6. 创建两个任务

   7. 结果

      ![](resource/jenkinsUser1.png)

6. 凭证管理

   1. 安装插件 Credentials Binding

   2. 安装插件 git

   3. 所在的docker容器中安装git   `yum install git`

   4. 添加凭证

      系统管理->凭证管理->点击域(全局)添加凭证

      ![](resource/jenkinsNewCredentials.png)

   5. 随便创建一个项目,

      点击配置->源码管理->选择git->设置ssh的url->选择刚才创建的凭证

   6. 在工程中点击构建后有如下日志

      ![](resource/jenkinsGitBuild.png)

      说明项目构建成功,项目目录在`/var/lib/jenkins/workspace`下

7. maven构建

   1. 容器中安装maven

      `mkdir /software`

      `cd /software `

      `wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.8.5/binaries/apache-maven-3.8.5-bin.tar.gz`

      `tar -zxvf apache-maven-3.8.5-bin.tar.gz`

      //这里可以配置一下自己的setting文件

      在 etc/profile 中配置

      `export PATH=$PATH:/software/apache-maven-3.8.5/bin`

      ` source /etc/profile`

      ![](resource/jenkinsMvnV.png)

   2. 配置全局变量

      系统管理->系统配置->全局属性->环境变量

      ![](resource/jenkinsEnviroments.png)

      ```
      JAVA_HOME
      /usr/lib/jvm/java-11-openjdk-11.0.14.1.1-1.el7_9.x86_64
      M2_HOME
      /software/apache-maven-3.8.5
      PATH+EXTRA
      $M2_HOME/bin
      ```

      

   3. 打开要构建的项目->配置->构建->执行shell

      `mvn clean package`

      构建之后可以看到maven正常执行

      ![](resource/jenkinsMaven.png)

      完成后去项目下面的target目录可以看到构建完成之后的文件

### 2.2 jenkins的使用

#### 2.2.1 jenkins支持的项目类型

jenkins中**常用**的构建类型有以下三种

+ 自由风格软件项目(FreeStyle Project)
+ Maven项目(Maven Project)
+ 流水线项目(Pipeline Project)

在之前的操作中 我们就使用了自由风格的软件项目,下面说明一下Mave项目和 **流水线项目**

个人认为其实自由风格和maven项目差距不大,都能完成基础的发布

**流水线项目则较难上手,但是灵活度极高**

#### 2.2.2 Maven项目

1. 安装插件 `Maven Integration`

   新建任务的时候选择maven项目

   build中的goals and options中输入

   `clean package -U -Dmaven.test.skip=true`

   其中-U 是强制更新jar包

2. 全局工具配置

   系统管理->全局工具配置->maven

   ![](resource/jenkinsMavenConfig.png)

   ```
   maven 3.8.5
   /software/apache-maven-3.8.5
   ```

   (同上)系统管理->全局工具配置->jdk

   ```
   jdk11
   /usr/lib/jvm/java-11-openjdk-11.0.14.1.1-1.el7_9.x86_64
   ```

   注:可能会提示**不是 JDK 目录** 忽略即可

   

3. 安装插件 `Publish Over SSH`

   在系统管理->系统配置->public over ssh中配置需要发送的服务器配置

   ![](resource/jenkinsCompanySsh.png)

4. 在项目的配置中添加构建后操作

   ![](resource/jenkinsSshConfig.png)

   以上步骤可以将maven项目推送到指定服务器目录下并执行

#### 2.2.3 pipeline项目

pipeline以代码的形式实现,使团队可以编辑,审查,迭代其流程,并且极大提高了 部署流程的灵活性

##### 2.2.3.1 然后创建pipeline

1. pipe脚本由groovy语言实现
2. pipeline支持两种语法:declarative(声明式)和scripted pipeline(脚本式)
3. pipeline有两种创建方式
   + 在jenkins web页面中输入脚本
   + 通过项目中的jenkinsfile文件控制

##### 2.2.3.2 pipeline 示例

1. 安装pipeline插件

2. 创建一个pipeline(流水线)任务

3. 创建完成后发现配置多了一个流水线

   ![](resource/jenkinsPipeline.png)

4. pipeline的两种语法示例

   这两种方式能完成的效果是一样的,相对来说声明式更加常用

   ![](resource/jenkinsPipeline2Style.png)

5. pipeline 语法片段生成器

   流水线下面有个**流水线语法**的链接,点击进入pipeline语法片段生成器,可以从这里看到语法的教程和说明

   声明式语法大体分为

   声明:上图中 agent部分

   所有阶段: stages

   单个阶段: stage

   步骤: steps (我们实际的操作写在这里边)

   逻辑很简单 ,以之前的项目为例,我们可以分为3个阶段

   + 阶段1:从git拉取代码
   + 阶段2:mave 打包
   + 阶段3:发送到目标服务器,并且执行

6. 从git拉取代码

   在片段生成器中找到 checkout 将值填好就可以生成拉取的阶段代码

   ![](resource/jenkinsPipelineCheckout.png)

   生成的结果如下

   ```groovy
   checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'github', url: 'git@github.com:hgsxxx/xxxdemo.git']]])
   ```

   将结果放到steps里即可,结果如下

   ```groovy
   pipeline {
       agent any
       stages {
           stage('git pull') {
               steps {
                  checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'github', url: 'git@github.com:hgsxxx/xxxdemo.git']]])
               }
           }
       }
   }
   ```

7. 打包项目 

   打包项目通过shell script实现

   ![](resource/jenkinsPipelineMvnPackage.png)

8. 发布项目

   ![](resource/jenkinsPipelineSshPublish.png)

9. 完成后的pipeline脚本为

   ```groovy
   pipeline {
       agent any
   
       stages {
           stage('git pull') {
               steps {
                  checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'github', url: 'git@github.com:xxxx/xxxx.git']]])
               }
           }
           stage('mvn package') {
               steps {
                  sh 'mvn -Dmaven.test.failure.ignore=true clean package'
               }
           }
           stage('ssh publish') {
               steps {
                  sshPublisher(publishers: [sshPublisherDesc(configName: 'xxxx', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: './jenkinstest/restart.sh', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: 'target', sourceFiles: 'target/demo-0.0.1-SNAPSHOT.jar')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
               }
           }
           
       }
   }
   ```

10. 点击立刻构建之后  成功打印日志 示例完成

    可以在流水线步骤中查看各个阶段的时间

    ![](resource/jenkinsPipelineSteps.png)

##### 2.2.3.3  通过Jenkinsfile完成部署

Jenkinsfile其实就是将写在jenkins配置中的pipeline脚本放到项目根目录中的Jenkinsfile文件下,实现在项目通过代码中控制发布的目的

1. pipeline项目中配置的pipeline脚本复制到项目的Jenkinsfile文件中,并提交git

2. 在配置中选择**pipeline script from SCM**

3. scm选择git 并配置地址

   ![](resource/jenkinsPipelineJenkinsfile.png)

4. 再次构建,结果和上次一致

#### 2.2.4 构建触发器

##### 2.2.4.1 jenkins内置构建触发器

1. 其他工程构建后触发

   顾名思义,将需要关注的工程名称输入文本框,就可以自动在目标工程构建之后触发本工程的构建

   ![](resource/jenkinsTriggerBuildAfterProject.png)

2. 定时构建

   输入一个类cron表达式就可以定时构建

3. 轮训scm

   这个也是输入一个类cron表达式,jenkins会根据表达式定时扫描项目的git 地址,发现新的提交就会自动构建,没有提交不会构建

   **这个方案需要频繁请求git仓库,性能不佳,后面会有更好的解决方案**

4. 触发远程构建

   设定一个token之后就可以通过固定的链接+token的形式通过http请求远程触发构建

##### 2.2.4.2 高级触发器

1. web hook trigger

   这个触发器可以通过git提交事件时,github/gitlib/等类似工具的主动调用来触发工程构建

   1. 安装插件 `Generic Webhook Trigger`

   2. 在项目的配置中构建触发器中设置插件

      ![](resource/jenkinsTriggerWebhook.png)

   3. 在github中设置webhook

      ![](resource/jenkinsTriggerGithubWebhook.png)

   4. 提交代码后发现成功触发构建

##### 2.2.4.3 参数化构建

1. 工程的配置中选择参数化构建![](resource/jenkinsPipelineParamBuild.png)

2. 在pipeline脚本中通过`${参数名}`引入参数

   ![](resource/jenkinsPipelineParamUse.png)

3. 构建成功后确认可以根据参数发布相应分支

## 3. sonarqube

一个**代码审查平台**

### 3.1 环境支持

需要一个postgresql数据库(现在的版本已经不支持mysql数据库了)

`docker pull postgres`

`docker run --name postgres -d -p 5432:5432 -e POSTGRES_USER=sonar -e POSTGRES_PASSWORD=sonar -e POSTGRES_DB=sonar --privileged=true postgres`

此时可以通过本机的5432端口访问postgres

用户名 sonar 密码 sonar

### 3.2 安装sonarqube

通过docker 安装sonarqube

1. 拉去镜像

   `docker pull sonarqube:9.3-community`

2. 启动容器

   `docker run -d --name sonar -p 9090:9000 -e ALLOW_EMPTY_PASSWORD=yes --privileged=true  sonarqube:9.3-community`

   注:如果有以下报错

   ```
   vm.max_map_count [65530] is too low, increase to at least [262144]
   ```

   解决方式:**宿主机器上执行**

   ```
   wsl -d docker-desktop
   sysctl -w vm.max_map_count=262144
   ```

3. 进入容器

   `docker exec -it sonar /bin/bash`

4. 更新配置

   `vi conf/sonar.properties`

   ```
   sonar.jdbc.username=sonar
   sonar.jdbc.password=sonar
   sonar.jdbc.url=jdbc:postgresql://10.243.149.2:5432/sonar
   ```

5. 重启容器

   `docker container restart sonar`

6. 访问本地9090端口可以看到sonar成功启动

   默认用户名密码都是 admin

   ![](resource\sonarqubeStart.png)

### 3.3 jenkins 集成 sonarqube

1. jenkins 安装 `SonarQube Scanner` 插件

2. 通过全局工具配置中自动将 sonarqube scanner安装到jenkins的所在服务器

3. 在sonarqube的页面中点击自己头像选择->myaccount->security->generate tokens 生成token 

   ![](resource/sonarqubeToken.png)

4. 在jenkins中添加凭证

   ![](resource/jenkinsSonarqubeToken.png)

5. 在系统管理->系统配置中添加sonarqube的服务器地址

   ![](resource/jenkinsSonarqubeServerAddress.png)

### 3.4 freestyle/maven项目使用sonarqube

安装插件之后,项目的配置中

post steps中可以看到 execute sonarqube scanner选项

![](resource/jenkinsMavenSonarqube.png)

在项目下创建`sonar-project.properties `文件

配置如下

```
sonar.projectKey = jenkins_test_key
sonar.projectName = jenkins_test_name
sonar.projectVersion = 1.0
# 待扫描路径
sonar.sources = .
sonar.exclusions = **/test/**,**/target/**

sonar.java.source = 1.8
sonar.java.target = 1.8
sonar.sourceEncoding = UTF-8
sonar.java.binaries = target/classes
```

工程配置如下

![](resource/jenkinsSonarqubePostSteps.png)

点击立刻构建 发现执行成功

此时去sonarqube中可以看到刚才构建项目的分析结果

![](resource/sonarqubeAnalysisResult.png)

### 3.5 pipeline项目使用sonarqube

这个写法官网不太好找,百度才能找到

可以在maven打包之前添加代码审核阶段

代码如下

```groovy
pipeline {
    agent any

    stages {
        stage('git pull') {
            steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: 'github', url: 'git@github.com:xxxxx/xxxx.git']]])
            }
        }
        stage('check code'){
            steps {
                script{
                    //引入sonarqube scanner
                    //这是全局工具设置中 新增sonarqube scanner的那个名字
                    scannerHome= tool 'sonarqube scanner'
                }
                //这是系统设置中 SonarQube installations的名字
                withSonarQubeEnv('sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        stage('mvn package') {
            steps {
               sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }
        }
        stage('ssh publish') {
            steps {
               sshPublisher(publishers: [sshPublisherDesc(configName: 'company 252', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'bash /jenkinstest/restart.sh', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: 'target', sourceFiles: 'target/demo-0.0.1-SNAPSHOT.jar')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
            }
        }
      
    }
}
```

构建成功并且可以在sonarqube 查看到分析结果

## 4 实战部署

实战中,我们直接部署jenkins镜像,不再使用centos镜像创建

1. linux服务器安装docker [官方地址](https://docs.docker.com/engine/install/centos/)

2. 在宿主服务器上创建如下目录

   `/software/jenkins/jenkins_home`

3. 启动容器

   `docker run -itd --name jenkins -p 8888:8080 -p 50000:50000 --restart always -v /software/jenkins/jenkins_home:/var/jenkins_home -u root jenkins/jenkins:lts-jdk11`

4. 暂时配置的管理员账户和密码都是admin

5. 安装上述所有的插件

6. 创建测试工程app-training-maven 和app-training-pipeline

7. 根据上述教程安装sonarqube  (用户名为 admin 密码为 123)

8. 测试通过







