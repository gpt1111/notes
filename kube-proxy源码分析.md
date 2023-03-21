# kube-proxy 源码分析

## ProxyServer
代理服务器
```go
type ProxyServer struct { 
    // k8s 客户端
    Client                 clientset.Interface
    EventClient            v1core.EventsGetter
    // iptable 操作接口
    IptInterface           utiliptables.Interface
    // ipvs 操作接口
    IpvsInterface          utilipvs.Interface
    // ipset 操作接口
    IpsetInterface         utilipset.Interface
    // 命令行执行接口
    execer                 exec.Interface
    // 代理服务器provider实现
    Proxier                proxy.Provider
    Broadcaster            events.EventBroadcaster
    Recorder               events.EventRecorder
    ...
}

// 服务器启动
func (s *ProxyServer) Run() error {
    ...
    // 生成informerFactory
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))

    // serviceConfig 实现了Service ResourceEventHandler{AddFunc、UpdateFunc、DeleteFunc}
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)
    
    // endpointSliceConfig 实现了EndpointSlice ResourceEventHandler{AddFunc、UpdateFunc、DeleteFunc}
	endpointSliceConfig := config.NewEndpointSliceConfig(informerFactory.Discovery().V1().EndpointSlices(), s.ConfigSyncPeriod)
	endpointSliceConfig.RegisterEventHandler(s.Proxier)
	go endpointSliceConfig.Run(wait.NeverStop)

	// This has to start after the calls to NewServiceConfig because that
	// function must configure its shared informer event handlers first.
	informerFactory.Start(wait.NeverStop)
    ...
	// Birth Cry after the birth is successful
	s.birthCry()
	go s.Proxier.SyncLoop()

	return <-errCh
}
```

