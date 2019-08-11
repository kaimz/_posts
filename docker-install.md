---
title: 安装 Docker
date: 2018-03-13 11:11:03
tags: [Docker]
categories: 学习笔记
---

安装之前需要注意的是系统内核版本，linux内核至少在3.10版本以上，使用 command `uname -r` 查看linux内核版本。

<!--more-->

### CentOS 安装
```bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，你可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将 [docker-ce-test] 下方的 enabled=0 修改为 enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2 : 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```
安装校验
```bash
root@iZbp12adskpuoxodbkqzjfZ:$ docker version
Client:
 Version:      17.03.0-ce
 API version:  1.26
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.0-ce
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64
 Experimental: false
```

加入开机自启
```shell
systemctl enable docker
```
docker安装教程网址


### ubuntu 安装
```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```
### Window 安装 
目前 win 10 pro 支持 typer-V

[阿里云下载](http://mirrors.aliyun.com/docker-toolbox/windows/docker-for-windows/)

在 控制面板 》 程序和功能 》 启用或关闭 Window 功能中勾选上 `Typer-V` 安装并重启，并且去[官网](https://yeasy.gitbooks.io/docker_practice/content/install/windows.html)下载 Docker 的安装包。
>注意的是，网站被墙，需要翻墙，不然下载龟速，或者去国内的镜像网站下载 
>且不是 win 10 pro 或企业版，不能启动 `Typer-V` 只能使用虚拟机环境了，可以下载 `Docker Toolbox`。

### Deepin 安装

我使用的是桌面版 15.5

`deepin` 使用官方的 `debian` 安装方式是安装不成功的。

```bash
# 1. 卸载已经安装的 docker
$ sudo apt-get remove docker docker-engine
# 2. 安装docker-ce与密钥管理与下载相关的依赖库
$ sudo apt-get install apt-transport-https ca-certificates curl python-software-properties software-properties-common
# 3. 下载并安装密钥。
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
# 4. 添加docker官方仓库
$ sudo add-apt-repository   "deb [arch=amd64] https://download.docker.com/linux/debian   wheezy   stable"
# 5. 更新仓库
$ sudo apt-get update
# 6. 安装docker-ce
$ sudo apt-get install docker-ce
```



### 国内镜像加速器

国内从 Docker Hub 拉取镜像有时会遇到困难，此时可以配置镜像加速器。
这里我配置的是阿里云的镜像加速。

获取自己的阿里云仓库的镜像 [地址](https://cr.console.aliyun.com/?spm=a2c4e.11153959.blogcont29941.9.520269d65b5sBo&accounttraceid=7944ca1b-ff8f-4239-91ba-79d103b8e92e#/imageList)  
在左侧的 `镜像加速器` 中可以查看专属加速器地址 。

最新的Linux 系统 ，systemd 的系统，在 `/etc/docker/daemon.json` 中写入，例如我的:
```json
{
  "registry-mirrors": [
    "https://2sloyw2o.mirror.aliyuncs.com"
  ]
}
```
然后重新加载配置文件和重启
```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

>对于使用` Windows 10` 的系统，在系统右下角托盘 Docker 图标内右键菜单选择 `Settings`，打开配置窗口后左侧导航菜单选择 `Daemon`。在 `Registry mirrors` 一栏中填写加速器地址， 保存后 Docker 就会重启并应用配置的镜像地址了。



使用 Docker toolbox 的话，进入虚拟机器更改配置文件：

```bash
$ docker-machine ssh default 
sudo sed -i "s|EXTRA_ARGS='|EXTRA_ARGS='--registry-mirror=加速地址 |g" /var/lib/boot2docker/profile 
exit 
$ docker-machine restart default
```



### Reference
* [Docker CE 镜像源站](https://yq.aliyun.com/articles/110806)
* [镜像加速器](https://yeasy.gitbooks.io/docker_practice/content/install/mirror.html)
