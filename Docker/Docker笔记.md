# Docker笔记

## 安装

### 下载

到官方网站下载即可：https://www.docker.com/

### mac安装

mac版的是dmg安装包，双击即可安装

### Linux安装

#### 把yum包更新到最新

```shell
$ yum update
```

#### 安装需要的软件包 

**注意：如果已经安装过，则跳过**

yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```shell
$ yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 设置yum源

```shell
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 可以查看所有仓库中所有docker版本，并选择特定版本安装

```shell
$ yum list docker-ce --showduplicates | sort -r
```

#### 安装Docker，命令：yum install docker-ce-版本号

```shell
# 指定安装的版本：17.12.1.ce
$ yum install docker-ce-17.12.1.ce

# 安装最新版本
$ yum install docker-ce
```

#### 启动Docker

```shell
# 启动 docker
$ systemctl start docker

# 添加到开机启动(如果不需要则不操作)
$ systemctl enable docker
```

#### 验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

```shell
$ docker version
```

```shell
# 如果出现下面信息则安装并启动成功
Client:
 Version:    17.12.1-ce
 API version:    1.35
 Go version:    go1.9.4
 Git commit:    7390fc6
 Built:    Tue Feb 27 22:15:20 2018
 OS/Arch:    linux/amd64

Server:
 Engine:
  Version:    17.12.1-ce
  API version:    1.35 (minimum version 1.12)
  Go version:    go1.9.4
  Git commit:    7390fc6
  Built:    Tue Feb 27 22:17:54 2018
  OS/Arch:    linux/amd64
  Experimental:    false
```

#### 如果失败，则卸载重新安装

```shell
$ yum remove docker \
             docker-client \
             docker-client-latest \
             docker-common \
             docker-latest \
             docker-latest-logrotate \
             docker-logrotate \
             docker-engine
```

## docker基本操作

### 启动容器(执行一个命令)

```bash
$ docker run 镜像 [命令] [参数]
# 参数说明：
#		-d 后台运行
#		-p 端口映射: -p 宿主机端口:docker容器端口
```

### 查看Docker基本配置信息

```shell
$ docker info
```

### 查看Docker日志

```shell
$ docker logs 容器id
#		参数说明：
#			-f 持续跟踪日志
```

### 交互式容器

```bash
# 启动交互式容器
$ docker run -i -t 镜像 /bin/bash

# 进入正在运行的容器
$ docker exec -it 容器id或者名称 /bin/bash
$ docker exec -it 容器id或者名称 /bin/sh
```

### 查看创建的容器

```bash
# -a 列出所有创建过的容器
# -l 列出最新创建的一个容器
# 不带参数则列出正在运行的容器
docker ps [-a][-l]
```

### 查看镜像/容器的配置信息

```bash
docker inspect [CONTAINER ID / NAMES][仓库名称:tag]
# 查看容器信息：docker inspect id
# 查看镜像信息：docker inspect ubuntu:16.04
```

### 自定义容器名称

```bash
# 默认情况下docker会随机给容器起一个名称
docker run --name=自定义容器名称
# 例如：docker run --name=docker001 ubuntu echo 'hello world'
```

### 重新启动已经停止的容器

```bash
# -i 以交互方式启动停止的容器
# 进入后输入：exit则完全终止了该容器，如果按ctrl+p ctrl+q 则退出并未终止容器
docker start [-i] id/容器名称
```

### 删除停止的容器

```bash
docker rm id/容器名称
# -f 强制删除

# 批量删除停止容器
docker rm $(docker ps -a -q)

# 删除所有停止的容器
$ docker container prune
```

### 查看容器的端口

```
docker port id/容器名称
```

### 删除未使用的镜像（none）

```shell
$ docker image prune
# 删除未使用镜像时，如果有容器(停止或者启动)使用了该镜像则无法删除，需要先删除容器
```

## 守护式容器

### 进入正在运行的容器

```bash
docker attach id/容器名称	
```

### 启动守护式容器（后台运行）

```bash
docker run -d 镜像 [命令] [参数]
# 例如：docker run -d --name=docker002 ubuntu /bin/sh -c "while true;do echo hello world; sleep 1; done"
```

### 查看容器日志

```bash
docker logs [-f][-t][--tail] id/容器名称
# -f --follows=true|false 默认false：跟踪日志变化并返回结果
# -t --timestamps=true|false 默认false：在返回的结果上加上时间戳
# --tail = "all"：返回结尾多少数量的日志，例如：docker logs --tail 10 容器名称
```

