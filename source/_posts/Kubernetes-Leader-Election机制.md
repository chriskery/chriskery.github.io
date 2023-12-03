---
title: Kubernetes Leader Election机制详解
date: 2023-12-02 23:35:14
tags:
tags:
- kubernetes
- leader election
categories:
- kubernetes
---
## 背景
分布式应用通常会创建多个service的副本以提高service的可靠性和可伸缩性，但通常有必要指定一个副本负责所有副本之间的协调，kubernetes的SIG已经提供了这方面的能力，主要是通过configmap/lease/endpoint的资源实现选Leader的功能。通常在`Leader Election`的选举过程中，会有一组确定的candidates（候选人）都竞相宣称自己是Leader，其中会有一个candidates选举成功。一旦选举获胜，Leader就会按照固定的间隔不断的发送“心跳”以维持其领导人的地位(在稳定的环境中，实例一旦成为了leader，通常情况是不会释放锁的，会保持一直运行的状态，这样有利于业务的稳定和Controller快速的对资源的状态变化做成相应的操作)。其他竞选失败的candidates会周期性地进行新的尝试，以确保当当前的Leader因为某种原因而失败，新的Leader能够被迅速的确定。

为了执行Kubernetes内部的Leader Election，需要用到Kubernetes API对象的两个属性：
- ResourceVersions - Every API object has a unique ResourceVersion, and you can use these versions to perform compare-and-swap on Kubernetes objects
- Annotations - Every API object can be annotated with arbitrary key/value pairs to be used by clients.

