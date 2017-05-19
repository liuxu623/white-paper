# 基于 Lain 集群创建一个全新的应用

> 本文会演示如何基于 LAIN 集群创建一个新应用，该应用是一个 HTTP 服务，当用户访问 `/` 时，返回 `Hello, world.`。

### 准备工作

- 首先需要部署一个本地集群，建议由两个节点组成。具体步骤见 [LAIN 的快速安装](install.html)
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../usermanual/install-lain-client.html)

### 有何不同

创建基于 LAIN 集群的应用，整体上的流程

- 规划/更新 lain.yaml 配置
- 编写业务代码
- 创建/更新持续集成任务
- 滚动部署到本地/开发/产品环境
- 回到 1 或者 2 

### 基础：lain.yaml

与创建一个运行于机器（物理机/虚拟机）的“野生”应用相比，基于 LAIN 集群的创建工作所增加的无非是一个应用内全局的配置文件，一般我们将它放置在应用的根目录下，称之为 lain.yaml.

lain.yaml 主要规定/描述了如下几件事情
- 为应用起一个好听的名儿
- 基础技术栈
- 构建过程
- 微服务拆分以及内部配置

以上几步描述了一个应用的配置，LAIN 集群根据此为应用调度资源，创建实例并运行。

为此，有必要了解一些 LAIN 的基本概念，和与之相对的 lain.yaml 中的配置

### LAIN 基本概念

- App，运行于 LAIN 集群中的一个应用，很显然对应现实中的一个应用，比如一个你自己的博客站点。一个 lain.yaml ，即是对一个 App 的描述。记得为应用起一个好听且 LAIN 集群内唯一的名字
- Proc，一个 LAIN App 包含若干自己的 Proc，对应于拆分出来的微服务。Proc 有自己 App 内唯一的名字，和类型。比如我们可能把博客站点，划分成一个前端 web 服务，和一个后端 db 服务。

App 和 Proc，分别对应于完整应用和微服务，就形成了 LAIN 上应用的骨架。


### 开始创建应用

回到开篇提到的 hello-world 应用，我们来一步一步完成它。

先给应用取个名字，在 LAIN 集群中，每个应用的名字是全局 unique 的。字符集要求是**小写字母以及数字，非连续 "-" 符号，且开头不能为数字**，
比如起名为 hello-world。

appname 一旦注册到 LAIN 集群，且这个集群开启了 auth，那么这个 appname 就会被你占据和管理，除非你把别人加入这个 app 的管理组，否则任何人不能控制该 app。

### 挑选技术栈

选用 golang 完成 hello-world，基础镜像可自行任意选取安装了 go 的基础镜像。

### 构建过程

golang 项目的构建采用 go 命令行工具，请参考相关文档。

### 规划应用服务

hello-world App 的功能很简单：监听 HTTP 请求，当用户访问 `/` 时，返回 "Hello, world."。
因此，只需要规划一个类型为 `web` 的 Proc 即可，起名为 `web`。`web` 类型的 Proc 可以理解为由
LAIN 集群守护的 daemon 程序，由集群运行指定的 cmd 命令，且保证实例始终存活；同时，LAIN
集群还会为含有 `web` 类型的 App 分配域名，比如此例中为 `hello-world.${LAIN-Domain}`。

> 当 proc.type 为 web 时，proc 的名字也必须为 web，即一个 app 只能有一个 web 类型的 proc，且其名字为 web。

至此，lain.yaml 有了大概的轮廓：

```
appname: hello-world

build:
    base: golang:1.8
    script:
        - go build -o hello-world  # 编译指令，类似于 Dockerfile 里的 RUN，WORKDIR 为 /lain/app

proc.web:
    type: web
    cmd: /lain/app/hello-world  # 因为 WORKDIR 为 /lain/app，所以编译好的程序在 /lain/app 目录下
    port: 8080  # hello-world 监听的端口
```

我们定义了一个 hello-world 应用，因为是 golang 技术栈，选用了 `golang:1.8` 作为基础镜像，并且确定了构建方法。

同时梳理了应用内唯一服务，即一个 `web` 类型的 Proc，该 Proc 实例由集群守护，始终执行着既定的任务；并且，LAIN
集群自动为该 App 分配了 `hello-world.${LAIN-Domain}` 的域名。

接下来需要做的事情就是 
- 完成应用业务代码
- 根据具体需求，完善配置细节
- 调试，ci，部署等

应用业务代码

业务需求主要包括
- 监听 HTTP 请求，当用户访问 `/` 时，返回 "Hello, world."

