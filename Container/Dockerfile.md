# Dockerfile

[TOC]

## 一、概述

Docker的镜像是通过`Dockerfile`的指示一步步构建的。`Dockerfile`仅仅是一个包含了构建操作的纯文本。

## 二、使用

* 使用`docker build`从`Dockerfile`构建镜像。

  ``` txt
  # PATH是本地的路径
  docker build [PATH]  
  # URL是Git repository
  docker build [URL]
  # 构建完成后指定repository和tag
  docker build -t shykes/myapp .  
  # 构建完后指定多个repository和tag
  docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .   
  ```

* Build操作是通过Docker Daemon运行的，而不是CLI。步骤：

  * 寻找.dockerignore文件，排除该文件说明要忽略的文件

  * 发送Dockerfile所在目录完整的上下文到Daemon，即所有的文件

  * 对Dockerfile语法进行初步校验

  * 一步一步运行Dockerfile的操作，提交每一步操作的结果生产一个新的镜像。每一层镜像基于上一层镜像构建，直到最终输出完整镜像的ID

    * Docker会尽可能使用缓存的镜像来加速构建过程

      ``` shell
      $ docker build -t svendowideit/ambassador .
      Sending build context to Docker daemon 15.36 kB
      Step 1/4 : FROM alpine:3.2
       ---> 31f630c65071
      Step 2/4 : MAINTAINER SvenDowideit@home.org.au
       ---> Using cache
       ---> 2a1c91448f5f
      Step 3/4 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
       ---> Using cache
       ---> 21ed6e7fbb73
      Step 4/4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
       ---> Using cache
       ---> 7ea8aef582cc
      Successfully built 7ea8aef582cc
      ```

    * 构建过程中的缓存仅在镜像间有链式关系的场景下才会使用，就是说这些缓存镜像已经在之前构建的过程中创建出来了或者通过`docker load`加载了整个镜像的构建过程和内容

    * 可以使用`--cache-from`指定特定的镜像作为缓存使用，这特定的镜像则没有上面的束缚

  * Daemon自动清除相关的上下文

## 三、格式

``` txt
# Comment
INSTRUCTION arguments
```

* 大小写不敏感，但指令(INSTRUCTION)一般是大写的

* Docker按顺序执行指令

* 第一个操作指令必须是`FROM`，

  * `FROM`前面可以有`ARG`指令，用于定义变量

* `#`一般是注释，除非是解析器指令(Parser directives)

  * 行内的`#`都视为参数，不再当作注释

    ``` txt
    # Comment
    RUN echo 'we are running some # of cool things'
    ```

## 四、解析器指令(Parser directives)

* 概述：解析器指令是可选的，并会影响`Dockerfile`后续行的处理方式。
  * 构建中不会生成新的中间镜像层，也不会在构建步骤中展示
  * 格式：类似于注释`directive=value`。
    * 大小写不敏感，习惯用小写
    * 不支持换行，行内可以有空格
    * 解析器指令后面一般跟随空白行用于和其它指令区分
  * 一个指令只能用一次
  * 只要解析器指令出现在注释、空行或者构建指令的后面，Docker就不再处理它，把它当作是注释。
    * 所有的解析器指令必须在`Dockerfile`的顶部
    * 如果第一行解析器指令有误，不再视为解析器指令，后续所有的都会被当作注释

