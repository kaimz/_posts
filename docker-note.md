---
title: 学习 Docker 笔记
date: 2018-04-13 22:12:03
tags: [Docker]
categories: 学习笔记
---

学习总结来自 Gitbook [Docker 从入门到实践的总结](https://yeasy.gitbooks.io/docker_practice/content/basic_concept/image.html)；
现在容器的工具太多，国内阿里，华为等都有自己的容器产品，说到容器大多数想到或者用到的是 Docker，所以就这样来学习吧。主要学习方向参考上面的书。

这篇书讲得很全，作为开发人员可以刚开始不用全部都学习完，只是把基础的内容学习一遍，加上尝试实践，和中间学习过程中感觉到理解困难的学习点记录了下，最主要还是注重基础，有  *shell*  脚本基础学习起来时要轻松很多，重点是理解原理，有条件加以实战，才知道原来是这样啊，也能发现发现很多问题。

<!--more-->

## 为什么要使用 Docker
### 什么是 Docker

Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得 Docker 技术比虚拟机技术更为轻便、快捷。

传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

### 为什么要用 Docker
1. 更高效的利用系统资源
2. 更高效的利用系统资源：直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。
3. 一致的运行环境
4. 持续交付和部署，一次创建或配置，可以在任意地方正常运行。
5. 更轻松的迁移，可以多平台运行。
6. 更轻松的维护和扩展，可以自定义镜像。


## 了解 Docker 的基本概念

### 镜像
Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。
### 分层存储
 Docker 设计时，充分利用 Union FS 的技术，将其设计为分层存储的架构。镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。

镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。
### 容器
容器是镜像运行时的实体，只是是以镜像为基础层，在其上创建一个当前**容器的存储层**。容器可以被创建、启动、停止、删除、暂停等。

## 基本使用
### 安装

参考 [安装 Docker](https://blog.wuwii.com/docker-install.html)

### 镜像加速

国内使用加速：

```json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

重启：

```bash
$ systemctl daemon-reload
$ systemctl restart docker
```

### 镜像

#### 查找镜像语法

```bash
$ docker search [OPTIONS] TERM
```

OPTIONS说明：

- **--automated :**只列出 automated build类型的镜像；

- **--no-trunc :**显示完整的镜像描述；

- **-s :**列出收藏数不小于指定值的镜像。

  

  我们首先想使用某个镜像，就可以去 DockerHub 下载，首先去查找镜像，例如我想使用 `nginx` 镜像并且收藏数大于100：

```bash
$ docker search nginx -s 100
Flag --stars has been deprecated, use --filter=stars=3 instead
NAME                                     DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
nginx                                    Official build of Nginx.                        8309                [OK]                
jwilder/nginx-proxy                      Automated Nginx reverse proxy for docker con…   1312                                    [OK]
richarvey/nginx-php-fpm                  Container running Nginx + PHP-FPM capable of…   544                                     [OK]
jrcs/letsencrypt-nginx-proxy-companion   LetsEncrypt container to use with nginx as p…   341                                     [OK]
kong                                     Open-source Microservice & API Management la…   173                 [OK]   
```

#### 获取镜像

从 Docker 的镜像仓库中拖取镜像 `docker pull`：
```bash
$ docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```
获取 nginx 镜像

```bash
$ docker pull nginx:latest
```

#### 列出镜像

要想列出已经下载下来的镜像，可以使用 `docker image ls ` 命令，或者 `docker images`，它会列出所有的镜像。
列表包含了 仓库名、标签、镜像 ID（唯一标识）、创建时间 以及 所占用的空间。

当然命令后面可以接条件，找出符合条件的镜像。

```bash
$ docker images [OPTIONS] [REPOSITORY[:TAG]]
```

OPTIONS说明：

- **-a :**列出本地所有的镜像（含中间映像层，默认情况下，过滤掉中间映像层）；
- **--digests :**显示镜像的摘要信息；
- **-f :**显示满足条件的镜像；
- **--format :**指定返回值的模板文件；
- **--no-trunc :**显示完整的镜像信息；
- **-q :**只显示镜像ID。

合理的格式获取镜像的参数，可以方便在脚本中的使用。

查看 nginx 镜像的 ID：

```bash
$ docker image ls nginx:latest -q
c5c4e8fa2cf7
```

#### 删除镜像

语法：

```bash
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```

1. 使用 ID 删除镜像：上面我们查看到 nginx 的 id 很长，但是实际上只要使用前三位以后能够确定到哪个镜像就行了 :

   ```bash
   $ docker image rm c5c
   ```

2. 使用镜像名删除镜像：也就是 `<仓库名>:<标签>`，来删除镜像。

   ```bash
   $ docker image rm nginx:latest
   ```

3. 使用镜像摘要删除镜像：

   ```bash

   $ docker image ls nginx --digests
   REPOSITORY          TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
   nginx               latest              sha256:e36d7f5dabf1429d84135bb8a8086908e1150f1a178c75719a9e0e53ebb90353   c5c4e8fa2cf7        6 days ago          109MB

   $ docker image rm nginx@sha256:e36d7f5dabf1429d84135bb8a8086908e1150f1a178c75719a9e0e53ebb90353
   Untagged: nginx@sha256:e36d7f5dabf1429d84135bb8a8086908e1150f1a178c75719a9e0e53ebb90353
   ```



现在我们就可以使用查询命令来配合使用删除：

```bash
$ docker image rm $(docker image ls nginx -q)
Untagged: nginx:latest
Untagged: nginx@sha256:e36d7f5dabf1429d84135bb8a8086908e1150f1a178c75719a9e0e53ebb90353
Deleted: sha256:c5c4e8fa2cf7d87545ed017b60a4b71e047e26c4ebc71eb1709d9e5289f9176f
Deleted: sha256:df08705f06272d44ac0364419532e581af1340fc54ef33423d3735abba422834
Deleted: sha256:220ece772fae32240b2b8491a072c7b30cc0c5c6b67ad73fba6c2968e4ecacd7
```

****



#### tag

给一个镜像打上指定的标签（它们拥有一样的 IMAGE ID，可以试一试）：

```bash
$ docker tag nginx:latest kronchan/nginx:v1.0 
$ docker image ls kronchan/nginx:v1.0        
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kronchan/nginx      v1.0                c5c4e8fa2cf7        6 days ago          109MB
```

**但是多个镜像有相同 ID 不能使用 ID 一次性删除：**`Error response from daemon: conflict: unable to delete c5c4e8fa2cf7 (must be forced) - image is referenced in multiple repositories`

#### 使用 Dockerfile 定制镜像

`Dockerfile`是由一系列命令和参数构成的脚本，一个`Dockerfile`里面包含了构建整个`image`的完整命令。Docker通过`docker build`执行`Dockerfile`中的一系列命令自动构建`image`。

##### FROM

所谓定制镜像，那一定是以一个镜像为基础，比如我们需要构建一个 java 项目，就必须要在 java 环境的基础上进行构建，其中基础镜像是必须指定的，`FROM` 就是指定**基础镜像**，因此一个 `Dockerfile` 中 `FROM` 是必备的指令，并且必须是第一条指令。

```dockerfile
FROM <image>[:tag]  
```



##### RUN

`RUN` 指令是用来执行命令行命令的。由于命令行的强大能力，`RUN` 指令在定制镜像时是最常用的指令之一。其格式有两种：

1. *shell* 格式：`RUN <命令>`，直接追加 shell 命令行；
2. *exec* 格式：`RUN ["可执行文件", "参数1", "参数2"]`，这更像是函数调用中的格式。

比如给 nginx 制定欢迎页的 `Dockerfile`：

```dockerfile
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

在文件所在目录执行命令：

```bash
$ docker build -t nginx:v1.0 .
```

注意后面的点不能少，点 `.` 表示使用当前目录作为上下文环境将文件进行打包，上传到 Docker 引擎服务器，进行构建镜像。

##### COPY

格式：

- `COPY <源路径>... <目标路径>`
- `COPY ["<源路径1>",... "<目标路径>"]`

和 `RUN` 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

`COPY` 指令将从构建上下文目录中 `<源路径>` 的文件/目录复制到新的一层的镜像内的 `<目标路径>` 位置。比如：

```
COPY package.json /usr/src/app/
```

`<源路径>` 可以是多个，甚至可以是通配符，其通配符规则要满足 Go 的 [`filepath.Match`](https://golang.org/pkg/path/filepath/#Match) 规则，如：

```
COPY hom* /mydir/
COPY hom?.txt /mydir/
```

`<目标路径>` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径（工作目录可以用 `WORKDIR` 指令来指定）。目标路径不需要事先创建，如果目录不存在会在复制文件前先行创建缺失目录。

此外，还需要注意一点，使用 `COPY` 指令，源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。

##### ADD 

一般认为拥有和 `COPY` 一样的功能，但是多了一个自动解压缩的功能。

##### CMD

容器启动命令，其格式有两种：

1. *shell* 格式：`RUN <命令>`，直接追加 shell 命令行；
2. *exec* 格式：`RUN ["可执行文件", "参数1", "参数2"]`，这更像是函数调用中的格式。



在启动容器的时候，可以使用下面命令覆盖`CMD` 缺省值：

```bash
$ sudo docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```



##### ENTRYPOINT 

它指定了当container执行时，需要启动哪些进程。

两种形式：

- ENTRYPOINT [“executable”, “param1”, “param2”] （*exec* 形式, 首选）
- ENTRYPOINT command param1 param2 (*shell* 形式)

*shell* 形式防止使用任何`CMD`或运行命令行参数，但是缺点是您的`ENTRYPOINT`将作`/bin/sh -c`的子命令启动，它不传递信号。这意味着可执行文件将不是容器的`PID 1`，并且不会接收Unix信号，因此您的可执行文件将不会从`docker stop <container>`接收到`SIGTERM`。

只有`Dockerfile`中最后一个`ENTRYPOINT`指令会有效果。

`docker run <image>`的命令行参数将附跟在 *exec* 形式的`ENTRYPOINT`中的所有元素之后，并将覆盖使用`CMD`指定的所有元素。这允许将参数传递到入口点，即`docker run <image> -d`将把`-d`参数传递给入口点。

入口点，和 `CMD` 一起使用比较好，

例子：

编写一个 `Dockerfile`:

```dockerfile
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```

构建镜像，分别运行两个容器：

1. `$ docker run -d --name test1 test/ubuntu:v1.0 `
2. ` $ docker run  -d --name test2 test/ubuntu:v1.0 -H`

```bash
$ docker container ls
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS                  PORTS                    NAMES
de2ac31d1219        test/ubuntu:v1.0              "top -b -H"              2 minutes ago       Up 2 minutes                                     test2
852e5d5119d0        test/ubuntu:v1.0              "top -b -c"              3 minutes ago       Up 3 minutes                                     test1
```

可以检查容器：

```bash
$ docker exec -it test2 ps aux 
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  36520  1584 ?        Ss   04:04   0:00 top -b -H
root         5  0.0  0.1  34412  1436 pts/0    Rs+  04:15   0:00 ps aux
```



启动容器的时候可以使用 `--entrypoint="": Overwrite the default entrypoint set by the image`，覆盖缺省的值。

TODO ，这方面还需要再了解下。

##### ENV

格式有两种：

- `ENV <key> <value>`
- `ENV <key1>=<value1> <key2>=<value2>...`

设置环境变量，后面可以直接调用：

```dockerfile
ENV VERSION=1.0

RUN echo $VERSION
```

`-e`参数：在启动容器 的时候使用 `-e VERSION=2.0` 可以覆盖值。 



在容器启动的时候，会缺省创建下面的变量：

| **Variable** | **Value**                                |
| ------------ | ---------------------------------------- |
| `HOME`       | Set based on the value of `USER`         |
| `HOSTNAME`   | The hostname associated with the container |
| `PATH`       | Includes popular directories, such as :`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` |
| `TERM`       | `xterm` if the container is allocated a psuedo-TTY |



##### ARG 

构建参数，格式：`ARG <参数名>[=<默认值>]`

`Dockerfile` 中的 `ARG` 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 `docker build` 中用 `--build-arg <参数名>=<值>` 来覆盖。

##### VOLUME 

格式为：

- `VOLUME ["<路径1>", "<路径2>"...]`
- `VOLUME <路径>`

一般上，容器是不保存任何文件的，因为容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。

我们将这些动态数据挂载到主机中，就可以使用挂载卷保存数据。也可以在运行的时候覆盖挂载卷的参数：

```bash
$ docker run -dit -v mydata:/data image
```

`mydata` 为 宿主中的挂载卷，将会挂载到 docker 容器中的 data 这个文件夹中，而且会覆盖 Dockerfile 中设置的匿名挂载卷。

##### EXPOSE 

格式为 `EXPOSE <端口1> [<端口2>...]`。

这只是一个**声明**，在运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 `docker run -P`时，会自动随机映射 `EXPOSE` 的端口。

运行时使用命令:

```bash
$ docker run d -p 8001:8000 IMAGE
```

其中 docker 容器的 8000 端口映射到宿主 8000 端口。

使用 `-P` 将内部容器所有开放的端口随机分发，你可以使用docker port来查找这个随机绑定端口。

```bash
--expose=[]: Expose a port or a range of ports from the container
            without publishing it to your host
-P=false   : Publish all exposed ports to the host interfaces
-p=[]      : Publish a container᾿s port to the host (format:
             ip:hostPort:containerPort | ip::containerPort |
             hostPort:containerPort | containerPort)
             (use 'docker port' to see the actual mapping)
--link=""  : Add link to another container (name:alias)
```



##### WORKDIR 

区别于 *shell* 中的 `cd`的切换命令：

```dockerfile
RUN cd /app
RUN echo "hello" > world.txt
```

由于 Dockerfile 中每个 RUN 都是开启一个容器，所以第二行的命令重启一个容器又是一个新的环境，把那个不知道你切换了文件目录，它还是在原来的位置。如果需要改变以后各层的工作目录的位置，那么应该使用 `WORKDIR` 指令。

启动容器的时候可以使用 `-w`覆盖默认缺省值。

##### USER 

格式：`USER <用户名 or UID>`

`USER`  改变后面构建层使用命令的**身份**，这个用户必须是事先建立好的，否则无法切换。

启动容器的时候可以使用`-u`覆盖缺省值。

##### MAINTAINER

格式：`MAINTAINER [name]`

`MAINTAINER`指令允许您设置生成的images的作者字段。

##### ONBUILD

// todo



##### 使用

`docker build`命令从`Dockerfile`和`context`构建image。`context`是`PATH`或`URL`处的文件。`PATH`本地文件目录。 `URL`是Git repository的位置。

`context`以递归方式处理。因此，`PATH`包括任何子目录，`URL`包括repository及submodules。一个使用当前目录作为`context`的简单构建命令：

```bash
$ docker build .
Sending build context to Docker daemon  6.51 MB
...
```

构建由Docker守护程序运行，而不是由CLI运行。构建过程所做的第一件事是将整个context（递归地）发送给守护进程。大多数情况下，最好是将`Dockerfile`和所需文件复制到一个空的目录，再到这个目录进行构建。

> `警告`：不要使用根目录`/`作为PATH，因为它会导致构建将硬盘驱动器的所有内容传输到Docker守护程序。

可以使用`.dockerignore`文件添加到`context`目录中来排除文件和目录。

一般的，`Dockerfile`位于`context`的根中。但使用`-f`标志可指定Dockerfile的位置。

```bash
$ docker build -f /path/to/a/Dockerfile .
```

如果build成功，您可以指定要保存新image的repository和tag：

```BASH
$ docker build -t shykes/myapp .
```

要在构建后将image标记为多个repositories，请在运行构建命令时添加多个`-t`参数：

```BASH
$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .
```

Docker守护程序一个接一个地运行`Dockerfile`中的指令，如果需要，将每个指令的结果提交到一个新image，最后输出新映像的ID。Docker守护进程将自动清理您发送的context。

请注意，每个指令独立运行，并导致创建一个新image - 因此`RUN cd /tmp`对下一个指令不会有任何影响。

只要有可能，Docker将重新使用中间images（缓存），就是以前构建镜像使用过的指令再次重复使用，加速`docker build`过程。

##### 实践

写一个简单的 Dockerfile：

```dockerfile
# my demo
FROM busybox
MAINTAINER kronchan1@gmail.com
ENV MKDIR=new-file \
    FILE=tmpfile
WORKDIR /kronchan
RUN mkdir ${MKDIR} \
    && touch ${MKDIR}/${FILE}
COPY file1.tar.gz .
ADD file2.tar .
```

开始构建：

```bash
$ ls
Dockerfile  file1.tar.gz  file2.tar
# 开始构建镜像
$ docker build -t test-image .
Sending build context to Docker daemon  3.072kB
Step 1/7 : FROM busybox
 ---> 8ac48589692a
Step 2/7 : MAINTAINER kronchan1@gmail.com
 ---> Using cache
 ---> ca58745123bb
Step 3/7 : ENV MKDIR=new-file     FILE=tmpfile
 ---> Using cache
 ---> f6383692900a
Step 4/7 : WORKDIR /kronchan
 ---> Using cache
 ---> f1f4e917cdcf
Step 5/7 : RUN mkdir ${MKDIR}     && touch ${MKDIR}/${FILE}
 ---> Using cache
 ---> 28963ac16c41
Step 6/7 : COPY file1.tar.gz .
 ---> Using cache
 ---> d4f4affd5b88
Step 7/7 : ADD file2.tar .
^[[A ---> f1ba7424ec6e
Successfully built f1ba7424ec6e
Successfully tagged test-image:latest
# 运行容器
$ docker run -it test-image
# 进入容器后，自动进入创建的工作目录
/kronchan # ls
file1.tar.gz  file2.tar     new-file
# 检查环境变量
/kronchan # echo $FILE
tmpfile
```

**突然发现，`ADD` 指令好像也不能像前面或者网络上前辈们介绍的说自动解压文件，暂时记录，能力有限，也不能说明白，记录下以后不要踩坑，复制文件还是使用 `COPY` 语义更标准。**

### 容器

#### 启动

语法： `docker run`

启动 nginx

```bash
$ docker run -dit nginx
```

`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上，

 `-i` 则让容器的标准输入保持打开。

`-d` 守护状态运行。



当利用 `docker run` 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止



可以利用 `docker container start [container ID or NAMES]` 命令，直接将一个已经终止的容器启动运行。



查看**守护状态运行**的容器的输出信息：

```bash
$ docker logs --help

Usage:  docker logs [OPTIONS] CONTAINER

Fetch the logs of a container

Options:
      --details        Show extra details provided to logs
  -f, --follow         Follow log output
      --since string   Show logs since timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
      --tail string    Number of lines to show from the end of the logs (default "all")
  -t, --timestamps     Show timestamps
      --until string   Show logs before a timestamp (e.g. 2013-01-02T13:23:37) or relative (e.g. 42m for 42 minutes)
```



#### 终止容器

 可以使用 `docker container stop [container ID or NAMES]` 来终止一个运行中的容器。

#### 进入容器

`exec` 命令

一般使用 `-i`，`-t`参数后就可以有终端和提示符。

#### 删除容器

1. 可以使用 `docker container rm [container ID or NAMES]` 来删除一个处于终止状态的容器。
2. 如果需要删除运行状态中的容器，加上参数 `-f` 强制删除；
3. `$ docker container prune` 清理所有处于终止状态的容器。

#### 导出和导入

1. 导出容器：

   ```bash
   $ docker export 180f > /root/springboot-docker.tar
   ```

2. 导入容器快照：

   ```bash
   $ docker import - kronchan/springboot-docker:v1.0
   ```

   还可以导入 url 中的文件作为镜像。

### Docker Hub

#### 推送镜像

目前 Docker 官方维护了一个公共仓库 [Docker Hub](https://hub.docker.com/)，但是国内推送的速度在没有翻墙的情况下比较尴尬，所以可以使用 [阿里云镜像服务](https://cr.console.aliyun.com/?spm=a2c4e.11153959.blogcont29941.9.520269d65b5sBo&accounttraceid=7944ca1b-ff8f-4239-91ba-79d103b8e92e#/imageList)。

1. 登陆 ：

   ```bash
   $ docker login registry.cn-hangzhou.aliyuncs.com
   ```

   然后输入账号密码。

2. 标记 TAG（可选）：

   ```bash
   $ docker tag [ImageId] com.wuwii/<image>[:镜像版本号]
   ```

3. 推送：

   ```bash
   $ docker push [image]
   ```

#### 提交构建文件到仓库

只需要将构建镜像的 `Dockerfile` 和其余相关的文件一同 `push` 到代码托管仓库，再次 `push` 下来就能重新构建镜像。

### 数据管理

#### 数据卷

`数据卷` 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

- `数据卷` 可以在容器之间共享和重用
- 对 `数据卷` 的修改会立马生效
- 对 `数据卷` 的更新，不会影响镜像
- `数据卷` 默认会一直存在，即使容器被删除


**Command：**

```bash
Usage:  docker volume COMMAND

Manage volumes

Options:


Commands:
  create      Create a volume 
  inspect     Display detailed information on one or more volumes
  ls          List volume
  prune       Remove all unused volumes 
  rm          Remove one or more volumes
```

#### 容器挂载数据卷

例如我启动一个 `jenkins` 容器，命名为 `my_jenkins`，将容器的 `var/jenkins_home/`目录挂载到数据卷 `my_jenkins`:

```bash
$ docker volume create my_jenkins_volume

$ docker run -p 7322:8080 -p 50000:50000 -v my_jenkins_volume:/var/jenkins_home/  --name my_jenkins -d jenkins 
```

> `-v my_jenkins_volume:/var/jenkins_home/` 可以理解是 `--mount source=my_jenkins_volume,target=/var/jenkins_home/`，后面一种更好理解，也是推荐使用的。

查询容器信息：

```bash
$ docker inspect my_jenkins

…………
 "Mounts": [
            {
                "Type": "volume",
                "Name": "my_jenkins_volume",
                "Source": "/var/lib/docker/volumes/my_jenkins_volume/_data",
                "Destination": "/var/jenkins_home",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ],
…………
```




#### 挂载主机文件作为数据卷

不光可以将数据卷挂载到容器中，我们还可以直接将宿主的文件直接作为数据卷使用，完成一样的效果。

上面的，我将宿主的文件夹 `/var/my_jenkins`，作为数据卷挂载到 `/var/my_jenkins`：

```bash
$ docker run -p 7322:8080 -p 50000:50000 -v /var/jenkins_home/:/var/jenkins_home/  --name my_jenkins -d jenkins 
```

> 也可以用  `--mount type=bind,source=/var/jenkins_home/,target=/var/jenkins_home/`

### 网络

当 Docker 启动时，会自动在主机上创建一个 `docker0` 虚拟网桥，实际上是 Linux 的一个 bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进行转发。

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/docker/network.png)

#### 外部访问容器

使用`-p`和`-P` 设置主机和容器的映射端口。

#### 容器互联

**使用自定义网络 （network）进行互联。**

运行一个容器并连接到新建的 `my-net` 网络

```bash
$ docker run -it --rm --name busybox1 --network my-net busybox sh
```

打开新的终端，再运行一个容器并加入到 `my-net` 网络

```bash
$ docker run -it --rm --name busybox2 --network my-net busybox sh
```

再打开一个新的终端查看容器信息

```bash
$ docker container ls

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
b47060aca56b        busybox             "sh"                11 minutes ago      Up 11 minutes                           busybox2
8720575823ec        busybox             "sh"                16 minutes ago      Up 16 minutes                           busybox1
```

下面通过 `ping` 来证明 `busybox1` 容器和 `busybox2` 容器建立了互联关系。

在 `busybox1` 容器输入以下命令

```bash
/ # ping busybox2
PING busybox2 (172.19.0.3): 56 data bytes
64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.072 ms
64 bytes from 172.19.0.3: seq=1 ttl=64 time=0.118 ms
```

用 ping 来测试连接 `busybox2` 容器，它会解析成 `172.19.0.3`。

同理在 `busybox2` 容器执行 `ping busybox1`，也会成功连接到。

```
/ # ping busybox1
PING busybox1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.064 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.143 ms
```

这样，`busybox1` 容器和 `busybox2` 容器建立了互联关系。

#### DNS

在自定义配置文件中加入：

```json
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```

以后每次启动容器都会自动配置上面的 `DNS`。

如果用户想要手动指定容器的配置，可以在使用 `docker run` 命令启动容器时加入如下参数：

`-h HOSTNAME` 或者 `--hostname=HOSTNAME` 设定容器的主机名，它会被写到容器内的 `/etc/hostname` 和 `/etc/hosts`。但它在容器外部看不到，既不会在 `docker container ls` 中显示，也不会在其他的容器的 `/etc/hosts` 看到。

`--dns=IP_ADDRESS` 添加 DNS 服务器到容器的 `/etc/resolv.conf` 中，让容器用这个服务器来解析所有不在 `/etc/hosts` 中的主机名。

`--dns-search=DOMAIN` 设定容器的搜索域，当设定搜索域为 `.example.com` 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 `host.example.com`。

> 注意：如果在容器启动时没有指定最后两个参数，Docker 会默认用主机上的 `/etc/resolv.conf` 来配置容器。

#### 容器访问外网

容器要想访问外部网络，需要本地系统的转发支持。在Linux 系统中，检查转发是否打开。

```
$sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

如果为 0，说明没有开启转发，则需要手动打开。

```
$sysctl -w net.ipv4.ip_forward=1
```

如果在启动 Docker 服务的时候设定 `--ip-forward=true`, Docker 就会自动设定系统的 `ip_forward` 参数为 1。

## Docker Compose

Docker镜像在创建之后，往往需要自己手动pull来获取镜像，然后执行run命令来运行。当服务需要用到多种容器，容器之间又产生了各种依赖和连接的时候，部署一个服务的手动操作是令人感到十分厌烦的。

Dcoker-Compose技术，就是通过一个`.yml`配置文件，将所有的容器的部署方法、文件映射、容器连接等等一系列的配置写在一个配置文件里，最后只需要执行`docker-compose up`命令就会像执行脚本一样的去一个个安装容器并自动部署他们，极大的便利了复杂服务的部署。

### 了解

`Compose` 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。

首先介绍几个术语。

- 服务 (`service`)：一个应用容器，实际上可以运行多个相同镜像的实例。
- 项目 (`project`)：由一组关联的应用容器组成的一个完整业务单元。

可见，一个项目可以由多个服务（容器）关联而成，`Compose` 面向项目进行管理。

### 安装

Linux 上默认是没有安装 `docker-compose`，查看版本：

```bash
$ docker-compose --version    
-bash: /usr/bin/docker-compose: No such file or directory
```

没有安装。

从 [官方 GitHub Release](https://github.com/docker/compose/releases) 处直接下载编译好的二进制文件，

使用二进制包安装，比如：

```bash
$ sudo curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

### 模板指令

使用的 `version: 3`

#### build

指定 `Dockerfile` 所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径）。 `Compose` 将会利用它自动构建这个镜像，然后使用这个镜像。

> `注意`：YAML布尔值（true，false，yes，no，on，off）必须用引号引起来，以便解析器将其解释为字符串。



```yaml
version: '3'
services:

  webapp:
    build:
      context: ./dir # Dockerfile所在目录， 或者远程仓库的地址
      dockerfile: Dockerfile-alternate # 指定 Dockerfile
      args:
        buildno: 1 # 只有在构建时候能使用的变量
      cache_from: # 设置镜像构建名
    	- alpine:latest 
    	- corp/web_app:3.14
```

build 不能和 image 一起去使用

#### cap_add, cap_drop

指定容器的内核能力（capacity）分配。

1. 让容器拥有所有能力：

   ```yaml
   cap_add:
     - ALL
   ```

2. 让容器移除某些能力：

   ```yaml
   cap_drop:
     - NET_ADMIN
     - SYS_ADMIN
   ```

#### command

覆盖容器启动后默认执行的命令。

#### cgroup_parent

为容器指定可选的父cgroup。

```yaml
cgroup_parent: m-executor-abcd
```

#### container_name

指定自定义容器名称，而不是生成的默认名称。

> 由于Docker容器名称必须是唯一的，因此如果您指定了自定义名称，则无法将服务扩展到1个容器之外。 尝试这样做会导致错误。

#### devices

指定设备映射关系。

```yaml
devices:
  - "/dev/ttyUSB1:/dev/ttyUSB0"
```

#### depends_on

Express之间的依赖关系，有两个效果：

- `docker-compose up` 将按照依赖顺序启动服务。 在下面的示例中，db和redis将在web之前启动。

- `docker-compose up SERVICE` 将自动包含SERVICE的依赖关系。 在以下示例中，docker-compose up web也将创建并启动db和redis。

  ```yaml
  version: '2'
  services:
    web:
      build: .
      depends_on:
        - db
        - redis
    redis:
      image: redis
    db:
      image: postgres
  ```

  > `注意`：在启动web之前，depends_on不会等待db和redis“就绪”，直到它们被启动。 如果您需要等待服务准备就绪，请参阅控制[启动顺序](https://docs.docker.com/compose/startup-order/)了解有关此问题的更多信息以及解决问题的策略。

#### dns

自定义DNS服务器。可以是单个值或列表。

```yaml
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 114.114.114.114
```

#### dns_search

配置 `DNS` 搜索域。可以是一个值，也可以是一个列表。

```yaml
dns_search: example.com

dns_search:
  - domain1.example.com
  - domain2.example.com
```

#### tmpfs

挂载一个或者多个 tmpfs 文件系统到容器。

```yaml
tmpfs: /run
tmpfs:
  - /run
  - /tmp
```

#### env_file

从文件添加环境变量。可以是单个值或列表。

如果已使用`docker-compose -f FILE`指定了一个Compose文件，则`env_file`中的路径相对于该文件所在的目录。

在环境中指定的环境变量会覆盖这些值。

```yaml
env_file: .env

env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

Compose期望env文件中的每一行都处于`VAR = VAL`格式。 以＃开头的行（即注释）将被忽略，空行也是如此。

```yaml
# Set Rails/Rack environment
RACK_ENV=development
```

> `注意`：如果您的service指定了build选项，则在build过程中将不会自动显示环境文件中定义的变量。 使用build的args子选项来定义构建时环境变量。

#### environment

设置环境变量。你可以使用数组或字典两种格式。

只给定名称的变量会自动获取运行 Compose 主机上对应变量的值，可以用来防止泄露不必要的数据。

```yaml
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```

#### expose

暴露端口，但不映射到宿主机，只被连接的服务访问。

仅可以指定内部端口为参数，

这个标签与Dockerfile中的EXPOSE指令一样，用于指定暴露的端口，但是只是作为一种参考，实际上docker-compose.yml的端口映射还得ports这样的标签。

```yaml
expose:
 - "3000"
 - "8000"
```

#### extra_hosts

添加主机名的标签，就是往 `/etc/hosts`文件中添加一些记录，与 `Docker client` 的 `--add-host`

```yaml
extra_hosts:
 - "googledns:8.8.8.8"
 - "dockerhub:52.1.157.61"
```

会在启动后的服务容器中 `/etc/hosts` 文件中添加相应 host。

#### healthcheck

通过命令检查容器是否健康运行。

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
```

#### image

指定为镜像名称或镜像 ID。如果镜像在本地不存在，`Compose` 将会尝试拉取这个镜像。

```yaml
image: ubuntu:latest
```

#### labels

为容器添加 Docker 元数据（metadata）信息。例如可以为容器添加辅助说明信息。

```yaml
labels:
  com.startupteam.description: "webapp for a startup team"
  com.startupteam.department: "devops department"
  com.startupteam.release: "rc3 for v1.0"
```

#### links

这个标签解决的是容器连接问题，与`Docker client`的 `--link`一样效果，会连接到其它服务中的容器。

```yaml
links:
 - db
 - db:database
 - redis
```

使用的别名将会自动在服务容器中的/etc/hosts里创建。例如：

```
172.12.2.186  db
172.12.2.186  database
172.12.2.187  redis
```

相应的环境变量也将被创建。

#### logging

配置日志选项。

```yaml
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

目前支持三种日志驱动类型。

```yaml
driver: "json-file"
driver: "syslog"
driver: "none"
```

`options` 配置日志驱动的相关参数。

```yaml
options:
  max-size: "200k"
  max-file: "10"
```

#### network_mode

设置网络模式。使用和 `docker run` 的 `--network` 参数一样的值。 

```yaml
network_mode: "bridge" # 桥接模式，这种模式下， docker 会默认创建一个 docker0 的网桥，从它中分配 ip 提供给容器使用
network_mode: "host" # 使用主机网络
network_mode: "none" # 只能访问本地网络，不能使用外网
network_mode: "service:[service name]" # 与其他某个 service 共享一个网络
network_mode: "container:[container name/id]" # 与其他某个容器共享一个网络
```

#### networks

配置容器连接的网络。

```yaml
version: "3"
services:

  some-service:
    networks:
     - some-network
     - other-network

networks:
  some-network:
  other-network:
```

#### pid

跟主机系统共享进程命名空间。打开该选项的容器之间，以及容器和宿主机系统之间可以通过进程 ID 来相互访问和操作。

```yaml
pid: "host"
```

#### ports

暴露端口信息。

使用宿主端口：容器端口 `(HOST:CONTAINER)` 格式，或者仅仅指定容器的端口（宿主将会随机选择端口）都可以。

```
ports:
 - "3000"
 - "8000:8000"
 - "49100:22"
 - "127.0.0.1:8001:8001"
```

*注意：当使用 HOST:CONTAINER 格式来映射端口时，如果你使用的容器端口小于 60 并且没放到引号里，可能会得到错误结果，因为 YAML 会自动解析 xx:yy 这种数字格式为 60 进制。为避免出现这种问题，建议数字串都采用引号包括起来的字符串格式。*

#### secrets

Docker命令行工具提供了`docker secret`命令来管理敏感信息，

从 Docker Compose V3.1开始，支持在容器编排文件中使用 secret，这可以方便地在不同容器中分享所需的敏感信息。

**`docker secret` 只能从`Docker Swarm`模式的`manager`节点调用，如果你在本机进行试验，请先执行 `docker swarm init `命令**。



```bash
$ docker secret --help

Usage:    docker secret COMMAND

Manage Docker secrets

Options:
      --help   Print usage

Commands:
  create      Create a secret from a file or STDIN as content
  inspect     Display detailed information on one or more secrets
  ls          List secrets
  rm          Remove one or more secrets
```

创建一个数据库密码：

```bash
$ echo "kronchan" |  docker secret create db_password -
yxadqo4xguucuyhd9oxc5t4q2
$ docker secret ls
ID                          NAME                DRIVER              CREATED             UPDATED
yxadqo4xguucuyhd9oxc5t4q2   db_password                             3 seconds ago       3 seconds ago
```

在服务编排中：

```yaml
version: "3"
services:

mysql:
  image: mysql
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_password # 缺省值
  secrets:
    - db_password
```



在 Swarm 集群中 ，例如，我用这个密码启动一个 MYSQL 服务，在 manage 节点中：

```bash
docker service create \
        --name mysql \
        --secret source=db_password,target=mysql_root_password \
        -e MYSQL_ROOT_PASSWORD_FILE="/run/secrets/mysql_root_password" \
        mysql:latest
```

这个过程分为两个步骤：

1. `source` 指定容器使用 secret 后，secret 会被解密并存放到**容器的文件系统**中，默认位置为 `/run/secrets/<secret_name>`，可以使用 `target` 重新定位。
2. 设置环境变量 `MYSQL_ROOT_PASSWORD_FILE` 指定从 `/run/secrets/mysql_root_password` 中读取并设置 MySQL 的管理员密码。

#### security_opt

指定容器模板标签（label）机制的默认属性（用户、角色、类型、级别等）。例如配置标签的用户名和角色名。

```yaml
security_opt:
    - label:user:USER
    - label:role:ROLE
```

#### stop_signal

设置另一个信号来停止容器。在默认情况下使用的是 `SIGTERM` 停止容器。

```yaml
stop_signal: SIGUSR1 # 缺省值
```

#### sysctls

配置容器内核参数。

```yaml
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

# other grammar
sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

#### ulimit

指定容器的 `ulimits` 限制值。

例如，指定最大进程数为 65535，指定文件句柄数为 20000（软限制，应用可以随时修改，不能超过硬限制） 和 40000（系统硬限制，只能 root 用户提高）。

```yaml
  ulimits:
    nproc: 65535
    nofile:
      soft: 20000
      hard: 40000
```

#### volumes

数据卷所挂载路径设置。可以设置宿主机路径 （`HOST:CONTAINER`） 或加上访问模式 （`HOST:CONTAINER:ro`）。

该指令中路径支持相对路径。

```yaml
volumes:
 # 只是指定一个路径，Docker 会自动在创建一个数据卷（这个路径是容器内部的）。
 - /var/lib/mysql
 # 使用绝对路径挂载数据卷
 - /opt/data:/var/lib/mysql
 # 以 Compose 配置文件为中心的相对路径作为数据卷挂载到容器。
 - cache/:/tmp/cache
 # 使用用户的相对路径（~/ 表示的目录是 /home/<用户目录>/ 或者 /root/）。
 - ~/configs:/etc/configs/:ro
 # 已经存在的命名的数据卷。
 - datavolume:/var/lib/mysql
```

例如编排一个 pqsql:

```yaml
version: "3"
services: 
  pq:
    image: postgres:9.5
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql
      - db_log:/var/log/postgresql

volumes:
  db_data:
  db_log:
```

举个例子，自动创建一个数据卷 `pq_db_data`，目录在 `/var/lib/docker/volumes/` 下。

将 容器 `pq` 的目录 `/var/lib/postgresql` 挂载到数据卷 `pq_db_data`。

#### volumes_from

从其它容器或者服务挂载数据卷，可选的参数是 :ro或者 :rw，前者表示容器只读，后者表示容器对数据卷是可读可写的。默认情况下是可读可写的。

```yaml
volumes_from:
  - service_name
  - service_name:ro
  - container:container_name
  - container:container_name:rw
```

#### entrypoint

指定服务容器启动后执行的入口文件。

```yaml
entrypoint: /code/entrypoint.sh
```

#### privileged

允许容器中运行一些特权命令。

```yaml
privileged: true
```

#### restart

指定容器退出后的重启策略为始终重启。该命令对保持服务始终运行十分有效，在生产环境中推荐配置为 `always` 或者 `unless-stopped`。

```yaml
restart: always
```

> 在 Swarm 中失效，Swarm Stacks use the `restart_policy:` under the `deploy:`

#### working_dir

指定容器中工作目录。

```yaml
working_dir
```

#### user

指定容器中运行应用的用户名。

```yaml
user: nginx
```

### 使用

#### 基本的使用格式

```bash
$ docker-compose [options] [COMMAND] [ARGS...]
```

`options`:

```bash
-f, --file FILE           指定启动模版文件(一个非docker-compose.yml命名的yaml文件,默认为docker-compose.yml)
-p, --project-name NAME   指定一个替代项目名称 (默认是directory名)
-d 以daemon的方式启动容器
--x-networking            (EXPERIMENTAL) 使用新的网络功能
--x-network-driver DRIVER (EXPERIMENTAL) 指定网络驱动程序，桥 (default: "bridge").
--verbose：输出详细信息
--version 打印版本并退出。
```

#### 常用命令

##### build

格式为 `docker-compose build [options] [SERVICE...]`。

构建（重新构建）项目中的服务容器。

服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。

可以随时在项目目录下运行 `docker-compose build` 来重新构建服务。

选项包括：

- `--force-rm` 删除构建过程中的临时容器。
- `--no-cache` 构建镜像过程中不使用 cache（这将加长构建过程）。
- `--pull` 始终尝试通过 pull 来获取更新版本的镜像。

##### config

验证 Compose 文件格式是否正确，若正确则显示配置，若格式错误显示错误原因。

##### down

此命令将会停止 `up` 命令所启动的容器，并移除网络。

##### exec

进入指定的容器。

##### help

获取命令的帮助信息。

##### images

列出这个 Compose 项目中包含的镜像。

##### kill

格式为 `docker-compose kill [options] [SERVICE...]`。

通过发送 `SIGKILL` 信号来强制停止服务容器。

支持通过 `-s` 参数来指定发送的信号，例如通过如下指令发送 `SIGINT` 信号。

```bash
$ docker-compose kill -s SIGINT
```

##### logs

格式为 `docker-compose logs [options] [SERVICE...]`。

查看服务容器的输出。默认情况下，docker-compose 将对不同的服务输出使用不同的颜色来区分。可以通过 `--no-color` 来关闭颜色。

该命令在调试问题的时候十分有用。

##### pause

暂停一个或多个容器。

##### port

格式为 `docker-compose port [options] SERVICE PRIVATE_PORT`。

打印某个容器端口所映射的公共端口。

选项：

- `--protocol=proto` 指定端口协议，tcp（默认值）或者 udp。
- `--index=index` 如果同一服务存在多个容器，指定命令对象容器的序号（默认为 1）。

##### ps

格式为 `docker-compose ps [options] [SERVICE...]`。

列出项目中目前的所有容器。

选项：

- `-q` 只打印容器的 ID 信息。

##### pull

格式为 `docker-compose pull [options] [SERVICE...]`。

拉取服务依赖的镜像。

选项：

- `--ignore-pull-failures` 忽略拉取镜像过程中的错误。

##### push

推送服务依赖的镜像到 Docker 镜像仓库。

##### restart

格式为 `docker-compose restart [options] [SERVICE...]`。

重启项目中的服务。

选项：

- `-t, --timeout TIMEOUT` 指定重启前停止容器的超时（默认为 10 秒）。

##### rm

格式为 `docker-compose rm [options] [SERVICE...]`。

删除所有（停止状态的）服务容器。推荐先执行 `docker-compose stop` 命令来停止容器。

选项：

- `-f, --force` 强制直接删除，包括非停止状态的容器。一般尽量不要使用该选项。
- `-v` 删除容器所挂载的数据卷。

##### run

格式为 `docker-compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]`。

在指定服务上执行一个命令。

例如：

```bash
$ docker-compose run ubuntu ping docker.com
```

将会启动一个 ubuntu 服务容器，并执行 `ping docker.com` 命令。

默认情况下，如果存在关联，则所有关联的服务将会自动被启动，除非这些服务已经在运行中。

该命令类似启动容器后运行指定的命令，相关卷、链接等等都将会按照配置自动创建。

两个不同点：

- 给定命令将会覆盖原有的自动运行命令；
- 不会自动创建端口，以避免冲突。

如果不希望自动启动关联的容器，可以使用 `--no-deps` 选项，例如

```
$ docker-compose run --no-deps web python manage.py shell
```

将不会启动 web 容器所关联的其它容器。

选项：

- `-d` 后台运行容器。
- `--name NAME` 为容器指定一个名字。
- `--entrypoint CMD` 覆盖默认的容器启动指令。
- `-e KEY=VAL` 设置环境变量值，可多次使用选项来设置多个环境变量。
- `-u, --user=""` 指定运行容器的用户名或者 uid。
- `--no-deps` 不自动启动关联的服务容器。
- `--rm` 运行命令后自动删除容器，`d` 模式下将忽略。
- `-p, --publish=[]` 映射容器端口到本地主机。
- `--service-ports` 配置服务端口并映射到本地主机。
- `-T` 不分配伪 tty，意味着依赖 tty 的指令将无法运行。

##### scale

格式为 `docker-compose scale [options] [SERVICE=NUM...]`。

设置指定服务运行的容器个数。

通过 `service=num` 的参数来设置数量。例如：

```
$ docker-compose scale web=3 db=2
```

将启动 3 个容器运行 web 服务，2 个容器运行 db 服务。

一般的，当指定数目多于该服务当前实际运行容器，将新创建并启动容器；反之，将停止容器。

选项：

- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

##### start

格式为 `docker-compose start [SERVICE...]`。

启动已经存在的服务容器。

##### stop

格式为 `docker-compose stop [options] [SERVICE...]`。

停止已经处于运行状态的容器，但不删除它。通过 `docker-compose start` 可以再次启动这些容器。

选项：

- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

##### top

查看各个服务容器内运行的进程。

##### unpause

格式为 `docker-compose unpause [SERVICE...]`。

恢复处于暂停状态中的服务。

##### up

格式为 `docker-compose up [options] [SERVICE...]`。

该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

链接的服务都将会被自动启动，除非已经处于运行状态。

可以说，大部分时候都可以直接通过该命令来启动一个项目。

默认情况，`docker-compose up` 启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很方便进行调试。

当通过 `Ctrl-C` 停止命令时，所有容器将会停止。

如果使用 `docker-compose up -d`，将会在后台启动并运行所有的容器。一般推荐生产环境下使用该选项。

默认情况，如果服务容器已经存在，`docker-compose up` 将会尝试停止容器，然后重新创建（保持使用 `volumes-from` 挂载的卷），以保证新启动的服务匹配 `docker-compose.yml` 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 `docker-compose up --no-recreate`。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 `docker-compose up --no-deps -d <SERVICE_NAME>` 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务。

选项：

- `-d` 在后台运行服务容器。
- `--no-color` 不使用颜色来区分不同的服务的控制台输出。
- `--no-deps` 不启动服务所链接的容器。
- `--force-recreate` 强制重新创建容器，不能与 `--no-recreate` 同时使用。
- `--no-recreate` 如果容器已经存在了，则不重新创建，不能与 `--force-recreate` 同时使用。
- `--no-build` 不自动构建缺失的服务镜像。
- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

##### version

格式为 `docker-compose version`。

打印版本信息。

### 实践

实战 `WordPress` 项目：

```yaml
version: "3"
services:

   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
    db_data:
```



## Docker Machine

docker技术是基于Linux内核的`cgroup`技术实现的，那么问题来了，在非Linux平台上是否就不能使用docker技术了呢？答案是可以的，不过显然需要借助虚拟机去模拟出Linux环境来。

Docker Machine 就是docker公司官方提出的，用于在各种平台上快速创建具有docker服务的虚拟机的技术，甚至可以通过指定driver来定制虚拟机的实现原理（一般是virtualbox）。

Docker Machine 是 Docker 官方编排（Orchestration）项目之一，负责在多种平台上快速安装 Docker 环境。

### 安装

查询版本信息：

```bash
$ docker-machine -v
```

在 Linux 64 位系统上直接下载对应的二进制包。

```bash
$ sudo curl -L https://github.com/docker/machine/releases/download/v0.13.0/docker-machine-`uname -s`-`uname -m` > /usr/local/bin/docker-machine
$ sudo chmod +x /usr/local/bin/docker-machine
```

完成后，查看版本信息。

```bash
$ docker-machine -v
docker-machine version 0.13.0, build 9ba6da9
```

### 创建本地主机驱动

1. 创建一个 `virtualbox` 类型的驱动，名为 `test1`，可以加上参数配置分配的硬件的信息:

   ```bash
   $ docker-machine create -d virtualbox test1
   Running pre-create checks...
   Error with pre-create check: "VBoxManage not found. Make sure VirtualBox is installed and VBoxManage is in the path"
   ```

   出现了错误，需要安装VirtualBox环境

   * 配置VirtualBox源

     ```bash
     $  vim /etc/yum.repos.d/virtualbox.repo    
     [virtualbox]
     name=Oracle Linux / RHEL / CentOS-$releasever / $basearch - VirtualBox
     baseurl=http://download.virtualbox.org/virtualbox/rpm/el/$releasever/$basearch
     enabled=1
     gpgcheck=0
     repo_gpgcheck=0
     gpgkey=https://www.virtualbox.org/download/oracle_vbox.asc
     ```

   * 安装VirtualBox

     * CentOS

       ```bash
       # 1. 先搜索
       # 2. 安装版本
       # 3. 重新加载配置
       $ yum search VirtualBox 
       Loaded plugins: fastestmirror
       Loading mirror speeds from cached hostfile
       ======================================= N/S matched: VirtualBox ========================================
       VirtualBox-4.3.x86_64 : Oracle VM VirtualBox
       VirtualBox-5.0.x86_64 : Oracle VM VirtualBox
       VirtualBox-5.1.x86_64 : Oracle VM VirtualBox
       VirtualBox-5.2.x86_64 : Oracle VM VirtualBox

         Name and summary matches only, use "search all" for everything.
       $ yum install -y VirtualBox-5.1
       $ /sbin/vboxconfig    
       # 中间出了点问题，说是少了
       # This system is not currently set up to build kernel modules (system extensions).
       #Running the following commands should set the system up correctly:
       #   yum install kernel-devel-3.10.0-693.2.2.el7.x86_64
       $ yum install kernel-devel-3.10.0-693.2.2.el7.x86_64
       # 重新再次加载
       $ /sbin/vboxconfig  
       # 启动成功，心累
       ```

       然后就可以创建了虚拟驱动了。

     * Ubuntu

       ```bash
       # 手上暂时没有 Ubuntu 系统没有测试，网上都是这么说的，记录下，没有验证，
       $ apt-get install virtualbox
       ```

   还有一种问题：

   ```bash
   "This computer doesn't have VT-X/AMD-v enabled. Enabling it in the BIOS is mandatory"
   ```

   [issues/4271](https://github.com/docker/machine/issues/4271)额，我之前是一直远程在阿里云上的服务器做的， 由于一直很讨厌 WINDOW 的命令模式，甩锅啦，现在好了，这个问题要修改 BIOS ，只好又回到 WINDOW 平台上做

2. 登陆到主机：

   ```bash
   $ docker-machine ls 
   NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
   default   *        virtualbox   Running   tcp://192.168.99.100:2376           v17.12.0-ce
   test1     -        virtualbox   Running   tcp://192.168.99.101:2376           v18.04.0-ce

   $ docker-machine ssh test1
   # 后面就可以操作 test1 了
   ```



补充，通过 `-d` 选项可以选择支持的驱动类型。

- amazonec2
- azure
- digitalocean
- exoscale
- generic
- google
- hyperv
- none
- openstack
- rackspace
- softlayer
- virtualbox
- vmwarevcloudair
- vmwarefusion
- vmwarevsphere



### 操作命令

- `active` 查看活跃的 Docker 主机
- `config` 输出连接的配置信息
- `create` 创建一个 Docker 主机
- `env` 显示连接到某个主机需要的环境变量
- `inspect` 输出主机更多信息
- `ip` 获取主机地址
- `kill` 停止某个主机
- `ls` 列出所有管理的主机
- `provision` 重新设置一个已存在的主机
- `regenerate-certs` 为某个主机重新生成 TLS 认证信息
- `restart` 重启主机
- `rm` 删除某台主机
- `ssh` SSH 到主机上执行命令
- `scp` 在主机之间复制文件
- `mount` 挂载主机目录到本地
- `start` 启动一个主机
- `status` 查看主机状态
- `stop` 停止一个主机
- `upgrade` 更新主机 Docker 版本为最新
- `url` 获取主机的 URL
- `version` 输出 docker-machine 版本信息
- `help` 输出帮助信息

## Docker Swarm

Docker 1.12 [Swarm mode](https://docs.docker.com/engine/swarm/) 已经内嵌入 Docker 引擎，成为了 docker 子命令 `docker swarm`。

`Swarm mode` 内置 kv 存储功能，提供了众多的新特性，比如：具有容错能力的去中心化设计、内置服务发现、负载均衡、路由网格、动态伸缩、滚动更新、安全传输等。

### 基本概念

#### 节点

运行 Docker 的主机可以主动初始化一个 `Swarm` 集群或者加入一个已存在的 `Swarm` 集群，这样这个运行 Docker 的主机就成为一个 `Swarm` 集群的节点 (`node`) 。

节点分为管理 (`manager`) 节点和工作 (`worker`) 节点。

管理节点用于 `Swarm` 集群的管理，`docker swarm` 命令基本只能在管理节点执行（节点退出集群命令 `docker swarm leave` 可以在工作节点执行）。一个 `Swarm` 集群可以有多个管理节点，但只有一个管理节点可以成为 `leader`，`leader` 通过 `raft` 协议实现。

工作节点是任务执行节点，管理节点将服务 (`service`) 下发至工作节点执行。管理节点默认也作为工作节点。你也可以通过配置让服务只运行在管理节点。

来自 Docker 官网的这张图片形象的展示了集群中管理节点与工作节点的关系。

![img](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)

#### 服务和任务

任务 （`Task`）是 `Swarm` 中的最小的调度单位，目前来说就是一个单一的容器。

服务 （`Services`） 是指一组任务的集合，服务定义了任务的属性。服务有两种模式：

- `replicated services` 按照一定规则在各个工作节点上运行指定个数的任务。
- `global services` 每个工作节点上运行一个任务

两种模式通过 `docker service create` 的 `--mode` 参数指定。

来自 Docker 官网的这张图片形象的展示了容器、任务、服务的关系。

![img](https://docs.docker.com/engine/swarm/images/services-diagram.png)

### 创建 Swarm 集群

前面我们了解到 Swarm 集群是由 `manager` 和 `worker` 组成的。

#### 初始化集群

初始化一个 manager，主机有多个网卡，拥有多个 IP，必须使用 `--advertise-addr` 指定 IP。：

```bash
$ docker-machine create -d virtualbox manager

docker@manager:~$ docker-machine ssh manager
docker@manager:~$ ip addr
# 主机有多个网卡，拥有多个 IP，必须使用 --advertise-addr 指定 IP。
docker@manager:~$ docker swarm init --advertise-addr 192.168.99.102
Swarm initialized: current node (wwyq0ym88dvuhxkljr4963kei) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3js3ppvjj191r9qx92uiva4l64z8xgy6sno69et5y9ri7a8es5-695418k4uks90r9j9bgwag2k3 192.168.99.102:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

#### 工作节点

1. 新增加一个工作节点 `worker1`：

   ```bash
   $ docker-machine create -d virtualbox worker1
   $ docker-machine ssh worker1

   docker@worker1:~$ docker swarm join --token SWMTKN-1-3js3ppvjj191r9qx92uiva4l64z8xgy6sno69et5y9ri7a8es5-695418k4uks90r9j
   9bgwag2k3 192.168.99.102:2377
   This node joined a swarm as a worker.
   ```

2. 然后按照上面的步骤，新增一个节点 `worker2`。

3. 进入 `manager` 节点 ，查看集群状态：

   ```bash
   docker@manager:~$ docker node ls
   ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
   wwyq0ym88dvuhxkljr4963kei *   manager             Ready               Active              Leader              18.04.0-ce
   dncrx0o8lbq0a5v8lcdwt2onl     worker1             Ready               Active                                  18.04.0-ce
   d9d9pjhwh86lz42klh41ud8ss     worker2             Ready               Active         
   ```

### 部署服务

我们使用 `docker service` 命令来管理 `Swarm` 集群中的服务，该命令只能在管理（manager）节点运行。

#### 新建服务

在上面我们创建的 Swarm 集群中的 `manager` 节点中 运行一个 nginx 服务：

```bash
$ docker service create --replicas 3 -p 80:80 --name nginx nginx:1.13.7-alpine
mqisdpsereiv0x3whj74lpbxi
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

#### 查询服务

```bash
# 查询服务
docker@manager:~$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
mqisdpsereiv        nginx               replicated          3/3                 nginx:1.13.7-alpine   *:80->80/tcp

# 查询进程
docker@manager:~$ docker service ps nginx
ID                  NAME                IMAGE                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
wyvxj6nyp435        nginx.1             nginx:1.13.7-alpine   manager             Running             Running 2 minutes ago
o9qb0pmi1epu        nginx.2             nginx:1.13.7-alpine   worker1             Running             Running 3 minutes ago
45uxv7c6s2b0        nginx.3             nginx:1.13.7-alpine   worker2             Running             Running 3 minutes ago
```

我的集群三个节点的 IP 分别是：

```bash
$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
manager   -        virtualbox   Running   tcp://192.168.99.102:2376           v18.04.0-ce
worker1   -        virtualbox   Running   tcp://192.168.99.103:2376           v18.04.0-ce
worker2   -        virtualbox   Running   tcp://192.168.99.104:2376           v18.04.0-ce
```

分别访问这几个节点的 ip 都能访问到熟悉 nginx 欢迎页。

 查询日志 `docker service logs <service>` 

#### 删除服务

使用 `docker service rm` 来从 `Swarm` 集群移除某个服务。

### 使用 compose 文件部署

**需要注意的 swarm 的compose 多出一些模板指令，而且以前 `docker-compose` 中的一些指令将失效。**

```yaml

```



部署服务使用 `docker stack deploy`，其中 `-c` 参数指定 compose 文件名。

```bash
$ docker stack deploy -c docker-compose-swarm.yml myweb-name
```

查看：

```
$ docker stack ls
NAME                SERVICES
wordpress           2
```

移除服务：

```bash
$ docker stack down
```



[服务编排和 Swarm集群的例子](https://github.com/kaimz/learning-code/tree/master/springboot-docker?1523630460079)



我们以在 `Swarm` 集群中部署 `WordPress` 为例进行说明。

```yaml
version: "3"

services:
  wordpress:
    image: wordpress
    ports:
      - 80:80
    networks:
      - overlay
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    deploy:
      mode: replicated
      replicas: 3

  db:
    image: mysql
    networks:
       - overlay
    volumes:
      - db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    deploy:
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

volumes:
  db-data:
networks:
  overlay:
```

在 `Swarm` 集群管理节点新建该文件，其中的 `visualizer` 服务提供一个可视化页面，我们可以从浏览器中很直观的查看集群中各个服务的运行节点。

## 远程 API 架构

### 了解

#### daemon

Docker的基础服务，比如容器的创建、查看、停止、镜像的管理，其实都是由docker的守护进程(daemon)来实现的。

每次执行的Docker指令其实都是通过向daemon发送请求来实现的。

daemon的运作（通信模式）主要有两种，一种是通过unix套接字（默认，但只能在本地访问到，比较安全），一种是通过监听tcp协议地址和端口来实现（这个可以实现在远程调用到docker服务）。

#### 远程 API

除了通过远程tcp协议访问远程主机上的docker服务外，docker还提供了一套基于HTTP的API，可以使用curl来实现操作远程主机上的docker服务，这为开发基于WEB的docker服务提供了便利。

### 开启远程 API

#### Linux

1.12版本后, 用户可以自行创建配置文件 `/etc/docker/daemon.json`，该文件不区分系统，是通用的，推荐使用。具体参考：[官方文档](https://docs.docker.com/engine/admin/)。不知道版本的可以通过 `$ dockerd version` 查看。

方式一： 首先，你需要创建 `/etc/docker/daemon.json` 文件，文件内容如下：

```json
{
  "hosts": [
    "tcp://0.0.0.0:2375",
    "unix:///var/run/docker.sock"
  ]
}
```

开启的是 `2375` 端口。

然后重启生效：

```bash
$ systemctl daemon-reload
$ systemctl restart docker
```



方式二 ： 当然也可以更改启动命令的方式 ：

```bash
# Ubuntu 系统
$ sudo vim /lib/systemd/system/docker.service
# CentOS
$ sudo vim /usr/lib/systemd/system/docker.service

# 更改下面的启动命令
ExecStart=/usr/bin/dockerd  -H tcp://0.0.0.0:2375  -H unix:///var/run/docker.sock
# 重启
$ systemctl daemon-reload
$ systemctl restart docker
```



使用的时候只需要在客户端的主机上加上环境变量 `DOCKER_HOST=tcp://xxx.xxx.xx.xx/2375`，就可以在主机上使用远程的 Docker  API。

#### Docker Toolbox

1. 在 window 或者 mac 上面使用 `Docker-toolbox`，则就比较简单了，,打开 `docker QuickStart Terminal` 终端，输入命令 ：

   ```bash
   $ docker-machine ls

   NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
   default   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.10.0-ce
   ```

   查询到 URL，根据上面的结果

    我使用的是 WONDOW 电脑的主机上加上环境变量 `DOCKER_HOST=tcp://192.168.99.100:2376 `；

   当然也可以使用 `VirtualBox` 的端口映射到 本地的 2375端口也行，docker 默认的 host 是 `127.0.0.1:2376`，这样环境变量也不用配置。

2. 我使用的是 WIN10 ，后面也是这个开发环境，打开 `PowerShell` 执行命令 docker version 命令发现一个问题：

   ```bash
   Get http://192.168.99.100:2376/v1.33/containers/json: malformed HTTP response "\x15\x03\x01\x00\x02\x02".
   * Are you trying to connect to a TLS-enabled daemon without TLS?
   ```

   发现它需要 TLS ，

   设置环境变量 `DOCKER_TLS_VERIFY=1`

3.  然后再次执行 docker version 命令，发现又出现 问题：

   ```bash
   The server probably has client authentication (--tlsverify) enabled. Please check your TLS client certification settings: Get https://192.168.99.100:2376/v1.35/version: remote error: tls: bad certificate
   ```

   没指定 证书 `cret`。 证书的路径在当前用户的 `/.docker/certs`下：

   设置环境变量 `DOCKER_CERT_PATH=C:/Users/KronChan/.docker/machine/certs`



配置好，我们就能直接能在自己的主机中 使用 虚拟机中的 Docker API。

## 一些其他的问题解决

### 更改Docker 默认镜像路径

一般的我们系统盘不会给很大的空间，然而 Docker 镜像占用的空间一般都是非常大的，所以我们需要将镜像和容器挂在到其他数据盘下。

docker 默认的数据目录都在`/var/lib/docker/` 下，我们只要将这个目录挂载到其他数据盘目录下。

```bash
# 前提关闭 docker 
# 1. 数据盘中新建一个目录，用于存放docker数据
# 2. 将原数据目录移动到新建的目录中
# 3. 创建软链接，将数据盘中的docker数据目录挂到/var/lib/docker/
# 4. 重启 docker 查询 Docker 信息
# 查询到结果 ‘Docker Root Dir: /media/kronchan/文件/kronchan/tools/docker’

```

### WARNING: No memory limit support

查询 `sudo docker info`的时候发现警告信息：`WARNING: No memory limit support`。

解决方案：

```bash
$ sudo vim /etc/default/grub
# 加入 GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
# 更新 grub
$ sudo grub-update
sudo: grub-update：找不到命令
$ sudo update-grub
Generating grub configuration file ...
Found theme: /boot/grub/themes/deepin/theme.txt
Found background image: /boot/grub/themes/deepin/background.png
Found linux image: /boot/vmlinuz-4.9.0-deepin13-amd64
Found initrd image: /boot/initrd.img-4.9.0-deepin13-amd64
Found deepin image: /boot/deepin/vmlinuz-4.13.4
Found initrd image: /boot/deepin/initrd.img-4.13.4
Adding boot menu entry for EFI firmware configuration
done
# 重启系统，然后检查查看问题是否存在
```



## Reference

* [Docker Documentation](https://docs.docker.com/)
* [Docker — 从入门到实践](https://legacy.gitbook.com/book/yeasy/docker_practice/details)
* [Docker 中文指南](http://www.widuu.com/docker/index.html)
* [远程连接docker daemon，Docker Remote API](https://deepzz.com/post/dockerd-and-docker-remote-api.html)
* [优雅地实现安全的容器编排 - Docker Secrets](https://yq.aliyun.com/articles/91396)

