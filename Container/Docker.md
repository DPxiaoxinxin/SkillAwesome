# Docker

[TOC]

## 一、前言

### 1.Docker是什么？

Docker是一个提供给开发者和系统管理员用容器(container)开发、部署和运行应用的平台。使用Linux容器去部署应用的方式叫做容器化(containerization)。

容器化可以让持续集成/持续部署(CI/CD)无缝化:

* 应用不需要系统依赖
* 更新可以推送给分布式应用的任意部分
* 资源密度可以得到优化

通过Docker，扩展应用就是运行一个新的进程(或者说可执行文件)，而不是运行一个更重的VM(虚拟机)

#### 1.1 Container带来的优势：

  * Flexible可伸缩：再复杂的应用也可以容器化
  * Lightweight轻量级：容器可以高效利用和共享主机内核
  * Interchangeable可替换：可以即时更新升级应用
  * Portable便携的：可以在本地编译，部署在云端，在任意地方运行
  * Scalable可扩展：可以增加和自动分配容器副本
  * Stackable可堆叠：可以即时和纵向地堆叠各种服务

#### 1.2 镜像（Image）和容器（Container）：

* Image是一个包含了运行程序需要的所有代码、运行空间、依赖库、环境变量和配置文件的可执行包文件
* Container通过运行Image开启，也就是它是Image的运行实例

#### 1.3 容器（Containers）和虚拟机（Virtual Machines）：

* Container跑在本地Linux上并与其它cotainer共享主机内核
  * 单独占用独立进程，并只使用其需要的内存，使其轻量化
* VM通过虚拟化技术(hypervisor)运行在一个虚拟的子操作系统上面
  * 各个子操作系统共享宿主的资源
  * VM会提供超过大多数程序运行所需要的资源环境
* 对比图：![]()

###二、Docker概述

### 1. 新的开发方式

以前开发应用，比如开发python应用，我们要做的第一件事就是在机器上安装python解释器。但是这里就有问题了，首先我们要确保机器的环境要符合应用运行所需，还要尽量与生产环境匹配。这样维护起来会比较困难。

通过Docker的话，你可以不用去安装Python环境，而是把它作为一个基础镜像。然后你可以把源代码和这个基础镜像一起构建，进而确保你的应用，环境依赖和运行状态都是统一的。

如何构造一个镜像？通过Dockerfile

### 2. 通过Dockerfile定义Container

#### 2.1 Dockerfile是什么？

* Dockerfile定义了容器里面环境的搭建过程
  * 与宿主机器绑定端口：容器里面的网络交互和硬盘都是虚拟化的，与宿主资源完成隔离。
  * 定义哪些文件需要拷贝进去环境
* 完成Dockerfile后，无论在哪运行，构建app的行为都严格安装Dockerfile的执行

#### 2.2 创建APP

当前目录下创建requirements.txt

``` txt
Flask
Redis
```

创建app.py

``` python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```



#### 2.3 定义一个Dockerfile

当前目录下创建Dockerfile文件，复制以下内容

``` shell
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
# 当前目录下的app.py和requirements.txt会被复制进镜像的/app目录下
ADD . /app

# Install any needed packages specified in requirements.txt
# 安装requirements.txt需要的环境依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
# 暴露镜像的80端口以便宿主可以访问app.py的输出
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

#### 2.4 构建App

确保当前目录有以下文件

``` shell
$ ls
Dockerfile		app.py			requirements.txt
```

运行Docker build命令，-t用于给镜像命名或打标签

``` shell
docker build -t friendlyhello .
```

查看存在本地镜像中心构建完成的镜像

``` shell
$ docker image ls

REPOSITORY            TAG                 IMAGE ID
friendlyhello         latest              326387cea398
```

#### 2.5 运行App

运行app时，需要把宿主机器上的端口与container的端口进行绑定

``` shell
docker run -p 4000:80 friendlyhello
```

由于该镜像是基于Python的，没有Redis，因此访问` curl http://localhost:4000`可以看到相应的错误信息。

查看和停止当前运行的容器：

``` shell
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED
1fa4ab2cf395        friendlyhello       "python app.py"     28 seconds ago

$ docker container stop 1fa4ab2cf395
```

#### 2.6 共享镜像

类似于Github，Docker注册中心(refistry)包含一系列仓库(repository)，每个仓库由一系列镜像迭代更新组成，不同在于这里的代码是构建过的。每个用户都可以自行在注册中心创建仓库。

* 登陆Dokcer，默认是公共的注册中心

  ``` shell
  $ docker login
  ```

* 给镜像打标签

  ``` shell
  # tag是可选的，但推荐使用，用于标识镜像版本
  docker tag image username/repository:tag

  # Example
  docker tag friendlyhello john/get-started:part2

  $ docker image ls

  REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
  friendlyhello            latest              d9e555c53008        3 minutes ago       195MB
  john/get-started         part2               d9e555c53008        3 minutes ago       195MB
  python                   2.7-slim            1c7128a655f6        5 days ago          183MB
  ...
  ```

* 发布镜像

  ``` shel
  docker push username/repository:tag
  ```

* 远端拉取和运行镜像

  ``` shell
  # 如果本地没有该镜像，会自动从远程仓库拉取
  docker run -p 4000:80 username/repository:tag
  ```

### 3 Services服务

#### 3.1 Service是什么？

在一个分布式系统中，不同功能的app称为service

* 比如一个最简单的网站，web为一个service，数据库为一个service，redis为一个service
* 每一种镜像是一个service，但service和镜像怎么运行没有关系。
  * 一个service可以由多个镜像运行的容器组成
  * 扩展一个service只需要提高容器的数量，进而占用更多宿主资源

在Docker平台可以通过docker-compose.yml定义、运行及扩展services

#### 3.2 定义Service

创建docker-compose.yml文件

``` txt
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    # 从注册中心下载镜像
    image: username/repo:tag
    deploy:
      # 运行5个实例作为service，该service名为web
      replicas: 5
      resources:
        # 限制每个实例最高使用10%的CPU及50M内存
        limits:
          cpus: "0.1"
          memory: 50M
      # 失败时立即重启
      restart_policy:
        condition: on-failure
    # 宿主的80端口与该servie的80端口绑定
    ports:
      - "80:80"
    # 该web下各容器通过webnet负债均衡共享80端口(容器内部通过临时端口链接到webnet)
    networks:
      - webnet
# 定义一个默认设置的webnet负载均衡网络，默认 round-robin方式
networks:
  webnet:
```

