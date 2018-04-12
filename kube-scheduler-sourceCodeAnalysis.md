本代码分析基于v1.9.4-beta0

 

\1. 进程启动过程

​    入口函数在/kubernetes/cmd/kube-scheduler/scheduler.go

​    在main函数中关键为如下程序：

​    **command := app.NewSchedulerCommand(**)   //生成scheduler命令行

   ** iferr := command.Execute(); err != nil**             //执行命令行

​    作用分别是新建scheduler命令行和执行scheduler命令行

​     

​    接下来再分析构建scheduler命令行的详细过程，app包的位置定位到 /kubernetes/cmd/kube-scheduler/app/server.go的如下函数 

​    **func NewSchedulerCommand() \*cobra.Command**  //读取并处理命令行参数

​    为后续构建SchedulerServer从命令行一侧读取所需的上下文环境和参数

**  **

​    再采用了cobra.Command框架后，新建SchedulerServer对象的新建没有之前版本那么明显，隐藏在了函数**funcNewSchedulerCommand() \*cobra.Command**中的程序

​    **cmdutil.CheckErr(opts.Run()) **从这段程序跳转到如下构建SchedulerServer的函数

​    **func (o \*Options) Run() error**

​    在Run()函数中有如下代码

**    server, err := NewSchedulerServer(config,o.master)**

**    if err != nil {**

**return err**

**     }**

**    stop := make(chan struct{})**

**    return server.Run(stop)**

**    **这段代码中可以清楚的看到新建了SchedulerServer对象，并且执行对象的Run函数

 

​    我们再进入NewSchedulerServer函数

​    **func NewSchedulerServer(config\*componentconfig.KubeSchedulerConfiguration, master string) (*SchedulerServer,error) {**

 

**return&SchedulerServer{**

**SchedulerName:                  config.SchedulerName,**

**Client:                         client,**

**InformerFactory:               informers.NewSharedInformerFactory(client, 0),**

**PodInformer:                   factory.NewPodInformer(client, 0, config.SchedulerName),**

**AlgorithmSource:                config.AlgorithmSource,**

**HardPodAffinitySymmetricWeight:config.HardPodAffinitySymmetricWeight,**

**EventClient:                    eventClient,**

**Recorder:                       recorder,**

**Broadcaster:                    eventBroadcaster,**

**LeaderElection:                 leaderElectionConfig,**

**HealthzServer:                  healthzServer,**

**MetricsServer:                  metricsServer,**

**}, nil**

**}**

 

​    发现NewSchedulerServer函数最后返回的是一个SchedulerServer对象，分析SchedulerServer对象一些属性

   ** typeSchedulerServer struct {**

**       SchedulerName       string     **//scheduler的名字

**       Client                       clientset.Interface   **  // kube 的client，用于调用kube的各类接口，具体可查看k8s.io/client-go/kubernetes/clientset.go

**InformerFactory       informers.SharedInformerFactory**     // SharedInformerFactory 提供各种资源各个API版本可共享的informer 

**       PodInformer             coreinformers.PodInformer    ** **// ****构建的PodiFormer，providesaccess to a shared informer and lister for pods.**

**       AlgorithmSource    componentconfig.SchedulerAlgorithmSource     // 调度算法**

**HardPodAffinitySymmetricWeightint32**

**       EventClient                    v1core.EventsGetter     **// event接口的eventsGetter函数，经过另一个接口最终返回event的接口函数

**       Recorder                      record.EventRecorder     **

**       Broadcaster                   record.EventBroadcaster     // EventBroadcaster knows how to receiveevents and send them to any EventSink, watcher, or log.**

**//LeaderElection is optional.**

**       LeaderElection\*leaderelection.LeaderElectionConfig    // leader选取**

**// HealthzServeris optional.**

**       HealthzServer \*http.Server        **// 健康检查用的http服务

**// MetricsServeris optional.**

**       MetricsServer \*http.Server        **// 性能指标度量的metric http接口

**}**

​    

​     返回到我们之前的函数 **func (o\*Options) Run() error，**我们看到这里新建了channel，并用这个channel作为参数将返回的SchedulerServer执行了Run函数

**    stop := make(chan struct{})**

**    return server.Run(stop)**

​    这边似乎还看不出stopchannel的作用，待会回过头来再看，直接到**server.Run(stop)**,该函数位于/kubernetes/cmd/kube-scheduler/app/server.go中的

​     **func (s \*SchedulerServer) Run(stop chanstruct{}) error**

**    **从这个函数的注释中说明此函数永远都不会退出，下面详细分析这段代码

​    

