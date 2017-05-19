# 第一个 LAIN 应用

> 本文会演示如何基于 LAIN 集群创建一个 LAIN 应用，它提供 HTTP 服务，当用户访问 `/` 时，返回 `Hello, LAIN.`。

## 准备工作

- 首先需要一个 LAIN 集群。如果没有的话可以参考[启动 LAIN 集群](../install/cluster.html)启动一个本地集群，建议由 2 个节点组成
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../install/lain-client.html)

## 业务代码

```
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Hello, LAIN.")
	})

	http.ListenAndServe(":8080", nil)
}
```

代码的逻辑为：
- 监听 `0.0.0.0:8080` 端口
- 收到 `/` 的 HTTP 请求时，返回 `Hello, LAIN.`

## lain.yaml

`lain.yaml` 是 LAIN 应用的配置文件，如下例所示：

```
appname: hello-world  # 应用名，在集群内唯一，由小写字母、数字和 `-` 组成，且开头不能为数字，不能有连续的 `-`

build:  # 描述如何构建 hello-world:build 镜像
  base: golang:1.8  # 基础镜像，类似于 Dockerfile 里的 FROM
  script:
    - go build -o hello-world  # 编译指令，类似于 Dockerfile 里的 RUN，WORKDIR 为 /lain/app

proc.web:  # 定义一个 proc，名字为 web
  type: web  # proc 类型为 web
  cmd: /lain/app/hello-world  # 因为 WORKDIR 为 /lain/app，所以编译好的程序在 /lain/app 目录下
  port: 8080  # hello-world 监听的端口
```

因为我们需要提供 HTTP 服务，所以定义一个 `web` 类型的 proc，LAIN 集群会为 web 类型的 proc 自动分配 ${appname}.${LAIN-domain} 的域名。

> proc.type 为 `web` 时，其名字也必须为 web，即一个 app 只能有一个 web 类型的 proc，且其名字为 web。