#### 3.3 运行分布式App

``` shell
$ docker swarm init
# getstartedlab为app名字
$ docker stack deploy -c docker-compose.yml getstartedlab
# 查看service, servie名字为：AppName_ServiceName，此处应该为getstartedlab_web
$ docker service ls
```

* 每一个单独运行service的容器称为task，每个task有根据副本数量独立递增的id

``` shell
docker service ps getstartedlab_web
```

运行`curl -4 http://localhost`查看该web服务，每一次请求被选中的task不同，展示的结果也不同

#### 3.4 扩展应用

改变yml文件中repicas的数量后重新运行docker stack deploy命令。Docker会在内部自动更新副本数量，不需要停止整个stack或任意容器。

```
docker stack deploy -c docker-compose.yml getstartedlab
```

#### 3.5 停止app和swarm

``` shell
docker stack rm getstartedlab
docker swarm leave --force
```

### 4 Swams集群

#### 4.1 Swam是什么？

Swarm是一组加入集群并都运行Docker的机器，每台机器都称之为node

* 构造：
  * swarm manager：swarm的管理节点。运行的docker命令都会通过swarm manager运行在集群上。
  * workers：通过manager认证的节点，提供工作环境
* 运行container策略：
  * emptiest node：先填充最少使用的机器
  * global：保证每一台机器至少有一个容器在执行
* 加入swarm的机器可以是物理机或虚拟机

#### 4.2 设置swarm

``` shell
# 当前机器：开启swarm模式，并设置该机器为manager节点
$ docker swarm init
# 其它机器：开启swarm模式，加入manager作为node节点
 docker swarm join --token [TOKEN] [manager IP]:2377
```

PS: 可以使用docker-machine创建其它节点并加入swarm，此处不再赘述

#### 4.3 在swam上部署app

``` shell
# 确保相关镜像已经发布到registry，否则其它node从本地和远端获取不到镜像，所有容器会运行在当前结点。或者把该镜像导入到其它node上
$ docker stack deploy -c docker-compose.yml getstartedlab
# 如果镜像存储在私有registry
docker login registry.example.com
docker stack deploy --with-registry-auth -c docker-compose.yml getstartedlab
# 查看servie运行情况
$ docker stack ps getstartedlab

ID            NAME                  IMAGE                   NODE   DESIRED STATE
jq2g3qp8nzwx  getstartedlab_web.1   john/get-started:part2  myvm1  Running
88wgshobzoxl  getstartedlab_web.2   john/get-started:part2  myvm2  Running
vbb1qbkb0o2z  getstartedlab_web.3   john/get-started:part2  myvm2  Running
ghii74p9budx  getstartedlab_web.4   john/get-started:part2  myvm1  Running
0prmarhavs87  getstartedlab_web.5   john/get-started:part2  myvm2  Running
```

#### 4.4 访问App

背后的机制：

* 每个加入swarm的node节点，相关的端口会加入内部的路由策略中，或者说是一个负载均衡机制。保证swarm上每一个绑定端口的service，无论哪个node节点运行容器，该端口总是能链接到自己。

架构图: ![]()

### 5 Stacks服务栈

#### 5.1 Stack是什么？

Stack是一系列共享依赖环境，一起被编排和扩展伸缩的服务的组合。

* 一个Stack有能力去定义或协调一个完整应用的功能；非常复杂的应用可能需要多个stack协作

#### 5.2 定义stack并部署

之前的docker-compose.yml就是stack的定义文件，不过只有一个service，现重新扩展它增加service

``` txt
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  # 与web同级的serice，名为visualizer，可以图形化展示services在swarm的运行状态
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    # 让visualizer可以访宿主Docker的socket文件
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    # 保证该visualizer只运行在manager节点上
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
  # redis提供持久化服务
  redis:
    image: redis
    ports:
      - "6379:6379"
    # 容器的/data与宿主的/home/docker/data绑定，作为数据存储的目录
    # 宿主的目录要提前保证存在
    volumes:
      - "/home/docker/data:/data"
    # 保证只在manager运行并使用同一个文件系统
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks:
  webnet:
```

* Redis能持久化数据的原因：
  * placement字段限制了运行的环境，保证只在同一台机器运行
  * volume绑定了宿主的资源，相关数据是存储在宿主的，容器销毁也不影响

### 6. Container、Service、Stack和Swarm的关系

* Swarm：Swarm是一系列node的集合，每个node上可能运行很多container，每个container隶属的service和stack可能不一样
* 通常情况下，一个App应用就是一个stack。特别复杂的app可能会有多个stack组成
  * 一个stack包含1或多个service，每个service的功能都是一致的
    * 一个service包含1或多个container，它们都是同一个image的实例
    * container是最小的运行单元，运行在node上
  * 一个stack分布在一个swam上

## 三、 其它

### 1 Swarm与Kubernetes的关系

* Swarm是Docker官方的容器集群服务
* Kubernetes是Google开源的自动部署、扩展和管理容器化应用程序的系统。简单说：K8s=Dcoker Swarm + Docker stack(容器编排)

### 2 Docker Compose

#### 2.1 Docker Compose是什么？

* Compose是一个定义和运行多个基于Docker容器应用的工具

  * 使用YAML定义App的services

* Compose可以工作在所有的环境：prd、stg、dev、test，以及CI

* 使用compose的步骤：

  * 用Dockerfile定义app的运行环境，生成可以被随处运行的镜像

  * 用docker-compose.yml定义app的services，这些services可以在单独的环境中协作运行

    * docker-compose.yml示例

      ``` txt
      version: '3'
      services:
        web:
          build: .
          ports:
          - "5000:5000"
          volumes:
          - .:/code
          - logvolume01:/var/log
          links:
          - redis
        redis:
          image: redis
      volumes:
        logvolume01: {}
      ```

  * 运行` docker-compose up`，Compose 会随之启动并运行app