**func (s\*SchedulerServer) Run(stop chan struct{}) error {**

**// To helpdebugging, immediately log version**

**       glog.Infof("Version: %+v", version.Get())    **// 列出了版本信息

 

**// Build ascheduler config from the provided algorithm source.**

**       schedulerConfig, err :=s.SchedulerConfig()      **// 创建scheduler配置

**if err != nil {**

**return err**

**}**

 

**// Create thescheduler.**

**       sched := scheduler.NewFromConfig(schedulerConfig)    ** // 从config中创建scheduler

 

**// Prepare theevent broadcaster.**

**if!reflect.ValueOf(s.Broadcaster).IsNil() &&!reflect.ValueOf(s.EventClient).IsNil() {**

**s.Broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface:s.EventClient.Events("")})**

**}**     // StartRecordingToSink 主要是负责发送从特定event 广播地收到的event到给定的sink池

 

**       // Start up the healthz server.   **// 开启健康检查的服务端

**ifs.HealthzServer != nil {**

**gowait.Until(func() {**

**glog.Infof("startinghealthz server on %v", s.HealthzServer.Addr)**

**err := s.HealthzServer.ListenAndServe()**

**if err != nil {**

**utilruntime.HandleError(fmt.Errorf("failedto start healthz server: %v", err))**

**}**

**},5\*time.Second, stop)**

**       }    ** // 我们看到这里用go携程启动了Until函数，Until函数将会循环定期执行其中定义的函数直到stop channel被关闭，Until的具体实现在/k8s.io/apimachinery/pkg/util/wait目录下

 

 

**       // Start up the metrics server.      **// 开启指标监控的服务端

**ifs.MetricsServer != nil {**

**gowait.Until(func() {**

**glog.Infof("startingmetrics server on %v", s.MetricsServer.Addr)**

**err :=s.MetricsServer.ListenAndServe()**

**if err != nil {**

**utilruntime.HandleError(fmt.Errorf("failedto start metrics server: %v", err))**

**}**

**},5\*time.Second, stop)**

**       }     ** // 我们看到这里用go携程启动了Until函数，Until函数将会循环定期执行其中定义的函数直到stop channel被关闭，Until的具体实现在/k8s.io/apimachinery/pkg/util/wait目录下

 

**// Start allinformers.**

**       go s.PodInformer.Informer().Run(stop)   ** // Run函数最后会定位到/k8s.io/client-go/tools/cache/shared_informer.go, Run starts the sharedinformer, which will be stopped when stopCh is closed.** ****PodInformer****用于watch/list non-terminal pods并缓存**

**       s.InformerFactory.Start(stop)    **// Start函数最后定位到/k8s.io/client-go/informers/internalinterfaces/factory_interfaces.go,SharedInformerFactory a small interface to allow for adding an informer withoutan import cycle

 

**// Wait for allcaches to sync before scheduling.**

**       s.InformerFactory.WaitForCacheSync(stop)   // 这个最后定位到函数/k8s.io/client-go/tools/cache/shared-informer.go，但该函数有返回，所以难确定是否是该函数**

**controller.WaitForCacheSync("scheduler",stop, s.PodInformer.Informer().HasSynced)**

**    **    // WaitForCacheSync is a wrapper around cache.WaitForCacheSyncthat generates log messages indicating that the controller identified bycontrollerName is waiting for syncs, followed by either a successful or failedsync.

 

**// Prepare areusable run function.**

**       run := func(stopCh <-chan struct{}) {     // ？？？？？**

**sched.Run()**

**<-stopCh**

**}**

 

**// If leaderelection is enabled, run via LeaderElector until done and exit.**

**ifs.LeaderElection != nil {**

**s.LeaderElection.Callbacks= leaderelection.LeaderCallbacks{**

**OnStartedLeading:run,**

**OnStoppedLeading:func() {**

**utilruntime.HandleError(fmt.Errorf("lostmaster"))**

**},**

**}**

**leaderElector,err := leaderelection.NewLeaderElector(\*s.LeaderElection)**

**if err != nil {**

**returnfmt.Errorf("couldn't create leader elector: %v", err)**

**}**

 

**leaderElector.Run()**

 

**returnfmt.Errorf("lost lease")**

**       }      **// 如果开启了LeaderElection选项，则在此函数内部开始Run函数，否则在下面的程序中开启Run函数，Run函数实际跳转到/kubernetes/pkg/scheduler/scheduler.go的函数  func (sched *Scheduler)Run() 

 

**// Leaderelection is disabled, so run inline until done.**

**       run(stop)    **//run(channel) 函数通过上面的sched.Run函数，实际跳转到/kubernetes/pkg/scheduler/scheduler.go的函数  func (sched *Scheduler) Run() 

