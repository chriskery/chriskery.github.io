---
title: kubernetes-Informer详解
date: 2023-12-11 21:16:37
tags:
- kubernetes
- informer
categories:
- kubernetes
---
## informer简介
在Kubernetes编程中,`informer`是Kubernetes Go client`client-go`官方library提供的一个高级别的抽象，来监听和处理k8s集群的资源变化。通过使用`informer`，开发者能有效的监听资源的变化，而无需通过手工`轮询`的方式。`informer`使用了`API Server`提供的函数接口，我们通过 apiserver 增删改某个 Kubernetes API 对象，该资源对应的控制器中的 Informer 会立即感知到这个事件并进行相应的处理

Informer 由三部分构成：

- Reflector：Informer 通过 Reflector 与 Kubernetes apiserver 建立连接并 ListAndWatch Kubernetes 资源对象的变化，并将此“增量” push 入 DeltaFIFO Queue
- DeltaFIFO Queue：Informer 从该队列中 pop 增量，或创建或更新或删除本地缓存（Local Store）
- Indexer：将增量中的 Kubernetes 资源对象保存到本地缓存中，并为其创建索引，这份缓存与 etcd 中的数据是完全一致的。控制器只从本地缓存通过索引读取数据，这样做减小了 apiserver 和 etcd 的压力。

![](https://pic.crazytaxii.com/21-03-31/22701732.jpg)

## Informer使用
informer的使用分为以下几步：
1. 引入相关依赖
```golang
import (
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
)
```
2. 初始化kubernetes clientset，与API server做交互
```golang
clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
```
3. 使用初始化好的clientset创建对应资源的informer(e.g., Pods, Deployments, etc.).
```golang
informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
podInformer := informerFactory.Core().V1().Pods()
```
4. 添加事件回调
```golang
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        // Handle added pod
    },
    UpdateFunc: func(oldObj, newObj interface{}) {
        // Handle updated pod
    },
    DeleteFunc: func(obj interface{}) {
        // Handle deleted pod
    },
})
```

5.启动informer并等待缓存同步。
```golang
stopCh := make(chan struct{})
defer close(stopCh)

informerFactory.Start(stopCh)
if !cache.WaitForCacheSync(stopCh, podInformer.Informer().HasSynced) {
    panic("Timed out waiting for caches to sync")
}
```

## 源码实现
在上面的代码示例种我们创建了一个`informerFactory`对象并通过调用`informerFactory.Start(stopCh)`初始化所有请求的informer，其内部的实现为：
```golang
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.shuttingDown {
		return
	}

	for informerType, informer := range f.informers {
		if !f.startedInformers[informerType] {
			f.wg.Add(1)
			// We need a new variable in each loop iteration,
			// otherwise the goroutine would use the loop variable
			// and that keeps changing.
			informer := informer
			go func() {
				defer f.wg.Done()
				informer.Run(stopCh)
			}()
			f.startedInformers[informerType] = true
		}
	}
}
```
start调用就是for循环了内部所有informers，如果未启动的话就启动一下。`informer.Run(stopCh)`的实现为：
```golang

func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	if s.HasStarted() {
		klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
		return
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,

			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```
`informer.Run(stopCh)`做了一些变量的初始化，并且调用`s.controller.Run(stopCh)`，其内部实现为：
```golang

// Run begins processing items, and will continue until a value is sent down stopCh or it is closed.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
			ResyncPeriod:    c.config.FullResyncPeriod,
			TypeDescription: c.config.ObjectDescription,
			Clock:           c.clock,
		},
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```
`s.controller.Run(stopCh)`里面做了两件事，
1. `wg.StartWithChannel(stopCh, r.Run)`:构建并运行了一个Reflector将资源对象/事件通知从`c.config.ListerWatcher` 送到 `c.config.Queue`里。并进行不定期的`Resync`操作。
2. `wait.Until(c.processLoop, time.Second, stopCh)`:周期性的从`c.config.Queue`里Pop出对象并且使用`Config's ProcessFunc`进行处理。
  
我们首先来看下`Reflector`的`r.Run`函数实现：
```golang
// Run repeatedly uses the reflector's ListAndWatch to fetch all the
// objects and subsequent deltas.
// Run will exit when stopCh is closed.
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
	klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.typeDescription, r.resyncPeriod, r.name)
}
```
`wait.BackoffUntil`是一个工具函数，其功能可以认为是周期性的调用回调函数（第一个参数)，如果回调函数调用失败，则等待一段时候再次调用回调函数。回调函数内部又调用了`r.ListAndWatch(stopCh)`方法：
```golang

// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	var w watch.Interface
	.....
	defer close(cancelCh)
	go r.startResync(stopCh, cancelCh, resyncerrc)
	return r.watch(w, stopCh, resyncerrc)
}

// watch simply starts a watch request with the server.
func (r *Reflector) watch(w watch.Interface, stopCh <-chan struct{}, resyncerrc chan error) error {
	for {
		if w == nil {
            ......
			w, err = r.listerWatcher.Watch(options)
			if err != nil {
				return err
			}
		}
		err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.typeDescription, r.setLastSyncResourceVersion, nil, r.clock, resyncerrc, stopCh)
		// Ensure that watch will not be reused across iterations.
		w.Stop()
		w = nil
	}
}
```
`r.ListAndWatch(stopCh)`内部调用`r.watch(w, stopCh, resyncerrc)`，`r.watch(w, stopCh, resyncerrc)`首先使用`r.listerWatcher`初始化实现了`watch.Interface`接口的w对象，并调用了`watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.typeDescription, r.setLastSyncResourceVersion, nil, r.clock, resyncerrc, stopCh`对资源对象进行真正的watch.