* 用途：

  * excape：Dockerfile的转义字符，可自定义

    * 格式：

      ```yaml
      # escape=\ (backslash)

      # 通常在windows下用`进行转义
      # escape=` (backtick) 
      ```

## 五、使用环境变量

* 格式：

  ``` txt
  $variable_name
  ${variable_name}
  ${foo}_bar

  # 如果变量已经设置就用变量的值，如果没设置就用word
  ${variable:-word}  
  # 如果变量已经设置就用word，如果没设置则为空字符串
  ${variable:+word}
  ```

* 可以使用环境变量的命令

  - `ADD`
  - `COPY`
  - `ENV`
  - `EXPOSE`
  - `FROM`
  - `LABEL`
  - `STOPSIGNAL`
  - `USER`
  - `VOLUME`
  - `WORKDIR`
  - ` ONBUILD`（需与上述命令结合使用）

* 环境变量的替换只会使用上一次命令定义的变量

  ``` txt
  ENV abc=hello
  ENV abc=bye def=$abc
  ENV ghi=$abc

  # 运行结果def是hello。而ghi才是bye
  ```

## 六、指令

### 1. FROM

* 初始化构建状态并给后续的指令提供基础镜像

* 格式：

  ``` txt
  FROM <image> [AS <name>]
  FROM <image>[:<tag>] [AS <name>]
  FROM <image>[@<digest>] [AS <name>]
  ```

* 多阶段构建：在`dockerfile`多次使用`FROM`来创建多个镜像或者基于先前镜像作为当前镜像的初始状态。

  * 每个`FROM`会清楚先前指令运行后的状态重新开始

  * 最后生成的镜像需要前面所有的`FROM`构建完成。（Simply make a note of the last image ID output by the commit before each new `FROM` instruction）

  * Example：

    ``` txt
    FROM muninn/glide:alpine AS build-env
    ADD . /go/src/app
    WORKDIR /go/src/app
    RUN glide install
    RUN go build -v -o /go/src/app/app-server

    FROM alpine
    RUN apk add -U tzdata
    RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
    # 从build-env复制文件
    COPY --from=build-env /go/src/app/app-server /usr/local/bin/app-server
    EXPOSE 80
    CMD ["app-server"]
    ```

    ``` yaml
    from debian as build-essential
    arg APT_MIRROR
    run apt-get update
    run apt-get install -y make gcc
    workdir /src

    # 基于第一个镜像
    from build-essential as foo
    copy src1 .
    run make

    # 基于第一个镜像
    from build-essential as bar
    copy src2 .
    run make

    # 分别从foo和bar镜像复制文件
    from alpine
    copy --from=foo bin1 .
    copy --from=bar bin2 .
    cmd ...
    ```

    ​

* tips:

  * `ARG`是唯一能出现在`FROM`面前的指令

    * `FROM`可以使用`ARG`定义的变量

      ``` txt
      ARG  CODE_VERSION=latest
      FROM base:${CODE_VERSION}
      CMD  /code/run-app

      FROM extras:${CODE_VERSION}
      CMD  /code/run-extras
      ```

    * `FROM`前`ARG`定义的变量在构建状态的作用域外，只可以在`FROM`使用。如果一定要使用，需要重新声明一次但不用赋值

      ``` txt
      ARG VERSION=latest
      FROM busybox:$VERSION
      ARG VERSION
      RUN echo $VERSION > image_version
      ```

  * 通过name给构建阶段命名，可选。这个名字可以在后续阶段中的`FROM`或`COPY --from=<name|index>`使用并指向这个名字所构建的镜像

  * tag和digest是可选的。如果忽略，构建器如果没找到tag值会报错，因此会默认tag为`latest`

### 2. RUN

* 在当前镜像的顶层执行命令并提交结果，该结果将会在Dockerfile的下一步继续使用。

  * 分层运行`RUN`指令并提交结果组成了Docker的核心概念：commits are cheap。容器可以在镜像历史的任意节点创建，很类似源代码管理。
  * 运行的缓存不会在下一次构建前自动失效，会继续使用。可以通过`docker build --no-cache`来设置缓存自动失效
    * 如果使用了`ADD`指令，缓存也会自动失效

* 格式：

  * shellform：`RUN <COMMAND>`

    * shell终端可通过命令行改了

    * 可使用`\`进行换行

      ``` txt
      RUN /bin/bash -c 'source $HOME/.bashrc; \
      echo $HOME'
      ```

    * 每个`RUN`只视为一行命令

      ``` txt
      RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
      ```

  * execform：`RUN ["executable", "param1", "param2"]`

    * 可以避免shell命令混杂一起
    * 不默认shell终端，需指定
    * 参数是一个JSON对象，意味所有字段必须用`""`
      * 某些场景需要转义`\`
      * ​

### 3. CMD

* **作为容器运行的默认操作**
  * 该默认操作包含一个可执行文件，也可以在有ENTRYPOINT忽略该可执行文件
  * 每个`Dockerfile`只能有一个`CMD`，如果有多个只有最后一个`CMD`有效


* 格式：
  * execform（推荐）：` CMD ["executable","param1","param2"]`
  * 作为ENTRYPOINT的默认参数：`CMD ["param1","param2"]`
    * 配合使用ENTRYPOINT，让容器每次都运行相同的可执行文件
    * 如果使用了该CMD模式，ENTRYPOINT也要使用JSON对象格式
  * shellform：`CMD command param1 param2`
    * 默认通过`/bin/sh -c`执行命令
* 与`RUN`的不同：`RUN`是运行在容器构建过程中的，运行命令并提交结果；`CMD`不在容器构建过程中运行，但指定了容器启动时的命令。

### 4. LABEL

* 给镜像加元数据/打标签

  * 镜像的标签继承了基础镜像和父级镜像的标签

  * 如果标签已存在，会被替换成新值

  * 使用`docker inspect`来查看镜像的标签

    ``` txt
    "Labels": {
        "com.example.vendor": "ACME Incorporated"
        "com.example.label-with-value": "foo",
        "version": "1.0",
        "description": "This text illustrates that label-values can span multiple lines.",
        "multi.label1": "value1",
        "multi.label2": "value2",
        "other": "value3"
    },
    ```

    ​

* 格式：` LABEL <key>=<value> <key>=<value> <key>=<value> ...`

  * 示例

    ``` txt
    # 使用""来包含空格
    LABEL "com.example.vendor"="ACME Incorporated"
    LABEL com.example.label-with-value="foo"
    LABEL version="1.0"
    # 使用\来换行
    LABEL description="This text illustrates \
    that label-values can span multiple lines."
    ```

  * 镜像可以有多个标签

    ```txt
    LABEL multi.label1="value1" multi.label2="value2" other="value3"
    ```


### 5. EXPOSE

* 通知Docker该容器在运行期间需要监听的端口，简单说指示容器暴露的端口

  * 暴露端口并不是发布端口。这是构建镜像和运行容器的区别，后者才需要指定发布的端口。

    * 通过`docker run -p HOST_PORT:CONTAINER_PORT`来发布端口
    * 使用`-P`参数来发布所有暴露的端口并与宿主高位端口绑定，所以端口可能不一样

  * 覆盖`EXPOSE`的设置：

    ``` txt
    docker run -p 80:80/tcp -p 80:80/udp ...
    ```

    ​

* 格式：` EXPOSE <port> [<port>/<protocol>...]`

  * 可指定监听的协议类型，默认是TCP协议

  * 同时暴露TCP和UDP，需要两条命令

    ``` txt
    EXPOSE 80/tcp
    EXPOSE 80/udp
    ```

* tips:

  * ` docker network`可以创建一个容器间互相交流的network而不需要暴露或发布端口，因为容器连接到network上可以通过任意端口互相交流

### 6.ENV

* 设置环境变量。该环境变量可以在构建的后续阶段使用。

  * 这些环境变量在容器运行时也一直保留。
  * 可使用` docker inspect`查看，或者使用`docker run --env <key>=<value>` 来改变

* 格式：

  * `ENV <key> <value>`：第一个空格后面的所有字段包含空格都视为value

    ``` txt
    ENV myName John Doe
    ENV myDog Rex The Dog
    ENV myCat fluffy
    ```

  * ` ENV <key>=<value> ...`

    ``` txt
    ENV myName="John Doe" myDog=Rex\ The\ Dog \
        myCat=fluffy
    ```

### 7.ADD

* 格式：

  * ``ADD [--chown=<user>:<group>] <src>... <dest>`

  * ` ADD [--chown=<user>:<group>]["<src>",... "<dest>"]`（包含空格的时候）

  * 规则：

    * <src>的路径必须在镜像构建的上下文中
    * 如果<src>是url而且<dest>不已` /`结尾，下载的文件将会以<dest>命名
    * 如果<src>是url而且<dest>以` /`结尾，下载的文件将从url的后缀推测命名
    * 如果<src>是一个文件夹，那么整个文件夹内的内容都会被复制，包含文件系统的元数据(filesystem metedata)
    * 如果<src>是一个压缩文件，它会被自动解压为文件夹后复制。对URL下载的压缩文件无效
    * 如果<src>是任意的一种文件，会包含它的元数据一起复制。在这场景下，如果<dest>以` /`结尾，复制的文件将以` <dest>/base(<src>)` 格式存储
    * 如果<src>包含多个资源，或者是一个目录，那么<dest>也必须是目录且以` /`结尾
    * 如果<dest>不以` /`结尾，会被视作常规文件而被复制替换
    * 如果<dest>不存在，将会自动创建

  * tips：

    * --chown只可以在Linux系统下使用

    * <src>可以包含通配符

      ```txt
      ADD hom* /mydir/        # adds all files starting with "hom"
      ADD hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"
      ```

    * <dest>是绝对路径。或者是相对于`WIRKDIR`的相对路径

      ```txt
      ADD test relativeDir/          # adds "test" to `WORKDIR`/relativeDir/
      ADD test /absoluteDir/         # adds "test" to /absoluteDir/
      ```

    * 如果<src>的文件或文件夹包含`[]`，需转义

      ``` txt
      ADD arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/
      ```

    * 加入的文件和文件夹默认的UID和GID是0，或者通过`--chown`指定，需保证这些需在容器中存在

      ``` txt
      ADD --chown=55:mygroup files* /somedir/
      ADD --chown=bin files* /somedir/
      ADD --chown=1 files* /somedir/
      ADD --chown=10:11 files* /somedir/
      ```

    * 如果<src>是URL，目标文件的权限将会是600。如果URL需要认证，或许需要通过`RUN WGET、RUN CURL`或其它工具