具体业务代码请参考 [laincloud/hello-world/main.go](https://github.com/laincloud/hello-world/tree/v0.0.1/main.go)。

### 完善 lain.yaml

#### 构建阶段

build 阶段除了常规的构建命令 script 之外，可以添加可选的 prepare 子阶段，用来缓存构建结果，供 lain debug/build 使用。

```
prepare:  # lain prepare 时，在 docker container 的 /lain/app/ 下执行下列命令，这一步的结果会默认缓存，以备 lain debug|build 使用，可用来安装稳定的系统依赖
    version: 20170516 # prepare 版本用 `build.prepare.version` 来定义，内容为字符串，base, script 或者 keep 发生变化时，请变更这个版本号，否则不用变动
    script: # 在 lain prepare 时于 /lain/app/ 路径下执行的顺序指令，注意这里类似 Dockerfile 的 RUN 语法，上一行的 `cd` 等操作不会对下面行的 `PWD` 产生影响
      - echo "prepare"
```

#### 规划编译产出

build 阶段完成构建，亦即编译工作。由于 build 阶段使用的镜像以及 build 过程中的操作，可能产生大量不需要带到生产环境的中间结果。

可以定义一个可选的 release 阶段，仅将需要的编译产出打包成干净的生产环境镜像。

例如

```
release: # 可以不写，如果不写，就把 build 阶段生成的 image 打上 release 的标签
    dest_base: centos:7   # release image 基于的 base image，如果不写，就把 script 执行完生成的中间结果 image 直接作为 release image
    copy:     # 指定把 script 运行结束后的中间结果 image 的路径 src 拷贝到 dest_base 中的 dest 中, src 可以是绝对路径或者是相对于 /lain/app 的路径，拷贝之前，dest image 中会自动创建 /lain/app 空目录
        - src: /lain/app/hello-world
          dest: /hello-world

# 如果不写src, dest，比如只写 - bin/*， 那么相当于把 src image 中的 bin/* 拷贝到 dest image 中的相同路径下
```

#### 最终的 lain.yaml

```
appname: anti-gfw
build:
    base: golang:1.8
    script:
        - go build -o hello-world
release:
  dest_base: centos:7
  copy:
    - src: /lain/app/hello-world
      dest: /hello-world
proc.web:
    type: web
    num_instances: 1
    cmd: /hello-world
```
 
> 最终的代码在[这里](https://github.com/laincloud/hello-world/tree/master)。

#### 本地调试 proc

在[安装 LAIN 客户端](../usermanual/install-lain-client.html)，之后，便可以使用 `lain` 命令了。

进入 hello-world 代码目录

- lain config show        # 查看当前配置
- lain dashboard local
- lain reposit local      # 往 local 环境注册本 app
- lain prepare
- lain build
- lain run gfw-watcher

> 因为 lain-cli 使用 git commit ID 标识应用版本，请先 `git init` 仓库，并在 `lain build` 之前执行 `git commit`。

顺利的话，即在本地启动了一个 Proc，docker ps 可以看到：

```
[vagrant@lain hello-world]$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                     NAMES
070820c9de56        hello-world:release   "/lain/app/hello-w..."   4 hours ago         Up 4 hours          0.0.0.0:32768->8080/tcp   hello-world.web.web
```

运行 lain debug web 可以以交互方式进入实例中进行 debug。

#### 部署应用

> 上述 lain build 成功之后

- lain tag local  # 为应用打标签，类似于 docker tag，作用为标识应用版本
- lain push local  # 将镜像推送到 registry.${LAIN-domain}，类似于 docker push
- lain deploy local  # 部署应用

> `lain tag local` 的进一步说明：
>  - 作用是将 lain build 阶段生成的 hello-world:build 和 hello-world:meta 镜像分别打标签为
>    registry.${LAIN-domain}/hello-world:build-${timestamp}-${commit-id} 和
>    registry.${LAIN-domain}/hello-world:meta-${timestamp}-${commit-id}，主要是为了
>    标识版本
>  - hello-world:build 镜像包含了编译成果，比如这里的 /lain/app/hello-world
>  - hello-world:meta 镜像包含了 lain.yaml，供 LAIN 集群调度时解析，用户不需要关心
>  - `local` 为 LAIN 集群的名字，通过 `lain config` 设置，如[安装 LAIN 客户端](../usermanual/install-lain-client.html)中所示

部署的过程是一个异步的过程，在 `lain deploy local` 之后可以使用 `lain ps local` 查询部署结果。

#### 访问 hello-world 服务

为了解析 `hello-world.lain.local`, 如果是以 vip 方式启动的集群，请执行：

```
echo "192.168.77.201 hello-world.lain.local" >> /etc/hosts
```

否则，请执行：

```
echo "192.168.77.21 hello-world.lain.local" >> /etc/hosts
```

然后就可以访问 `hello-world` 服务了：

```
curl http://hello-world.lain.local
# 返回 Hello, world.
```

我们还可以方便地扩容：

```
lain scale -n 2 local web  # 扩容
lain ps local  # 观察结果
```
