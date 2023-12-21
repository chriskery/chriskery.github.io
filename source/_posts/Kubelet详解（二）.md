---
title: Kubelet详解（二）- Pod同步流程
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
- 文件∶启动参数--config指定的配置目录下的文件（默认/etc/Kubernetes/manifests/）。
- HTTP endpoint（URL）∶启动参数--manifest-url设置。
- APIServer∶通过APIServer监听etcd目录，同步Pod清单。
- HTTP Server∶ kubelet 侦听 HTTP 请求，并响应简单的 API 以提交新的 Pod 清单

其中所有来自`APIServer`的Pod为非静态Pod，其余来源的Pod称之为`静态Pod`。静态 Pod 不通过 master 节点上的 apiserver 操作及管理，直接由特定节点上的 kubelet 进程来管理的。无法与我们常用的控制器 Deployment 或者 DaemonSet 等进行关联，它由 kubelet 进程自己来监控，当 pod 崩溃时重启该 pod。静态 pod 始终绑定在某个 kubelet ，并且始终运行在同一个节点上。

为了让这些不是由API Server管理的Pods也能在Kubernetes API中表示和监控，kubelet 会自动为每一个静态 pod 在 kubernetes 的 apiserver 上创建一个镜像 pod，也就是`mirror pod`。注意，镜像Pod是一个只读的资源，在API server上不能对其进行修改或删除

### syncLoopIteration

Kubelet的`syncLoopIteration`功能是Kubernetes节点代理kubelet内部的一个核心循环，负责同步节点上的Pod状态。在这个循环中，kubelet会执行一系列动作来确保节点上的Pods状态与其期望的状态一致。

```golang
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler, syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool
```

`syncLoopIteration`参数为各个事件源的channel，其具体含义为：
- configCh <-chan kubetypes.PodUpdate: 这是一个Go channel，kubelet通过它接收Pod更新信息。kubetypes.PodUpdate是一个结构体，包含了Pods的更新信息，例如Pod的创建、更新、删除事件。

- handler SyncHandler: SyncHandler是一个接口，它定义了处理Pod同步的一系列方法。syncLoopIteration函数会调用SyncHandler的方法来同步Pod的状态，比如启动新的Pod、更新现有的Pod、或者停止不再需要的Pod。

- syncCh <-chan time.Time: 这是一个定时器channel，kubelet使用它来定期执行同步操作。当这个channel发出信号时，它触发kubelet去检查和更新Pod的状态。

- housekeepingCh <-chan time.Time: 这个也是一个定时器channel，但它专门用于触发常规的节点维护任务，如垃圾收集、状态更新等。

- plegCh <-chan *pleg.PodLifecycleEvent: plegCh是Pod生命周期事件生成器（PLEG）的channel。PLEG负责检测容器的状态变化，并生成相应的事件。kubelet监听这个channel来获取容器状态变化的通知，并根据这些变化来更新Pod的状态。

## configCh

针对`configCh`传递过来的Pod的更新事件，其响应如下：
```golang
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.ErrorS(nil, "Kubelet does not support snapshot update")
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		kl.sourcesReady.AddSource(u.Source)

```

kubelet 将Pod的操作类型分为`kubetypes.ADD、kubetypes.UPDATE、kubetypes.REMOVE
、kubetypes.RECONCILE、kubetypes.DELETE、kubetypes.SE`，并根据事件的类型调用handler的对应接口进行处理。

### handler.HandlePodAdditions(u.Pods)
其函数实现为：
```golang
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		kl.podManager.AddPod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		// Only go through the admission process if the pod is not requested
		// for termination by another part of the kubelet. If the pod is already
		// using resources (previously admitted), the pod worker is going to be
		// shutting it down. If the pod hasn't started yet, we know that when
		// the pod worker is invoked it will also avoid setting up the pod, so
		// we simply avoid doing any work.
		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			activePods := kl.filterOutInactivePods(existingPods)

			if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {} else {
				// Check if we can admit the pod; if not, reject it.
				if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
					kl.rejectPod(pod, reason, message)
					continue
				}
			}
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodCreate,
			StartTime:  start,
		})
	}
}
```

1. 将待添加的pods按照创建时间进行排序
2. 依次遍历每个Pod
3. 获取`podManager`已经存在的pod，podManager缓存了该Kubelet所有已经`admitted`的pods和mirror pods
4. 将新的Pod添加到`podManager`中
5. 调用`podManager`获取到缓存里该Pod对应的pod和mirror pod，并判断是否是`mirror pod`，调用`kl.podWorkers.UpdatePod`传递更新事件。
6. 判断是否Pod的`Termination`请求，是的话调用`kl.podWorkers.UpdatePod`传递更新事件，否则检查是否应该Admit该Pod（资源限制、节点匹配、xxxx），然后再调用`kl.podWorkers.UpdatePod`。
7. 无论是 MirrorPod 还是正常 Pod ，最终都是调用 `kl.podWorkers.UpdatePod` 接口进行处理，他们的区别在于传进去的参数的不同。MirrorPod所有的=操作类型都被当做 UPDATE 来处理，更新它在 apiserver 上的状态。

### handler.HandlePodUpdates(u.Pods)
其实现如下：
```golang

// HandlePodUpdates is the callback in the SyncHandler interface for pods
// being updated from a config source.
func (kl *Kubelet) HandlePodUpdates(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.UpdatePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
		}

		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodUpdate,
			StartTime:  start,
		})
	}
}
```

主要做了两件事：
1. 更新`podManager`里的缓存
2. 判断是否是MirrorPod，最终调用 `kl.podWorkers.UpdatePod` 接口进行处理。

### handler.HandlePodRemoves(u.Pods)
其具体实现如下：
```golang

// HandlePodRemoves is the callback in the SyncHandler interface for pods
// being removed from a config source.
func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.RemovePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		// Deletion is allowed to fail because the periodic cleanup routine
		// will trigger deletion again.
		if err := kl.deletePod(pod); err != nil {
			klog.V(2).InfoS("Failed to delete pod", "pod", klog.KObj(pod), "err", err)
		}
	}
}
```

