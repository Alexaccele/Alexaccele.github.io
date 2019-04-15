---
title: Docker的使用以及命令总结
tags: Docker
categories: Docker
---

## 帮助相关

- docker –help 可以查看docker的所有命令（注意单词全拼的参数前都是两个‘-’号）
- docker info 查看docker信息
- docker version 查看docker版本号

### 最有用的帮助命令
docker XXX –help 查看具体命令的帮助手册（XXX代指各种命令）

## 镜像相关
### docker images 查看所有镜像
参数说明：

- -a :列出本地所有的镜像（含中间映像层）
- -q :只显示镜像ID。
- –digests :显示镜像的摘要信息
- –no-trunc :显示完整的镜像信息

### docker search XXX 根据关键字搜索docker镜像
参数说明：

- -s : 列出收藏数不小于指定值的镜像。
- –limit int ：最大显示结果数(默认25个)
- –no-trunc : 显示完整的镜像描述
- –automated : 只列出 automated build类型的镜像；


### docker pull XXX 下载镜像
可以使用docker pull XXX:Tag 具体指出下载某个Tag的镜像

### docker rmi ID 删除ID所指的镜像
参数说明：

-f：强制删除

例子：

- 删除单个镜像：docker rmi -f ID
- 删除多个镜像：docker rmi -f 镜像名1:TAG 镜像名2:TAG
- 删除全部镜像：docker rmi -f $(docker images -qa)


## 容器相关
### docker run [OPTIONS] IMAGE [COMMAND] [ARG…] 新建并启动容器
参数说明：

- –name=”容器新名字”: 为容器指定一个名称
- -d: 后台运行容器，并返回容器ID，也即启动守护式容器
- -i：以交互模式运行容器，通常与 -t 同时使用
- -t：为容器重新分配一个伪输入终端，通常与 -i 同时使用
- -P: 随机端口映射
- -p: 指定端口映射，有以下四种格式

- ip:hostPort:containerPort
- ip::containerPort
- hostPort:containerPort
- containerPort
启动交互式容器：使用镜像centos:latest以交互模式启动一个容器,在容器内执行/bin/bash命令。`docker run -it centos /bin/bash`

后台运行容器：`docker run -d centos`

### docker ps 查看所有正在运行的容器
参数说明：

- -a :列出当前所有正在运行的容器+历史上运行过的
- -l :显示最近创建的容器。
- -n：显示最近n个创建的容器。
- -q :静默模式，只显示容器编号。
- –no-trunc :不截断输出。
### 退出容器
- exit：容器停止退出
- Ctrl+P+Q：容器不停止退出
### 容器的启动停止和重启
- docker start ID/容器名 启动容器
- docker restart ID/容器名 重启容器
- docker stop ID/容器名 停止容器
- docker kill ID/容器名 强制停止容器

stop和kill的区别：stop是类似于电脑关机，会进行程序的一系列关闭操作
kill则类似于直接断电，强制停止容器，不进行一系列程序关闭所需要执行的操作

### 删除容器
- docker rm -f ID 删除指定ID的容器
- docker rm -f $(docker ps -q)批量删除全部正在运行的容器

### 重新进入容器的方法
docker exec -it ID BashShell 进入正在运行的容器并以命令行交互
例如：`docker exec -it ID /bin/bash`

docker attach ID 重新进入容器

区别：
attach 直接进入容器启动命令的终端，不会启动新的进程
exec 是在容器中打开新的终端，并且可以启动新的进程

### 从容器中拷贝文件到主机上
docker cp ID:容器内路径 目标主机路径
例如：`docker cp ID:/usr /home`

### docker commit 提交指定容器副本成为一个新镜像
    docker commit -m=“提交的描述信息” -a=“作者” 容器ID 要创建的目标镜像名:[标签名]
应用场景：当我们从hub上下载的镜像在本地成功运行后，对其中的内容进行了修改，例如配置参数等，当完成我们的修改后，可以使用docker commit命令，为这个修改后的容器生成一个新的镜像，从而当我们运行这个镜像时，里面的内容就是我们修改好的。

### 容器相关细节命令

    docker logs -f -t –tail ID 查看容器日志

参数说明：

- -t 是加入时间戳
- -f 跟随最新的日志打印
- –tail 数字 显示最后多少条

docker top ID 查看容器内进程

docker inspect ID 查看容器内部细节（JSON形式表示）

## 容器数据卷
容器数据卷是为了持久化容器中的数据而设计的，数据卷是完全独立于容器的生命周期，不会因为容器的删除而消失。

数据卷存在于宿主机中，独立于容器，和容器的生命周期是分离的，数据卷存在于宿主机的文件系统中，数据卷可以目录也可以是文件，容器可以利用数据卷与宿主机进行数据共享，实现了容器间的数据共享和交换。

数据卷的特点：

1. 容器启动的时候初始化的，如果容器使用的镜像包含了数据，这些数据也会拷贝到数据卷中。
1. 容器对数据卷的修改是及时进行的。
1. 数据卷的变化不会影响镜像的更新。数据卷是独立于联合文件系统，镜像是基于联合文件系统。镜像与数据卷之间不会有相互影响。
1. 数据卷是宿主机中的一个目录，与容器生命周期隔离。
### 命令添加
使用-v可以挂载一个本地的目录到容器中作为数据卷。
    docker run -it -v /宿主机绝对路径目录:/容器内目录 镜像名

使用-v运行的容器会在宿主机和容器内对应位置创建文件目录，此时若使用docker inspect命令则可以看到，两个目录已经成功连接、挂载并且同时具有读写权限，此时的容器和宿主机中的对应目录是数据共享的，并且在容器停止后，数据依然共享，且在容器重新启动后数据仍保持一致。

