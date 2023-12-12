---
title: kube-batch详解
date: 2023-12-03 01:17:01
tags:
- kubernetes
- kube-batch
categories:
- kubernetes
---
## 背景
Kubernetes提供了默认的kube scheduler调度器将未调度的 Pod 按照调度策略绑定到合适的 Node，其调度的流程如下图所示：
![kube scheduler调度流程](https://raw.githubusercontent.com/kubernetes/enhancements/master/keps/sig-scheduling/624-scheduling-framework/scheduling-framework-extensions.png)
kube scheduler调度主要分为以下几个部分：
- 预选过程:过滤掉不满足条件的节点，这个过程称为Predicates
- 优选过程:对通过的节点按照优先级排序，称之为Priorities

最后从中选择优先级最高的节点，将Pod调度到此节点上。


从上面的调度流程可以看到，Kube Scheduler 是基于单个Pod的调度，Kube Scheduler的调度目标是给单个Pod找到最优的节点，这个特性对于需要涉及到多个Pod的作业来说（分布式AI训练/大数据）是不够友好的。以AI为例，分布式训练需要所有 worker 都启动后，训练才能够开始进行。使用原生调度器，可能会出现以下问题：
- 一个任务包含 10 个 worker, 但是集群剩余资源只满足 9 个 worker。原生调度器会将任务的 9 个 worker 调度并启动，而最后一个 worker 一直无法启动。这样训练一直无法开始，9 个已经启动的 worker 资源就会被浪费。
- 两个任务，各包含 10 个 worker, 集群资源只能启动 10 个 worker。两个任务分别有 5 个 worker 被启动，但两个任务都无法开始训练。10 个 worker 的资源被浪费了。

Kubernetes社区提供了[kube batch](https://github.com/kubernetes-retired/kube-batch/edit/master/doc/usage/tutorial.md)调度器，帮助 Kubernetes弥补在“高性能计算”方面的不足，帮助Kubernetes构建统一的容器平台。

## 架构
![kube batch架构](https://pic3.zhimg.com/80/v2-057fae99cb1a1a620b98bb4b96f0127e_1440w.webp)

可以看到kube batch提供的两个核心能力为：
- 提供一个资源调度的中间层,为计算类作业提供了相应的调度算法支持
- 多租户之间的隔离

Kube Batch 内部包含如下几个模块：
- Cache，缓存 Kubernetes 集群内的 Node、Pod、PodGroup、Queue 等资源对象信息。
- Session，该模块用于管理 Kube Batch 的调度过程，每次调度称为一次 Session 过程。
- Action，一次调度包括多个 Action 操作，Action根据配置的Plugins内容将作业调度到具体的节点上。
- Plugin，用于实现不同调度策略，例如 DRF、Gang Scheduling。

当前kube batch支持的actions和plugins包括：
```golang
actions: "allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
- plugins:
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder
```

Kube Batch默认启动的配置包括allocate和backfill两个action，
```golang
var defaultSchedulerConf = `
actions: "allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
- plugins:
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder`
```
## 具体实现
```golang
func main() {
	......
	if err := app.Run(s); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}

// Run the kubeBatch scheduler
func Run(opt *options.ServerOption) error {
	// Start policy controller to allocate resources.
	sched, err := scheduler.NewScheduler(config,
		opt.SchedulerName,
		opt.SchedulerConf,
		opt.SchedulePeriod,
		opt.DefaultQueue)
	if err != nil {
		panic(err)
	}

	run := func(ctx context.Context) {
		sched.Run(ctx.Done())
		<-ctx.Done()
	}

	if !opt.EnableLeaderElection {
		run(context.TODO())
		return fmt.Errorf("finished without leader elect")
	}

	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: leaseDuration,
		RenewDeadline: renewDeadline,
		RetryPeriod:   retryPeriod,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				glog.Fatalf("leaderelection lost")
			},
		},
	})
	return fmt.Errorf("lost lease")
}
```
kube batch的[启动流程](https://github.com/kubernetes-retired/kube-batch/blob/1ebe60e4af4f164cab0818bc1530e541ed7aa1a5/cmd/kube-batch/app/server.go)：
1. 判断是否需要进行leader election，若不需要，调用sched.Run(ctx.Done())启动调度
2. 若需要，先进行leader election，leader再调用sched.Run(ctx.Done())启动调度

```golang
func (pc *Scheduler) Run(stopCh <-chan struct{}) {
     // Start cache for policy.
    go pc.cache.Run(stopCh)
    ......
    pc.actions, pc.pluginArgs = loadSchedulerConf(conf)

    go wait.Until(pc.runOnce, 1*time.Second, stopCh)
}

func (pc *Scheduler) runOnce() {
    ssn := framework.OpenSession(pc.cache, pc.pluginArgs)
    defer framework.CloseSession(ssn)

    for _, action := range pc.actions {
        action.Execute(ssn)
    }
}
```

sched的调度过程如上：
1. 调用cache模块，通过Informer同步集群内Pod/Node等相关信息。
2. 加载配置文件
3. 每隔`1*time.Second`调用一次`sched.runOnce()`，执行调度
4. `runOnce`依次执行OpenSession->action.Execute(ssn)->CloseSession()

### cache
cache的具体实现为：
```golang
// Run  starts the schedulerCache
func (sc *SchedulerCache) Run(stopCh <-chan struct{}) {
	go sc.pdbInformer.Informer().Run(stopCh)
	go sc.podInformer.Informer().Run(stopCh)
	go sc.nodeInformer.Informer().Run(stopCh)
	go sc.podGroupInformerv1alpha1.Informer().Run(stopCh)
	go sc.podGroupInformerv1alpha2.Informer().Run(stopCh)
	go sc.pvInformer.Informer().Run(stopCh)
	go sc.pvcInformer.Informer().Run(stopCh)
	go sc.scInformer.Informer().Run(stopCh)
	go sc.queueInformerv1alpha1.Informer().Run(stopCh)
	go sc.queueInformerv1alpha2.Informer().Run(stopCh)

	if options.ServerOpts.EnablePriorityClass {
		go sc.pcInformer.Informer().Run(stopCh)
	}

	// Re-sync error tasks.
	go wait.Until(sc.processResyncTask, 0, stopCh)

	// Cleanup jobs.
	go wait.Until(sc.processCleanupJob, 0, stopCh)
}
```
除了缓存pod/Node/PV/PVC等相关信息外，kube batch还定义了一系列的CRDs，以实现批量调度和多租户之间的隔离。

### session

```golang
// OpenSession start the session
func OpenSession(cache cache.Cache, tiers []conf.Tier) *Session {
	ssn := openSession(cache)
	
	for _, plugin := range ssn.plugins {
		plugin.OnSessionOpen(ssn)
	}
	return ssn
}


func openSession(cache cache.Cache) *Session {
	ssn := &Session{
		UID:   uuid.NewUUID(),
		cache: cache
	}

	snapshot := cache.Snapshot()

	ssn.Jobs = snapshot.Jobs
	ssn.Nodes = snapshot.Nodes
	ssn.Queues = snapshot.Queues
	return ssn
}
```
OpenSession首先调用了`openSession(cache cache.Cache)`获取到本次调度的session，`openSession(cache cache.Cache)`内部对当前缓存的cache进行了快照，并将快照里的信息保存在了session中。然后调用注册的plugins的`OnSessionOpen`方法通知新一轮的调度开始。

### action

以allocate为例，其实现为：
```golang
func (alloc *allocateAction) Execute(ssn *framework.Session) {
	queues := util.NewPriorityQueue(ssn.QueueOrderFn)
	for _, job := range ssn.Jobs {
		if queue, found := ssn.Queues[job.Queue]; found {
			queues.Push(queue)
		} else {
			glog.Warningf("Skip adding Job <%s/%s> because its queue %s is not found",
				job.Namespace, job.Name, job.Queue)
			continue
		}

	}

	for {
		if queues.Empty() {
			break
		}

		queue := queues.Pop().(*api.QueueInfo)

		jobs, found := jobsMap[queue.UID]

		job := jobs.Pop().(*api.JobInfo)
		
		tasks := pendingTasks[job.UID]

		for !tasks.Empty() {
			task := tasks.Pop().(*api.TaskInfo)
			predicateNodes := util.PredicateNodes(task, allNodes, predicateFn)
			if len(predicateNodes) == 0 {
				// Further Tasks should be checked because tasks are ordered in priority, so it affects taskPriority within Job,
				// so if one task fails predicates, it should not check further tasks in same job, should skip to next job.
				break
			}
			priorityList, err := util.PrioritizeNodes(task, predicateNodes, ssn.NodePrioritizers())
			if err != nil {
				glog.Errorf("Prioritize Nodes for task %s err: %v", task.UID, err)
				break
			}
			nodeName := util.SelectBestNode(priorityList)
            ......
		}
		// Added Queue back until no job in Queue.
		queues.Push(queue)
    }
}
```
可以看出，在单个Session中，Job首先按照Queue进行聚合，然后依次遍历每条队列里的每个Job，将这个Job内部的Task依次执行调度。调度过程也跟默认的Kube Scheduler调度器差不多:`PredicateNodes->PrioritizeNodes->SelectBestNode`。在调用Action之前，OpenSession为配置的Plugins注册了一系列Func，在执行不同Action的时候，使用这些Func对Task进行调度。