跟`HandlePodUpdates`实现差不多，只不过在最后调用`kl.deletePod(pod)`从Kubelet内部的状态里删除对应的Pod，其内部分为两个步骤：
1. 异步停止所有相关联的pod worker
2. 给`podKillingCh`发送signal通知pod进行kill的过程

### handler.HandlePodRemoves(u.Pods)
其具体实现如下：
```golang
func (kl *Kubelet) HandlePodReconcile(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		// Update the pod in pod manager, status manager will do periodically reconcile according
		// to the pod manager.
		kl.podManager.UpdatePod(pod)

		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			// Static pods should be reconciled the same way as regular pods
		}


		if status.NeedToReconcilePodReadiness(pod) {
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodSync,
				StartTime:  start,
			})
		}


		if eviction.PodIsEvicted(pod.Status) {
			if podStatus, err := kl.podCache.Get(pod.UID); err == nil {
				kl.containerDeletor.deleteContainersInPod("", podStatus, true)
			}
		}
	}
}
```

handler.HandlePodReconcile() 分 3 步走，先将 pod 状态更新到缓存中，然后判断 pod 状态是否需要重新reconcile ，则调用 `kl.podWorkers.UpdatePod` 完成同步，如果是被驱逐的，则删掉该Pod的所有容器。

## kl.podWorkers.UpdatePod

`kl.podWorkers.UpdatePod`处理pod的配置变化和pod的`termination`状态，其具体实现为：
```golang

// UpdatePod carries a configuration change or termination state to a pod. A pod is either runnable,
// terminating, or terminated, and will transition to terminating if: deleted on the apiserver,
// discovered to have a terminal phase (Succeeded or Failed), or evicted by the kubelet.
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
	// Handle when the pod is an orphan (no config) and we only have runtime status by running only
	// the terminating part of the lifecycle. A running pod contains only a minimal set of information
	// about the pod
	var isRuntimePod bool
	var uid types.UID
	var name, ns string
	if runningPod := options.RunningPod; runningPod != nil {
		if options.Pod == nil {
			// the sythetic pod created here is used only as a placeholder and not tracked
			if options.UpdateType != kubetypes.SyncPodKill {
				klog.InfoS("Pod update is ignored, runtime pods can only be killed", "pod", klog.KRef(runningPod.Namespace, runningPod.Name), "podUID", runningPod.ID, "updateType", options.UpdateType)
				return
			}
			uid, ns, name = runningPod.ID, runningPod.Namespace, runningPod.Name
			isRuntimePod = true
		} else {
			options.RunningPod = nil
			uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
			klog.InfoS("Pod update included RunningPod which is only valid when Pod is not specified", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		}
	} else {
		uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
	}

	p.podLock.Lock()
	defer p.podLock.Unlock()

	// decide what to do with this pod - we are either setting it up, tearing it down, or ignoring it
	var firstTime bool
	now := p.clock.Now()
	status, ok := p.podSyncStatuses[uid]
	if !ok {
		klog.V(4).InfoS("Pod is being synced for the first time", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		firstTime = true
		status = &podSyncStatus{
			syncedAt: now,
			fullname: kubecontainer.BuildPodFullName(name, ns),
		}
		// if this pod is being synced for the first time, we need to make sure it is an active pod
		if options.Pod != nil && (options.Pod.Status.Phase == v1.PodFailed || options.Pod.Status.Phase == v1.PodSucceeded) {
			// Check to see if the pod is not running and the pod is terminal; if this succeeds then record in the podWorker that it is terminated.
			// This is needed because after a kubelet restart, we need to ensure terminal pods will NOT be considered active in Pod Admission. See http://issues.k8s.io/105523
			// However, `filterOutInactivePods`, considers pods that are actively terminating as active. As a result, `IsPodKnownTerminated()` needs to return true and thus `terminatedAt` needs to be set.
			if statusCache, err := p.podCache.Get(uid); err == nil {
				if isPodStatusCacheTerminal(statusCache) {
					// At this point we know:
					// (1) The pod is terminal based on the config source.
					// (2) The pod is terminal based on the runtime cache.
					// This implies that this pod had already completed `SyncTerminatingPod` sometime in the past. The pod is likely being synced for the first time due to a kubelet restart.
					// These pods need to complete SyncTerminatedPod to ensure that all resources are cleaned and that the status manager makes the final status updates for the pod.
					// As a result, set finished: false, to ensure a Terminated event will be sent and `SyncTerminatedPod` will run.
					status = &podSyncStatus{
						terminatedAt:       now,
						terminatingAt:      now,
						syncedAt:           now,
						startedTerminating: true,
						finished:           false,
						fullname:           kubecontainer.BuildPodFullName(name, ns),
					}
				}
			}
		}
		p.podSyncStatuses[uid] = status
	}

	// RunningPods represent an unknown pod execution and don't contain pod spec information
	// sufficient to perform any action other than termination. If we received a RunningPod
	// after a real pod has already been provided, use the most recent spec instead. Also,
	// once we observe a runtime pod we must drive it to completion, even if we weren't the
	// ones who started it.
	pod := options.Pod
	if isRuntimePod {
		status.observedRuntime = true
		switch {
		case status.pendingUpdate != nil && status.pendingUpdate.Pod != nil:
			pod = status.pendingUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
		case status.activeUpdate != nil && status.activeUpdate.Pod != nil:
			pod = status.activeUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
		default:
			// we will continue to use RunningPod.ToAPIPod() as pod here, but
			// options.Pod will be nil and other methods must handle that appropriately.
			pod = options.RunningPod.ToAPIPod()
		}
	}

	// When we see a create update on an already terminating pod, that implies two pods with the same UID were created in
	// close temporal proximity (usually static pod but it's possible for an apiserver to extremely rarely do something
	// similar) - flag the sync status to indicate that after the pod terminates it should be reset to "not running" to
	// allow a subsequent add/update to start the pod worker again. This does not apply to the first time we see a pod,
	// such as when the kubelet restarts and we see already terminated pods for the first time.
	if !firstTime && status.IsTerminationRequested() {
		if options.UpdateType == kubetypes.SyncPodCreate {
			status.restartRequested = true
			klog.V(4).InfoS("Pod is terminating but has been requested to restart with same UID, will be reconciled later", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}
	}

	// once a pod is terminated by UID, it cannot reenter the pod worker (until the UID is purged by housekeeping)
	if status.IsFinished() {
		klog.V(4).InfoS("Pod is finished processing, no further updates", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		return
	}

	// check for a transition to terminating
	var becameTerminating bool
	if !status.IsTerminationRequested() {
		switch {
		case isRuntimePod:
			klog.V(4).InfoS("Pod is orphaned and must be torn down", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.DeletionTimestamp != nil:
			klog.V(4).InfoS("Pod is marked for graceful deletion, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.Status.Phase == v1.PodFailed, pod.Status.Phase == v1.PodSucceeded:
			klog.V(4).InfoS("Pod is in a terminal phase (success/failed), begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.terminatingAt = now
			becameTerminating = true
		case options.UpdateType == kubetypes.SyncPodKill:
			if options.KillPodOptions != nil && options.KillPodOptions.Evict {
				klog.V(4).InfoS("Pod is being evicted by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
				status.evicted = true
			} else {
				klog.V(4).InfoS("Pod is being removed by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			}
			status.terminatingAt = now
			becameTerminating = true
		}
	}

	// once a pod is terminating, all updates are kills and the grace period can only decrease
	var wasGracePeriodShortened bool
	switch {
	case status.IsTerminated():
		// A terminated pod may still be waiting for cleanup - if we receive a runtime pod kill request
		// due to housekeeping seeing an older cached version of the runtime pod simply ignore it until
		// after the pod worker completes.
		if isRuntimePod {
			klog.V(3).InfoS("Pod is waiting for termination, ignoring runtime-only kill until after pod worker is fully terminated", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}

		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
		}
		options.KillPodOptions = nil

	case status.IsTerminationRequested():
		if options.KillPodOptions == nil {
			options.KillPodOptions = &KillPodOptions{}
		}

		if ch := options.KillPodOptions.CompletedCh; ch != nil {
			status.notifyPostTerminating = append(status.notifyPostTerminating, ch)
		}
		if fn := options.KillPodOptions.PodStatusFunc; fn != nil {
			status.statusPostTerminating = append(status.statusPostTerminating, fn)
		}

		gracePeriod, gracePeriodShortened := calculateEffectiveGracePeriod(status, pod, options.KillPodOptions)

		wasGracePeriodShortened = gracePeriodShortened
		status.gracePeriod = gracePeriod
		// always set the grace period for syncTerminatingPod so we don't have to recalculate,
		// will never be zero.
		options.KillPodOptions.PodTerminationGracePeriodSecondsOverride = &gracePeriod

	default:
		// KillPodOptions is not valid for sync actions outside of the terminating phase
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
			options.KillPodOptions = nil
		}
	}

	// start the pod worker goroutine if it doesn't exist
	podUpdates, exists := p.podUpdates[uid]
	if !exists {
		// buffer the channel to avoid blocking this method
		podUpdates = make(chan struct{}, 1)
		p.podUpdates[uid] = podUpdates

		// ensure that static pods start in the order they are received by UpdatePod
		if kubetypes.IsStaticPod(pod) {
			p.waitingToStartStaticPodsByFullname[status.fullname] =
				append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
		}

		// allow testing of delays in the pod update channel
		var outCh <-chan struct{}
		if p.workerChannelFn != nil {
			outCh = p.workerChannelFn(uid, podUpdates)
		} else {
			outCh = podUpdates
		}

		// spawn a pod worker
		go func() {
			// TODO: this should be a wait.Until with backoff to handle panics, and
			// accept a context for shutdown
			defer runtime.HandleCrash()
			defer klog.V(3).InfoS("Pod worker has stopped", "podUID", uid)
			p.podWorkerLoop(uid, outCh)
		}()
	}

	// measure the maximum latency between a call to UpdatePod and when the pod worker reacts to it
	// by preserving the oldest StartTime
	if status.pendingUpdate != nil && !status.pendingUpdate.StartTime.IsZero() && status.pendingUpdate.StartTime.Before(options.StartTime) {
		options.StartTime = status.pendingUpdate.StartTime
	}

	// notify the pod worker there is a pending update
	status.pendingUpdate = &options
	status.working = true
	klog.V(4).InfoS("Notifying pod of pending update", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
	select {
	case podUpdates <- struct{}{}:
	default:
	}

	if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
		klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
		status.cancelFn()
		return
	}
}

```