可以为数据卷添加权限 `docker run -it -v /宿主机绝对路径目录:/容器内目录:ro 镜像名` （ro表示Read-Only）此时的数据卷在容器中只能查看，不能修改。在宿主机中则可以修改。

### DockerFile添加
可在Dockerfile中使用VOLUME指令来给镜像添加一个或多个数据卷

即在DockerFile中添加如下指令：


    VOLUME["/dataVolumeContainer","/dataVolumeContainer2","/dataVolumeContainer3"]
例如：

    # volume test
    FROM centos
    VOLUME ["/dataVolumeContainer1","/dataVolumeContainer2"]
    CMD echo "finished,--------success1"
    CMD /bin/bash
在写好dockerfile文件后，则可以使用docker build命令来生成镜像，执行命令如下：

`docker build -f /mydocker/dockerfile -t xxx/centos .`（‘.’表示当前目录下）
-f:指定dockerfile文件的位置

-t:生成的镜像命名空间

此时就生成了一个名为xxx/centos的镜像，运行后就能通过docker inspect命令查看容器的数据卷对应的宿主机目录。


> 注意：Docker挂载主机目录Docker访问出现cannot open directory .: Permission denied
> 
> 解决办法：在挂载目录后多加一个–privileged=true参数即可

## 数据卷容器
命名的容器挂载数据卷，其它容器通过挂载这个(父容器)实现数据共享，挂载数据卷的容器，称之为数据卷容器。
数据卷容器挂载了一个本地目录，其他容器连接这个容器来实现数据的共享（数据地址的拷贝）。

首先需要一个父容器作为数据卷容器


    docker run -it --name dc01 xxx/centos
再使用--volumes-from命令则使得子容器继承自父容器


    docker run -it --name dc02 --volumes-from dc01 xxx/centos
此时父容器dc01和子容器dc02中的数据就共享了，且所有修改都能同步。
即使删除任意一个父容器或子容器，数据也仍然共享。

结论：容器之间配置信息的传递，数据卷的生命周期一直持续到没有容器使用它为止

## DockerFile
Dockerfile是用来构建Docker镜像的构建文件，是由一系列命令和参数构成的脚本。

### 基本内容
1. 每条保留字指令都必须为大写字母且后面要跟随至少一个参数
1. 指令按照从上到下，顺序执行
1. #表示注释
1. 每条指令都会创建一个新的镜像层，并对镜像进行提交

### 执行大致流程
1. docker从基础镜像运行一个容器
1. 执行一条指令并对容器作出修改
1. 执行类似docker commit的操作提交一个新的镜像层
1. docker再基于刚提交的镜像运行一个新容器
1. 执行dockerfile中的下一条指令直到所有指令都执行完成


### 保留字
- FROM：基础镜像，当前新镜像是基于哪个镜像的
- MAINTAINER：镜像维护者的姓名和邮箱地址
- RUN：容器构建时需要运行的命令
- EXPOSE：当前容器对外暴露出的端口
- WORKDIR：指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点
- ENV：用来在构建镜像过程中设置环境变量
- ADD：将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
- COPY：类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置
- VOLUME：容器数据卷，用于数据保存和持久化工作
- CMD：指定一个容器启动时要运行的命令；Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效，CMD 会被 docker run 之后的参数替换
- ENTRYPOINT：指定一个容器启动时要运行的命令；ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数
- ONBUILD：当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发


### dockerfile案例

自定义镜像mycentos

    FROM centos  
    MAINTAINER zzyy<zzyy167@126.com>
    ENV MYPATH /usr/local
    WORKDIR $MYPATH
    RUN yum -y install vim
    RUN yum -y install net-tools
    EXPOSE 80
    CMD echo $MYPATH
    CMD echo "success--------------ok"
    CMD /bin/bash 


自定义镜像Tomcat9

    FROM centos
    MAINTAINERzzyy<zzyybs@126.com>
    #把宿主机当前上下文的c.txt拷贝到容器/usr/local/路径下
    COPY c.txt /usr/local/cincontainer.txt
    #把java与tomcat添加到容器中
    ADD jdk-8u171-linux-x64.tar.gz /usr/local/ADD apache-tomcat-9.0.8.tar.gz /usr/local/
    #安装vim编辑器
    RUN yum -y install vim
    #设置工作访问时候的WORKDIR路径，登录落脚点
    ENV MYPATH /usr/local
    WORKDIR $MYPATH
    #配置java与tomcat环境变量
    ENV JAVA_HOME /usr/local/jdk1.8.0_171
    ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.8
    ENV CATALINA_BASE /usr/local/apache-tomcat-9.0.8
    ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
    #容器运行时监听的端口
    EXPOSE  8080
    #启动时运行tomcat
    #ENTRYPOINT ["/usr/local/apache-tomcat-9.0.8/bin/startup.sh" ]
    #CMD ["/usr/local/apache-tomcat-9.0.8/bin/catalina.sh","run"]
    CMD /usr/local/apache-tomcat-9.0.8/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.8/bin/logs/catalina.out


## 总结
从应用软件的角度来看，Dockerfile、Docker镜像与Docker容器分别代表软件的三个不同阶段

- Dockerfile是软件的原材料
- Docker镜像是软件的交付品
- Docker容器则可以认为是软件的运行态。


Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。

1. Dockerfile，需要定义一个Dockerfile，Dockerfile定义了进程需要的一切东西。Dockerfile涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计namespace的权限控制)等等;
1. Docker镜像，在用Dockerfile定义一个文件之后，docker build时会产生一个Docker镜像，当运行 Docker镜像时，会真正开始提供服务;
1. Docker容器，容器是直接提供服务的。