## 启动选举
Kubernetes提供了[Endpoints、ConfigMap 和 Lease](https://github.com/kubernetes-retired/kube-batch/blob/1ebe60e4af4f164cab0818bc1530e541ed7aa1a5/vendor/k8s.io/client-go/tools/leaderelection/resourcelock/interface.go)三种资源锁，Leader Election 选主的实现方式就是基于这三种资源锁。
```golang
const (
	EndpointsResourceLock             = "endpoints"
	ConfigMapsResourceLock            = "configmaps"
	LeasesResourceLock                = "leases"
)
```
开始选举时需要通过那么在开始选举先需要`resourcelock.New`先通过方法 resourcelock.New 获取资源锁的对象。以kube-batch为例：

```golang
	rl, err := resourcelock.New(resourcelock.ConfigMapsResourceLock,
		opt.LockObjectNamespace,
		"kube-batch",
		leaderElectionClient.CoreV1(),
		leaderElectionClient.CoordinationV1(),
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: eventRecorder,
		})
```
也可以通过`resourcelock.NewFromKubeconfig`的方法创建资源锁，该方法对 resourcelock.New 进行了封装。以control manager为例：
```golang
rl, err := resourcelock.NewFromKubeconfig(resourceLock,  
   c.ComponentConfig.Generic.LeaderElection.ResourceNamespace,  
   leaseName,  
   resourcelock.ResourceLockConfig{  
      Identity:      lockIdentity,  
      EventRecorder: c.EventRecorder,  
   },  
   c.Kubeconfig,  
   c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration)
```
创建好资源锁后，通过`leaderelection.RunOrDie`启动选举：
```golang
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
```
其中LeaseDuration为租约时长，非leader的candidate等待LeaseDuration后会强制重新选举；RenewDeadline为leader刷新资源锁超时时间；RetryPeriod为调用资源锁间隔。Callbacks的定义如下：
```golang
type LeaderCallbacks struct {
	// OnStartedLeading is called when a LeaderElector client starts leading
	OnStartedLeading func(context.Context)
	// OnStoppedLeading is called when a LeaderElector client stops leading
	OnStoppedLeading func()
	// OnNewLeader is called when the client observes a leader that is
	// not the previously observed leader. This includes the first observed
	// leader when the client starts.
	OnNewLeader func(identity string)
}
```

## 选举主流程
启动选举后，RunOrDie 方法会调用 le.Run(ctx) 方法开始真正的选举流程，le.Run的接口实现为：
```golang
func (le *LeaderElector) Run(ctx context.Context) {
  ......  
  // 竞选获取锁，如果没有获取到，就一直等待
  if !le.acquire(ctx) {
    return // ctx signalled done
  }
  ctx, cancel := context.WithCancel(ctx)
  defer cancel()
  // 获取到锁后，需要调用回调函数中的OnStartedLeading，运行controller的代码
  go le.config.Callbacks.OnStartedLeading(ctx)
  
  // 获取到锁后，需要不断地进行renew操作
  le.renew(ctx)
}
```
### 竞选
`le.acquire`竞选获取锁，如果没有获取到(即没有成为Leader)，则会一直等待。其实现为：
```golang
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	// wait.JitterUntil间隔 RetryPeriod 执行一次
	wait.JitterUntil(func() {
		// 竞选
		succeeded = le.tryAcquireOrRenew(ctx)
		// leader 变化回调
		le.maybeReportTransition()
		// 竞选失败返回
		if !succeeded {
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		// 成功则退出竞选函数
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}
```
`wait.JitterUntil`间隔`RetryPeriod`执行一次, 若获取锁成功，则会调用ctx的cancel方法中止 wait.JitterUntil 循环。返回`succeeded`。
所以 acquire 函数只会有两种情况会返回：
- 当选 leader；
- ctx 被取消（外部要求中止选举流程）

### 抢锁
candidate抢锁关键的实现在于tryAcquireOrRenew，而tryAcquireOrRenew就是依赖锁的状态转移机制完成核心逻辑。其实现为：
```gonlang
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
  now := metav1.Now()
  leaderElectionRecord := rl.LeaderElectionRecord{
    HolderIdentity:       le.config.Lock.Identity(),
    LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
    RenewTime:            now,
    AcquireTime:          now,
  }
​
  // 1. obtain or create the ElectionRecord
  // 检查锁有没有
  oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
  if err != nil {
    // 没有锁的资源，就创建一个
    if !errors.IsNotFound(err) {
      klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
      return false
    }
    if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
      klog.Errorf("error initially creating leader election record: %v", err)
      return false
    }
    //对外宣称自己成为了leader
    le.setObservedRecord(&leaderElectionRecord)
​
    return true
  }
​
  // 2. Record obtained, check the Identity & Time
  if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
    // 这个机制很重要，会如果leader会不断正常renew这个锁，oldLeaderElectionRawRecord会一直发生变化，发生变化会更新le.observedTime
    le.setObservedRecord(oldLeaderElectionRecord)
    le.observedRawRecord = oldLeaderElectionRawRecord
  }
  
  // 如果还没超时并且此实例不是leader（leader是其他实例），那么就直接退出
  if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
    le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
    !le.IsLeader() {
    klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
    return false
  }
​
  // 3. We're going to try to update. The leaderElectionRecord is set to it's default
  // here. Let's correct it before updating.
  // 如果是leader，就更新时间RenewTime，保证其他实例（非主）可以观察到：主还活着
  if le.IsLeader() {
    leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
  } else {
  // 不是leader，那么锁就发生了转移
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
  }
  
  // 更新锁
  // update the lock itself
  if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
    klog.Errorf("Failed to update lock: %v", err)
    return false
  }
​
  le.setObservedRecord(&leaderElectionRecord)
  return true
}
```
`tryAcquireOrRenew`的流程
-   获取ElectionRecord对象。
    -   若ElectionRecord对象不存在，则创建新的ElectionRecord对象，创建成功后宣称成为Leader。
    -   若ElectionRecord对象存在，则将获取的oldLeaderElectionRecord更新到该le(LeaderElector)的observedRecord缓存中。
        -   若observedRecord不是该le创建且未过期，获取锁失败，退出。
        -   否则调用` le.config.Lock.Update(leaderElectionRecord)`使用该le(LeaderElector)的信息尝试更新 ElectionRecord对象，更新成功后宣称成为Leader。
        
#### 选举原理
`tryAcquireOrRenew`的代码的核心逻辑主要为更新`le.observedTime`,更新的时机为：
- 锁（ElectionRecord对象）不存在，创建锁成功时更新；
- 获取到锁记录和缓存的不同，说明上次尝试获取锁到现在的间隔内 leader 变化了，更新缓存；
- leader 超期没续期且当前节点抢锁成功，更新缓存。

如果锁对象不存在或者leader在任期时间内未更新锁对象，则该le会执行抢锁的逻辑，尝试成为新的leader。

## Leader续期
在获取到锁成为 leader 后，会进入 le.renew(ctx) 方法进行定期续期操作。其实现如下：
```golang
func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	// 定期续期，每 RetryPeriod 执行一次续期
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
		// 执行续期，直到成功或超时
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			return le.tryAcquireOrRenew(timeoutCtx), nil
		}, timeoutCtx.Done())

		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		// 若renew成功，则Until会不断循环
		if err == nil {
			klog.V(5).Infof("successfully renewed lease %v", desc)
			return
		}
		// 若renew失败，引发超时错误，说明续期失败，退出续期流程
		le.config.Lock.RecordEvent("stopped leading")
		le.metrics.leaderOff(le.config.Name)
		klog.Infof("failed to renew lease %v: %v", desc, err)
		cancel()
	}, le.config.RetryPeriod, ctx.Done())

	// if we hold the lease, give it up
	if le.config.ReleaseOnCancel {
		le.release()
	}
}
```
renew 方法如果在任期内续期失败则会退出续期循环，主流程结束，其它candidate会尝试成为新的leader。旧的leader若想重新参与竞选，则需要再次调用`leaderelection.RunOrDie`, Kubernetes的组件一般直接退出，通过重启Pod实现重新参与选举。
