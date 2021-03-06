Docker Tips
===

### 1. CLI

##### 1.1 美化 docker ps 的输出

将 Docker ps 的输出通过管道到 less -S，这样表格式的行就不会被折叠。
docker ps - a | less -S

##### 1.2 刷新日志

docker 的日志不会即时刷新，除非使用 -F 选项：
docker logs &lt;containerid> -F

##### 1.3 从 docker inspect 中获取一个单一的值

docker inspect 默认会输出大量的 JSON 格式的数据。可以用 jq，来得到某一特定键的值。或者你可以使用内置的 go 模板功能：
最后一个 docker 容器现在运转正常吗？

docker inspect --format '{{.State.Running}}' $(docker ps -lq)

##### 1.4 使用 docker exec 而不是 sshd 或者 nsenter

exec 在 1.3 版本中添加的新功能，可以在容器里面运行一个新的进程。这样就不必运行 sshd 或者在主机上安装 nscenter。

### 2. Dockerfiles

##### 2.1 docker build支持git仓库

你不但可以从本地的 Dockerfile 中创建 Docker 镜像，你还可以简单的给 docker build 指定一个仓库的 URL，然后 docker build 会为你做完余下的事情。

##### 2.2 没有软件包列表

默认的镜像（如 Ubuntu）是不包含软件包列表的，目的是让镜像体积更小。因此需要在任何的基础的 Dockerfile 中需要使用 apt-get update。

##### 2.3 留意软件包的版本

注意软件包的安装，因为这些命令也是会缓存起来的。意味着如果你清空了缓存，你可能会得到不同的版本；或者如果缓存长期不更新，你可能不会得到最新的安全更新。

##### 2.4 小体积的基础镜像

在 Docker Hub 上有一个官方的真正零体积的 Docker 镜像，它的名字叫做 scratch。所以如果你有这种需求，可以让你的镜像从零开始。而大多数的情况下，你最好还是从 busybox 开始，其大小只有 2.5M。

##### 2.5 FROM 默认会获取最新的

如果在 FROM 关键字后你没有指定一个版本的 tag，那么默认就会获取最新的。请注意这点，并确保尽可能的指定一个特定的版本。

##### 2.6 shell 或者是 exec 模式

在 Dockerfile 中可以通过两种方式来指定命令（如 CMD RUN 等）。如果你仅仅写下命令那么 Docker 会将其包裹在 sh -c 命令中执行。你也可以写成一个字符串数组的形式。数组的写法不需要依赖容器中的 shell，因为其会使用 go 的 exec。Docker 的开发者建议使用后一种方式。

##### 2.7 ADD vs COPY

ADD 和 COPY 都能在创建容器的时候添加本地的文件。但是ADD有一些额外的魔力，如添加远程的文件、unzip 或者 untar 一些文件包等。使用 ADD 之前请了解这种差别。

##### 2.8 WORKDIR 和 ENV

每个命令都会创建一个新的临时镜像并在新的 shell 中运行，所以如果你在 Dockerfile 中不能运行 cd <directory> 或者 export <var>=<value>。使用WORKDIR 在多个命令中设置工作目录并使用ENV来设置环境变量。

##### 2.9 CMD 和 ENTRYPOINT

CMD 是当一个镜像在运行时默认会执行的命令。默认的 ENTRYPOINT 是 /bin/sh -c，然后 CMD 会以参数的形式被传入。我们可以在 Dockerfile 中覆盖ENTRYPOINT以让我们的容器像在接受命令行参数（默认的参数在Dockerfile中的CMD指定）。
Dockerfile中

ENTRYPOINT /bin/ls
CMD ["-a"]
我们覆盖了命令行但是 netrypoint 仍然是 ls

docker run training/ls -l

##### 2.10 将ADD置于末尾

如果文件发生改变，ADD会让缓存失效。不要在 Dockerfile 中添加经常变化的东西，以避免让缓存失效。将你的代码放在最后，将库和依赖放在最前。对于 Node.js 的应用来说，这意味着将 package.json 放在前面，运行 nmp install 然后添加代码。

### 3. Docker的网络

Docker 有一个内置的 IP 池，用来指定容器的 ip 地址。它对外是不可见的，通过桥接的网口可以访问到。

##### 3.1 查找端口的映射

docker run 接收显式的端口映射作为参数，或者你可以通过 -P 选项来映射所有的端口。第二种做法的好处是能防止冲突。可以通过以下命令查找指定的端口：
docker port containerID portNumber

或者

docker inspect --format '{{.NetworkSettings.Ports}}'
containerID

##### 3.2 容器的IP地址

每一个容器都拥有自己属于私有网段的 IP 地址（默认是 172.17.0.0/16）。IP 可能会在重启的时候发生变化，如果你想知道其地址，可以用：
docker inspect --format '{{.NetworkSettings.IPAddress}}' containerID
Docker 会尝试检查冲突，在需要的情况下会使用不同的网段的地址。

##### 3.3 接管主机的网络

docker run --net=host 能重用网络。但是请不要这么做。

### 4. 卷（volume）

一个绕过目录或者单一文件写时复制（copy-on-write）的文件系统，接近零负载（绑定挂载）。

##### 4.1 卷的内容在 docker commit 的时候不会被保存

在镜像建立后写入你的卷没有太多的意义。

##### 4.2 卷默认是可读可写的

但是有一个 :ro 的标志。

##### 4.3 卷和容器是分开存在的

卷只要有一个容器使用他们就会存在。可以在容器之间通过 --volumes-from 选项共享。

##### 4.4 挂载你的 docker.sock

你可以仅仅挂载 docker.sock 就能让你的容器访问到 Docker 的 API。然后你可以在该容器中运行 Docker 的命令。这样容器甚至还可以杀死自己，在一个容器里面运行一个 Docker 的守护者进程是没有必要的。

### 5. 安全

##### 5.1 以 root 身份运行 Docker

Docker API 能给 root 的访问权限，因为你可以将/映射成一个卷，然后读或者写。或者你可以通过 --net host 接管宿主机的网络。不要暴露 Docker API 如果你需要请使用 TLS。

##### 5.2 Dockerfile中的USER

默认下 Docker 可以以 root 的身份运行任何命令，但是你可以使用 USER。Docker 没有用户的命名空间，因此容器将用户看作是宿主机上的用户。但是仅仅是 UID 因而你需要在容器里面添加该用户。

##### 5.3 使用TLS操作Docker API

Docker 1.3 版本添加了对 TLS 的支持。他们使用手动的验证机制：客户端和服务端都有一个 Key。把 Key 看做是 root 用户的密码。从 1.3 版本开始，Boot2docker 默认使用 TLS 并且会为你生成 key。

另外生成 Key 需要 OpenSSL 1.0.1 以上版本的支持，然后 Docker daemon 进程需要加上 --tls-verify 选项运行，Docker 会使用安全的端口（2376）。