* Compose功能：

  * 在一台宿主机器上拥有多个隔离的环境
    * Compose使用项目名(project name)来隔离环境
    * 默认的项目名是项目文件夹的名字，使用命令行工具可以用` -p` 指定
  * 容器创建后可以保留数据(Volumn Data)
    * ` docker-compose up`运行后，如果找到先前运行的任意容器，compose会复制旧容器的卷数据到新容器中，进而保证卷中的任意数据不会丢失
  * 只在容器发生变化时候重新创建
    * compose会缓存相关配置用于创建容器。当重启一个没有改变的service，compose会复用现存的容器。
    * 复用现存容器意味着可以迅速对环境做改变
  * 在不同环境设置变量和变量组合
    * compose支持在compose文件设置变量
    * 针对不同的环境或用户定制化变量组合
    * 使用` extends` 参数可以扩展compose文件或使用多个compose文件

* 单机部署

  * compose一般侧重于开发和测试流程，适用于单机部署，对生产环境复杂的架构可能无能为力。
  * 如涉及到容灾部署，即app要运行多份实例在不同机器上，可以使用Dokcer Machine或者Dcoker Swarm。

#### 2.2 Compse安装

参考官方文档[Install Docker Compose](https://docs.docker.com/compose/install/)

#### 2.3 定义使用Compose

##### step1：初始化

``` shell
$ mkdir composetest
$ cd composetest
```

``` shell
$ vi app.py

import time

import redis
from flask import Flask


app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)


def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)


@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

``` shell
$ vi requirements.txt

flask
redis
```

##### step2: 创建Dockerfile

``` shell
$ vi Dockerfile

FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

##### step3: 创建Compose

``` shell
$ vi docker-compose.yml

version: '3'
services:
  web:
  	# 使用当前目录下Dockerfile构建的镜像
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

##### step4: 构建和启动app

``` shell
$ docker-compose up
$ curl http://0.0.0.0:5000
$ curl http://0.0.0.0:5000
$ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
composetest_web         latest              e2c21aa48cc1        4 minutes ago       93.8MB
python                  3.4-alpine          84e6077c7ab6        7 days ago          82.5MB
redis                   alpine              9d8fa9aa0e5b        3 weeks ago         27.5MB
$ docker-compose down
```

##### step5: 编辑compose绑定资源

通过volumes绑定当前目录和容器的/code，这样在宿主改了源代码，也不用重新打包镜像

``` shell
$ vi docker-compose.yml

version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
```

##### step6: 重新构建和启动app

``` shell
$ docker-compose up
```

##### step7: 更新应用

``` shell
$ vi app.py

# 编辑hello函数的return
return 'Hello from Docker! I have been seen {} times.\n'.format(count)

$ curl http://0.0.0.0:5000
```

#### 2.4 Compose 文件

##### 1) Version 3版本

###### 概述

* 这篇引用介绍compose文件最顶层的key，例如build、deploy、depends_on、networks等等，并介绍每个key的option。在compose文件中，格式为：` <key>: <option>: <value>`
* 文件后缀以.yml和.yaml命名
* compose文件中service的定义将在service的容器启动时执行
  * 大多数定义传递给` dcoker container create`作为参数运行
    * Dcokerfile定义的options，将为默认执行，不需要在compose重复定义
  * 少部分像network和volumn传递给` docker network create` 和 `docker volumn create`
  * 可以使用环境变量，格式为` ${VARIABLE}`

###### Service配置

* build：在镜像构建时候应用

  ``` yaml
  version: '3'
  services:
    webapp:
      build: ./dir
  ```

  * context：包含Dockerfile的目录、url或git repository

    ``` yaml
    build:
      context: ./dir
    ```

  * dockerfile：选择特定的Dockerfile文件

    ```yaml
    build:
      context: .
      dockerfile: Dockerfile-alternate
    ```

  * args：构建过程中的环境变量

    * 在Dockerfile声明和使用参数

      ``` txt
      ARG buildno
      ARG gitcommithash

      RUN echo "Build number: $buildno"
      RUN echo "Based on commit: $gitcommithash"
      ```

    * 在compose文件定义参数

      ``` yaml
      build:
        context: .
        args:
          buildno: 1
          gitcommithash: cdc3b19

      build:
        context: .
        args:
          - buildno=1
          - gitcommithash=cdc3b19
          
      # 如忽略value则使用环境变量的值
      args:
        - buildno
        - gitcommithash
      ```

      ​

* image：构建后的镜像名字将以webapp和tag命名

  ``` yaml
  build: ./dir
  image: webapp:tag
  ```

* command：覆盖Dockerfile的command

* configs

* container_name：定义container名字

* credential_spec

* deploy：定义service的部署和运行规则。**Version3独有**。只在swarm mode和docker stack deploy下有效，` docker-composer up/run会忽略` 

  ``` yaml
  version: '3'
  services:
    redis:
      image: redis:alpine
      deploy:
        replicas: 6
        update_config:
          parallelism: 2
          delay: 10s
        restart_policy:
          condition: on-failure
  ```

  * ENDPOINT_MODE：外部客户端链接到swarm发现service的规则

    * endpoint_mode: vip。默认配置。Docker给每一个service绑定vip(virtual ip)作为客户端访问service的前端。Docker的路由策略会让每次请求在可工作的worker节点选择，客户端不需要知道service包含了多少节点。
    * endpoint_mode: dnsrr。DNS round-roubin(DNSRR)不再使用vip。它给每个service绑定DNS记录，每次DNS查询直接返回一系列ip和端口，客户端再根据需要链接其中一个。通常在想自定义负载均衡策略或混合使用win和linux的app使用。

    ``` yaml
    version: "3.3"

    services:
      wordpress:
        image: wordpress
        ports:
          - "8080:80"
        networks:
          - overlay
        deploy:
          mode: replicated
          replicas: 2
          endpoint_mode: vip

      mysql:
        image: mysql
        volumes:
           - db-data:/var/lib/mysql/data
        networks:
           - overlay
        deploy:
          mode: replicated
          replicas: 2
          endpoint_mode: dnsrr

    volumes:
      db-data:

    networks:
      overlay:
    ```

  * labels: 给service绑定一个标签，但是不会给service里面的container绑定

    ``` yaml
    version: "3"
    services:
      web:
        image: web
        deploy:
          labels:
            com.example.description: "This label will appear on the web service"
    ```

    * 如果想给container绑定标签，要在deploy标签外使用

      ``` shell
      version: "3"
      services:
        web:
          image: web
          labels:
            com.example.description: "This label will appear on all containers for the web service"
      ```

  * mode：容器在节点的部署方式

    * global：默认值。每个节点部署一个容器
    * replicated：只部署特定数量的容器

    ``` yaml
    version: '3'
    services:
      worker:
        image: dockersamples/examplevotingapp_worker
        deploy:
          mode: global
    ```

  * placement：service的部署环境限制

    ``` yaml
    version: '3.3'
    services:
      db:
        image: postgres
        deploy:
          placement:
            constraints:
              - node.role == manager
              - engine.labels.operatingsystem == ubuntu 14.04
            preferences:
              - spread: node.labels.zone
    ```

  * replicas：service是replicated模式，指定service运行时容器的数量

    ``` yaml
    version: '3'
    services:
      worker:
        image: dockersamples/examplevotingapp_worker
        networks:
          - frontend
          - backend
        deploy:
          mode: replicated
          replicas: 6
    ```

  * resources：资源限制。官网是说限制service？到底是限制service还是container？

    ``` yaml
    version: '3'
    services:
      redis:
        image: redis:alpine
        deploy:
          resources:
            # 最高不超过50m的内存和50%的cpu
            limits:
              cpus: '0.50'
              memory: 50M
            # 至少预留25%的cpu和20m的内存以供使用
            reservations:
              cpus: '0.25'
              memory: 20M
    ```

    * Out Of Memory Exceptions(OOME)：如果service或container尝试去使用超过系统提供的内存，可能会抛出该异常，并且container或者Docker daemon可能被内核的OOM killer杀掉。因此要确保应用有合适的内存运行

  * restart_policy：定义container在退出时怎么重启

    * condition：触发条件。none，on-failure，any(default)
    * delay：两次尝试重启的间隔，默认0
    * max_attemps：尝试重启的次数（默认不放弃）。如果在window窗口内重启失败，该重启不计入max_attemps，window窗口前的才计入（If the restart does not succeed within the configured `window`, this attempt doesn’t count toward the configured `max_attempts` value. For example, if `max_attempts` is set to ‘2’, and the restart fails on the first attempt, more than two restarts may be attempted.）
    * window：决定重启是否成功之前的等待时间，即重启后等待n秒再判断重启状态（默认是immediately）

    ``` yaml
    version: "3"
    services:
      redis:
        image: redis:alpine
        deploy:
          restart_policy:
            condition: on-failure
            delay: 5s
            max_attempts: 3
            window: 120s
    ```

    ​

  * update_config：配置service怎么被更新的。特别适用于滚动更新。、

    * parrallelism：同时更新的container数量
    * delay：两组container更新的间隔
    * failuer_action：更新失败的策略。continue、rollback、pause(defalut)
    * minitor：每次任务更新后监控失败的时间。默认0s
    * max_failure_ratio：更新时最大的失败率
    * order：更新时的操作顺序。stop-first，开启新任务时旧任务先停止(deault)；start-first，优先启动新任务，会与正在运行的任务短暂交叠。

    ``` yaml
    version: '3.4'
    services:
      vote:
        image: dockersamples/examplevotingapp_vote:before
        depends_on:
          - redis
        deploy:
          replicas: 2
          update_config:
            parallelism: 2
            delay: 10s
            order: stop-first
    ```

* devices：device的绑定。与docker create的--device参数一致

  ``` yaml
  devices:
    - "/dev/ttyUSB0:/dev/ttyUSB0"
  ```

* depends_on：service间的依赖策略。以下例子db和redis先于web启动，但是不等待它们处于ready状态，仅需要它们已启动

  * ` docker-composer up`启动时按照该顺序来。
  * ` docker-composer up SERVICE`自动包含该service的依赖。
  * swarm mode下会忽略该字段。

  ``` yaml
  version: '3'
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

* dns：定义dns服务器

  ``` shell
  dns: 8.8.8.8
  dns:
    - 8.8.8.8
    - 9.9.9.9
  ```

* dns_search：定义dns查询的域名

  ``` shell
  dns_search: example.com
  dns_search:
    - dc1.example.com
    - dc2.example.com
  ```

* tmpfs：绑定临时的文件系统到container中。swarm mode下会被忽略

  ``` yaml
  tmpfs: /run
  tmpfs:
    - /run
    - /tmp
    
  - type: tmpfs
    target: /app
    tmpfs:
    	size: 1000
  ```

* entrypoint：覆盖默认的entrypoint

  ``` yaml
  entrypoint: /code/entrypoint.sh

  entrypoint:
      - php
      - -d
      - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
      - -d
      - memory_limit=-1
      - vendor/bin/phpunit
  ```

* env_file：环境变量文件

* environment：环境变量

  ``` yaml
  environment:
    RACK_ENV: development
    SHOW: 'true'
    SESSION_SECRET:

  environment:
    - RACK_ENV=development
    - SHOW=true
    - SESSION_SECRET
  ```

* expose: 暴露的端口只在依赖的服务间可以访问，宿主机器是无法访问到的。只有内部的端口可以被配置

  ``` yaml
  expose:
   - "3000"
   - "8000"
  ```

* external_links：连接不在compose文件定义的container

  ``` yaml
  external_links:
   - redis_1
   - project_db_1:mysql
   - project_db_1:postgresql
  ```

* extra_hosts：绑定hostname。与docker client 的--add host参数一致

  ``` shell
  extra_hosts:
   - "somehost:162.242.195.82"
   - "otherhost:50.31.209.229"
   
  #将会更新容器内部的/etc/hosts
  162.242.195.82  somehost
  50.31.209.229   otherhost
  ```

* healthcheck：健康检查

  ``` yaml
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost"]
    interval: 1m30s
    timeout: 10s
    retries: 3
    start_period: 40s
  ```

* image：指定启动container的镜像。如果镜像不在本地，docker会尝试拉取它。除非使用了build

  ``` yaml
  image: redis
  image: ubuntu:14.04
  image: tutum/influxdb
  image: example-registry.com:4000/postgresql
  image: a4bc65fd
  ```

* isolation：container的隔离策略。Linux下默认值是default。window下为default，process和hyperv

* labels：给container绑定标签。推荐使用反向DNS避免这些label与其它也使用的软件冲突。

  ``` yaml
  labels:
    com.example.description: "Accounting webapp"
    com.example.department: "Finance"
    com.example.label-with-empty-value: ""

  labels:
    - "com.example.description=Accounting webapp"
    - "com.example.department=Finance"
    - "com.example.label-with-empty-value"
  ```

* links：连接其它service内的容器。该标签可能会在后续被移除，不在考虑。swarm mode也被忽视。

* logging：日志配置

  ``` yaml
  logging:
    # driver: none, syslog, json-file(default)
    driver: syslog
    options:
      syslog-address: "tcp://192.168.0.42:123"
      max-size: "200k"
      max-file: "10"
  ```

* network_mode：网络模式。与--net参数一致。swarm mode忽略

  ``` yaml
  network_mode: "bridge"
  network_mode: "host"
  network_mode: "none"
  network_mode: "service:[service name]"
  network_mode: "container:[container name/id]"
  ```

* networks：要加入的network，在顶层networks标签定义

  ``` yaml
  services:
    some-service:
      networks:
       - some-network
       - other-network
  ```

  * alias：network中service的别名
  * IPV4_ADDRESS, IPV6_ADDRESS

* pid：设置pid模式绑定到主机的pid中。开启后容器可以与宿主机器共享pid地址空间。开启这个功能的容器可以访问并操作其它也共享命名空间的容器。（Containers launched with this flag can access and manipulate other containers in the bare-metal machine’s namespace and vise-versa.）

* ports：暴露容器端口

  * 短语法：` HOST:CONTAINER` 格式，或者只有container端口(会绑定主机的临时端口)

    ``` yaml
    ports:
     - "3000"
     - "3000-3005"
     - "8000:8000"
     - "9090-9091:8080-8081"
     - "49100:22"
     - "127.0.0.1:8001:8001"
     - "127.0.0.1:5000-5010:5000-5010"
     - "6060:6060/udp"
    ```

  * 长语法:

    ``` yaml
    ports:
    	# container内部的端口
      - target: 80
        # 公开暴露的端口
        published: 8080
        # 端口协议
        protocol: tcp
        # host在每个节点暴露主机端口，ingress是在swarm mode下的负载均衡
        mode: host
    ```

* secrets：每个service的访问权限控制。secret必须存在或者在顶级key配置里定义，否则stack部署会失败。

  * 短语法：

    ``` yaml
    version: "3.1"
    services:
      redis:
        image: redis:latest
        deploy:
          replicas: 1
        secrets:
          - my_secret
          - my_other_secret
    secrets:
      my_secret:
        file: ./my_secret.txt
      my_other_secret:
        external: true
    ```

  * 长语法：

    ``` yaml
    version: "3.1"
    services:
      redis:
        image: redis:latest
        deploy:
          replicas: 1
        secrets:
          - my_secret
          - my_other_secret
    secrets:
      my_secret:
        file: ./my_secret.txt
      my_other_secret:
        external: true
    ```

* security_opt：覆盖每个容器的标签方案(labeling scheme)。swarm下忽视

  ``` yaml
  security_opt:
    - label:user:USER
    - label:role:ROLE
  ```

* stop_grace_period：当尝试去停止容器但却没捕捉到SIGTERM信号时，在发送SIGKILL信号前等待多长时间。默认情况下是10s

  ``` yaml
  stop_grace_period: 1s
  stop_grace_period: 1m30s
  ```

* stop_signal：给容器设置一个停止信号。默认是SIGTERM

  ``` yaml
  stop_signal: SIGUSR1
  ```

* sysctls：定义容器的内核参数。swarm下忽略

  ``` yaml
  sysctls:
    net.core.somaxconn: 1024
    net.ipv4.tcp_syncookies: 0

  sysctls:
    - net.core.somaxconn=1024
    - net.ipv4.tcp_syncookies=0
  ```

* ulimits：覆盖容器默认的ulimits。swarm下忽略

  ``` yaml
  ulimits:
    nproc: 65535
    nofile:
      soft: 20000
      hard: 40000
  ```

* userns_mode：如果docker daemon配置了用户的命名空间，关闭该service的用户命名空间。swarm下忽略

  ``` yaml
  userns_mode: "host"
  ```

* volumes：给service绑定主机目录或命名卷(named volumes)

  * tips：

    * 如果仅仅是给一个service绑定主机路径，不需要在顶级key中的volumn定义
    * 如果需要在多个service共用volumn，需要在顶级key中的volumn定义命名卷

    ``` yaml
    version: "3.2"
    services:
      web:
        image: nginx:alpine
        volumes:
          - type: volume
            source: mydata
            target: /data
            volume:
              nocopy: true
          - type: bind
            source: ./static
            target: /opt/app/static

      db:
        image: postgres:latest
        volumes:
          - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
          - "dbdata:/var/lib/postgresql/data"

    volumes:
      mydata:
      dbdata:
    ```

  * 短语法：定义路径` HOST:CONTAINER`，或者路径+权限控制` HOST:CONTAINER:ro`

    ``` yaml
    volumes:
      # Just specify a path and let the Engine create a volume
      - /var/lib/mysql

      # Specify an absolute path mapping
      - /opt/data:/var/lib/mysql

      # Path on the host, relative to the Compose file
      - ./cache:/tmp/cache

      # User-relative path
      - ~/configs:/etc/configs/:ro

      # Named volume
      - datavolume:/var/lib/mysql
    ```

  * 长语法：

    * bind：bind类型下的额外参数。可定义propagation来设置propagation
    * volume：volume类型下的额外参数。可定义nocopy
    * tmpfs：tmpfs类型下的额外参数。可定义size来设置tmpfs卷的大小

    ``` yaml
    version: "3.2"
    services:
      web:
        image: nginx:alpine
        ports:
          - "80:80"
        volumes:
            # 挂卷的类型,volume, bind或者tmpfs
          - type: volume
          	# 挂载源、主机的路径或者命名卷。tmpfs下不可用
            source: mydata
            # 挂载到容器上的路径
            target: /data
            # volume类型下的额外参数
            volume:
              # 当volume创建时不从container中复制数据
              nocopy: true
          - type: bind
            source: ./static
            target: /opt/app/static

    networks:
      webnet:

    volumes:
      mydata:
    ```

  * Note：

    * 如果service有多个容器部署在多个节点上，使用命名卷时，docker会给service的每个任务建立一个匿名卷。在容器销毁后这些匿名卷也会被移除而不能持久化数据。

    * 持久化数据的方式：1. 使用命名卷和一个卷驱动(volume driver)做到多主机感知，这样任何节点都可以访问到数据。2.给service设置约束保证该service只部署在一个有当前卷节点上

      ``` yaml
      version: "3"
      services:
        db:
          image: postgres:9.4
          volumes:
            - db-data:/var/lib/postgresql/data
          networks:
            - backend
          deploy:
            placement:
              constraints: [node.role == manager]
      ```

* restart：重启策略

  ```yaml
  # 默认方式。在任何环境下都不会重启
  restart: "no"
  # 任意环境下都会重启
  restart: always
  # 当退出码(exit code)是on-failure时重启
  restart: on-failure
  restart: unless-stopped
  ```

* domainname, hostname, ipc, mac_address, privileged, read_only, shm_size, stdin_open, tty, user, working_dir

  ``` yaml
  user: postgresql
  working_dir: /code

  domainname: foo.com
  hostname: foo
  ipc: host
  mac_address: 02:42:ac:11:65:43

  privileged: true
  ```


  read_only: true
  shm_size: 64M
  stdin_open: true
  tty: true
  ```

* 延迟时间定义方式：某些配置下可用

  ``` yaml
  2.5s
  10s
  1m30s
  2h32m
  5h34m56s
  ```

* 字节数定义方式：某些配置下可用。

  ``` yaml
  2b
  1024kb
  2048k
  300m
  1gb
  ```

###### Volume 配置

创建命名卷，以便可以在多个service共用，并且这些命名卷可以通过docker的API进行管理。

``` yaml
# db及其backup共用同个卷，以便即时备份数据
version: "3"

services:
  db:
    image: db
    volumes:
      - data-volume:/var/lib/db
  backup:
    image: backup-service
    volumes:
      - data-volume:/var/lib/backup/data

# data-volume没有配置，Engine会使用默认的驱动(大多数是local)
volumes:
  data-volume:
```

* driver：定义该卷的驱动。Engine的默认驱动大部分是local驱动。如果驱动不可用，engine会在`docker-compose up`创建卷时报错。

  ``` yaml
   driver: foobar
  ```

* driver_opts：定义变量传递给驱动。这些变量都是基于驱动相关的。

  ``` yaml
   driver_opts:
     foo: "bar"
     baz: 1
  ```

* external：如果设置为true，意味着该volume已经在compose外面创建过了。`docker-compose up`不会去创建它，而且它不存在的话将会报错。

  * 不能与`driver` ,`driver_opts`共同使用
  * 在swarm mode下，如果外部的卷不存在，该卷将会在service被调度到的节点上自动创建

  ``` yaml
  version: '2'

  services:
    db:
      image: postgres
      volumes:
        - data:/var/lib/postgresql/data

  volumes:
    data:
      external: true
  ```

  ``` yaml
  # 在compose重命名卷
  volumes:
    data:
      external:
        name: actual-name-of-volume
  ```

* labels：给卷加标签。推荐使用反向DNS避免标签与其它使用同样标签的冲突

  ``` yaml
  volumes:
    data:
      external:
        name: actual-name-of-volume
  ```

* names：给卷自定义名字。

  * 该名字可以在包含特殊字符的网络配置使用
  * The name is used as is and will **not** be scoped with the stack name.

  ``` yaml
  version: '3.4'
  volumes:
    data:
      name: my-app-data
    data1:
      external: true
      name: my-app-data
  ```

###### Network配置

* driver：默认配置取决于当前使用的docker engine。大多数情况下，在单主机默认是bridge，swarm下默认是overlay

  ``` yaml
  driver: overlay
  ```

  * bridge：在单主机上使用bridge网络

  * overlay：创建一个命名网络可以在swarm的多节点使用

  * HOST OR NONE：使用主机的网络，或者不用网络。跟 `docker run --net=host` 和 `docker run --net=none`效果一致。仅在`docker stack`下可用，如果使用`docker-compose`，用network_mode代替此配置

    ``` yaml
    services:
      web:
        ...
        networks:
          hostnet: {}

    networks:
      hostnet:
        external: true
        name: host
    ```

    ``` yaml
    services:
      web:
        ...
        networks:
          nonet: {}

    networks:
      nonet:
        external: true
        name: none
    ```

* driver_opts：网络驱动的可选参数

* attachable：只在overlay模式下使用。如果为` true`，除了service，单独的containers可以加入到这个网络。如果单独的containers加入这个overlay网络，Docker damon所有加入该overlay网络的service和独立containers可以互相交流。

  ``` yaml
  networks:
    mynet1:
      driver: overlay
      attachable: true
  ```

* ipam：自定义ipam配置

  * driver：定义驱动
    * config：0个或多个配置块列表，每个配置块都包含以下key
      * subnet：再CIDR中代表网段的subnet配置

  ``` yaml
  ipam:
    driver: default
    config:
      - subnet: 172.28.0.0/16
  ```

* internal：默认情况下，docker也把自己加入bridge网络以便提供与外部的通信。如果想建立的一个外部隔离的overlay网络，此选项设置为`true`。

* labels：设置标签

* external：如果为`true`，意味该netwrok已经再compose外部创建。

  * 不能与以下key共用。`driver`, `driver_opts`, `ipam`, `internal`

  ``` yaml
  version: '2'

  services:
    proxy:
      build: ./proxy
      networks:
        - outside
        - default
    app:
      build: ./app
      networks:
        - default

  networks:
    outside:
      external: true
  ```

* name：给该network自定义名字。

###### configs配置

顶级key中的config定义或导入stack中services访问到的config配置。(The top-level `configs` declaration defines or references [configs](https://docs.docker.com/engine/swarm/configs/) that can be granted to the services in this stack.)。config有两种类型：`file` 和 ` external`

* file：创建指定文件路径的config
* external：如果为`true`，意味该config已经创建。
* name：config对象在Docker中的名字。该字段可以被含有特殊字符的参考配置(reference configs )中使用。

``` yaml
configs:
  # 当stack被部署时创建
  my_first_config:
    file: ./config_data
  # 已经在docker中存在
  my_second_config:
    external: true
  # 引用外部redis_config
  my_third_config:
    external:
      name: redis_config
```

###### secrets配置

顶级key中的secrets定义或导入stack中services访问到的secret配置。参照config配置

* file
* external
* name

###### 变量替换

配置文件可以包含环境变量。

* 使用：

  * compose在` docker-compose`运行的时候寻找并使用这些变量。
  * 如果在环境变量中没找到，将默认为空字符串。
  * 相关环境变量也可以定义在.env文件中

  ``` yaml
  db:
    image: "postgres:${POSTGRES_VERSION}"
  ```

* 语法：

  * ` $VARIABLE` 或者 ` ${VARIABLE}`
  * `${VARAIABLE:-default}`：当变量没有设置或为空使用默认值
  * `${VARAIABLE-default}`：当变量没有设置使用默认值
  * `${VARAIABLE:?err}`：当变量没有设置或为空时，带着err信息退出
  * `${VARAIABLE?err}`：当变量没有设置时，带着err信息退出
  * 使用`$$` 转义成一个$

###### 其它配置

* **复**用字段：使用`x-`字段去重复使用配置块。这些`x-`配置块需定义在头部，使用过程中会被Compose忽略，然后在需要的地方生成对应的配置插入。

  * 假设需要重复引用以下logging配置

    ``` yaml
    logging:
      options:
        max-size: '12m'
        max-file: '5'
      driver: json-file
    ```

  * 可以这样写Compose文件

    ``` yaml
    version: '2.1'
    x-logging:
      &default-logging
      options:
        max-size: '12m'
        max-file: '5'
      driver: json-file

    services:
      web:
        image: myapp/web:latest
        logging: *default-logging
      db:
        image: mysql:latest
        logging: *default-logging
    ```

  * 也可以覆盖重写某些字段。此处用到了yaml的merge语法

    ``` yaml
    version: '2.1'
    x-volumes:
      &default-volume
      driver: foobar-storage

    services:
      web:
        image: myapp/web:latest
        volumes: ["vol1", "vol2", "vol3"]
    volumes:
      vol1: *default-volume
      vol2:
        << : *default-volume
        name: volume02
      vol3:
        << : *default-volume
        driver: default
        name: volume-local
    ```

##### 2) 不同Compose版本间的差异

* Version 1：

  * 遗留的格式
  * 顶部没有version字段
  * 所有的services定义在文件根部
  * 不能声明volumes、networks或者build字段
    * 不能用到networking的优势，所有的container默认使用bridge网络并且使用ip进行通信。
    * 需要使用`links`去连接容器

  ``` yaml
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
    links:
     - redis
  redis:
    image: redis
  ```

* Version2:

  * 需要指定版本，如` version:'2'`

  * 所有的service必须在`services`字段定义

  * 可以在`volumes`下定义命名卷，在`networks`下定义网络

  * 默认情况下，每个容器加入应用级别的默认网络，并且可以通过与service名一样的主机名被发现。意味着version1中的`links`字段不再需要

  * Simple example:

    ``` yaml
    version: '2'
    services:
      web:
        build: .
        ports:
         - "5000:5000"
        volumes:
         - .:/code
      redis:
        image: redis
    ```

  * Extended example：

    ``` yaml
    version: '2'
    services:
      web:
        build: .
        ports:
         - "5000:5000"
        volumes:
         - .:/code
        networks:
          - front-tier
          - back-tier
      redis:
        image: redis
        volumes:
          - redis-data:/var/lib/redis
        networks:
          - back-tier
    volumes:
      redis-data:
        driver: local
    networks:
      front-tier:
        driver: bridge
      back-tier:
        driver: bridge
    ```

* Version3：

  * 最新并且推荐的版本。可以在Compose和swarm模式下使用。
  * 需要指定版本，如` version:'2'`

* tips：

  * 可以尝试使用` docker-compose --compatibilty`尝试迁移，但不要在生产环境使用。
  * 未接触过Compose直接学Version3，需要时再回顾其它版本


#### 2.5 在Swarm使用Compose

* 局限性

  * 所有的镜像必须推送到registry中，否则其它节点获取不到该镜像

  * service的多依赖：

    * 如果service需要和其它service协调调度，swarm可能会把这些依赖service分派到不同节点，导致依赖的service不能协同调度。

    * swarm提供一系列调度以及依赖的关键字，运行您控制容器部署到哪个节点。这些通过`environment`变量设置。

      ``` yaml
      # Schedule containers on a specific node
      environment:
        - "constraint:node==node-1"

      # Schedule containers on a node that has the 'storage' label set to 'ssd'
      environment:
        - "constraint:storage==ssd"

      # Schedule containers where the 'redis' image is already pulled
      environment:
        - "affinity:image==redis"
      ```

    * Example:

      ``` txt
      # 错误版本
      version: "2"
      services:
        foo:
          image: foo
          volumes_from: ["bar"]
          network_mode: "service:baz"
        bar:
          image: bar
        baz:
          image: baz
          
       # 更新版本
       version: "2"
      services:
        foo:
          image: foo
          volumes_from: ["bar"]
          network_mode: "service:baz"
          environment:
            - "constraint:node==node-1"
        bar:
          image: bar
          environment:
            - "constraint:node==node-1"
        baz:
          image: baz
          environment:
            - "constraint:node==node-1"
      ```


#### 2.6 复用Compose文件

* 扩展整个Compose文件通过使用多个Compose

  * 默认情况下，Compose读取两个文件docker-compose.yml和可选的docker-compose.override.yml。

    * 前者包含最基本的配置
    * 后者包含需要覆盖、合并或新增的字段
    * 通过`-f`参数指定yaml文件，Compose会合并相应的文件。需确保所有的文件的路径时相对第一个基础文件的。简单做法：放在同一个目录下面

  * Examples：

    * 基础配置，docker-compose.yml

      ``` yaml
      web:
        image: example/my_web_app:latest
        links:
          - db
          - cache

      db:
        image: postgres:latest

      cache:
        image: redis:latest
      ```

    * 覆盖配置：docker-compose.overrride.yml。使用` docker-compose up`会自动读取该文件。

    * 生产配置：docker-compose.prod.yml。建议存储在不同的gir repo或者被不同团队管理

      ``` yaml
      web:
        ports:
          - 80:80
        environment:
          PRODUCTION: 'true'

      cache:
        environment:
          TTL: '500'
      ```

    * 部署：将会读取并合并`docker-compose.yml`和`docker-compose.prod.yml`两个文件

      ``` yaml
      docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
      ```

      ​

* 扩展独立的services通过` extends`字段。不支持Version3 忽略

#### 2.7 Compose中的网络

* 概述：默认情况下，Compose会给app设置一个单独的网络。service下的每个容器都会加入这个网络并且彼此都是可达的，可以通过和容器名一样主机名来访问。

  * 例如，运行一个名为myapp的项目

    ``` yaml
    version: "3"
    services:
      web:
        build: .
        ports:
          - "8000:8000"
      db:
        image: postgres
        ports:
          - "8001:5432"
    ```

  * 运行`docker-compose up`时

    * 创建myapp_default的默认网络
    * 使用web配置创建一个名为web容器，加入默认网络
    * 使用db配置创建一个名为db的容器，加入默认网络

  * 每个容器都可以通过主机名web和db搜索容器，并得到相应的ip列表

    * 例如访问postgres可以通过` postgres://db:5432`
    * 内部的网络通信通过CONTAINER_PORT来交流，外部访问内部通过HOST_PORT来交流
    * 在web容器中，通过` postgres://db:5432`来连接db，在主机中，通过`postgres://{DOCKER_IP}:8001`来连接db
    * 如果容器升级，旧的容器会被移除新的容器会使用新的ip，但是容器名字是一样的。因此通过容器名查找ip不会受到影响，旧的ip地址也会停止工作
    * 如果容器升级，还有任意其它容器与旧容器保持连接，这些连接将会关闭。容器会去探测这些变动，重新查询新的ip和重连。

  * tips：

    * ` links`可以给sevice设置别名，其它service可已通过该别名寻找

      ``` yaml
      version: "3"
      services:

        web:
          build: .
          links:
            - "db:database"
        db:
          image: postgres
      ```

* 多网络(multi-host networking)模式：

  * 当在swarm集群中部署一个应用，可以使用内建的overlay驱动去开启multi-host在多个容器间交流，而不用改变compose文件配置或应用源码

  * Swarm集群默认都开启overlay驱动。

  * 配置自定义网络Example：

    * 使用顶级key中的networks
    * 每个service通过service级别中的network标明要通过哪个网络连接
    * 以下例子中proxy和db无法互相通信，它们链接到不同的网络
    * networks也可以配置ip地址

    ``` yaml
    version: "3"
    services:

      proxy:
        build: ./proxy
        networks:
          - frontend
      app:
        build: ./app
        networks:
          - frontend
          - backend
      db:
        image: postgres
        networks:
          - backend

    networks:
      frontend:
        # Use a custom driver
        driver: custom-driver-1
      backend:
        # Use a custom driver which takes special options
        driver: custom-driver-2
        driver_opts:
          foo: "1"
          bar: "2"
    ```

* 自定义默认网络

  ``` yaml
  version: "3"
  services:

    web:
      build: .
      ports:
        - "8000:8000"
    db:
      image: postgres

  networks:
    default:
      # Use a custom driver
      driver: custom-driver-1
  ```

* 使用已有的网络

  ``` yaml
  networks:
    default:
      external:
        name: my-pre-existing-network
  ```

#### 2.8 在生产环境中使用Compose

* prd的compose文件可能需要但不局限于以下改变：

  * 移除与宿主源码的绑定，保证容器内部的源码不受外部影响
  * 绑定宿主不同的端口
  * 设置不同的环境变量，比如减少冗余的日志，开启邮件服务等等
  * 设置重启策略`restart:alwayts`去避免宕机
  * 加入额外的service例如logging

* 例如，定义额外的production.yml，包含prd中需要覆盖或新增的变量

  ``` sehll
  docker-compose -f docker-compose.yml -f production.yml up -d
  ```

* 部署更新：

  * 当改变源码的时候，需要重新构建镜像并重新创建容器。

  * 比如重新部署一个web service

    ``` shell
    # 重建web镜像
    $ docker-compose build web
    # 停止web容器、销毁、重新创建，创建时不重建web依赖的容器
    $ docker-compose up --no-deps -d web
    ```

    ​

#### 2.9 Compose和Swarm模式下stack的区别

两者都是为了部署和管理容器，也都使用Compose文件(其中stack只能使用Version3)

* Compose：
  * 是一个Python应用，通过Docker API来操作
  * 目的是在单台机器上定义和运行多个容器
* Stack：
  * 内嵌到Docker Engine的功能
  * 在swarm模式(Docker的调度和编排工具)下使用，因此需要额外的配置(replicas、deploy、roles)
  * 不限制在单台机器上

## Docker命令

###  List Docker CLI commands

``` shell
docker
docker container --help
```

### Display Docker version and info

``` shell
docker --version
docker version
docker info
```

### Docker build

``` shell
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
```

### Execute Docker image

``` shell
docker run hello-world
docker run -p 4000:80 friendlyhello  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker run username/repository:tag                   # Run image from a registry
docker container ls -q                                      # List container IDs
```

### Docker images

``` shell
docker image ls
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
```

### Docker containers (running, all, all in quiet mode)

``` shell
docker container ls
docker container ls --all
docker container ls -aq
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
```

### Docker registry

``` shell
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
```

### Docker stack

``` shell
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker stack rm <appname>                             # Tear down an application
docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
```

### Docker Swarm

``` shell
docker swam init
docker swarm leave --force      # Take down a single node swarm from the manager
```

### Docker Node

``` shell
docker node ls                # View nodes in swarm (while logged on to manager)
```



### Docker Service

``` shell
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
```

### Others

``` shell
docker inspect <task or container>                   # Inspect task or container
```

### Docker Compose

``` shell
docker-compose up -d								# 后台启动compose
docker-compose run web env							# 在web Service中执行一次env命令
docker-compose stop									# 停止compose
docker-compose down --volumes      					# 卸载挂上的卷 
```

####