[laincloud/hello-world@v1.0.0](https://github.com/laincloud/hello-world/tree/v1.0.0) 的完整代码在这里。

## 本地运行

```
[vagrant@lain ~]$ cd ${hello-world-project}  # 进入工程目录

[vagrant@lain hello-world]$ lain build  # 构建 hello-world:build 镜像，生成编译结果
>>> Building meta and release images ...
>>> found shared prepare image at remote and local, sync ...
>>> generating dockerfile to /Users/bibaijin/Projects/go/src/github.com/laincloud/hello-world/Dockerfile
>>> building image hello-world:build ...
Sending build context to Docker daemon 6.656 kB
Step 1/4 : FROM registry.lain.local/hello-world:prepare-0-1494908044
 ---> 7406706a7f21
Step 2/4 : COPY . /lain/app/
 ---> 404266ec2a0d
Removing intermediate container c506a14acde2
Step 3/4 : WORKDIR /lain/app/
 ---> 5e850af78540
Removing intermediate container 7dafe4bd9c3a
Step 4/4 : RUN ( go build -o hello-world )
 ---> Running in 00e857f8b812
 ---> 5c6985b36760
Removing intermediate container 00e857f8b812
Successfully built 5c6985b36760
>>> build succeeded: hello-world:build
>>> tag hello-world:build as hello-world:release
>>> generating dockerfile to /Users/bibaijin/Projects/go/src/github.com/laincloud/hello-world/Dockerfile
>>> building image hello-world:meta ...
Sending build context to Docker daemon 6.656 kB
Step 1/2 : FROM scratch
 --->
Step 2/2 : COPY lain.yaml /lain.yaml
 ---> b43de7c3cbcd
Removing intermediate container 15eb844d61eb
Successfully built b43de7c3cbcd
>>> build succeeded: hello-world:meta
>>> Done lain build.

[vagrant@lain hello-world]$ lain run web  # 在本地运行
>>> run proc hello-world.web.web with image hello-world:release
docker: Error response from daemon: Conflict. The container name "/hello-world.web.web" is already in use by container c1fe8419c7846628021a55c792dfea51b17ecbc4a5e72777dc752f2ac614a43d. You have to remove (or rename) that container to be able to reuse that name..
See 'docker run --help'.
>>> container name: hello-world.web.web
>>> port mapping:
>>> 8080/tcp -> 0.0.0.0:32768
```

> lain-cli 的所有命令均需要在包含 lain.yaml 文件的目录下运行。

`docker ps` 时，可以看到：

```
CONTAINER ID        IMAGE                 COMMAND                  CREATED              STATUS              PORTS                     NAMES
c1fe8419c784        hello-world:release   "/lain/app/hello-w..."   About a minute ago   Up About a minute   0.0.0.0:32768->8080/tcp   hello-world.web.web
```

上面的输出表示 lain 通过 docker 把 hello-world.web.web 容器里的 8080 端口映射到了主机的 32768，所以，可以在主机上访问：

```
[vagrant@lain hello-world]$ curl http://localhost:32768
Hello, LAIN.
```

得到了预期的结果。

## 部署到 LAIN 集群

从上一小节可以看到，本地运行没有问题，现在可以部署到 LAIN 集群了：

```
[vagrant@lain ~]$ cd ${hello-world-project}  # 进入工程目录

[vagrant@lain hello-world]$ lain build  # 构建 hello-world:build 镜像，生成编译结果

[vagrant@lain hello-world]$ lain tag local  # 类似于 docker tag，为 hello-world:build 和 hello-world:meta 镜像打上版本标签
>>> Taging meta and relese image ...
>>> tag hello-world:meta as registry.lain.local/hello-world:meta-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7
>>> tag hello-world:release as registry.lain.local/hello-world:release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7
>>> Done lain tag.

[vagrant@lain hello-world]$ lain push local  # 类似于 docker push，将镜像推送到 LAIN 集群
>>> Pushing meta and release images ...
>>> pushing image registry.lain.local/hello-world:meta-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 ...
The push refers to a repository [registry.lain.local/hello-world]
905261a88a60: Layer already exists
meta-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7: digest: sha256:b374000a6bbabb65a80b989856cbbed752987a7b0f1fd69239b77f20c2cd6406 size: 524
>>> pushing image registry.lain.local/hello-world:release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 ...
The push refers to a repository [registry.lain.local/hello-world]
f475d32089ec: Layer already exists
9db04493db70: Layer already exists
edac683c8e67: Layer already exists
0372f18510d4: Layer already exists
c0b53d6ac422: Layer already exists
bcf20a0a17f3: Layer already exists
9d039e60afe3: Layer already exists
a172d29265f3: Layer already exists
e6562eb04a92: Layer already exists
596280599f68: Layer already exists
5d6cbe0dbcf9: Layer already exists
release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7: digest: sha256:d2c009a5ca5a553719149b42a799113fb1d166ac248597f293314fab48563d8e size: 2626
>>> Done lain push.

[vagrant@lain hello-world]$ lain deploy local  # 将应用部署到 LAIN 集群
>>> Begin deploy app hello-world to dev ...
upgrading... Done.
>>> app hello-world deploy operation:
>>>     last version: 1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7
>>>     this version: 1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7
>>> if shit happened, rollback your app by:
>>>     lain deploy -v 1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7
```

> - local 为 LAIN 集群的名字，请参考[安装 LAIN 客户端](../install/lain-client.html)中的设置
> - `lain tag` 为镜像打标签，标签格式为
>   `registry.${LAIN-domain}/${appname}:(release/meta)-${timestamp}-${git-commit-id}`，
>   目的是标记镜像版本
> - `release` 镜像包含了编译成果，将来会以这个镜像为基础运行容器
> - `meta` 镜像包含 `lain.yaml` 文件，用于 LAIN 集群解析，用户不需要关心
> - 部署的过程是一个异步的过程，在 `lain deploy local` 之后可以使用 `lain ps local` 查询部署结果。

此时，可以通过以下命令访问 `hello-world`：

```
[vagrant@lain hello-world]$ curl -H "Host: hello-world.lain.local" http://192.168.77.201
Hello, LAIN.
```

或者可以先更改 `/etc/hosts` 文件，然后直接使用域名访问：

```
[vagrant@lain hello-world]$ echo "192.168.77.201 hello-world.lain.local" >> /etc/hosts
[vagrant@lain hello-world]$ curl http://hello-world.lain.local
Hello, LAIN.
```

> 上面的 `192.168.77.201` 是本地 LAIN 集群的虚拟 IP，没有以 `vip` 方式启动，请使用 `192.168.77.21`

得到了 `Hello, LAIN.` 的响应，符合我们的预期。

## 扩容与更改预留内存

此外，我们还可以对 `hello-world` 进行扩容：

```
[vagrant@lain hello-world]$ lain scale -n 2 local web
>>> Start to scale Proc web of App hello-world in Lain Cluster dev with domain lain.local
>>> Scaling...
>>> {'num_instances': 2}
    status_code: 202
    message: PodGroupSpec will be patched and rescheduled.
>>> proc status:
+-----------|---------------------------------------------------------------------------------------------+
| procname  | web                                                                                         |
| proctype  | web                                                                                         |
| state     | healthy                                                                                     |
| memory    | 64 MB                                                                                       |
| cpu       | 0                                                                                           |
| image     | registry.lain.local/hello-world:release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 |
| instances | 2                                                                                           |
+-----------|---------------------------------------------------------------------------------------------+
├─> /hello-world.web.web.v9-i1-d0 (172.20.0.39) @ Node 10.106.170.101
└─> /hello-world.web.web.v9-i2-d0 (172.20.0.91) @ Node 10.106.170.104
>>> Outline of scale result:
>>>   scale of num_instances success
>>>     params: {'num_instances': 2}
```

或者改变预留内存：

```
[vagrant@lain hello-world]$ lain scale -m 64M local web
>>> Start to scale Proc web of App hello-world in Lain Cluster dev with domain yxapp.xyz
>>> Scaling......
>>> {'memory': '64M'}
    status_code: 202
    message: PodGroupSpec will be patched and rescheduled.
>>> proc status:
+-----------|---------------------------------------------------------------------------------------------+
| procname  | web                                                                                         |
| proctype  | web                                                                                         |
| state     | healthy                                                                                     |
| memory    | 64 MB                                                                                       |
| cpu       | 0                                                                                           |
| image     | registry.lain.local/hello-world:release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 |
| instances | 2                                                                                           |
+-----------|---------------------------------------------------------------------------------------------+
├─> /hello-world.web.web.v9-i1-d0 (172.20.0.39) @ Node 10.106.170.101
└─> /hello-world.web.web.v9-i2-d0 (172.20.0.91) @ Node 10.106.170.104
>>> Outline of scale result:
>>>   scale of cpu_or_memory success
>>>     params: {'memory': '64M'}
```

用 `lain ps local` 查看下部署结果吧！

```
[vagrant@lain hello-world]$ lain ps local
>>> App info:
╒══════════════╤═════════════════════════════════════════════════════╕
│ appname      │ hello-world                                         │
├──────────────┼─────────────────────────────────────────────────────┤
│ state        │ healthy                                             │
├──────────────┼─────────────────────────────────────────────────────┤
│ apptype      │ app                                                 │
├──────────────┼─────────────────────────────────────────────────────┤
│ metaversion  │ 1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 │
├──────────────┼─────────────────────────────────────────────────────┤
│ updatetime   │ 2017-05-23 11:08:14                                 │
├──────────────┼─────────────────────────────────────────────────────┤
│ deploy_error │                                                     │
╘══════════════╧═════════════════════════════════════════════════════╛
>>> Proc list:
+-----------|---------------------------------------------------------------------------------------------+
| procname  | web                                                                                         |
| proctype  | web                                                                                         |
| state     | healthy                                                                                     |
| memory    | 64 MB                                                                                       |
| cpu       | 0                                                                                           |
| image     | registry.lain.local/hello-world:release-1495536989-71b265e40e5eaf9783ed42a2e4b42fba47dc71a7 |
| instances | 2                                                                                           |
+-----------|---------------------------------------------------------------------------------------------+
├─> /hello-world.web.web.v9-i1-d0 (172.20.0.39) @ Node 10.106.170.101
└─> /hello-world.web.web.v9-i2-d0 (172.20.0.91) @ Node 10.106.170.104
```
