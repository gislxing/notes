# Docker笔记

## 安装

### 下载

到官方网站下载即可：https://www.docker.com/

### mac安装

mac版的是dmg安装包，双击即可安装

### Linux安装

使用命令下载：

```bash
sudo wget -qO- hppts://get.docker.com | sh
或者
curl -sSL https://get.docker.com/ubuntu/ | sudo sh
```

修改使其他用户也可运行docker（docker默认只有root用户才能运行）：

```bash
sudo usermod -aG docker 用户名
```

或者使用下列命令：

1. 创建`docker`用户组：`sudo groupadd docker`
2. 将当前用户添加到上面的用户组中: `sudo gpsswd -a 用户名 docker`
3. 重新启动`docker`服务: `sudo service docker restart`

如果没有安装`curl`需要安装

```bash
查看是否安装了curl:
whereis curl
安装curl:
sudo apt-get install -y curl
```

### 验证是否安装成功

执行命令：`docker --version`，如果现实对应安装docker版本号则说明安装ok

## docker基本操作

### 启动容器(执行一个命令)

```bash
docker run 镜像 [命令] [参数]
```

### 查看Docker基本配置信息

```
docker info
```

### 启动交互式容器

```bash
docker run -i -t 镜像 /bin/bash
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

# 批量删除容器
docker rm $(docker ps -a -q)
```

### 查看容器的端口

```
docker port id/容器名称
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
docker push NAME[:tag]
```

## 构建镜像

### 通过commit命令构建镜像

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

### 使用Dockerfile文件构建镜像

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
	-t, --tag list            指定构建出的镜像的名称
# 例如(在当前目录构建镜像)
$ docker build -t="df-test" .
```