**returnfmt.Errorf("finished without leader elect")**

**}**

 

​    我们接着看 func (sched*Scheduler) Run() 函数

   ** //Run begins watching and scheduling. It waits for cache to be synced, thenstarts a goroutine and returns immediately.**

**func (sched\*Scheduler) Run() {**

**if!sched.config.WaitForCacheSync() {**

**return**

**}**

 

**ifutilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {**

**gosched.config.VolumeBinder.Run(sched.bindVolumesWorker,sched.config.StopEverything)**

**       }     **// Run starts a goroutine to handle thebinding queue with the given function,一直对StopEverything这个比较好奇，作为Scheduler结构体里的属性，Close this to shut down thescheduler，定义如下StopEverything chan struct{}，

 

**gowait.Until(sched.scheduleOne, 0, sched.config.StopEverything)**

**}    **  // Until函数循环监听直到stopchannel被关闭，并周期执行f 函数，Until函数本身是JitterUntil函数的糖衣函数，JitterUntil函数可以直接定位到 /k8s.io/apimachinery/pkg/util/wait/wait.go，此处我们发现开启了调度流程函数schedulerOne

 

跳转到函数**func (sched \*Scheduler) scheduleOne()** ，从注释中我们发现schedulerOne服务一个pod的整个调度流程，解读这段代码

**// scheduleOne does the entire scheduling workflow for a singlepod.  It is serialized on the schedulingalgorithm's host fitting.**

**func (sched \*Scheduler) scheduleOne() {**

**       pod := sched.config.NextPod()   **// 获取一个待调度的Pod, NextPod()方法是阻塞的

**       if pod.DeletionTimestamp!= nil {    **// 首先，判断这个Pod是否正在被删除中，如果是，则返回，跳过调度。

**sched.config.Recorder.Eventf(pod, v1.EventTypeWarning,"FailedScheduling", "skip schedule deleting pod: %v/%v",pod.Namespace, pod.Name)**

**glog.V(3).Infof("Skip schedule deleting pod: %v/%v",pod.Namespace, pod.Name)**

**              return       **// 如果pod正在被删除中，那就跳过调度

**}**

 

**glog.V(3).Infof("Attempting to schedule pod: %v/%v",pod.Namespace, pod.Name)**

// 开始尝试调度pod,以命名空间和pod名称区分

 

**// Synchronously attempt to find a fit for the pod.**

**start := time.Now()**

**       suggestedHost, err :=sched.schedule(pod)   ** // 此处的schedule()方法中，会调用注册的初选和优选的算法，最终返回一个节点的名称存放到suggestedHost变量中，schedule过程的详细算法我们在这个函数后再分析

**if err != nil {**

**// schedule() may have failed because the pod would not fit on anyhost, so we try to**

**// preempt, with the expectation that the next time the pod is triedfor scheduling it**

**// will fit due to the preemption. It is also possible that adifferent pod will schedule**

**// into the resources that were preempted, but this is harmless.**

**if fitError, ok := err.(\*core.FitError); ok {**

**preemptionStartTime := time.Now()**

**sched.preempt(pod, fitError)**

**metrics.PreemptionAttempts.Inc()**

**metrics.SchedulingAlgorithmPremptionEvaluationDuration.Observe(metrics.SinceInMicroseconds(preemptionStartTime))**

**}**

**return**

**}**

**       metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInMicroseconds(start))           ** // 记录算法延迟的度量

**// Tell the cache to assume that a pod now is running on a givennode, even though it hasn't been bound yet.**

**// This allows us to keep scheduling without waiting on binding tooccur.**

**       assumedPod :=pod.DeepCopy()   ** // 即使该pod还未真正绑定到节点上，我们先假设这个pod已经在指定的节点上运行了。这是为了更新shcedulerCache, 与绑定操作（需要花费一些时间）以异步方式进行。 

 

**// Assume volumes first before assuming the pod.**

**//**

**// If no volumes need binding, then nil is returned, and continue toassume the pod.**

**//**

**// Otherwise, error is returned and volume binding is startedasynchronously for all of the pod's volumes.**

**// scheduleOne() returns immediately on error, so that it doesn't continueto assume the pod.**

**//**

**// After the asynchronous volume binding updates are made, it willsend the pod back through the scheduler for**

**// subsequent passes until all volumes are fully bound.**

**//**

**// This function modifies 'assumedPod' if volume binding isrequired.**

**err = sched.assumeAndBindVolumes(assumedPod, suggestedHost)**

**if err != nil {**

**return**

**}**

 

**// assume modifies `assumedPod` by setting NodeName=suggestedHost**