`RuntimePod`是指正在被container runtime执行，但kubelet无法获取到对应Pod的相关配置信息的Pod。对于此种Pod，kubelet会执行`termination`操作。

1. 判断Pod是否为`RuntimePod`
2. 将Pod的相关状态信息更新到`p.podSyncStatuses[uid]`
3. 如果能从缓存中拿到该 pod 的缓存，而且 pod 处于终止状态，而这时候 options.UpdateType == kubetypes.SyncPodCreate ，则说明使用相同 uid 的 pod 需要被重启，会被后续的程序处理。
4. 检查 pod 是否转换到`Terminating`状态。并根据相关状态去更新`options`和`status`的值。
5. kubelet 会为每一个Pod创建一条单独的`goroutine`负责监听该Pod的更新事件并负责后续的状态更新，`p.podUpdates`负责跟踪kubelet所有启动的此类goroutine。若待更新的Pod不存在此`goroutine`，则调用`p.podWorkerLoop(uid, outCh)`创建。
6. 每次更新实际都是将更新事件发送到 podUpdates 通道，这个通道的另一端就是 `p.podWorkerLoop(uid, outCh)` 在监听。

## p.podWorkerLoop(uid, outCh)
`podWorkerLoop`负责管理 Pod 的顺序状态更新，直到达到最终状态后退出。这个循环主要负责推动 Pod 经过以下四个主要阶段：
- 等待开始（Wait to start）：确保同一时间不会有两个具有相同 UID 或全名的 Pod 同时运行。

- 同步（Sync）：通过协调期望的 Pod 规格和 Pod 的运行时状态来安排 Pod 的设置，这是一个组织 Pod 设置过程的阶段。

- 终止中（Terminating）：确保 Pod 中所有正在运行的容器都被停止。

- 已终止（Terminated）：清理必须释放的任何资源，以便 Pod 可以被删除。

