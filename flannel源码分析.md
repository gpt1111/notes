# flannel 源码分析

## subnet.Manager
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