* 从<src>复制文件、文件夹或者URL的文件加入到在镜像文件系统中的<dest>

  * 可以指定多个src资源，但如果它们是文件或文件夹，它们的路径是相对于构建时的上下文的
  *  The first encountered `ADD` instruction will invalidate the cache for all following instructions from the Dockerfile if the contents of `<src>` have changed. This includes invalidating the cache for `RUN` instructions. See the [`Dockerfile` Best Practices guide](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#/build-cache) for more information.

### 8.COPY

* 格式：
  * `COPY [--chown=<user>:<group>] <src>... <dest>`
  * `COPY [--chown=<user>:<group>]["<src>",... "<dest>"]`（包含空格）
  * 可选参数：`--from=<name|index>`。用于设置先前的构建镜像中的源地址替代当前的上下文环境。
    * 如果先前构建的镜像不能被发现，将会使用同名的新镜像
* 从<src>复制文件、文件夹加入到在镜像文件系统中的<dest>
* 与`ADD`差异不大，细节可参考`ADD`命令。两者的区别：
  * `CMD`的<src>不能为URL
  * `CMD`不会自动解压<src>
  * `CMD`可指定` --from`参数用于从先前构建的镜像复制文件

### 9.ENTRYPOINT

* 允许把配置容器作为一个可执行文件

  * 仅仅最后一个`ENTRYPOINT`才有效

* 格式：

  * execform: `ENTRYPOINT ["executable", "param1", "param2"] `（推荐）
  * shellform: `ENTRYPOINT command param1 param2`
    * 会阻止`CMD`或者`RUN`添加参数，并且shell默认为不会传递任何信号的`/bin/sh -c`，意味着不能通过``docker stop <container>`接收SIGTERM信号。

* CMD与ENTRYPOINT的交互

  * 前提：

    * 至少要定义`CMD`和`ENTRYPOINT`的一个
    * 当容器是可执行时需要定义`ENTRYPOINT`
    * `CMD`需作为`ENTRYPOINT`的默认参数或者是容器的可执行命令
    * `CMD`可以被运行容器的命令覆盖参数

  * ``ENTRYPOINT` 和 `CMD`结合的交互影响：

    |                                | No ENTRYPOINT              | ENTRYPOINT exec_entry p1_entry | ENTRYPOINT [“exec_entry”, “p1_entry”]    |
    | ------------------------------ | -------------------------- | ------------------------------ | ---------------------------------------- |
    | **No CMD**                     | *error, not allowed*       | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry                      |
    | **CMD [“exec_cmd”, “p1_cmd”]** | exec_cmd p1_cmd            | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry exec_cmd p1_cmd      |
    | **CMD [“p1_cmd”, “p2_cmd”]**   | p1_cmd p2_cmd              | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry p1_cmd p2_cmd        |
    | **CMD exec_cmd p1_cmd**        | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd |

### 10.VOLUME

* 创建一个挂载点并声明该挂载点是来自外部宿主或者其他容器的，用于共享资源。

  * `docker run`会初始化新创建的挂载点并复制先前镜像中已经修改后存在的数据。(The `docker run` command initializes the newly created volume with any data that exists at the specified location within the base image. )

    ``` txt
    # 创建一个/myvol的挂载点并复制已有的greeting到挂载点中
    FROM ubuntu
    RUN mkdir /myvol
    RUN echo "hello world" > /myvol/greeting
    VOLUME /myvol

    ```

    ​

* 格式：`VOLUME ["/data"]`

  * 必须是JSON的数组对象
  * 可以有多个挂载点

* tips:

  * windows容器中的volumes：
    * 必须是不存在或者空的文件夹
    * 驱动大于C盘(a drive other than `C:`)
  * 在Dockerfile改变volume：
    * 在构建过程中，声明volume后的挂载点中数据的任意变化都会放弃。
  * 在docker运行的时候指定宿主目录：挂载点与主机目录的绑定在每个宿主上是不一样的。这保证了镜像的可移植性。因此不可以在dockerfile指定要绑定的宿主目录，需要在创建或运行容器的时候指定宿主目录

* Docker文件系统的工作模式：

  Docker镜像是由多个文件系统（只读层）叠加而成。当我们启动一个容器的时候，Docker会加载只读镜像层并在其上（译者注：镜像栈顶部）添加一个读写层。如果运行中的容器修改了现有的一个已经存在的文件，那该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是已经被读写层中该文件的副本所隐藏。当删除Docker容器，并通过该镜像重新启动时，之前的更改将会丢失。在Docker中，只读层及在顶部的读写层的组合被称为[Union File System](https://docs.docker.com/terms/layer/#union-file-system)（联合文件系统）。

### 11. USER

* 当运行镜像以及其它命令如`RUN`, `CMD` and `ENTRYPOINT`指定用户以及可选的群组。

  * 用户必须先创建

* 格式：

  ``` txt
  USER <user>[:<group>] or
  USER <UID>[:<GID>]
  ```

### 12.WORKDIR

* 给dockerfile的后续命令设置工作目录。如果该目录不存在，将会自动创建，哪怕后续不会用到。
  * 可以声明多次工作目录
  * 如果路径是相对路径，则基于先前的工作目录
  * 可以解析环境变量
* 格式：` WORKDIR /path/to/workdir`

### 13.ARG

* 定义一个变量，该变量在用户构建期间使用` docker build --build-arg <varname>=<value>`传递到构建器

  * 可以多次声明ARG
  * 不推荐使用ARG来传递任何私密的值。会被` docker history`看到。

* 格式：` ARG <name>[=<default value>]`

  * 可以指定默认值。如果构建过程中没有传入，则使用默认值
  * 作用域：从定义的那一行起开始生效，在构建结束阶段失效。
    * 作用域外的默认为空字符串
    * 如果有多个构建状态使用多个变量，需定义多次。

* 使用：

  * ` ARG`和` ENV`可以声明同一个变量，但环境变量总是覆盖传递给`ARG`变量的值。

    ``` txt
    1 FROM ubuntu
    2 ARG CONT_IMG_VER
    3 ENV CONT_IMG_VER v1.0.0
    4 RUN echo $CONT_IMG_VER

    $ docker build --build-arg CONT_IMG_VER=v2.0.1 .

    # 输出v1.0.0
    ```

  * 通过命令行传递变量并持久化到最终的镜像中。Example:

    ``` txt
    1 FROM ubuntu
    2 ARG CONT_IMG_VER
    3 ENV CONT_IMG_VER ${CONT_IMG_VER:-v1.0.0}
    4 RUN echo $CONT_IMG_VER
    ```

* 预定义的变量

  * Docker预先定义的变量，不需要再重复定义：
    - `HTTP_PROXY`
    - `http_proxy`
    - `HTTPS_PROXY`
    - `https_proxy`
    - `FTP_PROXY`
    - `ftp_proxy`
    - `NO_PROXY`
    - `no_proxy`
  * 预定义的变量可以通过`--build-arg <varname>=<value>`传递
  * 预定义的变量不会出现在` docker history`中，减少泄露信息的风险。如果需要出现，可以在` dockerfile`重复定义

* 变量对构建过程中缓存的影响

  * `ARG`不像`ENV`保存在构建的镜像中，但如果它的值发生变化与先前构建时传递的值不同，先前的镜像缓存就会失效。

### 14.ONBUILD

* 给镜像加一个触发器，该触发器会在该镜像被其它构建当作基础镜像时触发执行命令

  * 任何构建指令都可以注册成触发器
    * `ONBUILD、FROM、MAINTAINER`除外

* 格式：` ONBUILD [INSTRUCTION]`

  ```txt

  ```

* 工作过程：

  * 遇到` ONBUILD`时，构建器会把该trigger加入到镜像的元数据里面，不影响当前的构建
  * 当前镜像构建完成后，所有的触发器会被保存在镜像的manifest中的`OnBuild`key下面。可以通过`docker inspect`命令查看
  * 这个镜像被下一个构建使用`FROM`当作基础镜像时，在处理完`FROM`额的过程中，会寻找`ONBUILD`触发器，并按注册的顺序执行它们。如果其中一个执行失败，`FROM`会终止并导致整个构建失败。
  * 在触发器执行后，它们会从最终的镜像被删除。换句话说，触发器不会从爷孙关系继承。

* 使用场景：

  * 应用程序的构建环境或守护进程可能需要根据用户的配置来自定义

  * Example：

    * 假设一个镜像是可以重复使用的python应用程序构建器，它可能需要添加源码到特定目录中，并需要在后续指定一个可执行文件。但不可以在当前构建中添加源码，因为它们还不存在。
    * 当然也可以通过复制等操作来把源码复制到当前镜像，但是这样效率太低，也没法定制化。
    * 因此可以使用`ONBUILD`来满足该需求。ONBUILD执行的时候已经存在于下一个镜像的上下文中

    ``` txt
    [...]
    ONBUILD ADD . /app/src
    ONBUILD RUN /usr/local/bin/python-build --dir /app/src
    [...]
    ```

### 15.STOPSIGNAL

* 设置通知容器退出的信号
  * 该信号必须是内核系统调用的数字，如9，或者信号名如`SIGKILL`
* 格式：` STOPSIGNAL signal`

### 16.HEALTHCHECK

* 指示Docker测试容器去检查它是否仍在工作

  * 比如检查一个web是否处于死循环而无法处理新的连接，而他的服务进程仍在运行
  * 当容器有healthcheck时候，会有一个health status健康状态加入到normal status正常状态中。
    * 初始化状态是starting
    * 任何时候通过健康检查，变成healthy(无论先前的状态是什么)
    * 如果经过特定数量的连续失败后，变成unhealthy
  * 一个dockerfile只能存在一条命令，如多条则最后一条有效

* 格式：

  * `HEALTHCHECK [OPTIONS] CMD command `在容器内部通过运行命令检查容器健康状态
    * `--interval=DURATION` (default: `30s`)：检测周期。容器启动后第一次检测在一个周期后，之后每次都要再隔一个周期
    * `--timeout=DURATION` (default: `30s`)：检测的超时时间。超过的认作失败
    * `--start-period=DURATION` (default: `0s`)：容器启动时需要的初始化时间。此期间的任意失败都不会计算到最大的失败次数里面去；而如果状态是正常的，认作容器是启动的，并且所有的失败次数将被计算到最大的失败次数里面去。
    * `--retries=N` (default: `3`)：最大的失败次数。达到后认为该容器是unhealthy
  * ` HEALTHCHECK NONE`：关闭从上一个基础镜像继承而来的健康检测
  * tips：
    * CMD之后的参数可以是shell格式也可以是exec array格式

* 使用：

  * 检测命令的退出状态指示了容器的健康状态

    * 0：success
    * 1：unhealthy
    * 2：保留。不使用该状态码

  * Example：

    ``` txt
    # 每隔5分钟检测web的主页，超时时间为3s
    HEALTHCHECK --interval=5m --timeout=3s \
      CMD curl -f http://localhost/ || exit 1
    ```

  * 输出应该尽可能短。因为该命令的输出都会写在存储于健康状态的stdout和stderr中，可以通过`docker inspect`查看。

    * 只有前4096个字节会被存储

  * 当容器的健康状态改变的时候，新的状态会通过` health_status`事件生成

### 17.SHELL

* 覆盖使用shellform命令的默认shell终端

  * Linux默认：`["/bin/sh", "-c"]`
  * windows默认：`["cmd", "/S", "/C"]`

* 格式：`SHELL ["executable", "parameters"]`

  * 必须是JSON格式
  * 该命令可以出现多次，每次覆盖前面的shell终端并对后续命令起作用
  * 当使用`RUN`, `CMD`和 `ENTRYPOINT`使用shellform时，` shell`命令对他们也有效
  * 可以修改shell终端的操作行为：` SHELL cmd /S /C /V:ON|OFF`

* 使用场景：

  * 特别适合有两种终端的windows：`CMD`和`powershell`，或者可选的终端`sh`

    ``` txt
    FROM microsoft/windowsservercore

    # Executed as cmd /S /C echo default
    RUN echo default

    # Executed as cmd /S /C powershell -command Write-Host default
    RUN powershell -command Write-Host default

    # Executed as powershell -command Write-Host hello
    SHELL ["powershell", "-command"]
    RUN Write-Host hello

    # Executed as cmd /S /C echo hello
    SHELL ["cmd", "/S", "/C"]
    RUN echo hello
    ```

    ​

  * 也可以用在linux上选择终端，如zsh、cash

## 七、实践

### 1. 容器应该是短暂的

* 容器应该尽可能短暂。意味着，我们可以通过最小的组织和配置来停止、销毁、新建、或者原地更新容器。
* 可参考[12 Factor app](https://12factor.net/)

### 2.步骤

#### 1.1 构建上下文

* 使用` docker build`时，当前工作的目录称作build context(构建上下文)。
  * 默认dockerfile是存在于当前目录的，但可以通过`-f`参数指定路径
* 无论dockerfile存在哪，当前目录下的所有文件和文件夹都会传送进Docker Daemon当作构建上下文
* 上下文中包含不必要的文件会导致构建上下文和最终镜像过大，提高构建时间和容器运行大小等。

### # 1.2 使用.dockerigonre文件

* 排除不必要的文件

#### 1.3 使用multi-stage多阶段构建

* 在Docker17.05以上的版本，可是使用多阶段构建来彻底减少最终镜像的大小，而不需要跳过一些步骤来减少中间层的数量或者删除构建过程中的临时文件。

* 镜像只会根据最后一阶段建立，你可以在大幅时间受益于构建缓存并最小化镜像层数。

* 构建阶段可能包含很多层，从较低次数的变更到频繁的变更：

  * 搭建app需要安装的工具
  * 安装或更新依赖库
  * 生成应用

* Example：

  ```
  FROM golang:1.9.2-alpine3.6 AS build

  # Install tools required to build the project
  # We need to run `docker build --no-cache .` to update those dependencies
  RUN apk add --no-cache git
  RUN go get github.com/golang/dep/cmd/dep

  # Gopkg.toml and Gopkg.lock lists project dependencies
  # These layers are only re-built when Gopkg files are updated
  COPY Gopkg.lock Gopkg.toml /go/src/project/
  WORKDIR /go/src/project/
  # Install library dependencies
  RUN dep ensure -vendor-only

  # Copy all project and build it
  # This layer is rebuilt when ever a file has changed in the project directory
  COPY . /go/src/project/
  RUN go build -o /bin/project

  # This results in a single layer image
  FROM scratch
  COPY --from=build /bin/project /bin/project
  ENTRYPOINT ["/bin/project"]
  CMD ["--help"]
  ```

#### 1.4 避免安装不必要的packages

* 避免安装不必要的packages，减少复杂度、依赖、文件大小和构建时间

#### 1.5 每个容器只关心一件事

* 使用多个容器的解耦应用更容易水平扩展及复用容器。例如，一个web可以用三种容器组成：web、db、缓存
* 尽可能保证容器干净和模块化。在大多数情况下，一个容器一个进程可能是最佳的。
  * Apache和Celery可能使用多进程

#### 1.6 最小化镜像的层数

* 尽可能减小镜像的层数：
  * 只有`RUN`, `COPY`, and `ADD`可以创建镜像层。其它的命令只创建中间的临时镜像，也不在增加构建的大小
  * 使用多阶段构建去复制只需要的包到最终镜像。而这可以在中间过程安装工具和调试信息但不会给增加最终镜像的大小。

#### 1.7 使用多行参数

* 尽可能使用多行参数来减少后续的更改。避免安装多余的包并且更容易升级，也更容易阅读。

  ``` txt
  RUN apt-get update && apt-get install -y \
    bzr \
    cvs \
    git \
    mercurial \
    subversion
  ```

#### 1.8 构建缓存

* 构建镜像的过程中，每一个指令Dcoker都会在缓存中寻找已存在的镜像去重复使用。
  * 通过` --no-cache=true`避免使用缓存
* 为了让Dcoker使用缓存，明白缓存能做什么和不能做什么、怎么匹配镜像是非常重要的。Dcoker构建缓存的规则：
  * 从一个已经在缓存中的父级镜像开始，下一个指令将会比较从该父级镜像继承的所有孩子镜像是否使用同一条指令。如果不是，缓存失效。
  * 大部分情况下，只需要简单比较下dockerfile指令和子镜像其中一条指令就够了。但是，某些指令需要更复杂的检测和解释
  * ` ADD`和` COPY`命令，会去检测镜像中每个文件的内容并计算它们的校验和(checksum)。文件的最后修改和最后访问次数在它们的checksum中没有考虑。在缓存寻找中，这些checksum会和已经存在的镜像的checksum比较。如果有任意改变，比如内容和元数据，缓存就会失效。
  * 除了` ADD`和` COPY`的其它命令，缓存检查不会去对比容器中的文件去决定缓存是否匹配。只是对比命令的字符串是否匹配
    * 比如`RUN apt-get -y update`不会去对比容器中文件的内容
  * 只要一次缓存没有匹配上，dockerfile后续的命令都会建立新的镜像并不再使用缓存