其代码实现为：
```golang
func (p *podWorkers) podWorkerLoop(podUID types.UID, podUpdates <-chan struct{}) {
	var lastSyncTime time.Time
	for range podUpdates {
		ctx, update, canStart, canEverStart, ok := p.startPodSync(podUID)
		// If we had no update waiting, it means someone initialized the channel without filling out pendingUpdate.
		if !ok {
			continue
		}
		// If the pod was terminated prior to the pod being allowed to start, we exit the loop.
		if !canEverStart {
			return
		}
		// If the pod is not yet ready to start, continue and wait for more updates.
		if !canStart {
			continue
		}

		podUID, podRef := podUIDAndRefForUpdate(update.Options)

		klog.V(4).InfoS("Processing pod event", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
		var isTerminal bool
		err := func() error {
			// The worker is responsible for ensuring the sync method sees the appropriate
			// status updates on resyncs (the result of the last sync), transitions to
			// terminating (no wait), or on terminated (whatever the most recent state is).
			// Only syncing and terminating can generate pod status changes, while terminated
			// pods ensure the most recent status makes it to the api server.
			var status *kubecontainer.PodStatus
			var err error
			switch {
			case update.Options.RunningPod != nil:
				// when we receive a running pod, we don't need status at all because we are
				// guaranteed to be terminating and we skip updates to the pod
			default:
				// wait until we see the next refresh from the PLEG via the cache (max 2s)
				// TODO: this adds ~1s of latency on all transitions from sync to terminating
				//  to terminated, and on all termination retries (including evictions). We should
				//  improve latency by making the pleg continuous and by allowing pod status
				//  changes to be refreshed when key events happen (killPod, sync->terminating).
				//  Improving this latency also reduces the possibility that a terminated
				//  container's status is garbage collected before we have a chance to update the
				//  API server (thus losing the exit code).
				status, err = p.podCache.GetNewerThan(update.Options.Pod.UID, lastSyncTime)

				if err != nil {
					// This is the legacy event thrown by manage pod loop all other events are now dispatched
					// from syncPodFn
					p.recorder.Eventf(update.Options.Pod, v1.EventTypeWarning, events.FailedSync, "error determining status: %v", err)
					return err
				}
			}

			// Take the appropriate action (illegal phases are prevented by UpdatePod)
			switch {
			case update.WorkType == TerminatedPod:
				err = p.podSyncer.SyncTerminatedPod(ctx, update.Options.Pod, status)

			case update.WorkType == TerminatingPod:
				var gracePeriod *int64
				if opt := update.Options.KillPodOptions; opt != nil {
					gracePeriod = opt.PodTerminationGracePeriodSecondsOverride
				}
				podStatusFn := p.acknowledgeTerminating(podUID)

				// if we only have a running pod, terminate it directly
				if update.Options.RunningPod != nil {
					err = p.podSyncer.SyncTerminatingRuntimePod(ctx, update.Options.RunningPod)
				} else {
					err = p.podSyncer.SyncTerminatingPod(ctx, update.Options.Pod, status, gracePeriod, podStatusFn)
				}

			default:
				isTerminal, err = p.podSyncer.SyncPod(ctx, update.Options.UpdateType, update.Options.Pod, update.Options.MirrorPod, status)
			}

			lastSyncTime = p.clock.Now()
			return err
		}()

		var phaseTransition bool
		switch {
		case errors.Is(err, context.Canceled):
			// when the context is cancelled we expect an update to already be queued
			klog.V(2).InfoS("Sync exited with context cancellation error", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)

		case err != nil:
			// we will queue a retry
			klog.ErrorS(err, "Error syncing pod, skipping", "pod", podRef, "podUID", podUID)

		case update.WorkType == TerminatedPod:
			// we can shut down the worker
			p.completeTerminated(podUID)
			if start := update.Options.StartTime; !start.IsZero() {
				metrics.PodWorkerDuration.WithLabelValues("terminated").Observe(metrics.SinceInSeconds(start))
			}
			klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
			return

		case update.WorkType == TerminatingPod:
			// pods that don't exist in config don't need to be terminated, other loops will clean them up
			if update.Options.RunningPod != nil {
				p.completeTerminatingRuntimePod(podUID)
				if start := update.Options.StartTime; !start.IsZero() {
					metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
				}
				klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
				return
			}
			// otherwise we move to the terminating phase
			p.completeTerminating(podUID)
			phaseTransition = true

		case isTerminal:
			// if syncPod indicated we are now terminal, set the appropriate pod status to move to terminating
			klog.V(4).InfoS("Pod is terminal", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
			p.completeSync(podUID)
			phaseTransition = true
		}

		// queue a retry if necessary, then put the next event in the channel if any
		p.completeWork(podUID, phaseTransition, err)
		if start := update.Options.StartTime; !start.IsZero() {
			metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
		}
		klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
	}
}
```

