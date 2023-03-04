# Docker

## docker的卸载与安装

### 卸载

{% embed url="https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine" %}
docker uninstall
{% endembed %}

### 安装

{% embed url="https://docs.docker.com/engine/install/ubuntu/#installation-methods" %}
docker install
{% endembed %}

### 镜像加速服务

{% embed url="https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors" %}
aliyun image speedup service
{% endembed %}



## docker 的命令

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>docker command</p></figcaption></figure>

### 镜像

1. 从远端仓库（docker hub）拉去镜像

```bash
docker pull redis
```

* docker pull redis == docker pull redis:latest(最新版)，特定版本需要指定  。
* REPOSITORY(名) TAG (标签) IMAGE ID(镜像id) CREATED(镜像的创建时间) SIZE(大小)

2. 查看所有的镜像

```bash
docker images
```

3. 镜像启动

```bash
docker run i
```

4. 删除镜像

```bash
docker rmi redis
```

* 多个镜像名称相同时，需要指定标签
* 如果容器正在启动，需要先删除容器，或者强制删除

```bash
docker rmi -f redis
```

* 批量删除镜像

```bash
docker rmi -f $(docker images -aq)
```

* 移除游离镜像，dangling:游离镜像(没有镜像名字的)

```bash
docker image prune
```

* 镜像重命名

```bash
docker tag 原镜像:标签 新镜像名:标签
```

### 容器

1. 删除容器

```bash
docker rm id 
```

2. 查看所有运行的容器(包含暂停的)

```bash
docker ps
```

* 查看所有的容器

```bash
docker ps -a
```

3. 创建容器create，需要手动的来启动start

{% code overflow="wrap" %}
```bash
docker create [OPTIONS] IMAGE [COMMAND] [ARG...] 
docker create [设置项] 镜像名 [启动] [启动参数...]

//按照redis:latest镜像启动一个容器 
docker create redis
docker create --name myredis -p 6379(主机的端口):6379(容器的端口) redis(镜像)
docker create --name myredis -p 6379:6379 redis
// 本地没有容器时候，去docker hub 拉一个
```
{% endcode %}

4. 启动创建好的容器start

```bash
docker start id
```

5. 暂停一个容器

```bash
docker pause id
```

6. 恢复一个容器

```bash
docker unpause id 
```

{% hint style="info" %}
容器的状态

Created(新建)、Up(运行中)、Pause(暂停)、Exited(退出)
{% endhint %}

7. 退出一个容器

```bash
// 优雅停机，做完任务
docker stop id
// 强制杀死
docker kill id
```

8. 直接运行一个主机，直接在后台启动

{% code overflow="wrap" %}
```bash
// 默认是前台启动的，一 般加上-d 让他后台悄悄启动, 虚拟机的很多端口绑定容器的一个端口是允许的
docker run --name myredis2 -p 6379:6379 -p 8888:6379 -d redis
docker run -d == docker create + docker start  
```
{% endcode %}

9. attach

docker 容器里面安装了nginx，要对nginx的所有修改都要进容器&#x20;

docker attach 绑定的是控制台. 可能导致容器停止，不要用这个。

10. exec

```bash
docker exec -it --privileged mynginx4 /bin/bash
```

* 以特权方式进入容器

### 日志

1. 查看日志

```bash
docker logs id
```

2. 追踪某个镜像

```bash
docker logs -f id
```

### 查看详情

docker container inspect 容器名 = docker inspect 容器

docker inspect image /network/volume ....

### 容器与宿主机复制

{% code overflow="wrap" %}
```bash
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH:把容器里面的复制出来 
docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH:把外部的复制进去
```
{% endcode %}

SRC\_PATH 指定为一个文件

* DEST\_PATH不存在： 文件名为 DEST\_PATH，内容为SRC的内容
* DEST\_PATH 不存在并且以 / 结尾:报错&#x20;
* DEST\_PATH 存在并且是文件:目标文件内容被替换为SRC\_PATH的文件内容。&#x20;
* DEST\_PATH 存在并且是目录:文件复制到目录内，文件名为SRC\_PATH指定的名字

SRC\_PATH 指定为一个目录

* DEST\_PATH 不存在: DEST\_PATH 创建文件夹，复制源文件夹内的所有内容
* DEST\_PATH 存在是文件:报错
* DEST\_PATH 存在是目录
  * SRC\_PATH 不以 /. 结束:源文件夹复制到目标里面
  * SRC\_PATH 以 /. 结束:源文件夹里面的内容复制到目标里面。

```bash
docker cp index.html mynginx4:/usr/share/nginx/html 
docker cp mynginx4:/etc/nginx/nginx.conf nginx.conf
```

### 比较不同

```bash
docker diff containder id
```

* A：添加
* D：删除
* C：修改





