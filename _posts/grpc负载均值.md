



### 一、负载均衡算法

不实时计算负载的算法：轮询、加权轮询、随机、加权随机、哈希、一致性哈希

实时计算负载的算法：最快响应时间、最少连接数、最少请求数算法

### 二、gRPC 与负载均衡算法结合

gRPC 负载均衡算法最终会调用  buildLoadBalancingPolicy 进一步去判断, gRPC 的负载均衡需要实现以下接口,pick 中实现负载均衡算法。但是gRPC 无法实现用户请求的负载均衡算法。

```go
type Picker interface {
 
   Pick(info PickInfo) (PickResult, error)
}
```

```golang
type PickerBuilder interface {
   // Build returns a picker that will be used by gRPC to pick a SubConn.
   Build(info PickerBuildInfo) balancer.Picker
}
```

```go
	resolveBuild := NewRegistryBuild(registry, SetOptionCtxTimeOut(50*time.Second))
	balancer.Register(base.NewBalancerBuilder("BALANCE_ROTATION", &balanceRotationBuild{}, 	          base.Config{HealthCheck: true}))
	dial, err := grpc.Dial("regrity:///user-server",
		grpc.WithInsecure(),
		grpc.WithResolvers(resolveBuild),
		grpc.WithDefaultServiceConfig(`{"LoadBalancingPolicy":"BALANCE_ROTATION"}`),
	)
```

### 三、gRPC 负载均衡算法

###### 1、轮询

```go
type balanceRotationBuild struct {
}

func (b *balanceRotationBuild) Build(info base.PickerBuildInfo) balancer.Picker {
	cs := info.ReadySCs
	subConlist := make([]balancer.SubConn, 0, len(cs))
	for k := range cs {
		subConlist = append(subConlist, k)
	}
	t := pick{
		subConlist: subConlist,
		lenSub:     int64(len(subConlist)),
		index:      -1,
	}
	return &t
}

type pick struct {
	subConlist []balancer.SubConn
	lenSub     int64
	index      int64
}

func (p *pick) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
	if p.lenSub == 0 {
		return balancer.PickResult{}, balancer.ErrNoSubConnAvailable
	}
	atomic.AddInt64(&p.index, 1)
	con := p.subConlist[p.index%p.lenSub]
	return balancer.PickResult{
		SubConn: con,
		Done: func(info balancer.DoneInfo) {
			if info.Err != nil {
				return
			}
		},
	}, nil
}

```

###### 2、加权轮询 (平滑加权轮询算法)

平滑的加权轮询就是考虑动态调整权重来平滑实现的效果。算法基本原理是：

- 设置三个值：weight(权重)、currentWeight (当前权重)、efficientWeight (有效权重)
- efficientWeight 动态调整
- 每次挑选实例的时候，计算所有的实例的 efficientWeight 作为 totalWeight
- 对于每一个实例，更新currentWeight为 currentWeight+efficientWeight
- 挑选 currentWeight 最大的那个节点作为最终节点，并更新它的currentWeight 为 currentWeight-totalWeight

weight 权重应该是在注册中心设定的，在服务发现时，从注册中心拿出

```go
type balanceWeightBuild struct {
}

func (b balanceWeightBuild) Build(info base.PickerBuildInfo) balancer.Picker {
	result := make([]*SubConnInfo, 0, len(info.ReadySCs))
	for k, v := range info.ReadySCs {
		weight := v.Address.Attributes.Value("weight").(string)
		w, _ := strconv.Atoi(weight)
		result = append(result, &SubConnInfo{
			Conn:            k,
			Weight:          w,
			CurrentWeight:   w,
			EffectiveWeight: w,
		})
	}
	return &balancepick{
		Address: result,
	}
}

type balancepick struct {
	Address []*SubConnInfo
}

type SubConnInfo struct {
	Conn            balancer.SubConn
	Weight          int
	CurrentWeight   int
	EffectiveWeight int
	lock            sync.Mutex
}

func (b *balancepick) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
	if len(b.Address) == 0 {
		return balancer.PickResult{}, balancer.ErrNoSubConnAvailable
	}
	totalWeitht := 0
	var current *SubConnInfo
	for _, v := range b.Address {
		v.lock.Lock()
		totalWeitht += v.EffectiveWeight
		v.CurrentWeight += v.EffectiveWeight
		if current == nil || current.CurrentWeight < v.CurrentWeight {
			current = v
		}
		v.lock.Unlock()
	}
	current.lock.Lock()
	current.CurrentWeight -= totalWeitht
	current.lock.Unlock()
	return balancer.PickResult{
		SubConn: current.Conn,
		Done: func(info balancer.DoneInfo) {
			current.lock.Lock()
			if info.Err == nil && current.EffectiveWeight == math.MaxInt {
				current.EffectiveWeight--
				return
			}
			if info.Err != nil && current.EffectiveWeight == 0 {
				current.EffectiveWeight++
				return
			}
			if info.Err != nil {
				current.EffectiveWeight--
			} else {
				current.EffectiveWeight++
			}
			current.lock.Unlock()

		},
	}, nil
}
```

###### 3、负载均衡：随机

就是随机选择服务器

```golang
type balanceRandomBuild struct {
}

func (b balanceRandomBuild) Build(info base.PickerBuildInfo) balancer.Picker {
   result := make([]balancer.SubConn, len(info.ReadySCs))
   for k := range info.ReadySCs {
      result = append(result, k)
   }
   return RandomPic{
      conlist: result,
      lenList: len(result),
   }
}

type RandomPic struct {
   conlist []balancer.SubConn
   lenList int
}

func (r RandomPic) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
   if r.lenList == 0 {
      return balancer.PickResult{}, balancer.ErrNoSubConnAvailable
   }
   index := rand.Intn(r.lenList + 1)
   return balancer.PickResult{
      SubConn: r.conlist[index],
      Done: func(info balancer.DoneInfo) {
         if info.Err != nil {
            return
         }
      },
   }, nil
}
```

###### 4、最小链接数

选择一个链接最小的节点，但是需要注意的是链接最小的节点并不一定是最优的那个节点

```golang
type balanceActiveBuild struct {
}

func (b balanceActiveBuild) Build(info base.PickerBuildInfo) balancer.Picker {
   cs := info.ReadySCs
   subConlist := make([]*actionvNode, 0, len(cs))
   for k := range cs {
      subConlist = append(subConlist, &actionvNode{
         con: k,
      })
   }
   return actionvPick{
      list: subConlist,
   }
}

type actionvNode struct {
   con balancer.SubConn
   cnt uint32
}

type actionvPick struct {
   list []*actionvNode
}

func (a actionvPick) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
   var currentNode *actionvNode
   for _, v := range a.list {
      if atomic.LoadUint32(&v.cnt) <= v.cnt {
         currentNode = v
      }
   }
   atomic.AddUint32(&currentNode.cnt, 1)
   return balancer.PickResult{
      SubConn: currentNode.con,
      Done: func(info balancer.DoneInfo) {
         atomic.AddUint32(&currentNode.cnt, -1)
      },
   }, nil
}
```
