---
title: kubernetes-klog简介
date: 2023-12-19 22:51:09
tags:
- kubernetes
- klog
categories:
- kubernetes
---
## 背景
[klog](https://github.com/kubernetes/klog.git)是Kubernetes中用于记录日志的日志库。它旨在为Kubernetes生态系统中的不同组件和服务提供一致和统一的日志记录体验。名称“klog”代表“Kubernetes logger”。 

## 功能简介
- klog 支持不同的日志级别，包括 `INFO`、`WARNING`、`ERROR` 、`FATAL`和 `VERBOSE`。开发人员可以设置日志级别以根据他们的需求控制日志的详细程度。写入到高严重等级日志文件的消息也会被写入到所有严重等级比它低的日志文件中.
- klog提供了由 -v 和 -vmodule=file=2 标志控制的 V-style 日志。
- klog 支持结构化日志，这意味着日志消息可以包含键值对。这使得使用外部工具过滤和分析日志变得更加容易。
- klog 允许通过命令行标志控制其行为。这些标志可以在启动 Go 应用程序时设置，以配置日志级别 (例如：--v=2)、日志格式和其他与日志记录相关的设置。

## 使用
以下是在 Go 应用程序中使用 `klog` 的简单示例：

```golang
package main

import (
	"k8s.io/klog/v2"
)

func main() {
	klog.Info("这是一条信息消息")
	klog.Warning("这是一条警告消息")
	klog.Error("这是一条错误消息")
	// 可以使用 klog 包的其他功能进行额外的配置和日志记录。
	
	klog.V(1).Info("这是一条Level为1的消息")
	klog.V(2).Info("这是一条Level为2的消息")
}
```

通过设置 --v 标志，你可以调整日志的详细程度：

```shell
./your_app --v=2
```

例如，当我们想要查看 kubernetes的组件的详细信息时，我们可以通过调整`--v`标志调整其日志输出级别：
```golang
root@k8s002:pts/20->/root (0)
> kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-7f89b7bc75-992fg         1/1     Running   0          85d
coredns-7f89b7bc75-zq9xn         1/1     Running   0          85d
etcd-k8s002                      1/1     Running   0          85d
kube-apiserver-k8s002            1/1     Running   0          85d
kube-controller-manager-k8s002   1/1     Running   28         85d
kube-proxy-lcwzq                 1/1     Running   0          85d
kube-scheduler-k8s002            1/1     Running   28         85d
-----------------------------------------------
root@k8s002:pts/20->/root (0)
> kubectl get pods -n kube-system --v=6
I1219 23:22:41.474328   25577 loader.go:379] Config loaded from file:  /root/.kube/config
I1219 23:22:41.491635   25577 round_trippers.go:445] GET https://47.123.6.56:6443/api/v1/namespaces/kube-system/pods?limit=500 200 OK in 11 milliseconds
NAME                             READY   STATUS    RESTARTS   AGE
coredns-7f89b7bc75-992fg         1/1     Running   0          85d
coredns-7f89b7bc75-zq9xn         1/1     Running   0          85d
etcd-k8s002                      1/1     Running   0          85d
kube-apiserver-k8s002            1/1     Running   0          85d
kube-controller-manager-k8s002   1/1     Running   28         85d
kube-proxy-lcwzq                 1/1     Running   0          85d
kube-scheduler-k8s002            1/1     Running   28         85d
```