### 查看容器内进程

```bash
docker top id/容器名称
```

### 在运行中的容器中启动新的进程

```bash
docker exec [-d][-i][-t] id/容器名称 [命令] [参数]
```

### 停止守护式容器

```bash
# 发送消息给容器，等待容器自己停止
docker stop id/容器名称

# 立即停止容器
docker kill id/容器名称
```

## 在容器中部署静态网站-Nginx部署流程

### 设置容器映射端口

```bash
docker run [-P][-p] 参数
# -P 为容器暴漏的所有端口映射
# -p 指定映射的容器端口，例如：docker run -p 80
# 参数格式如下：
	# containerport # 只指定容器端口，宿主机端口随机
    # ip:hostport:containerport #指定容器ip、指定宿主机port、指定容器port
    # ip::containerport #指定容器ip、未指定宿主机port（随机）、指定容器port
    # hostport:containerport #未指定ip、指定宿主机port、指定容器port
```

###1. 创建映射80端口的守护式容器

```bash
# 启动一个容器，并将容器的80端口映射到宿主机的8080端口
docker run -it -p 8080:80 /bin/bash
```

### 2.安装Nginx

```bash
apt-get update
或者 apt update

apt-get install nginx
```

### 3.安装vim

```bash
apt install vim
```

### 4.创建静态页面

```bash
# 创建存放静态页面的目录
mkdir -pv /var/www/html

# 在上面的目录中创建一个html文件
vim /var/www/html/index.html
# 并输入以下内容并保存
<html>
        <head>
                <title>test in docker nginx</title>
        </head>
        <body>
                <h2>this is web in docker Nginx</h2>
        </body>
</html>
```

### 5.修改Nginx配置文件

```bash
# 查看nginx的安装位置
whereis nginx
# 查看nginx的配置文件
ls /etc/nginx
ls /etc/nginx/sites-enabled/
# 修改里面的default文件
vim /etc/nginx/sites-enabled/default
# 修改 root 为静态文件的目录(第4步创建的目录)
root /var/www/html
```

### 6.运行Nginx

```bash
# 启动nginx
nginx
# 查看nginx是否启动
ps -ef
```

在宿主机浏览器输入：<http://localhost:8080/> 查看web是否运行正常

## 查看、删除镜像

### 查看镜像列表

```bash
docker images [options][仓库名称]
# -a 列出所有镜像
# -f 显示的过滤条件
# --no-trunc 不使用截断的形式显示id
# -q 只显示镜像的唯一id
# 例如: docker images -a ubuntu
```

### 删除镜像

```bash
$ docker rmi [options]镜像:tag[镜像:tag...]
# options取值如下:
# 	-f 强制删除镜像
# 	--no-prune 不删除未打tag的父镜像	

# 批量删除镜像
$ docker rmi $(docker images -q -a)
$ docker image prune
```

## 获取和推送镜像

### 查找镜像

```bash
# Docker Hub
https://hub.docker.com
# 命令查找
docker search [options] term
Options:
-f, --filter key=value   根据提供的条件筛选输出
	目前支持的过滤器是：
        stars（int - 镜像收藏的星数）
        is-automated（boolean - true或false） - 镜像是否自动化
        is-official（boolean - true或false） - 镜像是否官方的
	--format string   使用Go模板进行漂亮的打印搜索
	--limit int       最大搜索返回的搜索结果（默认25）
	--no-trunc        显示描述的时候不截断
# 查找收藏数大于100的镜像: docker search -f "stars=100" ubuntu
```

### 拉去镜像

```bash
docker pull [options] NAME[:tag]
Options:
  -a, --all-tags                下载仓库中所有标记的镜像
      --disable-content-trust   跳过镜像验证（默认为true）
```

### 推送镜像

```bash
# 首先登陆 https://hub.docker.com/
$ docker login

# 推送自己的镜像
$ docker push NAME[:tag]
```

## 构建镜像

### commit命令构建镜像

```bash
# commit 是通过容器构建镜像
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
Options:
  -a, --author string    Author (e.g., "John Hannibal Smith
                         <hannibal@a-team.com>")
  -c, --change list      对构建的镜像应用Dockerfile指令
  -m, --message string   描述信息
  -p, --pause            提交期间暂停容器（默认为true）
  
以下为通过一个容器构建镜像的例子：
	1. 启动一个交互式容器：docker run -it --name=test -p 8080:80 ubuntu /bin/bash
	2. apt update
	3. apt install nginx
	4. exit
	5. docker ps -l
	6. docker commit -a "zbs" -m "nginx" ctest n_test
	7. 查看上面构建的镜像: docker images
	8. 然后可以使用命令可以反复使用该镜像: docker run -d -p 80 n_test nginx -g "daemon off;"
```

