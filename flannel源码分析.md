# flannel 源码分析

## subnet.Manager
子网管理器
```go
type Manager interface {
    // 获取整个集群网络配置 10.214.0.0/16
    GetNetworkConfig(ctx context.Context) (*Config, error)  
    // 生成每台主机子网配置环境变量文件/run/flannel/subnet.env  10.214.10.0/24
    HandleSubnetFile(path string, config *Config, ipMasq bool, sn ip.IP4Net, ipv6sn ip.IP6Net, mtu int) error
    // 获取当前主机子网配置租约
    AcquireLease(ctx context.Context, attrs *LeaseAttrs) (*Lease, error)
    // 刷新租约
    RenewLease(ctx context.Context, lease *Lease) error
    // watch租约
    WatchLease(ctx context.Context, sn ip.IP4Net, sn6 ip.IP6Net, receiver chan []LeaseWatchResult) error
    WatchLeases(ctx context.Context, receiver chan []LeaseWatchResult) error
    CompleteLease(ctx context.Context, lease *Lease, wg *sync.WaitGroup) error

    Name() string
}
``` 

#### LocalManager 通过etcd Register实现 subnet.Manager
```go
type LocalManager struct {
    // 实现子网配置的后端register，localManager通过etcd实现register
    registry               Registry
    previousSubnet         ip.IP4Net
    previousIPv6Subnet     ip.IP6Net
    subnetLeaseRenewMargin int
}
// register接口
type Registry interface {
    // 获取集群子网配置
    getNetworkConfig(ctx context.Context) (string, error)
    // 获取已注册子网
    getSubnets(ctx context.Context) ([]Lease, int64, error)
    // 获取子网
    getSubnet(ctx context.Context, sn ip.IP4Net, sn6 ip.IP6Net) (*Lease, int64, error)
    // 创建子网
    createSubnet(ctx context.Context, sn ip.IP4Net, sn6 ip.IP6Net, attrs *LeaseAttrs, ttl time.Duration) (time.Time, error)
    // 更新子网
    updateSubnet(ctx context.Context, sn ip.IP4Net, sn6 ip.IP6Net, attrs *LeaseAttrs, ttl time.Duration, asof int64) (time.Time, error)
    // 删除子网
    deleteSubnet(ctx context.Context, sn ip.IP4Net, sn6 ip.IP6Net) error
    // watch子网
    watchSubnets(ctx context.Context, leaseWatchChan chan []LeaseWatchResult, since int64) error
    watchSubnet(ctx context.Context, since int64, sn ip.IP4Net, sn6 ip.IP6Net, leaseWatchChan chan []LeaseWatchResult) error
    leasesWatchReset(ctx context.Context) (LeaseWatchResult, error)
}

// etcdSubnetRegistry结构体通过etcd存储实现register接口
type etcdSubnetRegistry struct {
    cliNewFunc   etcdNewFunc
    mux          sync.Mutex
    kvApi        etcd.KV
    cli          *etcd.Client
    etcdCfg      *EtcdConfig
    networkRegex *regexp.Regexp
}
```

#### LocalManager 获取子网租约过程
```go
func (m *LocalManager) AcquireLease(ctx context.Context, attrs *subnet.LeaseAttrs) (*subnet.Lease, error) {
    // 获取集群网络配置
    config, err := m.GetNetworkConfig(ctx)
    ...
    // 获取租约
    for i := 0; i < raceRetries; i++ {
        l, err := m.tryAcquireLease(ctx, config, attrs.PublicIP, attrs)
    }
    ...
}

func (m *LocalManager) tryAcquireLease(ctx context.Context, config *subnet.Config, extIaddr ip.IP4, attrs *subnet.LeaseAttrs) (*subnet.Lease, error) {
    // 获取已存在子网
    leases, _, err := m.registry.getSubnets(ctx)
    ...
    // 查找对应子网是否已存在
    // Try to reuse a subnet if there's one that matches our IP
    if l := findLeaseByIP(leases, extIaddr); l != nil {
        // Make sure the existing subnet is still within the configured network
        if isSubnetConfigCompat(config, l.Subnet) && isIPv6SubnetConfigCompat(config, l.IPv6Subnet) {
            ...
            exp, err := m.registry.updateSubnet(ctx, l.Subnet, l.IPv6Subnet, attrs, ttl, 0)
            ...
            return l, nil
        } else {
            ...
            if err := m.registry.deleteSubnet(ctx, l.Subnet, l.IPv6Subnet); err != nil {
                return nil, err
            }
        }
    }
    // 不存在则创建子网
    // no existing match, check if there was a previous subnet to use
    var sn ip.IP4Net
    var sn6 ip.IP6Net
    // 主机之前分配的子网存在
    if !m.previousSubnet.Empty() {
        // use previous subnet
        if l := findLeaseBySubnet(leases, m.previousSubnet); l == nil {
            ...
        }
    }
    if sn.Empty() {
        // 从可分配子网中，随机选择一个子网分配
        // no existing match, grab a new one
        sn, sn6, err = m.allocateSubnet(config, leases)
        ...
    }
    //在register中创建子网
    exp, err := m.registry.createSubnet(ctx, sn, sn6, attrs, subnetTTL)
    ...
}
```

