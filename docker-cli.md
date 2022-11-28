# **Docker 命令行的基本用法**

## **目录**

- [**1 拉取镜像**](#1-拉取镜像)
- [**2 运行容器**](#2-运行容器)
- [**3 在已启动的容器中启动新程序**](#3-在已启动的容器中启动新程序)
- [**4 制作镜像**](#4-制作镜像)
    - [**4.1 命令行**](#41-命令行)
    - [**4.2 编写 Dockerfile**](#42-编写-dockerfile)
    - [**4.3 示例：制作一个简单的镜像**](#43-示例制作一个简单的镜像)

## **1 拉取镜像**

镜像为容器提供了一个隔离的文件系统，它包含运行程序所需要的全部文件（可执行文件、配置、脚本、动态库等），同时也包含运行环境的配置（环境变量、初始工作路径、默认的启动命令等）

```bash
docker pull <image>:<tag>
```

- `image` 为镜像名称

- `tag` 为镜像标签，通常以版本号作为标签

    ```bash
    docker pull ubuntu:latest # 拉取最新的 ubuntu 镜像
    docker pull ubuntu:22.04  # 拉取 22.04 版本的 ubuntu 镜像
    ```

    > PS：通过 `docker image ls` 可查看拉取的镜

## **2 运行容器**

`docker run` 命令将创建一个容器，容器的启动命令是将成为主进程，主进程结束后，容器的生命周期即结束，容器中其它进程也将被终止。

```bash
docker run [options] <image> [cmd]
```

`docker run` 有众多的 options，这里仅介绍一些重要的，完整的请参考 [Docker Reference - docker run](https://docs.docker.com/engine/reference/commandline/run/)

- `--interactive , -i`

    始终保持 `STDIN` 打开。对于用交互式程序（如 `bash`）作为主程序的容器，关闭 `STDIN` 即退出程序，容器出也随即退出，在这种情况下需要使用该参数。

    ```bash
    # 以交互式方式启动容器，ubuntu 的默认启动命令为 /bin/bash
    # 输入 exit 或者按 CTRL + D 可退出 bash 程序，从而退出容器
    docker run -i ubuntu:latest
    ```

- `--tty , -t`

    为容器分配伪终端。对于需要交互式终端的程序（如 `bash`），在容器中为其分配伪终端后，其行为才能像正常的 `shell` 一样与用户进行交互。

    ```bash
    # 与仅使用 -i 参数启动相比较，使用 -t 后有完整的交互式终端
    docker run -it ubuntu:latest
    ```

- `--name`

    设置容器的名称。在未指定该参数时，将使用随机名称对容器进行命名。

    ```bash
    # 创建一个命名为 test-name 的容器
    docker run -it --name test-name ubuntu:latest
    ```

- `--detach , -d`

    在后台运行容器。容器启动后会当前的命令行的终端，通过该参数可以使容器启动后自动切换到后台运行。

    ```bash
    # 在后台运行 ubuntu 容器，并且会输出容器的ID
    docker run -itd --name test-detach ubuntu:latest

    # attach 到创建的容器
    docker attach test-detach
    ```

    > PS：启动容器时未指定`-d`参数，但指定了`-it`参数时，可以通过快捷键 `CTRL + P + Q` 将占用当前终端的容器切换到后台运行。

- `--rm`

    容器退出时自动删除。

    ```bash
    # 创建容器：睡眠 3 秒，退出后自动删除
    docker run --name test-rm ubuntu:latest sleep 3
    ```

    > PS：可使用 `docker ps -a` 列出当前创建的容器与状态

- `--volume , -v`

    挂载一个卷到容器中。卷可以是主机文件系统上的路径，也可以是通过 `docker volume` 创建的命名卷。


    ```bash
    # 创建容器：将主机路径 /host/path 挂载到容器路径 /container/path，并在容器中执行 ls 命令
    docker run --rm -v /host/path:/container/path ubuntu:latest ls /container/path
    ```
    
    - 示例：挂载 volume 到容器中
    
    ```bash
    # 创建 volume
    docker volume create my-volume
    
    # 创建容器：将 my-volume 挂载到容器路径 /container/path，并在容器中执行 ls 命令
    docker volume --rm -v my-volume:/container/path ubuntu:latest ls /container/path
    ```
    
    > PS：volume 可用于存储数据、配置等，当升级镜像版本后重新挂载 volume 从而达到保持配置和数据不变的效果

- `--env , -e`

    设置容器的环境变量

    ```bash
    # 创建容器：为容器的 PATH 环境变量追加 /tmp 目录
    docker run --rm -e "PATH=$PATH:/tmp" ubuntu:latest bash -c 'echo $PATH'
    ```

- `--workdir , -w`

    设置容器的工作目录

    ```bash
    # 创建容器：设置容器的工具目录为 /tmp
    docker run --rm -w /tmp ubuntu:latest pwd
    ```

- `--restart`

    设置容器的重启策略

    |策略|效果|
    |-|-|
    |**no**|不自动重启，这是默认值|
    |**on-failure[:max_retries]**|当容器退出状态不是0时自动重启。可选择设置重启尝试的最大次数|
    |**always**|总是自动重启，不管当前容器是何退出状态|
    |**unless-stopped**|除非该容器被停止，否则总是自动重启，不管当前容器是何退出状态|

    ```bash
    # 创建容器：睡眠 1 秒后退出，并不断重启
    docker --restart always ubuntu:latest sleep 1
    ```

- `--cpus`

    设置容器使用的 CPU 数量，可以带小数

    ```bash
    # 创建容器：循环打印 pwd，CPU 使用量限制为 50%，容器命名为 test
    docker --rm -d --name test --cpus 0.5 ubuntu:latest bash -c 'while :; do pwd>/dev/null; done'

    # 查看容器中进程的 CPU 的使用量
    docker exec -it test top

    # 停止容器
    docker stop test
    ```

- `--cpuset-cpus`

    设置容器允许使用的 CPU 核心。

    ```bash
    # 创建容器：绑定到 1-4,8-12 核上运行
    docker --rm -it --cpuset-cpus 1-4,8-12 ubuntu:latest
    ```

- `--cpuset-mems`

    设置容器鸡毛使用 NUMA 结点的内存

    ```bash
    # 创建容器：限定使用使用 NUMA 0 的内存
    docker --rm -it --cpuset-mems 0 ubuntu:latest
    ```

- `--memory , -m`

    设置允许使用的内存大小

    ```bash
    # 创建容器：限定使用使用的内存大小为 1GB
    docker --rm -it -m 1G ubuntu:latest
    ```

- `--publish , -p`

    发布容器的端口，即端口转发。容器中的网络默认使用桥接网络，需要经过 NAT，其它主机想要访问容器中的服务需要通过端口转发来完成。

    ```bash
    # 创建容器：将主机 5566 端口的转发到容器的 7788 端口，UDP、TCP 端口都作转发
    docker run -it --rm -p 5566:7788/tcp -p 5566:7788/udp ubuntu:latest
    ```

- `--shm-size`

    设置容器的`/dev/shm`（共享内存）大小。默认值比较小，需要使用大量共享内存的情况下应该设置该值。

    ```bash
    # 创建容器：设置共享内存大小为1GB
    docker run -it --rm --shm-size 1G ubuntu:latest
    ```

## **3 在已启动的容器中启动新程序**

容器在启动后可能通过 `docker exec` 来增加新的进程，通常该方法可用于容器内的维护。

```bash
docker exec [options] [continaer-name|container-id] <cmd> [arg ...]
```

`docker exec` 的详细参数可参考 [Docker Reference - docker exec](https://docs.docker.com/engine/reference/commandline/exec/)

```bash
# 创建容器：用长时间睡眠来模拟业务
docker run -d --name test-exec ubuntu:latest sleep 9999

# 在容器中执行一个新 bash
docker exec -it test-exec /bin/bash
```

## **4 制作镜像**

### **4.1 命令行**
`docker build` 命令可用于制作镜像，镜像格式是符合开放容器标准（OCI）的，也可应用于其它支持 OCI 的容器平台。

```bash
docker build [options] <path>
```

`docker build` 的 options 中比较重要在此外列出，详细参数可参考 [Docker Reference - docker build](https://docs.docker.com/engine/reference/commandline/build/)

- `--tag , -t`

    该参数不仅指定标签，同时指定镜像名称，格式为 `name:tag`

    ```bash
    # 使用当前路径下默认的 ./Dockerfile 来构造镜像，并指定镜像名为 my-image，标签为 1.0
    docker build -t my-image:1.0
    ```

- `--file , -f`

    指定用于构建镜像的 Dockerfile 路径，默认路径为 <path>/Dockerfile

    ```bash
    # 使用 ./my-dockerfile 来构建镜像
    docker build -f my-dockerfile -t my-image:2.0 .
    ```

### **4.2 编写 Dockerfile**

Docker 通过读取 Dockerfile 中的指令来构造镜像。Dockerfile 中的指令是大小写不敏感的，通常用大写来表示命令，小写表示参数。Dockerfile 中的指令是按照先后顺序来执行的。除了用于 `FROM` 指令参数的声明外，通常Dockerfile 必须以 `FROM` 指令开头。Dockerfile 中的注释是以 `#` 开始。

Dockerfile 的详细语法可参考 [Docker Reference - Dockerfile](https://docs.docker.com/engine/reference/builder/)

- `FROM`

    ```dockerfile
    FROM [--platform=<platform>] <image> [AS <name>]
    FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
    FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
    ```

    `FROM` 指令以 Base Image 为参数，image 必须为一个本地有效的镜像或者可以从公共镜像仓库拉取的镜像。

    * `ARG` 指令是唯一可以出现在 `FROM` 之前的指令
    * `FROM` 可以在 Dockerfile 中多次出现，用于构造多个镜像或者作为其它镜像的依赖。一行新的 `FROM` 指令将提交前面的镜，并清理掉之前指令的状态。
    * `AS name` 可以代指 `FROM` 所构建的镜像名（真实的镜像名是在 `docker build` 的命令行参数中设置的），在后面的 `FROM` 和 `COPY --from=<name>` 指令中可以用 `name` 来表示之前构造的镜像
    * `tag` 和 `digest` 可以省略，在省略的情况下将使用默认值 `latest` 标签。
    * `--platform` 可用于设置镜像的平台，例如 `linux/amd64`，`linux/arm64` 或者 `windows/amd64`。默认情况下，使用的时构建镜像的平台。
    * `image` 值为 `scratch` 时，表示没有 Base Image

    ```dockerfile
    ARG  CODE_VERSION=latest
    FROM base:${CODE_VERSION}
    CMD  /code/run-app
    
    FROM extras:${CODE_VERSION}
    CMD  /code/run-extras
    ```

- `RUN`

    ```dockerfile
    # shell 格式
    RUN <command>

    # exec 格式
    RUN ["executable", "param1", "param2"]
    ```

    `RUN` 指令在当前构建的镜像上运行命令，将产生的结果也将被提交到新镜像中。在 shell 格式的命令中可以使用 `\` 连接多行命令。在 exec 格式的命令中可名句转义

    ```dockerfile
    RUN /bin/bash -c 'source $HOME/.bashrc; \
        echo $HOME'

    RUN ["/bin/bash", "-c", "echo hello"]
    ```

- `CMD`

    ```dockerfile
    # exec 格式
    CMD ["executable","param1","param2"]

    # 作为 ENTRYPOINT 的默认参数
    CMD ["param1","param2"]

    # shell 格式
    CMD command param1 param2
    ```

    在 Dockerfile 中只能有一个 CMD，如果有多个，则仅最后一个起作用。`CMD` 的作为是为容器提供一个默认的执行命令。可以在 `CMD` 中指定可执行文件，也可以在指定 `ENTRYPOINT` 后作为参数。

    ```dockerfile
    FROM ubuntu

    # 在 shell 格式中实际执行的是在 `/bin/sh -c` 的命令中
    CMD echo "This is a test." | wc -
    ```

    ```dockerfile
    FROM ubuntu

    # 如果不想再套一层 shell，则可以使用 exec 格式
    CMD ["/usr/bin/wc","--help"]
    ```

- `EXPOSE`

    ```dockerfile
    EXPOSE <port> [<port>/<protocol>...]
    ```

    `EXPOSE` 指令用于通知 Docker 容器在运行过程中将监听什么端口。该命令并不会增加端口转发规则，它只是一个提示，告诉外部，容器内部监听了哪些端口，当然也可不告诉外部端口监听的情况。通过 `-P` （大写P）参数可将 `EXPOSE` 端口映射到主机高位的一个随机端口。

    ```dockerfile
    EXPOSE 80/tcp
    EXPOSE 80/udp
    ```

- `ENV`

    ```dockerfile
    ENV <key>=<value> ...
    ```

    `ENV` 指令用于设置环境变量的值。

    ```dockerfile
    ENV MY_NAME="John Doe"
    ENV MY_DOG=Rex\ The\ Dog
    ENV MY_CAT=fluffy
    ```

    Dockerfile 中的环境变量会保留到容器启动。若要环境变量仅在 `docker build` 阶段启作用，可以使用 `ARG` 指令

    ```dockerfile
    ARG DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y ...
    ```

    或者在 `RUN` 指令中使用一次性的环境变量

    ```dockerfile
    RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y ...
    ```

- `ADD`

    ```dockerfile
    ADD [--chown=<user>:<group>] [--checksum=<checksum>] <src>... <dest>
    ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
    ```

    `ADD` 指令从 `<src>` 拷贝文件、目录或者远程文件 URL 到镜像的文件系统路径 `<dest>` 目录下，`<dest>` 是相对于 `WORKDIR` 的路径。`ADD` 指令遵循以下几个原则：

    * `<src>` 路径必须在构造的上下文中（即 `docker build` 指定的目录之内），无法使用 `../something` 或者 `/something`
    * 如果 `<src>` 为 URL，且 `<dest>` 不以 `/` 结尾，则文件下载保存为 `<dest>`
    * 如果 `<src>` 为 URL，且 `<dest>` 以 `/` 结尾，则文件下载保存为 `<dest>/<filename>`
    * 如果 `<src>` 为目录，则整个目录以及文件系统元数据都将被复制
    * 如果 `<src>` 为本地 `tar` 压缩文档（支持 `gzip`、`bzip2`、`xz`），则 `<src>` 被解压到目录中。来自远程 URL 的文件不会被解压
    * 如果 `<src>` 为文件，则 `<src>` 的内容及元数据将被复制。如果 `<dest>` 不以 `/` 结尾，则 `<src>` 被复制到 `<dest>` 目录下
    * 如果有多个 `<src>`，则 `<dest>` 必须以 `/` 结尾
    * 如果 `<dest>` 不以 `/` 结尾，则它被认为是文件，`<src>` 的内容被复制到 `<dest>`
    * 如果 `<dest>` 不存在，则会自动创建

    ```dockerfile
    # 复制 hom 开头的文件到镜像的 /mydir/ 路径
    ADD hom* /mydir/

    # 使用 ? 匹配任务字符
    ADD hom?.txt /mydir/

    # 复制 test.txt 到 <WORKDIR>/relativeDir/ 路径下
    ADD test.txt relativeDir/

    # 使用 Go 语言的转义：复制 arr[0].txt 到 /mydir/
    ADD arr[[]0].txt /mydir/

    # 复制文件，并设置文件所有者
    ADD --chown=55:mygroup files* /somedir/
    ADD --chown=bin files* /somedir/
    ADD --chown=1 files* /somedir/
    ADD --chown=10:11 files* /somedir/
    ```

- `COPY`

    ```dockerfile
    COPY [--chown=<user>:<group>] <src>... <dest>
    COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
    ```

    `COPY` 用于复制文件，与 `ADD` 功能相近，`COPY`指令遵循以下几个原则：

    * `<src>` 路径必须在构造的上下文中（即 `docker build` 指定的目录之内），无法使用 `../something` 或者 `/something`
    * 如果 `<src>` 为目录，则整个目录中的内容及文件系统元数据都将被复制，但不复制 `<src>` 本身
    * 如果 `<src>` 为文件，且 `<dest>` 以 `/` 结尾，则 `<src>` 被复制么 `<dest>` 目录下
    * 如果有多个 `<src>`，则 `<dest>` 必须以 `/` 结尾
    * 如果 `<dest>` 不存在，则会自动创建

- `ENTRYPOINT`

    ```dockerfile
    # shell 格式
    ENTRYPOINT command param1 param2

    # exec 格式
    ENTRYPOINT ["executable", "param1", "param2"]
    ```

    `ENTRYPOINT` 为容器启动时运行的命令，通过 `docker run` 设置的 `cmdline` 将被以 exec 格式追加到 `ENTRYPOINT` 后面，并且为覆盖 `CMD`。通过 `docker run --entrypoint` 可以覆盖 `ENTRYPOINT`。

    shell 格式的 `ENTRYPOINT` 将会禁止 `CMD` 或者 `docker run` 的 `cmdline` 参数。而且程序是以 `sh -c` 的形式启动，且不传递 Linux 信号，程序也不会获取 `PID 1` 这个进程号，因此接收不到 `docker stop` 发送的 `SIGTERM` 信号。（`docker stop` 会在等一段时间，最后将以 `SIGKILL` 强制结束容器中的进程）

- `WORKDIR`

    ```dockerfile
    WORKDIR /path/to/workdir
    ```

    设置容器的工作目录

- `VOLUME`

    ```dockerfile
    VOLUME ["/data"]
    VOLUME /data
    VOLUME /data1 /data2
    ```

    `VOLUME` 指令创建一个挂载点，并标记为使用了外部的挂载。`docker run` 在没使用使用 `-v` 参数时将自动创建匿名 `volume`（容器被删除时，关联的匿名 `volume` 将自动删除）。也可使用 `-v` 挂载外部目录或者已存在的 `volume`

### **4.3 示例：制作一个简单的镜像**

以制作一个简单的 `bash` 程序 docker 镜像为例，该示例不使用基础镜像，展示如何从零开始制作一个镜像：

* 创建工作目录及子目录

    ```bash
    mkdir my-docker-image
    cd my-docker-image
    ```

* 复制 `bash` 程序到 `my-docker-image/bin`

    ```bash
    mkdir bin
    cp /bin/bash bin/bash
    ```

* 查看 `bash` 的依赖库

    ```bash
    ldd bin/bash
    ```

   得到的结果如下：

    ```text
    linux-vdso.so.1 (0x00007ffde7dd2000)
    libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007f07fd4cf000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f07fd2a7000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f07fd66f000)
    ```

* 将依赖库复制到对应目下

    ```bash
    mkdir -p lib64
    mkdir -p lib/x86_64-linux-gnu
    cp /lib/x86_64-linux-gnu/libtinfo.so.6 lib
    cp /lib/x86_64-linux-gnu/libc.so.6 lib
    cp /lib64/ld-linux-x86-64.so.2 lib64
    ```

    > PS：linux-vdso.so.1 为 Linux 内核中的虚拟库，不需要复制

* 编写 Dockerfile

    ```dockerfile
    # 指明不使用基础镜像
    FROM scratch
    
    # 复制
    # 不能使用 COPY bin lib lib64 /
    # COPY 的 <src> 如果是目录，则复制 <src> 目录下的内容，不包含 <src> 自身
    
    COPY bin bin
    COPY lib lib
    COPY lib64 lib6
    
    # 这里要使用 exec 格式，shell 格式是在 /bin/sh -c 环境下执行，本次制作的镜像里没用 /bin/sh 文件
    CMD  [ "/bin/bash" ]
    ```

* 创建镜像

    ```bash
    # 创建镜像
    docker build -t my-image:1.0 .

    # 查看镜像
    docker image ls
    ```

* 运行容器

    ```bash
    docker run -it --rm my-image:1.0
    ```

* 在容器中运行命令

    由于镜像只打包了一个 bash 程序，大部分 Linux 常见命令无法支持，只能使用 bash 内建的命令，用于演示已经足够

    ```bash
    # 查看当前工作目录
    pwd
    
    # 列出 / 目录下的文件
    echo /*
    
    # 退出容器
    exit
    ```