`watch.Interface`的接口定义为：
```golang
// Interface can be implemented by anything that knows how to watch and report changes.
type Interface interface {
	// Stop stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// ResultChan returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, the implementation will close this channel and
	// release any resources used by the watch.
	ResultChan() <-chan Event
}
```
核心的函数为`ResultChan`可以返回一个`chan Event`, 被监听的资源的所有事件都可以通过这个chan获取到。

`watchHandler`的实现为：

```golang

// watchHandler watches w and sets setLastSyncResourceVersion
func watchHandler(start time.Time,
	w watch.Interface,
	store Store,
	expectedType reflect.Type,
	expectedGVK *schema.GroupVersionKind,
	name string,
	expectedTypeName string,
	setLastSyncResourceVersion func(string),
	exitOnInitialEventsEndBookmark *bool,
	clock clock.Clock,
	errc chan error,
	stopCh <-chan struct{},
) error {
	eventCount := 0
	if exitOnInitialEventsEndBookmark != nil {
		// set it to false just in case somebody
		// made it positive
		*exitOnInitialEventsEndBookmark = false
	}

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrors.FromObject(event.Object)
			}
			if expectedType != nil {
				if e, a := expectedType, reflect.TypeOf(event.Object); e != a {
					utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", name, e, a))
					continue
				}
			}
			if expectedGVK != nil {
				if e, a := *expectedGVK, event.Object.GetObjectKind().GroupVersionKind(); e != a {
					utilruntime.HandleError(fmt.Errorf("%s: expected gvk %v, but watch event object had gvk %v", name, e, a))
					continue
				}
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
				continue
			}
			resourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				err := store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Modified:
				err := store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", name, event.Object, err))
				}
			case watch.Bookmark:
				// A `Bookmark` means watch has synced here, just update the resourceVersion
				if _, ok := meta.GetAnnotations()["k8s.io/initial-events-end"]; ok {
					if exitOnInitialEventsEndBookmark != nil {
						*exitOnInitialEventsEndBookmark = true
					}
				}
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
			}
			setLastSyncResourceVersion(resourceVersion)
			if rvu, ok := store.(ResourceVersionUpdater); ok {
				rvu.UpdateResourceVersion(resourceVersion)
			}
			eventCount++
			if exitOnInitialEventsEndBookmark != nil && *exitOnInitialEventsEndBookmark {
				watchDuration := clock.Since(start)
				klog.V(4).Infof("exiting %v Watch because received the bookmark that marks the end of initial events stream, total %v items received in %v", name, eventCount, watchDuration)
				return nil
			}
		}
	}

	watchDuration := clock.Since(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", name)
	}
	klog.V(4).Infof("%s: Watch close - %v total %v items received", name, expectedTypeName, eventCount)
	return nil
}

```
整个watch就是在不断的从`event, ok := <-w.ResultChan()`并且将其塞进`store`里面，这个store就是我们在上面看到的`NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
			ResyncPeriod:    c.config.FullResyncPeriod,
			TypeDescription: c.config.ObjectDescription,
			Clock:           c.clock,
		},
	)`里的`c.config.Queue`，对应上面informer流程图中的第2步——`AddObject`.

