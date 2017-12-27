---
title: Kubernetes调度器
date: 2017-12-28 00:32:06
tags:
---
###  K8S 调度器
按照 K8S 的设计原则， 所有 Controller 的实现都是通过 watch Api Server，监听资源的变化并做出操作。
调度器亦是如此，在内存中维护一个podQueue，并watch pod变化，当有未调度pod增加时， 将节点加入podQuene 中。定期从 podQuene 中取 pod, 寻找适合这个 pod 的节点，执行 binding 在该节点上部署 pod。

### 启动
在/k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/server.go 中
* 创建两个 client， 一个用于和 apiServer 交互， 一个用于做该模块选leader
* 根据配置设置调度策略 `algorithmprovider.ApplyFeatureGates()`
* 启动监控的 http 端口 `startHTTP`
* 启动 informer，有podInformer 和 其他（sharedInformer） `go podInformer.Informer().Run(stop)
	informerFactory.Start(stop)`
* 创建调度器实例
* 初始化本地缓存数据，直到完全同步 `informerFactory.WaitForCacheSync(stop)`
* 运行调度程序 （如果是高可用配置，监听 leaderElectionClient的LeaderStarting事件，执行，当成为 leader 时再运行调度程序）


### Config
* CreateScheduler是创建调度器实例的方法。通过factory.NewConfigFactory 创建一个ConfigFactory对象。ConfigFactory中过程中会做如下几件事：
	1. 创建schedulerCache实例，作为调度过程中的内存缓存
	* 为创建过程中做为传入的各种 informer 增加事件监听。
	* podInformer 监听已分配的pod， 维护到schedulerCache中
	* 监听未分配的pod， 维护到podQueue中。 
	* 监听 node 变化，维护到schedulerCache。 
	* 监听pv，pvc变化，并将相应的指标缓存清理
	* 监听 service 变化，将所有调度的缓存指标清理
	* 保留以上的 lister，用于调度过程中获取资源信息
* 根据变量和ConfigFactory实例，创建Config实例。 有两种获取策略方式，一种是configMap，一种是注册插件提供的 provider。初始化 pod调度 信息缓存，根据策略获取算法。
* 将config实例赋值给Scheduler实例。


### Scheduler
Scheduler实现了几个关键方法：
* scheduleOne 作为定时任务执行，在scheduleOne中取podQuene 中的下一个 pod。
* schedule  计算 pod合适的node。 通过 config中存的算法对象的Schedule方法。
* preempt  当schedule失败，执行Preempt抢占其他 node
* assume   当pod 被schedule成功，预先选择node，将信息更新到内存
* bind   调 apiServer 将 pod 绑定到节点上


### 调度策略以及参数生成
```
--- github.com/kubernetes/kubernetes/plugin/pkg/scheduler/factory/factory.go ---
// configFactory的CreateFromKeys方法：

// 注册predicate和priority方法
predicateFuncs, err := f.GetPredicates(predicateKeys)
priorityConfigs, err := f.GetPriorityFunctionConfigs(priorityKeys)

// 生成Producer， 用于执行predicate和priority的传参
priorityMetaProducer, err := f.GetPriorityMetadataProducer()
predicateMetaProducer, err := f.GetPredicateMetadataProducer()

```

predicates注册为fitPredicateMap中的函数返回值。可见predicate和priority注册内容都应该是一个返回算法函数的函数: 

```
--- github.com/kubernetes/kubernetes/plugin/pkg/scheduler/factory/plugins.go ---
func getFitPredicateFunctions(names sets.String, args PluginFactoryArgs) (map[string]algorithm.FitPredicate, error) {

	......
	predicates := map[string]algorithm.FitPredicate{}
	for _, name := range names.List() {
		factory, ok := fitPredicateMap[name]
		if !ok {
			return nil, fmt.Errorf("Invalid predicate name %q specified - no corresponding function found", name)
		}
		predicates[name] = factory(args)
	}
	return predicates, nil
}

```
以下是一个predicate注册的例子：

```
--- github.com/kubernetes/kubernetes/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go ---

factory.RegisterFitPredicateFactory(
	predicates.NoVolumeZoneConflictPred,
	func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
		return predicates.NewVolumeZonePredicate(args.PVInfo, args.PVCInfo, args.StorageClassInfo)
	},
),
		
--- k8s.io/kubernetes/plugin/pkg/scheduler/algorithm/predicates/predicates.go ---

func NewVolumeZonePredicate(pvInfo PersistentVolumeInfo, pvcInfo PersistentVolumeClaimInfo) algorithm.FitPredicate {
	c := &VolumeZoneChecker{
		pvInfo:  pvInfo,
		
		pvcInfo: pvcInfo,
	}
	return c.predicate
}

func (c *VolumeZoneChecker) predicate(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
	.........
}
```