### Dockerfile文件构建镜像

1. 创建Dockerfile文件

```bash
# 1.在Dockerfile存放目录创建文件
$ vim Dockerfile

# 2.输入以下内容

# first Dockerfile for test

from ubuntu:18.04
maintainer zbs "zbs1016@163.com"
run apt-get update
run apt-get install -y nginx
expose 80
```

2. 构建镜像

```bash
# 使用 docker build 命令构建镜像
$ docker build [OPTIONS] PATH | URL | -
	--force-rm                始终移除中间容器
	--no-cache                构建镜像的时候不使用缓存
	--pull                    尝试拉取镜像的最新版本
	-q, --quiet               构建过程中不输出，在构建成功的时候输出镜像ID
	--rm                      构建成功后删除中间容器(default true)
	-t, --tag list            镜像的名字及标签，通常 name:tag 或者 name 格式

# 例如(在当前目录构建镜像)
# 说明：hub.docker仓库名称/镜像名称[:版本号]
$ docker build -t runoob/ubuntu:v1 . 
```

#### Dockerfile 指令

##### FROM 指定基础镜像

格式：

- FROM <image>
- FROM <image>:<tag>
- FROM <image>@<digest>

使用 FROM 指令指定基础镜像，FROM 指令类似 Java 里的 extends 关键字，FROM 指令必须指定且必须在其它指令之前，FROM 指令之后的的所有指令都依赖于所指定的镜像

##### COPY

复制指令，从上下文目录中复制文件或者目录到容器里指定路径。

格式：

```
COPY [--chown=<user>:<group>] <源路径1>...  <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",...  "<目标路径>"]
```

`[--chown=<user>:<group>]`：可选参数，用户改变复制到容器内文件的拥有者和属组。

`<源路径>`：源文件或者源目录，这里可以是通配符表达式，其通配符规则要满足 Go 的 filepath.Match 规则。例如：

```
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

**<目标路径>**：容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建。

##### ADD

ADD 指令和 COPY 的使用格式一致（同样需求下，官方推荐使用 COPY）。功能也类似，不同之处如下：

- ADD 的优点：在执行 <源文件> 为 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，会自动复制并解压到 <目标路径>。

- ADD 的缺点：在不解压的前提下，无法复制 tar 压缩文件。会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。具体是否使用，可以根据是否需要自动解压来决定。

##### RUN 执行命令

用于执行后面跟着的命令行命令。有以下俩种格式：

```dockerfile
# <命令行命令> 等同于，在终端操作的 shell 命令。
RUN <命令行命令>

RUN ["可执行文件", "参数1", "参数2"]
# 例如：
# RUN ["./test.php", "dev", "offline"] 等价于 RUN ./test.php dev offline
```

RUN <command> 在 shell 终端中运行，在 Linux 中默认是 /bin/sh -c，在 Windows 中是 cmd /s /c

##### CMD

类似于 RUN 指令，用于运行程序，但二者运行的时间点不同:

- CMD 在docker run 时运行。
- RUN 是在 docker build。

**作用**：为启动的容器指定默认要运行的程序，程序运行结束，容器也就结束。CMD 指令指定的程序可被 docker run 命令行参数中指定要运行的程序所覆盖。

**注意**：如果 Dockerfile 中如果存在多个 CMD 指令，仅最后一个生效。

格式：

```
CMD <shell 命令> 
CMD ["<可执行文件或命令>","<param1>","<param2>",...] 
CMD ["<param1>","<param2>",...]  # 该写法是为 ENTRYPOINT 指令指定的程序提供默认参数
```

推荐使用第二种格式，执行过程比较明确。第一种格式实际上在运行的过程中也会自动转换成第二种格式运行，并且默认可执行文件是 sh。

##### ENTRYPOINT

类似于 CMD 指令，但其不会被 docker run 的命令行参数指定的指令所覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序。

但是, 如果运行 docker run 时使用了 --entrypoint 选项，此选项的参数可当作要运行的程序覆盖 ENTRYPOINT 指令指定的程序。

**优点**：在执行 docker run 的时候可以指定 ENTRYPOINT 运行所需的参数。

**注意**：如果 Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效。

格式：

```
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]
```

可以搭配 CMD 命令使用：一般是变参才会使用 CMD ，这里的 CMD 等于是在给 ENTRYPOINT 传参，以下示例会提到。

示例：

假设已通过 Dockerfile 构建了 nginx:test 镜像：

```
FROM nginx

