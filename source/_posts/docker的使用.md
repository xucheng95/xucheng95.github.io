---
title: docker的使用
date: 2024-06-13T10:42:37+08:00
tags: Docker
categories: Docker
mathjax: False
---

## Docker简介
&emsp;&emsp;Docker是一种开源的容器化平台，旨在自动化应用程序的部署、扩展和管理。Docker将应用程序及其依赖环境打包在一个轻量级、可移植的容器中，确保应用在不同环境中的一致运行。

## Docker基本概念
1. 镜像 (Image): Docker镜像是一个只读模板，包含运行应用程序所需的所有文件和配置。镜像可以从Docker Hub等镜像仓库下载，也可以通过Dockerfile自行构建。

2. 容器 (Container): 容器是镜像的一个运行实例，包含应用程序的代码和所有依赖。容器是独立、隔离的环境，可以在任何支持Docker的操作系统上运行。

3. Dockerfile: Dockerfile是一个文本文件，包含构建Docker镜像所需的所有命令。通过Dockerfile可以自动化构建镜像的过程。

4. 仓库 (Repository): Docker仓库用来存储和分发Docker镜像。Docker Hub是最常用的公共仓库，用户也可以搭建私有仓库。

5. Docker引擎 (Docker Engine): Docker引擎是Docker的核心组件，负责镜像的构建和容器的运行。它包括守护进程 (daemon)、REST API 和命令行接口 (CLI)。

## Docker基本用法
1. Docker安装\
    &emsp;&emsp;Docker可以在Linux、macOS和Windows上安装。以下是Ubuntu上的安装步骤：
    ```Shell
    sudo apt update
    sudo apt install apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt update
    sudo apt install docker-ce
    sudo systemctl status docker
    ```
    &emsp;&emsp;安装完成后，可以使用以下命令检查Docker是否成功安装：
    ```
    docker --version
    ```
2. 常用命令
   * 拉取镜像
    ```Shell
    docker pull <image_name>
    # 拉取官方的nginx镜像
    docker pull nginx
    ```

   * 运行容器
    ```Shell
    docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
    # 运行一个nginx容器
    docker run -d -p 80:80 nginx
    ```

   * 查看正在运行的容器
    ```Shell
    docker ps
    ```

   * 停止容器
    ```Shell
    docker stop <container_id>
    ```

   * 删除容器
    ```Shell
    docker rm <container_id>
    ```

   * 查看镜像
    ```Shell
    docker images
    ```

   * 删除镜像
    ```Shell
    docker rmi <image_id>
    ```

3. 使用Dockerfile构建镜像\
    &emsp;&emsp;编写一个简单的Dockerfile：
    ```Dockerfile
    # 使用官方的python基础镜像
    FROM python:3.8-slim
    # 设置工作目录
    WORKDIR /app
    # 复制当前目录内容到工作目录
    COPY . /app
    # 安装依赖
    RUN pip install --no-cache-dir -r requirements.txt
    # 指定容器启动时运行的命令
    CMD ["python", "app.py"]
    ```
    
    &emsp;&emsp;使用Dockerfile构建镜像：
    ```Shell
    docker build -t my-python-app .
    ```

    &emsp;&emsp;运行构建的镜像：
    ```Shell
    docker run -d -p 5000:5000 my-python-app
    ```

4. Docker Compose \
    &emsp;&emsp;Docker Compose 是一个用于定义和运行多容器Docker应用的工具。通过使用一个 docker-compose.yml文件，可以定义一组关联的服务，然后使用一个命令启动它们。

    &emsp;&emsp;示例docker-compose.yml文件：
    ```YAML
    version: '3'
    services:
    web:
        image: nginx
        ports:
        - "80:80"
    app:
        build: .
        ports:
        - "5000:5000"
    ```

    &emsp;&emsp;使用docker-compose启动服务：
    ```Shell
    docker-compose up
    ```

5. 运行容器的详细选项 \
   &emsp;&emsp;在运行容器时，有很多命令选项，可以让你更精细地控制容器的行为。以下是一些常用的docker run命令选项及其详细说明：
   * 基本选项
     * -d / --detach：在后台运行容器，返回容器ID
     ```Shell
     docker run -d nginx
     ```
     * -it：以交互模式运行容器并分配一个伪TTY（终端）。通常用于需要手动输入的应用程序或进入容器内的shell。
     ```Shell
     docker run -it ubuntu /bin/bash
     ```
     * --name：为容器指定一个名称。默认情况下，Docker会为容器生成一个随机名称。
     ```Shell
     docker run --name my-nginx nginx
     ```
     * --rm：容器停止后自动删除容器。适用于临时任务。
     ```Shell
     docker run --rm ubuntu echo "Hello, World!"
     ```

   * 端口和网络相关选项
     * -p / --publish：将容器的端口映射到主机的端口。格式为主机端口:容器端口。
     ```Shell
     docker run -d -p 8080:80 nginx
     ```
     * -P / --publish-all：随机映射容器的所有公开端口到主机端口。
     ```Shell
     docker run -d -P nginx
     ``` 
     * --network：指定容器连接的网络。默认情况下，容器连接到默认的bridge网络。
     ```Shell
     docker run -d --network my-network nginx
     ``` 

   * 数据卷相关选项
     * -v / --volume：挂载一个主机目录或卷到容器中。格式为主机目录:容器目录。
     ```Shell
     docker run -d -v /my/host/data:/data nginx
     ``` 
     * --mount：更灵活的挂载选项，支持命名卷、绑定挂载和tmpfs挂载。格式较复杂，但更具可读性和功能性。
     ```Shell
     docker run -d --mount type=bind,source=/my/host/data,target=/data nginx
     ``` 

   * 资源限制相关选项
     * -m / --memory：限制容器使用的内存。可以用b, k, m, g等单位。
     ```Shell
     docker run -d -m 512m nginx
     ```
     * --cpus：限制容器使用的CPU。可以是一个具体值或小数形式。
     ```Shell
     docker run -d --cpus="1.5" nginx
     ```

   * 环境变量相关选项
     * -e / --env：设置环境变量。可以多次使用此选项设置多个环境变量。
     ```Shell
     docker run -d -e MY_VAR=my_value nginx
     ```
     * --env-file：从文件中读取环境变量。文件格式为每行一个KEY=VALUE。
     ```Shell
     docker run -d --env-file ./env.list nginx
     ```

   * 其他常用选项
     * --restart：设置容器的重启策略。常见的策略有 no (默认，不重启)、on-failure (失败时重启)、always (总是重启)、unless-stopped (除非手动停止，否则总是重启)。
     ```Shell
     docker run -d --restart unless-stopped nginx
     ```
     * --link：链接到另一个容器，允许容器间通信。该选项在Docker的新网络模式下已经不推荐使用，建议使用Docker网络来进行容器间通信。
     ```Shell
     docker run -d --link some-other-container nginx
     ``` 

   * 示例综合使用 \
     &emsp;&emsp;下面是一个综合使用上述选项的示例：
     * 运行数据库容器：
     ```Shell
     docker run -d --name my-db \
        -e MYSQL_ROOT_PASSWORD=my-secret-pw \
        -e MYSQL_DATABASE=mydatabase \
        -v /my/host/db:/var/lib/mysql \
        mysql:5.7
     ``` 
     * 运行Web应用容器，并链接到数据库容器：
     ```Shell
     docker run -d --name my-webapp \
        -p 8080:80 \
        --link my-db:db \
        -v /my/host/webapp:/usr/share/nginx/html \
        nginx
     ``` 


