# `lain.yaml` 的更多配置

lain.yaml 提供了一些选项来优化构建过程。

## `build.prepare`

[laincloud/hello-world/lain.yaml@v2.0.0](https://github.com/laincloud/hello-world/blob/v2.0.0/lain.yaml)
中需要下载 `github.com/go-redis/redis`，如果网络较慢的话会拖慢 `lain build` 时间，这时我们可以添加
`build.prepare`，来缓存下载结果。

```
build:
  base: golang:1.8
  prepare:  # lain prepare 时以 build.base 为基底镜像，执行 build.prepare.script，生成 registry.${LAIN-domain}/hello-world:prepare-${build.prepare.version}-${timestamp}，作为 lain build 的新基底镜像
    version: 201705231514  # lain build 时会选取 version 最大的 prepare 镜像作为基底镜像
    script:
      - go get -u github.com/go-redis/redis  # 安装依赖
  script:  # 以 registry.${LAIN-domain}/hello-world:prepare-201705231514-${timestamp} 为基底镜像
    - go build -o hello-world  # 编译，类似于 Dockerfile 里的 RUN，WORKDIR 为 /lain/app
```

## `release`

Docker 的镜像文件是分层存储的。如果在生产环境的话，我们可能会希望使用统一的 release 基底镜像，
来减小 registry.${LAIN-domain} 存储的镜像体积：

```
release:  # 运行容器时使用的镜像，不写时默认为 hello-world:build
  dest_base: centos:7  # 团队内部可能会希望采用统一的基底镜像，以减小镜像仓库存储的镜像体积
  copy:  # 将 build 镜像里的编译结果复制到 release 镜像
    - src: /lain/app/hello-world  # build 镜像里的编译结果
      dest: /hello-world  # release 镜像里的路径
```

[laincloud/hello-world@v2.1.0](https://github.com/laincloud/hello-world/tree/v2.1.0) 的完整代码在这里。
试着多次运行 `lain build`，速度是不是加快了？