## backend.Manager 
主机子网后端实现管理器
```go
// 接口定义
type Manager interface {
    GetBackend(backendType string) (Backend, error)
}
// 具体实现
type manager struct {
    ctx      context.Context
    // 子网管理器，获取主机子网
    sm       subnet.Manager
    extIface *ExternalInterface
    mux      sync.Mutex
    // 具体子网后端实现
    active   map[string]Backend
    wg       sync.WaitGroup
}
```

#### vxlan模式子网后端
```go
type VXLANBackend struct {
    // 子网管理器
    subnetMgr subnet.Manager
    // 外网接口
    extIface  *backend.ExternalInterface
}
// vxlan网络注册
func (be *VXLANBackend) RegisterNetwork(ctx context.Context, wg *sync.WaitGroup, config *subnet.Config) (backend.Network, error) {
    ...
    // 生成vxlan虚拟设备
    var dev, v6Dev *vxlanDevice
    var err error
    if config.EnableIPv4 {
        devAttrs := vxlanDeviceAttrs{
            vni:       uint32(cfg.VNI),
            name:      fmt.Sprintf("flannel.%v", cfg.VNI),
            MTU:       cfg.MTU,
            vtepIndex: be.extIface.Iface.Index,
            vtepAddr:  be.extIface.IfaceAddr,
            vtepPort:  cfg.Port,
            gbp:       cfg.GBP,
            learning:  cfg.Learning,
        }

        dev, err = newVXLANDevice(&devAttrs)
        if err != nil {
            return nil, err
        }
        dev.directRouting = cfg.DirectRouting
    }
    ...
    // 获取子网租约
    lease, err := be.subnetMgr.AcquireLease(ctx, subnetAttrs)
    
    // 通过获取的子网租约配置vxlan设备
    if config.EnableIPv4 {
        net, err := config.GetFlannelNetwork(&lease.Subnet)
        if err != nil {
            return nil, err
        }
        if err := dev.Configure(ip.IP4Net{IP: lease.Subnet.IP, PrefixLen: 32}, net); err != nil {
            return nil, fmt.Errorf("failed to configure interface %s: %w", dev.link.Attrs().Name, err)
        }
    }
    ...
    // 生成主机网络
    return newNetwork(be.subnetMgr, be.extIface, dev, v6Dev, ip.IP4Net{}, lease, cfg.MTU)
}

type network struct {
    backend.SimpleNetwork
    dev       *vxlanDevice
    v6Dev     *vxlanDevice
    subnetMgr subnet.Manager
    mtu       int
}
// 通过WatchLeases获取subnet更新事件，设置网络
func (nw *network) handleSubnetEvents(batch []subnet.Event) {
	for _, event := range batch {
        ...
		switch event.Type {
		case subnet.EventAdded:
			if event.Lease.EnableIPv4 {
				if directRoutingOK {
                   ...
				} else {
                    // 设置arp
					if err := retry.Do(func() error {
						return nw.dev.AddARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})
					}); err != nil {
						log.Error("AddARP failed: ", err)
						continue
					}
                    // 设置fdb
					if err := retry.Do(func() error {
						return nw.dev.AddFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})
					}); err != nil {
                        ...
					}

					// Set the route - the kernel would ARP for the Gw IP address if it hadn't already been set above so make sure
					// this is done last.
					// 设置路由
					if err := retry.Do(func() error {
						return netlink.RouteReplace(&vxlanRoute)
					}); err != nil {
                        ...
					}
				}
			}
            ...
		case subnet.EventRemoved:
            ...
		default:
			log.Error("internal error: unknown event type: ", int(event.Type))
		}
	}
}
```
