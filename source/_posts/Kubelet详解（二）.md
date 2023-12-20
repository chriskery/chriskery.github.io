---
title: Kubelet详解（二）
date: 2023-12-20 15:23:07
tags:
- kubernetes
- kubelet
categories:
- kubernetes

---
本章节介绍一下Kubelet对Pod的同步流程。
## 相关概念
### Mirror Pod
在浏览Kubelet的相关源码的时候，我们经常会看到如下的代码调用：
```golang
	GetPodByMirrorPod(*v1.Pod) (*v1.Pod, bool)
	// GetMirrorPodByPod returns the mirror pod for the given static pod and
	// whether it was known to the pod manager.
	GetMirrorPodByPod(*v1.Pod) (*v1.Pod, bool)
	// GetPodAndMirrorPod returns the complement for a pod - if a pod was provided
	// and a mirror pod can be found, return it. If a mirror pod is provided and
	// the pod can be found, return it and true for wasMirror.
	GetPodAndMirrorPod(*v1.Pod) (pod, mirrorPod *v1.Pod, wasMirror bool)
```
这里面的**mirror pod**很容易让人一头雾水。在介绍**mirror pod**之前，我们首先需要知道Kubelet所监听的Pods的来源，主要分为三种：
```golang
// define file config source
	if kubeCfg.StaticPodPath != "" {
		klog.InfoS("Adding static pod path", "path", kubeCfg.StaticPodPath)
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.FileSource))
	}

	// define url config source
	if kubeCfg.StaticPodURL != "" {
		klog.InfoS("Adding pod URL with HTTP header", "URL", kubeCfg.StaticPodURL, "header", manifestURLHeader)
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.HTTPSource))
	}

	if kubeDeps.KubeClient != nil {
		klog.InfoS("Adding apiserver pod source")
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, nodeHasSynced, cfg.Channel(ctx, kubetypes.ApiserverSource))
	}
```

Kubelet启动后，会负责所有调度到对应机器上的Pod的生命周期管理，获取Pod清单列表的方式包括：
● 文件∶启动参数--config指定的配置目录下的文件（默认/etc/Kubernetes/manifests/）。
● HTTP endpoint（URL）∶启动参数--manifest-url设置。
● APIServer∶通过APIServer监听etcd目录，同步Pod清单。
● HTTP Server∶ kubelet 侦听 HTTP 请求，并响应简单的 API 以提交新的 Pod 清单

其中所有来自`APIServer`的Pod为非静态Pod，其余来源的Pod称之为`静态Pod`。静态 Pod 不通过 master 节点上的 apiserver 操作及管理，直接由特定节点上的 kubelet 进程来管理的。无法与我们常用的控制器 Deployment 或者 DaemonSet 等进行关联，它由 kubelet 进程自己来监控，当 pod 崩溃时重启该 pod。静态 pod 始终绑定在某个 kubelet ，并且始终运行在同一个节点上。

为了让这些不是由API Server管理的Pods也能在Kubernetes API中表示和监控，kubelet 会自动为每一个静态 pod 在 kubernetes 的 apiserver 上创建一个镜像 pod，也就是`mirror pod`。注意，镜像Pod是一个只读的资源，在API server上不能对其进行修改或删除