1. 调用`p.startPodSync(podUID)`判断是否pod是否已经启动，若未启动，则返回能否启动，同时在`p.startPodSync`内部更新了pod的缓存状态。
2. ` p.podCache.GetNewerThan(update.Options.Pod.UID, lastSyncTime)`从缓存中根据 uid 获取 pod 的状态信息
3. 根据 `update.WorkType`` 的不同调用不同的接口来同步 pod ，有以下接口。
- p.syncPod() ，对新的 pod 进行同步。
- p.syncTerminatedPod() ，对已退出状态的 pod 进行同步。
- p.syncTerminatingPod() ，对正在退出状态的 pod 进行同步。
- p.SyncTerminatingRuntimePod() ，对`RuntimePod`进行同步
4. 根据同步类型的完成，不同的同步接口回调不同的完成接口。
- p.completeTerminatingRuntimePod() ，当退出状态的孤儿 pod 完成同步时调用。
- p.completeTerminating() ，当退出状态的 pod 完成同步时调用。
- p.completeSync() ，当完成新 pod 的同步时调用。
- p.completeWork() ，完成同步工作时调用。


### p.syncPod()
`p.syncPod()` 负责单个Pod的状态同步，这是一个可重入的方法，负责推进Pod的状态直到期望的状态为止，其代码实现为：
```golang
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	ctx, otelSpan := kl.tracer.Start(ctx, "syncPod", trace.WithAttributes(
		attribute.String("k8s.pod.uid", string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		attribute.String("k8s.pod.name", pod.Name),
		attribute.String("k8s.pod.update_type", updateType.String()),
		attribute.String("k8s.namespace.name", pod.Namespace),
	))
	klog.V(4).InfoS("SyncPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer func() {
		klog.V(4).InfoS("SyncPod exit", "pod", klog.KObj(pod), "podUID", pod.UID, "isTerminal", isTerminal)
		otelSpan.End()
	}()

	// Latency measurements for the main workflow are relative to the
	// first time the pod was seen by kubelet.
	var firstSeenTime time.Time
	if firstSeenTimeStr, ok := pod.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]; ok {
		firstSeenTime = kubetypes.ConvertToTimestamp(firstSeenTimeStr).Get()
	}

	// Record pod worker start latency if being created
	// TODO: make pod workers record their own latencies
	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// This is the first time we are syncing the pod. Record the latency
			// since kubelet first saw the pod if firstSeenTime is set.
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
		} else {
			klog.V(3).InfoS("First seen time not recorded for pod",
				"podUID", pod.UID,
				"pod", klog.KObj(pod))
		}
	}

	// Generate final API pod status with pod and status manager status
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. (See #24576)
	// TODO(random-liu): After writing pod spec into container labels, check whether pod is using host network, and
	// set pod IP to hostIP directly in runtime.GetPodStatus
	podStatus.IPs = make([]string, 0, len(apiPodStatus.PodIPs))
	for _, ipInfo := range apiPodStatus.PodIPs {
		podStatus.IPs = append(podStatus.IPs, ipInfo.IP)
	}
	if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
		podStatus.IPs = []string{apiPodStatus.PodIP}
	}

	// If the pod is terminal, we don't need to continue to setup the pod
	if apiPodStatus.Phase == v1.PodSucceeded || apiPodStatus.Phase == v1.PodFailed {
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		isTerminal = true
		return isTerminal, nil
	}

	// If the pod should not be running, we request the pod's containers be stopped. This is not the same
	// as termination (we want to stop the pod, but potentially restart it later if soft admission allows
	// it later). Set the status and phase appropriately
	runnable := kl.canRunPod(pod)
	if !runnable.Admit {
		// Pod is not runnable; and update the Pod and Container statuses to why.
		if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
			apiPodStatus.Phase = v1.PodPending
		}
		apiPodStatus.Reason = runnable.Reason
		apiPodStatus.Message = runnable.Message
		// Waiting containers are not creating.
		const waitingReason = "Blocked"
		for _, cs := range apiPodStatus.InitContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
		for _, cs := range apiPodStatus.ContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
	}

	// Record the time it takes for the pod to become running
	// since kubelet first saw the pod if firstSeenTime is set.
	existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
		!firstSeenTime.IsZero() {
		metrics.PodStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
	}

	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// Pods that are not runnable must be stopped - return a typed error to the pod worker
	if !runnable.Admit {
		klog.V(2).InfoS("Pod is not runnable and must have running containers stopped", "pod", klog.KObj(pod), "podUID", pod.UID, "message", runnable.Message)
		var syncErr error
		p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err := kl.killPod(ctx, pod, p, nil); err != nil {
			if !wait.Interrupted(err) {
				kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
				syncErr = fmt.Errorf("error killing pod: %w", err)
				utilruntime.HandleError(syncErr)
			}
		} else {
			// There was no error killing the pod, but the pod cannot be run.
			// Return an error to signal that the sync loop should back off.
			syncErr = fmt.Errorf("pod cannot be run: %v", runnable.Message)
		}
		return false, syncErr
	}

	// If the network plugin is not ready, only start the pod if it uses the host network
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
		return false, fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	// ensure the kubelet knows about referenced secrets or configmaps used by the pod
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		if kl.secretManager != nil {
			kl.secretManager.RegisterPod(pod)
		}
		if kl.configMapManager != nil {
			kl.configMapManager.RegisterPod(pod)
		}
	}

	// Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	pcm := kl.containerManager.NewPodContainerManager()
	// If pod has already been terminated then we need not create
	// or update the pod's cgroup
	// TODO: once context cancellation is added this check can be removed
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// When the kubelet is restarted with the cgroups-per-qos
		// flag enabled, all the pod's running containers
		// should be killed intermittently and brought back up
		// under the qos cgroup hierarchy.
		// Check if this is the pod's first sync
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		// Don't kill containers in pod if pod's cgroups already
		// exists or the pod is running for the first time
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
			if err := kl.killPod(ctx, pod, p, nil); err == nil {
				if wait.Interrupted(err) {
					return false, err
				}
				podKilled = true
			} else {
				klog.ErrorS(err, "KillPod failed", "pod", klog.KObj(pod), "podStatus", podStatus)
			}
		}
		// Create and Update pod's Cgroups
		// Don't create cgroups for run once pod if it was killed above
		// The current policy is not to restart the run once pods when
		// the kubelet is restarted with the new flag as run once pods are
		// expected to run only once and if the kubelet is restarted then
		// they are not expected to run again.
		// We don't create and apply updates to cgroup if its a run once pod and was killed above
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					klog.V(2).InfoS("Failed to update QoS cgroups while syncing pod", "pod", klog.KObj(pod), "err", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
					return false, fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	if kubetypes.IsStaticPod(pod) {
		deleted := false
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kubepod.IsMirrorPodOf(mirrorPod, pod) {
				// The mirror pod is semantically different from the static pod. Remove
				// it. The mirror pod will get recreated later.
				klog.InfoS("Trying to delete pod", "pod", klog.KObj(pod), "podUID", mirrorPod.ObjectMeta.UID)
				podFullName := kubecontainer.GetPodFullName(pod)
				var err error
				deleted, err = kl.mirrorPodClient.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
				if deleted {
					klog.InfoS("Deleted mirror pod because it is outdated", "pod", klog.KObj(mirrorPod))
				} else if err != nil {
					klog.ErrorS(err, "Failed deleting mirror pod", "pod", klog.KObj(mirrorPod))
				}
			}
		}
		if mirrorPod == nil || deleted {
			node, err := kl.GetNode()
			if err != nil || node.DeletionTimestamp != nil {
				klog.V(4).InfoS("No need to create a mirror pod, since node has been removed from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else {
				klog.V(4).InfoS("Creating a mirror pod for static pod", "pod", klog.KObj(pod))
				if err := kl.mirrorPodClient.CreateMirrorPod(pod); err != nil {
					klog.ErrorS(err, "Failed creating a mirror pod for", "pod", klog.KObj(pod))
				}
			}
		}
	}

	// Make data directories for the pod
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.ErrorS(err, "Unable to make pod data directories for pod", "pod", klog.KObj(pod))
		return false, err
	}

	// Wait for volumes to attach/mount
	if err := kl.volumeManager.WaitForAttachAndMount(ctx, pod); err != nil {
		if !wait.Interrupted(err) {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.ErrorS(err, "Unable to attach or mount volumes for pod; skipping pod", "pod", klog.KObj(pod))
		}
		return false, err
	}

	// Fetch the pull secrets for the pod
	pullSecrets := kl.getPullSecretsForPod(pod)

	// Ensure the pod is being probed
	kl.probeManager.AddPod(pod)

	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		// Handle pod resize here instead of doing it in HandlePodUpdates because
		// this conveniently retries any Deferred resize requests
		// TODO(vinaykul,InPlacePodVerticalScaling): Investigate doing this in HandlePodUpdates + periodic SyncLoop scan
		//     See: https://github.com/kubernetes/kubernetes/pull/102884#discussion_r663160060
		if kl.podWorkers.CouldHaveRunningContainers(pod.UID) && !kubetypes.IsStaticPod(pod) {
			pod = kl.handlePodResourcesResize(pod)
		}
	}

	// TODO(#113606): connect this with the incoming context parameter, which comes from the pod worker.
	// Currently, using that context causes test failures. To remove this todoCtx, any wait.Interrupted
	// errors need to be filtered from result and bypass the reasonCache - cancelling the context for
	// SyncPod is a known and deliberate error, not a generic error.
	todoCtx := context.TODO()
	// Call the container runtime's SyncPod callback
	result := kl.containerRuntime.SyncPod(todoCtx, pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		// Do not return error if the only failures were pods in backoff
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				// Do not record an event here, as we keep all event logging for sync pod failures
				// local to container runtime, so we get better errors.
				return false, err
			}
		}

		return false, nil
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && isPodResizeInProgress(pod, &apiPodStatus) {
		// While resize is in progress, periodically call PLEG to update pod cache
		runningPod := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err, _ := kl.pleg.UpdateCache(&runningPod, pod.UID); err != nil {
			klog.ErrorS(err, "Failed to update pod cache", "pod", klog.KObj(pod))
			return false, err
		}
	}

	return false, nil
}