**err = sched.assume(assumedPod, suggestedHost)**

**if err != nil {**

**return**

**}**

**// bind the pod to its host asynchronously (we can do this b/c ofthe assumption step above).**

**      ** //创建一个协程goruntime,异步地绑定pod（注意，该shceduleOne方法也是在一个协程中）

**go func() {**

**err := sched.bind(assumedPod, &v1.Binding{**

**ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name:assumedPod.Name, UID: assumedPod.UID},**

**Target: v1.ObjectReference{**

**Kind: "Node",**

**Name: suggestedHost,**

**},**

**              })    ** // 这里调用了bind函数执行具体的bind步骤，此函数就在同一文件中，我们稍后具体分析

**metrics.E2eSchedulingLatency.Observe(metrics.SinceInMicroseconds(start))**

**if err != nil {**

**glog.Errorf("Internal error binding pod: (%v)", err)**

**}**

**}()**

**}**

 

从上面这个schedulerOne函数中，我们具体再来分析如下两个函数

func (sched *Scheduler) schedule(pod *v1.Pod) (string, error)  ，这个具体实现调度算法的函数，以及

func (sched *Scheduler) bind(assumed *v1.Pod, b *v1.Binding) error ，这个具体执行bind操作的函数

 

​      先分析调度函数

**// schedule implements the scheduling algorithm and returns thesuggested host.**

**func (sched \*Scheduler) schedule(pod *v1.Pod) (string, error) {**

**host, err := sched.config.Algorithm.Schedule(pod,sched.config.NodeLister)**

**if err != nil {**

**glog.V(1).Infof("Failed to schedule pod: %v/%v",pod.Namespace, pod.Name)**

**pod = pod.DeepCopy()**

**sched.config.Error(pod, err)**

**sched.config.Recorder.Eventf(pod, v1.EventTypeWarning,"FailedScheduling", "%v", err)**

**sched.config.PodConditionUpdater.Update(pod, &v1.PodCondition{**

**Type:    v1.PodScheduled,**

**Status:  v1.ConditionFalse,**

**Reason: v1.PodReasonUnschedulable,**

**Message: err.Error(),**

**})**

**return "", err**

**}**

**return host, err**

**}**

 

以上这个函数中最关键的是程序**host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)**，其中

**func (g \*genericScheduler) Schedule(pod *v1.Pod, nodeListeralgorithm.NodeLister) (string, error) **是完成具体调度的程序，位于/kubernetes/pkg/scheduler/core/generic_scheduler.go

这个函数里面主要的调度过程有

（1）**podPassesBasicChecks(pod, g.pvcLister)**

（2）**filteredNodes, failedPredicateMap, err :=findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders,g.predicateMetaProducer, g.equivalenceCache, g.schedulingQueue,g.alwaysCheckAllPredicates)**

（3）**priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap,metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)**

从程序名字中我们也能够看出前两个是初步筛选分别筛选pvc和node，第三部是最优筛选，scheduler还支持自定义的调度算法，我们后续将找到它的入口，这里我们先详细分析一下**findNodesThatFit**和**PrioritizeNodes**函数。

**findNodesThatFit**函数主要Filters thenodes to find the ones that fit based on the given predicate functions Eachnode is passed through the predicate functions to determine if it is a fit

在**findNodesThatFit**函数的参数里面，我们看到了参数predicateFuncsmap[string]algorithm.FitPredicate 可以从中查看predicate的筛选函数，要确定是在哪个地方初始化了这些predicate函数，我们定位到了/pkg/scheduler/algorithmprovider/default/default.go的init函数里面func init () {}（**什么时候init需要再确认一下**），我们可以发现默认的default predicate都定义在了/pkg/scheduler/algorithm/predicates/predicates.go

在执行PrioritizeNodes函数之后，我们发现并没有直接返回结果priorityList，而是继续跳转到函数** func (g \*genericScheduler) selectHost(priorityListschedulerapi.HostPriorityList) (string, error) **这个函数中将给priorityList里面的node打分，分数最高的返回。

 

 

 

**还有很多问题未解决如下，后面将一一答复哈：**

\1. 如果绑定失败，retrying 是如何触发的呢？

\2. retrying的筛选逻辑，使用的算法是否和之前一致呢？

\3. 如果调度过程中节点挂了怎么办？

\4. scheduler是如何拿到各个节点的资源信息的？

\5. 绑定之后，Kubelet是如何接收Pod的？

\6. 如何添加自己定制的算法？

 

**参考：**

<https://www.jianshu.com/p/a8bbfecd4f96>

<https://www.jianshu.com/p/d0d979c76fb4>

​    

​    

​    

 