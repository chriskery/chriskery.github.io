---
title: CRI详解
date: 2023-11-30 22:05:21
tags:
- kubernetes
- container
categories:
- container
---
`CRI`（Container Runtime Interface 容器运行时接口）本质上就是 Kubernetes 定义的一组与容器运行时进行交互的接口，所以只要实现了这套接口的容器运行时都可以对接到 Kubernetes 平台上来。不过 Kubernetes 推出 CRI 这套标准的时候还没有现在的统治地位，所以有一些容器运行时可能不会自身就去实现 CRI 接口，于是就有了 `shim（垫片）`， 一个 shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上，其中 `dockershim` 就是 Kubernetes 对接 Docker 到 CRI 接口上的一个垫片实现。

![cri shim](https://picdn.youdianzhishi.com/images/20210809172030.png)

Kubelet 通过 gRPC 框架与容器运行时或 shim 进行通信，其中 kubelet 作为客户端，CRI shim（也可能是容器运行时本身）作为服务器。

CRI定义的 API([RuntimeService](https://github.com/kubernetes/kubernetes/blob/release-1.5/pkg/kubelet/api/v1alpha1/runtime/api.proto)) 主要包括两个 gRPC 服务，`ImageService` 和 `RuntimeService`，`ImageService` 服务主要是拉取镜像、查看和删除镜像等操作，`RuntimeService` 则是用来管理 Pod 和容器的生命周期，以及与容器交互的调用（exec/attach/port-forward）等操作，可以通过 kubelet 中的标志 `--container-runtime-endpoint` 和 `--image-service-endpoint` 来配置这两个服务的套接字。

![kubelet cri](https://picdn.youdianzhishi.com/images/20210809173134.png)

不过这里同样也有一个例外，那就是 Docker，由于 Docker 当时的江湖地位很高，Kubernetes 是直接内置了 `dockershim` 在 kubelet 中的，所以如果你使用的是 Docker 这种容器运行时的话是不需要单独去安装配置适配器之类的，当然这个举动似乎也麻痹了 Docker 公司。

![dockershim](https://picdn.youdianzhishi.com/images/20210809173555.png)

现在如果我们使用的是 Docker 的话，当我们在 Kubernetes 中创建一个 Pod 的时候，首先就是 kubelet 通过 CRI 接口调用 `dockershim`，请求创建一个容器，kubelet 可以视作一个简单的 CRI Client, 而 dockershim 就是接收请求的 Server，不过他们都是在 kubelet 内置的。

`dockershim` 收到请求后, 转化成 Docker Daemon 能识别的请求, 发到 Docker Daemon 上请求创建一个容器，请求到了 Docker Daemon 后续就是 Docker 创建容器的流程了，去调用 `containerd`，然后创建 `containerd-shim` 进程，通过该进程去调用 `runc` 去真正创建容器。

其实我们仔细观察也不难发现使用 Docker 的话其实是调用链比较长的，真正容器相关的操作其实 containerd 就完全足够了，Docker 太过于复杂笨重了，当然 Docker 深受欢迎的很大一个原因就是提供了很多对用户操作比较友好的功能，但是对于 Kubernetes 来说压根不需要这些功能，因为都是通过接口去操作容器的，所以自然也就可以将容器运行时切换到 containerd 来。

![切换到containerd](https://picdn.youdianzhishi.com/images/20210810094948.png)

切换到 containerd 可以消除掉中间环节，操作体验也和以前一样，但是由于直接用容器运行时调度容器，所以它们对 Docker 来说是不可见的。 因此，你以前用来检查这些容器的 Docker 工具就不能使用了。

你不能再使用 `docker ps` 或 `docker inspect` 命令来获取容器信息。由于不能列出容器，因此也不能获取日志、停止容器，甚至不能通过 `docker exec` 在容器中执行命令。

当然我们仍然可以下载镜像，或者用 `docker build` 命令构建镜像，但用 Docker 构建、下载的镜像，对于容器运行时和 Kubernetes，均不可见。为了在 Kubernetes 中使用，需要把镜像推送到镜像仓库中去。

从上图可以看出在 containerd 1.0 中，对 CRI 的适配是通过一个单独的 `CRI-Containerd` 进程来完成的，这是因为最开始 containerd 还会去适配其他的系统（比如 swarm），所以没有直接实现 CRI，所以这个对接工作就交给 `CRI-Containerd` 这个 shim 了。

然后到了 containerd 1.1 版本后就去掉了 `CRI-Containerd` 这个 shim，直接把适配逻辑作为插件的方式集成到了 containerd 主进程中，现在这样的调用就更加简洁了。

![containerd cri](https://picdn.youdianzhishi.com/images/20210810095546.png)

与此同时 Kubernetes 社区也做了一个专门用于 Kubernetes 的 CRI 运行时 [CRI-O](https://cri-o.io/)，直接兼容 CRI 和 OCI 规范。

![cri-o](https://picdn.youdianzhishi.com/images/20210810100752.png)

这个方案和 containerd 的方案显然比默认的 dockershim 简洁很多，不过由于大部分用户都比较习惯使用 Docker，所以大家还是更喜欢使用 `dockershim` 方案。

但是随着 CRI 方案的发展，以及其他容器运行时对 CRI 的支持越来越完善，Kubernetes 社区在 2020 年 7 月份就开始着手移除 dockershim 方案了：https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim，现在的移除计划是在 1.20 版本中将 kubelet 中内置的 dockershim 代码分离，将内置的 dockershim 标记为`维护模式`，当然这个时候仍然还可以使用 dockershim，目标是在 1.23/1.24 版本发布没有 dockershim 的版本（代码还在，但是要默认支持开箱即用的 docker 需要自己构建 kubelet，会在某个宽限期过后从 kubelet 中删除内置的 dockershim 代码）。

那么这是否就意味这 Kubernetes 不再支持 Docker 了呢？当然不是的，这只是废弃了内置的 `dockershim` 功能而已，Docker 和其他容器运行时将一视同仁，不会单独对待内置支持，如果我们还想直接使用 Docker 这种容器运行时应该怎么办呢？可以将 dockershim 的功能单独提取出来独立维护一个 `cri-dockerd` 即可，就类似于 containerd 1.0 版本中提供的 `CRI-Containerd`，当然还有一种办法就是 Docker 官方社区将 CRI 接口内置到 Dockerd 中去实现。

但是我们也清楚 Dockerd 也是去直接调用的 Containerd，而 containerd 1.1 版本后就内置实现了 CRI，所以 Docker 也没必要再去单独实现 CRI 了，当 Kubernetes 不再内置支持开箱即用的 Docker 的以后，最好的方式当然也就是直接使用 Containerd 这种容器运行时，而且该容器运行时也已经经过了生产环境实践的，接下来我们就来学习下 Containerd 的使用。

转载自：https://www.qikqiak.com/post/containerd-usage/