ENTRYPOINT ["nginx", "-c"] # 定参
CMD ["/etc/nginx/nginx.conf"] # 变参 
```

1、不传参运行

```
$ docker run  nginx:test
```

容器内会默认运行以下命令，启动主进程。

```
nginx -c /etc/nginx/nginx.conf
```

2、传参运行

```
$ docker run  nginx:test -c /etc/nginx/new.conf
```

容器内会默认运行以下命令，启动主进程(/etc/nginx/new.conf:假设容器内已有此文件)

```
nginx -c /etc/nginx/new.conf
```

##### ENV

设置环境变量，定义了环境变量，那么在后续的指令中，就可以使用这个环境变量。

格式：

```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

以下示例设置 NODE_VERSION = 7.2.0 ， 在后续的指令中可以通过 $NODE_VERSION 引用：

```
ENV NODE_VERSION 7.2.0

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc"
```

##### ARG

构建参数，与 ENV 作用一至。不过作用域不一样。ARG 设置的环境变量仅对 Dockerfile 内有效，也就是说只有 docker build 的过程中有效，构建好的镜像内不存在此环境变量。

构建命令 docker build 中可以用 --build-arg <参数名>=<值> 来覆盖。

格式：

```
ARG <参数名>[=<默认值>]
```

##### VOLUME

定义匿名数据卷。在启动容器时忘记挂载数据卷，会自动挂载到匿名卷。

作用：

- 避免重要的数据，因容器重启而丢失，这是非常致命的。
- 避免容器不断变大。

格式：

```
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>
```

在启动容器 docker run 的时候，我们可以通过 -v 参数修改挂载点。

##### EXPOSE

仅仅只是声明端口。

作用：

- 帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射。
- 在运行时使用随机端口映射时，也就是 docker run -P 时，会自动随机映射 EXPOSE 的端口。

格式：

```
EXPOSE <端口1> [<端口2>...]
```

##### WORKDIR

指定工作目录。用 WORKDIR 指定的工作目录，会在构建镜像的每一层中都存在。（WORKDIR 指定的工作目录，必须是提前创建好的）。

docker build 构建镜像过程中的，每一个 RUN 命令都是新建的一层。只有通过 WORKDIR 创建的目录才会一直存在。

格式：

```
WORKDIR <工作目录路径>
```

##### USER

用于指定执行后续命令的用户和用户组，这边只是切换后续命令执行的用户（用户和用户组必须提前已经存在）。

格式：

```
USER <用户名>[:<用户组>]
```

##### HEALTHCHECK

用于指定某个程序或者指令来监控 docker 容器服务的运行状态。

格式：

```
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令

HEALTHCHECK [选项] CMD <命令> : 这边 CMD 后面跟随的命令使用，可以参考 CMD 的用法。
```

##### ONBUILD

用于延迟构建命令的执行。简单的说，就是 Dockerfile 里用 ONBUILD 指定的命令，在本次构建镜像的过程中不会执行（假设镜像为 test-build）。当有新的 Dockerfile 使用了之前构建的镜像 FROM test-build ，这是执行新镜像的 Dockerfile 构建时候，会执行 test-build 的 Dockerfile 里的 ONBUILD 指定的命令。

格式：

```
ONBUILD <其它指令>
```

## Docker-compose

官方仓库：https://github.com/docker/compose

官方文档：https://docs.docker.com/ee/

Compose是用于定义和运行多容器Docker应用程序的工具。通过Compose，您可以使用Compose文件配置应用程序的服务。然后，使用单个命令，从您的配置中创建并启动所有服务

### 安装Docker Compose

#### Mac Windows 安装

Mac 和 Windows 不需要单独安装，安装的 Docker 桌面版包含 docker-compose

#### Linux 安装

1. 下载

   ```shell
   sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   ```


2. 给 `docker-compsoe` 可执行权限

   ```shell
   sudo chmod +x /usr/local/bin/docker-compose
   ```

3. 是否安装成功

   ```shell
   docker-compose --version
   ```
   
#### 卸载   

```shell
sudo rm /usr/local/bin/docker-compose
```



