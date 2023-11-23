---
layout: post
title: "Dokcer operations"
category: tools
date: 2022-05-26
---

### 私有仓库的鉴权问题

docker login -u `your_email` -p `your_cli_secret` `仓库地址`

### 关于Arch的docker使用

开始docker.service

```bash
sudo systemctl start docker 

sudo systemctl enable docker 
```

然后就直接拉取镜像就好

### Docker Operations

将当前用户添加到docker用户组。

为了避免每次使用`docker`命令都需要加上`sudo`权限，可以将当前用户加入安装中自动创建的`docker`用户组(可以参考[官方文档](https://docs.docker.com/engine/install/linux-postinstall/))：

```bash
sudo usermod -aG docker $USER
```

#### 镜像（images）

1. `docker pull ubuntu:20.04`：拉取一个镜像
2. `docker images`：列出本地所有镜像
3. `docker image rm ubuntu:20.04` 或 `docker rmi ubuntu:20.04`：删除镜像`ubuntu:20.04`
4. `docker [container] commit CONTAINER IMAGE_NAME:TAG`：创建某个`container`的镜像
5. `docker save -o ubuntu:20.04.tar ubuntu:20.04`：将镜像`ubuntu:20.04`导出到本地文件`ubuntu:20.04.tar`中
6. `docker load -i ubuntu:20.04.tar`：将镜像`ubuntu:20.04`从本地文件`ubuntu:20.04.tar`中加载出来

#### 容器(container)

1. `docker [container] create -it ubuntu:20.04`：利用镜像`ubuntu:20.04`创建一个容器。

2. `docker ps -a`：查看本地的所有容器

3. `docker [container] start CONTAINER`：启动容器

4. `docker [container] stop CONTAINER`：停止容器

5. `docker [container] restart CONTAINER`：重启容器

6. `docker [contaienr] run -itd ubuntu:20.04`：创建并启动一个容器

7. `docker [container] attach CONTAINER`进入容器
   - 先按`Ctrl-p`，再按`Ctrl-q`可以挂机容器

8. `docker [container] exec CONTAINER COMMAND`：在容器中执行命令

9. `docker [container] rm CONTAINER`：删除容器

10. `docker container prune`：删除所有已停止的容器

11. `docker export -o xxx.tar CONTAINER`：将容器`CONTAINER`导出到本地文件`xxx.tar`中

12. `docker import xxx.tar image_name:tag`：将本地文件`xxx.tar`导入成镜像，并将镜像命名为`image_name:tag`

13. `docker export/import`与`docker save/load`的区别：

    - `export/import`会丢弃历史记录和元数据信息，仅保存容器当时的快照状态
    - `save/load`会保存完整记录，体积更大

14. `docker top CONTAINER`：查看某个容器内的所有进程

15. `docker stats`：查看所有容器的统计信息，包括CPU、内存、存储、网络等信息

16. `docker cp xxx CONTAINER:xxx` 或 `docker cp CONTAINER:xxx`在本地和容器间复制文件

17. `docker rename CONTAINER1 CONTAINER2`：重命名容器

18. `docker update CONTAINER --memory 500MB`：修改容器限制

### 可以参考的命令

```bash
docker run -it --name linux_update  --memory="3072M" --memory-swap="-1"  --cpu-shares=8 -d -v /home/han/work/sdk_update:/root [your_images] /bin/bash
```

## Ref

1. [docker for beginner](https://docker-curriculum.com/)

2. [docker overview official](https://docs.docker.com/get-started/)