## iptable proxier
iptable 模式实现的proxier
```go
// proxier 接口
type Provider interface {
    // 对EndPointSlice进行处理的handler接口，注册到serviceConfig.RegisterEventHandler
    config.EndpointSliceHandler 
    // 对Endpoint进行处理的handler接口，注册到endpointSliceConfig.RegisterEventHandler
    config.ServiceHandler    
    config.NodeHandler

    // Sync immediately synchronizes the Provider's current state to proxy rules.
    Sync()
    // SyncLoop runs periodic work.
    // This is expected to run as a goroutine or as the main loop of the app.
    // It does not return.
    SyncLoop()
}

type Proxier struct {
    // endPoint更新Tracker
	endpointsChanges *proxy.EndpointChangeTracker
	// service更新Tracker
	serviceChanges   *proxy.ServiceChangeTracker

	mu           sync.Mutex // protects the following fields
	// ServicePortName到 ServicePort的映射
	svcPortMap   proxy.ServicePortMap
	// ServicePortName到 []EndPoint的映射
	endpointsMap proxy.EndpointsMap

	// iptable设置接口
	iptables       utiliptables.Interface
	// 命令行执行接口
	exec           utilexec.Interface
    ... 
    // iptable数据缓存
	iptablesData             *bytes.Buffer
	existingFilterChainsData *bytes.Buffer
	// filter表中存在的链，按行缓存
	filterChains             utilproxy.LineBuffer
	// filter表中存在的规则，按行缓存
	filterRules              utilproxy.LineBuffer
	// nat表中存在的链，按行缓存
	natChains                utilproxy.LineBuffer
	// nat表中存在的规则，按行缓存
	natRules                 utilproxy.LineBuffer
    ...
}

// 定时根据svcPortMap、endpointsMap数据更新iptable规则
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
    ...
    // 从serviceChanges中把有变动的命名空间servicePort更新到svcPortMap中
	serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
	// 从endpointsChanges中把有变动的命名空间[]EndPoint更新到endpointMap中
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)
	...
	// 使用iptables 确保链和规则
	if !tryPartialSync {
	    ...
		for _, jump := range append(iptablesJumpChains, iptablesKubeletJumpChains...) {
			if _, err := proxier.iptables.EnsureChain(jump.table, jump.dstChain); err != nil {
				klog.ErrorS(err, "Failed to ensure chain exists", "table", jump.table, "chain", jump.dstChain)
				return
			}
			args := jump.extraArgs
			if jump.comment != "" {
				args = append(args, "-m", "comment", "--comment", jump.comment)
			}
			args = append(args, "-j", string(jump.dstChain))
			if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jump.table, jump.srcChain, args...); err != nil {
				klog.ErrorS(err, "Failed to ensure chain jumps", "table", jump.table, "srcChain", jump.srcChain, "dstChain", jump.dstChain)
				return
			}
		}
	}

    // 重置链、规则行缓存
	proxier.filterChains.Reset()
	proxier.filterRules.Reset()
	proxier.natChains.Reset()
	proxier.natRules.Reset()

    //行缓存写入链名
	// Write chain lines for all the "top-level" chains we'll be filling in
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeExternalServicesChain, kubeForwardChain, kubeNodePortsChain, kubeProxyFirewallChain} {
		proxier.filterChains.Write(utiliptables.MakeChainLine(chainName))
	}
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeNodePortsChain, kubePostroutingChain, kubeMarkMasqChain} {
		proxier.natChains.Write(utiliptables.MakeChainLine(chainName))
	}
    ...
	// 创建 service-port.规则
	for svcName, svc := range proxier.svcPortMap {
        // iptable写入service链
		if hasInternalEndpoints {
			proxier.natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", string(internalTrafficChain))
		} else {
			// No endpoints.
			proxier.filterRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", internalTrafficFilterComment,
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", internalTrafficFilterTarget,
			)
		}
        ...
        // iptables 写入endpoint规则
		// Generate the per-endpoint chains.
		for _, ep := range allLocallyReachableEndpoints {
			epInfo, ok := ep.(*endpointsInfo)
			if !ok {
				klog.ErrorS(nil, "Failed to cast endpointsInfo", "endpointsInfo", ep)
				continue
			}

			endpointChain := epInfo.ChainName

			// Create the endpoint chain
			proxier.natChains.Write(utiliptables.MakeChainLine(endpointChain))
			activeNATChains[endpointChain] = true

			args = append(args[:0], "-A", string(endpointChain))
			args = proxier.appendServiceCommentLocked(args, svcPortNameString)
			// Handle traffic that loops back to the originator with SNAT.
			proxier.natRules.Write(
				args,
				"-s", epInfo.IP(),
				"-j", string(kubeMarkMasqChain))
			// Update client-affinity lists.
			if svcInfo.SessionAffinityType() == v1.ServiceAffinityClientIP {
				args = append(args, "-m", "recent", "--name", string(endpointChain), "--set")
			}
			// DNAT to final destination.
			args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", epInfo.Endpoint)
			proxier.natRules.Write(args)
		}
	}
    ...
    
	// 重置iptables缓存，写入同步规则
	proxier.iptablesData.Reset()
	proxier.iptablesData.WriteString("*filter\n")
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")
	proxier.iptablesData.WriteString("*nat\n")
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")

	// iptables restore恢复
	err = proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	...
}
```

