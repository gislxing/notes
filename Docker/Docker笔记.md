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
2. 将当前用户添加到上面的用户组中: `sudo glassed -a 用户名 docker`
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

### 查看镜像的配置信息

```bash
docker inspect [CONTAINER ID / NAMES]
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
docker start [-i] id/容器名称
```

### 删除停止的容器

```bash
docker rm id/容器名称
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