接下来我们看下`w, err = r.listerWatcher.Watch(options)`的listerWatcher是怎么来的。在初始化`Reflector`的时候我们通过`c.config.ListerWatcher`对`r.listerWatcher`进行了初始化，那`c.config.ListerWatcher`又是怎么来的呢？通过上面分析的`func (s *sharedIndexInformer) Run()`函数，我们知道`c.config.ListerWatcher`来自于sharedIndexInformer的`listerWatcher`。在我们的示例代码中，我们使用了`podInformer.Informer()`生成了一个pod的informer对象，其具体实现为：
```golang
func (f *podInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
}

func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}
// NewFilteredPodInformer constructs a new informer for Pod type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}

```
通过k8s的代码的代码生成工具生成的相关代码中，都会有NewFilteredXXXInformer 函数，其中调用了 `NewSharedIndexInformer`，`ListWatch` 对象就是在这里初始化的。

具体的List&Watch的流程为：
1. 首先 r.listerWatcher.List 获取所有资源数据`list`。本质上是调用 Kubernetes API 资源 NewFilteredPodInformer 时初始化 ListWatch 时传入的 ListFunc 函数，通过 ClientSet 客户端与 apiserver 交互并获取 Pod资源列表数据。
2. 调用resourceVersion = listMetaInterface.GetResourceVersion()
获取资源版本号。k8s内的每种资源都有资源版本号。注意，pod和pods是两种资源。当 Kubernetes 资源对象变化时，其资源版本也会发生变化
3. 调用meta.ExtractList 将资源数据`list`转换为资源对象列表`items`。这里就是做一些类型转换。类似于将pods 资源对象转化为pod数组。
4. 调用r.syncWith 将资源对象和资源版本号存储到 DeltaFIFO 队列中。这里是直接调用的`r.store.Replace(found, resourceVersion)`。
5. 调用`r.setLastSyncResourceVersion(resourceVersion)`设置当前的资源版本号。

至此，已经完成了资源对象的List操作，接下来进行资源对象的Watch操作。这里我们之前看到过，就是调用`r.watch()`，内部又调用`w, err = r.listerWatcher.Watch(options)`返回了了一个实现了watch接口的对象`w`，最终会调用到我们通过`NewFilteredPodInformer`生成的`WatchFunc`函数回调。

来看下`client.CoreV1().Pods(namespace).Watch`的函数实现：
```golang
// Watch returns a watch.Interface that watches the requested pods.
func (c *pods) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
    var timeout time.Duration
    if opts.TimeoutSeconds != nil {
        timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
    }
    opts.Watch = true
    return c.client.Get().
        Namespace(c.ns).
        Resource("pods").
        VersionedParams(&opts, scheme.ParameterCodec).
        Timeout(timeout).
        Watch(ctx)
}
```
再来看下 `c.client.Get().
        Namespace(c.ns).
        Resource("pods").
        VersionedParams(&opts, scheme.ParameterCodec).
        Timeout(timeout).
        Watch(ctx)` 的实现:
```golang
// Watch attempts to begin watching the requested location.
// Returns a watch.Interface, or an error.
func (r *Request) Watch(ctx context.Context) (watch.Interface, error) {

    url := r.URL().String()
    req, err := http.NewRequest(r.verb, url, r.body)
    if err != nil {
        return nil, err
    }
    req = req.WithContext(ctx)
    req.Header = r.headers
    client := r.c.Client
    if client == nil {
        client = http.DefaultClient
    }
    r.backoff.Sleep(r.backoff.CalculateBackoff(r.URL()))
    resp, err := client.Do(req)
    // a lot of code here
    if resp.StatusCode != http.StatusOK {
        defer resp.Body.Close()
        if result := r.transformResponse(resp, req); result.err != nil {
            return nil, result.err
        }
        return nil, fmt.Errorf("for request %s, got status: %v", url, resp.StatusCode)
    }

    contentType := resp.Header.Get("Content-Type")
    mediaType, params, err := mime.ParseMediaType(contentType)
    if err != nil {
        klog.V(4).Infof("Unexpected content type from the server: %q: %v", contentType, err)
    }
    objectDecoder, streamingSerializer, framer, err := r.c.content.Negotiator.StreamDecoder(mediaType, params)
    if err != nil {
        return nil, err
    }

    frameReader := framer.NewFrameReader(resp.Body)
    watchEventDecoder := streaming.NewDecoder(frameReader, streamingSerializer)

    return watch.NewStreamWatcher(
        restclientwatch.NewDecoder(watchEventDecoder, objectDecoder),
        // use 500 to indicate that the cause of the error is unknown - other error codes
        // are more specific to HTTP interactions, and set a reason
        errors.NewClientErrorReporter(http.StatusInternalServerError, r.verb, "ClientWatchDecoding"),
    ), nil
}


func (r *Request) newStreamWatcher(resp *http.Response) (watch.Interface, error) {
	contentType := resp.Header.Get("Content-Type")
	mediaType, params, err := mime.ParseMediaType(contentType)
	if err != nil {
		klog.V(4).Infof("Unexpected content type from the server: %q: %v", contentType, err)
	}
	objectDecoder, streamingSerializer, framer, err := r.c.content.Negotiator.StreamDecoder(mediaType, params)
	if err != nil {
		return nil, err
	}

	handleWarnings(resp.Header, r.warningHandler)

	frameReader := framer.NewFrameReader(resp.Body)
	watchEventDecoder := streaming.NewDecoder(frameReader, streamingSerializer)

	return watch.NewStreamWatcher(
		restclientwatch.NewDecoder(watchEventDecoder, objectDecoder),
		// use 500 to indicate that the cause of the error is unknown - other error codes
		// are more specific to HTTP interactions, and set a reason
		errors.NewClientErrorReporter(http.StatusInternalServerError, r.verb, "ClientWatchDecoding"),
	), nil
}
```

很典型的 Golang HTTP 请求代码，但是 Kubernetes apiserver 首次应答的 HTTP Header 中会携带上 Transfer-Encoding: chunked，表示分块传输，客户端会保持这条 TCP 连接并等待下一个数据块。如此 apiserver 会主动将监听的 Kubernetes 资源对象的变化不断地推送给客户端：


```shell
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

这样，Informer就已经完成了`List&Watch`的过程。每一种资源的informer都会创建一个对应Relector，Relector`List & Watch `某些资源对象，并将其推进对应的DeltaFifo队列里面。

`Relector`进行List & Watch的是在一条单独的go routine中执行中，informer会同时启动一个无限循环的函数，从DeltaFifo中取数据并进行后续的处理:

```golang
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}

```