## ServiceChangeTracker & ServiceMap
Service更新追踪和ServiceMap
```
type ServiceChangeTracker struct {
    // lock protects items.
    lock sync.Mutex
    // 按照命名空间缓存有哪些service更新了，然后按svcPortName-svcPort缓存
    items map[types.NamespacedName]*serviceChange
    // service转换函数
    makeServiceInfo         makeServicePortFunc
    processServiceMapChange processServiceMapChangeFunc
    ipFamily                v1.IPFamily
    recorder events.EventRecorder
}

// 根据serviceInformer回调event，生成serviceChang
func (sct *ServiceChangeTracker) Update(previous, current *v1.Service) bool {
    // service更新缓存到items
	change, exists := sct.items[namespacedName]
	if !exists {
		change = &serviceChange{}
		change.previous = sct.serviceToServiceMap(previous)
		sct.items[namespacedName] = change
	}
	change.current = sct.serviceToServiceMap(current)
    ...
	return len(sct.items) > 0
}

type ServiceMap map[ServicePortName]ServicePort
// 合并changes中缓存的serviceChange到ServiceMap
func (sm ServiceMap) Update(changes *ServiceChangeTracker) (result UpdateServiceMapResult) {
    result.UDPStaleClusterIP = sets.NewString()
    sm.apply(changes, result.UDPStaleClusterIP)

    // TODO: If this will appear to be computationally expensive, consider
    // computing this incrementally similarly to serviceMap.
    result.HCServiceNodePorts = make(map[types.NamespacedName]uint16)
    for svcPortName, info := range sm {
        if info.HealthCheckNodePort() != 0 {
            result.HCServiceNodePorts[svcPortName.NamespacedName] = uint16(info.HealthCheckNodePort())
        }
    }

    return result
}
```

## EndpointChangeTracker & EndpointsMap
endpoint更新追踪器和EndpointsMap
```
type EndpointChangeTracker struct {
    // lock protects items.
    lock sync.Mutex
    // hostname is the host where kube-proxy is running.
    hostname string
    // endpoints更新缓存
    items map[types.NamespacedName]*endpointsChange
    // endpoint转换函数
    makeEndpointInfo          makeEndpointFunc
    processEndpointsMapChange processEndpointsMapChangeFunc
    // endpointSlice更新缓存
    endpointSliceCache *EndpointSliceCache
    // ipfamily identify the ip family on which the tracker is operating on
    ipFamily v1.IPFamily
    recorder events.EventRecorder
    // Map from the Endpoints namespaced-name to the times of the triggers that caused the endpoints
    // object to change. Used to calculate the network-programming-latency.
    lastChangeTriggerTimes map[types.NamespacedName][]time.Time
    // record the time when the endpointChangeTracker was created so we can ignore the endpoints
    // that were generated before, because we can't estimate the network-programming-latency on those.
    // This is specially problematic on restarts, because we process all the endpoints that may have been
    // created hours or days before.
    trackerStartTime time.Time
}
// 根据endpointSliceInformer回调event，生成endpointSliceCache
func (ect *EndpointChangeTracker) EndpointSliceUpdate(endpointSlice *discovery.EndpointSlice, removeSlice bool) bool {
    ...
	changeNeeded := ect.endpointSliceCache.updatePending(endpointSlice, removeSlice)
	if changeNeeded {
		metrics.EndpointChangesPending.Inc()
		// In case of Endpoints deletion, the LastChangeTriggerTime annotation is
		// by-definition coming from the time of last update, which is not what
		// we want to measure. So we simply ignore it in this cases.
		// TODO(wojtek-t, robscott): Address the problem for EndpointSlice deletion
		// when other EndpointSlice for that service still exist.
		if removeSlice {
			delete(ect.lastChangeTriggerTimes, namespacedName)
		} else if t := getLastChangeTriggerTime(endpointSlice.Annotations); !t.IsZero() && t.After(ect.trackerStartTime) {
			ect.lastChangeTriggerTimes[namespacedName] =
				append(ect.lastChangeTriggerTimes[namespacedName], t)
		}
	}

	return changeNeeded
}

type EndpointsMap map[ServicePortName][]Endpoint
合并changes中缓存的endpointsChange到EndpointsMap
func (em EndpointsMap) Update(changes *EndpointChangeTracker) (result UpdateEndpointMapResult) {
	result.StaleEndpoints = make([]ServiceEndpoint, 0)
	result.StaleServiceNames = make([]ServicePortName, 0)
	result.LastChangeTriggerTimes = make(map[types.NamespacedName][]time.Time)

	em.apply(
		changes, &result.StaleEndpoints, &result.StaleServiceNames, &result.LastChangeTriggerTimes)
	return result
}
```
