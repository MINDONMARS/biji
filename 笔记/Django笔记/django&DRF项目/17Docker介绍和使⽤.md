# Docker介绍和使⽤

[TOC]

## 1.Docker介绍

**特点**
上⼿快
职责的逻辑分类
快速⾼效的开发⽣命周期
⿎励使⽤⾯向服务的架构

### 1.Docker中⽂社区 

http://www.docker.org.cn/index.html

### 2.简介

使⽤的是容器技术

- 容器和宿主机之间的隔离更加彻底
- 容器有独⽴的⽹络和存储栈
- 容器拥有⾃⼰的资源管理能⼒
- 同⼀台宿主机中的多个容器可以友好的共存

直接在宿主主机的操作系统上调⽤硬件资源

- 可以在计算机中安装linux
- docker可以直接运⾏在linux
- 是以容器的形式实现沙盒

不会虚拟化硬件和操作系统

- 所以操作速度快

应⽤、镜像都在Docker容器中

### 3.Docker组件

#### 1.Docker 客户端和服务器

- Docker 是⼀个客户端-服务器(C/S)架构程序。
- Docker 客户端只需要向 Docker 服务器 或者守护进程发出请求，服务器或者守护进程将完成所有⼯作并返回结果。

#### 2.Docker镜像

VMware虚拟机和ubuntu镜像   类⽐   Docker和ubuntu镜像

#### 3.Registry（注册中⼼）

- Docker ⽤ Registry 来保存⽤户构建的镜像。
- ⽐如：当我们需要在Docker中运⾏mysql进程时，只需要向Docker服务器发送请求，获取到注册中⼼的官⽅的mysql的镜像

#### 4.Docker容器

- Docker 可以帮助你构建和部署容器，你只需要把⾃⼰的应⽤程序或者服务打包放进容器即可。
- 容器是基于镜像启动起来的，容器中可以运⾏⼀个或多个进程。
- ⽐如：我们需要在Docker中运⾏已经获取到的mysql镜像，只需要使⽤Docker开启⼀个容器来运⾏镜像即可，mysql进程就在容器中运⾏

#### 5.Docker容器和镜像关系

![1536494643368](C:\Users\ADMINI~1\AppData\Local\Temp\1536494643368.png)

### 4.Docker安装

#### 1.Ubuntu中使⽤源码安装Docker

```
sudo apt-key add gpg
sudo dpkg -i docker-ce_17.03.2~ce-0~ubuntu-xenial_amd64.deb
```

#### 2.检查Docker CE是否安装正确

```
sudo docker run hello-world
```

#### 3.给Docker设置⽤户权限

为了避免每次命令都输⼊sudo，可以设置⽤户权限，注意执⾏后须注销重新登录

```
sudo usermod -a -G docker $USER
```

#### 4.启动和停⽌

```python
# 启动docker
sudo service docker start
# 停⽌docker
sudo service docker stop
# 重启docker
sudo service docker restart
```

### 5.Docker镜像操作

#### 1.展示镜像列表 

```
sudo docker image ls
```

#### 2.拉取镜像

```
sudo docker image pull hello-world
hello-world是镜像名字
```

#### 3.删除镜像

```
sudo docker image rm hello-world
hello-world是镜像名字
```

### 6.Docker容器操作

#### 1.创建容器

```
sudo docker run [option] 镜像名 [向启动容器中传⼊的命令]
sudo docker run -it --name=myubuntu ubuntu /bin/bash
退出容器 exit
```

#### 2.创建守护式容器

```
sudo docker run -dit --name=myubuntu2 ubuntu
进⼊守护式容器
sudo docker exec -it 容器名或容器id 进⼊后执⾏的第⼀个命令
sudo docker exec -it myubuntu2 /bin/bash
退出容器 exit
```

#### 3.查看容器

```
查看正在运⾏的容器 sudo docker container ls
查看所有创建的容器 sudo docker container ls --all
```

#### 4.停⽌和启动容器

```python
# 停⽌⼀个已经在运⾏的容器
sudo docker container stop 容器名或容器id
# 启动⼀个已经停⽌的容器
sudo docker container start 容器名或容器id
# kill掉⼀个已经在运⾏的容器
sudo docker container kill 容器名或容器id
```

#### 5.删除容器

```
sudo docker container rm 容器名或容器id
```

#### 6.将容器存为镜像

```
sudo docker commit 容器名 镜像名
sudo docker commit myubuntu2 ubuntu2
```

#### 7.打包⽣成的镜像

```
sudo docker save -o 保存的⽂件名 镜像名
sudo docker save -o ubuntu2.tar ubuntu2
```

#### 8.使⽤打包的镜像 

```
sudo docker load -i ./ubuntu2.tar
```