```
//   - If the pod is being created, record pod worker start latency
//   - Call generateAPIPodStatus to prepare an v1.PodStatus for the pod
//   - If the pod is being seen as running for the first time, record pod
//     start latency
//   - Update the status of the pod in the status manager
//   - Stop the pod's containers if it should not be running due to soft
//     admission
//   - Ensure any background tracking for a runnable pod is started
//   - Create a mirror pod if the pod is a static pod, and does not
//     already have a mirror pod
//   - Create the data directories for the pod if they do not exist
//   - Wait for volumes to attach/mount
//   - Fetch the pull secrets for the pod
//   - Call the container runtime's SyncPod callback
//   - Update the traffic shaping for the pod's ingress and egress limits
//

1. 调用`generateAPIPodStatus`为Pod生成一个`v1.PodStatus`结构体，并把信息更新到缓存中。
2. 调用`kl.canRunPod(pod)`对 pod 进行软准入校验，检查扩展资源、节点是否处于正在关闭、security context 白名单、驱逐、拓扑等信息，如果检查不通过则终止掉 pod 。
3. 创建和更新Pod的`Cgroups`
4. 判断是否是Static Pod，且 mirror pod 不存在，则创建 mirror pod 。
5. 如果 pod 的相关数据目录不存在，则为 pod 创建相关数据目录。
6. 等待 volumes 挂载/卸载，拉取 secrets 或 configmap ，获取镜像拉取`pullSecrets`。
7. 调用容器运行时的 `kl.containerRuntime.SyncPod()` 接口创建容器。

## p.podSyncer.SyncTerminatedPod()
`SyncTerminatedPod`负责清除已经处于terminated 状态的pod的相关资源，其代码实现如下：
```golang
func (kl *Kubelet) SyncTerminatedPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus) error {
	ctx, otelSpan := kl.tracer.Start(ctx, "syncTerminatedPod", trace.WithAttributes(
		attribute.String("k8s.pod.uid", string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		attribute.String("k8s.pod.name", pod.Name),
		attribute.String("k8s.namespace.name", pod.Namespace),
	))
	defer otelSpan.End()
	klog.V(4).InfoS("SyncTerminatedPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatedPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// generate the final status of the pod
	// TODO: should we simply fold this into TerminatePod? that would give a single pod update
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, true)

	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// volumes are unmounted after the pod worker reports ShouldPodRuntimeBeRemoved (which is satisfied
	// before syncTerminatedPod is invoked)
	if err := kl.volumeManager.WaitForUnmount(ctx, pod); err != nil {
		return err
	}
	klog.V(4).InfoS("Pod termination unmounted volumes", "pod", klog.KObj(pod), "podUID", pod.UID)

	if !kl.keepTerminatedPodVolumes {
		// This waiting loop relies on the background cleanup which starts after pod workers respond
		// true for ShouldPodRuntimeBeRemoved, which happens after `SyncTerminatingPod` is completed.
		if err := wait.PollUntilContextCancel(ctx, 100*time.Millisecond, true, func(ctx context.Context) (bool, error) {
			volumesExist := kl.podVolumesExist(pod.UID)
			if volumesExist {
				klog.V(3).InfoS("Pod is terminated, but some volumes have not been cleaned up", "pod", klog.KObj(pod), "podUID", pod.UID)
			}
			return !volumesExist, nil
		}); err != nil {
			return err
		}
		klog.V(3).InfoS("Pod termination cleaned up volume paths", "pod", klog.KObj(pod), "podUID", pod.UID)
	}

	// After volume unmount is complete, let the secret and configmap managers know we're done with this pod
	if kl.secretManager != nil {
		kl.secretManager.UnregisterPod(pod)
	}
	if kl.configMapManager != nil {
		kl.configMapManager.UnregisterPod(pod)
	}

	// Note: we leave pod containers to be reclaimed in the background since dockershim requires the
	// container for retrieving logs and we want to make sure logs are available until the pod is
	// physically deleted.

	// remove any cgroups in the hierarchy for pods that are no longer running.
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		name, _ := pcm.GetPodContainerName(pod)
		if err := pcm.Destroy(name); err != nil {
			return err
		}
		klog.V(4).InfoS("Pod termination removed cgroups", "pod", klog.KObj(pod), "podUID", pod.UID)
	}

	kl.usernsManager.Release(pod.UID)

	// mark the final pod status
	kl.statusManager.TerminatePod(pod)
	klog.V(4).InfoS("Pod is terminated and will need no more status updates", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```

1. 卸载存储卷。
2. 注销 secrets 和 configmap 。
3. 移除 cgroup 资源限制，回收资源。
4. 移除分配的user namespace
5. 标注 pod 的最终状态。


### p.podSyncer.SyncTerminatingPod
`SyncTerminatingPod`负责停止Pod内所有运行的容器，容器停止后，Pod的状态被置为`Terminated`，之后可以进行其余资源的清理。其代码实现为：
```golang
func (kl *Kubelet) SyncTerminatingPod(_ context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, gracePeriod *int64, podStatusFn func(*v1.PodStatus)) error {
	// TODO(#113606): connect this with the incoming context parameter, which comes from the pod worker.
	// Currently, using that context causes test failures.
	ctx, otelSpan := kl.tracer.Start(context.Background(), "syncTerminatingPod", trace.WithAttributes(
		attribute.String("k8s.pod.uid", string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		attribute.String("k8s.pod.name", pod.Name),
		attribute.String("k8s.namespace.name", pod.Namespace),
	))
	defer otelSpan.End()
	klog.V(4).InfoS("SyncTerminatingPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatingPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
	if podStatusFn != nil {
		podStatusFn(&apiPodStatus)
	}
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	if gracePeriod != nil {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", *gracePeriod)
	} else {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", nil)
	}

	kl.probeManager.StopLivenessAndStartup(pod)

	p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
	if err := kl.killPod(ctx, pod, p, gracePeriod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
		// there was an error killing the pod, so we return that error directly
		utilruntime.HandleError(err)
		return err
	}

	// Once the containers are stopped, we can stop probing for liveness and readiness.
	// TODO: once a pod is terminal, certain probes (liveness exec) could be stopped immediately after
	//   the detection of a container shutdown or (for readiness) after the first failure. Tracked as
	//   https://github.com/kubernetes/kubernetes/issues/107894 although may not be worth optimizing.
	kl.probeManager.RemovePod(pod)

	// Guard against consistency issues in KillPod implementations by checking that there are no
	// running containers. This method is invoked infrequently so this is effectively free and can
	// catch race conditions introduced by callers updating pod status out of order.
	// TODO: have KillPod return the terminal status of stopped containers and write that into the
	//  cache immediately
	podStatus, err := kl.containerRuntime.GetPodStatus(ctx, pod.UID, pod.Name, pod.Namespace)
	if err != nil {
		klog.ErrorS(err, "Unable to read pod status prior to final pod termination", "pod", klog.KObj(pod), "podUID", pod.UID)
		return err
	}
	var runningContainers []string
	type container struct {
		Name       string
		State      string
		ExitCode   int
		FinishedAt string
	}
	var containers []container
	klogV := klog.V(4)
	klogVEnabled := klogV.Enabled()
	for _, s := range podStatus.ContainerStatuses {
		if s.State == kubecontainer.ContainerStateRunning {
			runningContainers = append(runningContainers, s.ID.String())
		}
		if klogVEnabled {
			containers = append(containers, container{Name: s.Name, State: string(s.State), ExitCode: s.ExitCode, FinishedAt: s.FinishedAt.UTC().Format(time.RFC3339Nano)})
		}
	}
	if klogVEnabled {
		sort.Slice(containers, func(i, j int) bool { return containers[i].Name < containers[j].Name })
		klog.V(4).InfoS("Post-termination container state", "pod", klog.KObj(pod), "podUID", pod.UID, "containers", containers)
	}
	if len(runningContainers) > 0 {
		return fmt.Errorf("detected running containers after a successful KillPod, CRI violation: %v", runningContainers)
	}

	// NOTE: resources must be unprepared AFTER all containers have stopped
	// and BEFORE the pod status is changed on the API server
	// to avoid race conditions with the resource deallocation code in kubernetes core.
	if utilfeature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) {
		if err := kl.UnprepareDynamicResources(pod); err != nil {
			return err
		}
	}

	// Compute and update the status in cache once the pods are no longer running.
	// The computation is done here to ensure the pod status used for it contains
	// information about the container end states (including exit codes) - when
	// SyncTerminatedPod is called the containers may already be removed.
	apiPodStatus = kl.generateAPIPodStatus(pod, podStatus, true)
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// we have successfully stopped all containers, the pod is terminating, our status is "done"
	klog.V(4).InfoS("Pod termination stopped all running containers", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```

1. 调用 `kl.killPod()` 容器运行时终止该 pod 上的所有容器，该接口会在容器完全终止前阻塞住。
2. 停止 `liveness` 和 `startup` 探针。
3. 更新 pod 状态到缓存中，检查是否所有容器都已经终止。
4. 如果收到错误返回，则会在下一次循环继续。

### p.podSyncer.SyncTerminatingRuntimePod()
`SyncTerminatingRuntimePod`负责清理找不到配置的`RuntimePod`的Pod的容器，其代码实现为：
```golang
func (kl *Kubelet) SyncTerminatingRuntimePod(_ context.Context, runningPod *kubecontainer.Pod) error {
	// TODO(#113606): connect this with the incoming context parameter, which comes from the pod worker.
	// Currently, using that context causes test failures.
	ctx := context.Background()
	pod := runningPod.ToAPIPod()
	klog.V(4).InfoS("SyncTerminatingRuntimePod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatingRuntimePod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// we kill the pod directly since we have lost all other information about the pod.
	klog.V(4).InfoS("Orphaned running pod terminating without grace period", "pod", klog.KObj(pod), "podUID", pod.UID)
	// TODO: this should probably be zero, to bypass any waiting (needs fixes in container runtime)
	gracePeriod := int64(1)
	if err := kl.killPod(ctx, pod, *runningPod, &gracePeriod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
		// there was an error killing the pod, so we return that error directly
		utilruntime.HandleError(err)
		return err
	}
	klog.V(4).InfoS("Pod termination stopped all running orphaned containers", "pod", klog.KObj(pod), "podUID", pod.UID)
	return nil
}
```
确保 kubelet 识别到它已经没有任何容器运行了。

### p.completeTerminated(podUID)

这一步是终止 podWorker ，关停 PodUpdates channel 通道，并将 pod 从缓存中移除。
```golang
func (p *podWorkers) completeTerminated(podUID types.UID) {
	p.podLock.Lock()
	defer p.podLock.Unlock()

	klog.V(4).InfoS("Pod is complete and the worker can now stop", "podUID", podUID)

	p.cleanupPodUpdates(podUID)

	status, ok := p.podSyncStatuses[podUID]
	if !ok {
		return
	}
	if status.terminatingAt.IsZero() {
		klog.V(4).InfoS("Pod worker is complete but did not have terminatingAt set, likely programmer error", "podUID", podUID)
	}
	if status.terminatedAt.IsZero() {
		klog.V(4).InfoS("Pod worker is complete but did not have terminatedAt set, likely programmer error", "podUID", podUID)
	}
	status.finished = true
	status.working = false

	if p.startedStaticPodsByFullname[status.fullname] == podUID {
		delete(p.startedStaticPodsByFullname, status.fullname)
	}
}
```

### p.completeTerminating(podUID)
这一意味着 pod 上的容器已经全部终止，且容器不会在后面的同步逻辑中继续启动了，更新 pod 相关状态，准备被后续的清理逻辑接管，并确保不被后面的同步逻辑同步，确保能被 kubelet 始别到它已经没有任何容器运行了
```golang
func (p *podWorkers) completeTerminating(podUID types.UID) {
	p.podLock.Lock()
	defer p.podLock.Unlock()

	klog.V(4).InfoS("Pod terminated all containers successfully", "podUID", podUID)

	status, ok := p.podSyncStatuses[podUID]
	if !ok {
		return
	}

	// update the status of the pod
	if status.terminatingAt.IsZero() {
		klog.V(4).InfoS("Pod worker was terminated but did not have terminatingAt set, likely programmer error", "podUID", podUID)
	}
	status.terminatedAt = p.clock.Now()
	for _, ch := range status.notifyPostTerminating {
		close(ch)
	}
	status.notifyPostTerminating = nil
	status.statusPostTerminating = nil

	// the pod has now transitioned to terminated and we want to run syncTerminatedPod
	// as soon as possible, so if no update is already waiting queue a synthetic update
	p.requeueLastPodUpdate(podUID, status)
}
```

### p.completeSync(podUID)

这一步是处理在自然的生命周期内，pod 被终止了。类似驱逐、api驱使删除等非自然生命周期的处理，是走 UpdatePod() 这个接口的。
```golang
func (p *podWorkers) completeSync(podUID types.UID) {
	p.podLock.Lock()
	defer p.podLock.Unlock()

	klog.V(4).InfoS("Pod indicated lifecycle completed naturally and should now terminate", "podUID", podUID)

	status, ok := p.podSyncStatuses[podUID]
	if !ok {
		klog.V(4).InfoS("Pod had no status in completeSync, programmer error?", "podUID", podUID)
		return
	}

	// update the status of the pod
	if status.terminatingAt.IsZero() {
		status.terminatingAt = p.clock.Now()
	} else {
		klog.V(4).InfoS("Pod worker attempted to set terminatingAt twice, likely programmer error", "podUID", podUID)
	}
	status.startedTerminating = true

	// the pod has now transitioned to terminating and we want to run syncTerminatingPod
	// as soon as possible, so if no update is already waiting queue a synthetic update
	p.requeueLastPodUpdate(podUID, status)
}
```

### p.completeWork(podUID, phaseTransition, err)
当 pod 同步出现错误时调用该接口，接口逻辑是检查错误类型，根据错误类型将 pod 立即或者按一定时间间隔重新加入到同步队列，直到 pod 完成同步

```golang
func (p *podWorkers) completeWork(podUID types.UID, phaseTransition bool, syncErr error) {
	// Requeue the last update if the last sync returned error.
	switch {
	case phaseTransition:
		p.workQueue.Enqueue(podUID, 0)
	case syncErr == nil:
		// No error; requeue at the regular resync interval.
		p.workQueue.Enqueue(podUID, wait.Jitter(p.resyncInterval, workerResyncIntervalJitterFactor))
	case strings.Contains(syncErr.Error(), NetworkNotReadyErrorMsg):
		// Network is not ready; back off for short period of time and retry as network might be ready soon.
		p.workQueue.Enqueue(podUID, wait.Jitter(backOffOnTransientErrorPeriod, workerBackOffPeriodJitterFactor))
	default:
		// Error occurred during the sync; back off and then retry.
		p.workQueue.Enqueue(podUID, wait.Jitter(p.backOffPeriod, workerBackOffPeriodJitterFactor))
	}

	// if there is a pending update for this worker, requeue immediately, otherwise
	// clear working status
	p.podLock.Lock()
	defer p.podLock.Unlock()
	if status, ok := p.podSyncStatuses[podUID]; ok {
		if status.pendingUpdate != nil {
			select {
			case p.podUpdates[podUID] <- struct{}{}:
				klog.V(4).InfoS("Requeueing pod due to pending update", "podUID", podUID)
			default:
				klog.V(4).InfoS("Pending update already queued", "podUID", podUID)
			}
		} else {
			status.working = false
		}
	}
}

```