通用参数基本就是 get 中 资源的Lister 和 informer

```
--- github.com/kubernetes/kubernetes/plugin/pkg/scheduler/factory/factory.go ---

func (f *configFactory) getPluginArgs() (*PluginFactoryArgs, error) {
	return &PluginFactoryArgs{
		PodLister:                      f.podLister,
		ServiceLister:                  f.serviceLister,
		ControllerLister:               f.controllerLister,
		ReplicaSetLister:               f.replicaSetLister,
		StatefulSetLister:              f.statefulSetLister,
		NodeLister:                     &nodeLister{f.nodeLister},
		NodeInfo:                       &predicates.CachedNodeInfo{NodeLister: f.nodeLister},
		PVInfo:                         &predicates.CachedPersistentVolumeInfo{PersistentVolumeLister: f.pVLister},
		PVCInfo:                        &predicates.CachedPersistentVolumeClaimInfo{PersistentVolumeClaimLister: f.pVCLister},
		StorageClassInfo:               &predicates.CachedStorageClassInfo{StorageClassLister: f.storageClassLister},
		VolumeBinder:                   f.volumeBinder,
		HardPodAffinitySymmetricWeight: f.hardPodAffinitySymmetricWeight,
	}, nil
}

```

### 算法的调度实现
```
--- src/k8s.io/kubernetes/plugin/pkg/scheduler/core/generic_scheduler.go ---
Schedule函数

nodes, err := nodeLister.List()

......
// 更新缓存，用于后面的算法
err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	
......
// 寻找合适的节点
filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders, g.predicateMetaProducer, g.equivalenceCache)
......
// 给节点设置优先级
metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
......
// 选择节点
return g.selectHost(priorityList)
```

findNodesThatFit 函数， 寻找适合的一组节点

```
--- src/k8s.io/kubernetes/plugin/pkg/scheduler/core/generic_scheduler.go ---

checkNode := func(i int) {
	nodeName := nodes[i].Name
	fits, failedPredicates, err := podFitsOnNode(pod, meta, nodeNameToInfo[nodeName], predicateFuncs, ecache)
	if err != nil {
		predicateResultLock.Lock()
		errs[err.Error()]++
		predicateResultLock.Unlock()
		return
	}
	if fits {
		filtered[atomic.AddInt32(&filteredLen, 1)-1] = nodes[i]
	} else {
		predicateResultLock.Lock()
		failedPredicateMap[nodeName] = failedPredicates
		predicateResultLock.Unlock()
	}
}
workqueue.Parallelize(16, len(nodes), checkNode)
......
return filtered, failedPredicateMap, nil
		
```

PrioritizeNodes函数，给节点设置优先级

```
--- src/k8s.io/kubernetes/plugin/pkg/scheduler/core/generic_scheduler.go ---

	// 遍历执行
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Function != nil {
			// DEPRECATED
			wg.Add(1)
			go func(index int, config algorithm.PriorityConfig) {
				defer wg.Done()
				var err error
				results[index], err = config.Function(pod, nodeNameToInfo, nodes)
				if err != nil {
					appendError(err)
				}
			}(i, priorityConfig)
		} else {
			results[i] = make(schedulerapi.HostPriorityList, len(nodes))
		}
	}
	// 支持 map-reduce 的方式处理
	// map 是所有节点的所有priority方法同时执行
	processNode := func(index int) {
		nodeInfo := nodeNameToInfo[nodes[index].Name]
		var err error
		for i := range priorityConfigs {
			if priorityConfigs[i].Function != nil {
				continue
			}
			results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
			if err != nil {
				appendError(err)
				return
			}
		}
	}
	workqueue.Parallelize(16, len(nodes), processNode)
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Reduce == nil {
			continue
		}
		wg.Add(1)
		go func(index int, config algorithm.PriorityConfig) {
			defer wg.Done()
			// reduce 是所有priority方法并行执行一次，每个方法中需要计算所有节点的优先级并且写值
			if err := config.Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
				appendError(err)
			}
		}(i, priorityConfig)
	}
	wg.Wait()
	
.......
// 按照算得的优先级和权重计算最终分数
for i := range nodes {
		result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
		for j := range priorityConfigs {
			result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
		}
	}

```


#### Tips
* 所有gorouting 的启动都带一个stop channel 的参数， 为了保证退出时能够回收对应的 gorouting。
* 使用工厂模式创建configFactory，在创建 Config 实例，再创建Scheduler。 没有明白这里工厂模式的意义在哪。
* 其他值得一看的地方： 
	* Informer 的实现
	* 选主模块的实现


### 参考
http://cizixs.com/2017/07/19/kubernetes-scheduler-source-code-analysis
http://cizixs.com/2017/03/10/kubernetes-intro-scheduler