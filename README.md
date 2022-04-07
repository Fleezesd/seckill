##  秒杀系统的设计与实现

满足 大并发   高性能  和 高可用的分布式系统

而秒杀系统考虑  1.瞬时大并发

​		2.超卖

​		3.性能

而这三方面的架构核心理念  需要通过缓存 异步  限流 来保证服务的高并发和高可用

### 1. 项目架构

整套架构 分为

```go
从原来的单体架构 转为现在的微服务架构
1. 专门的秒杀系统
2. 依赖统一的用户和鉴权系统
3. 系统都注册到统一的服务注册中心
4. 依赖统一的配置中心进性服务配置
5. 服务都统一在API网关之后
6. 专门的链路监控系统 监控整个系统的运行情况
```

交互流程

```go
前端/移动端应用 通过 网关 与后端进行网络交互  网络请求   api-gateway  
			|
接入系统有 用户鉴权 负载均衡 以及 限流和熔断器 (即每个请求处理需要的基础组件)  pkg  初始化所需组件
			|
后端核心逻辑有 用户登录 秒杀处理  秒杀活动管理 系统降级等  sk-app   sk-admin  sk-core...
			(这些服务注册到服务注册中心, 通过配置中心 进性自身业务数据的配置)
			|
链路监控时刻监控着系统的状态   zipkin 埋点监控
			|
最底层是 缓存层的Redis 以及 数据持久化层Mysql和Zookeeper
```



### 2.流程简介

#### 1. 秒杀业务系统

```go
用户进行秒杀时, 先与业务系统进行交互     
秒杀业务系统 主要负责对  1. 对请求进行限流
					 2. 用户黑白名单过滤
					 3. 并发限制和用户数据签名校验

业务流程
1. 从Zookeeper中加载 秒杀活动数据 到内存中
2. 从Redis中加载 黑名单数据 到内存中
3. 对用户进行  黑名单限制 限流
4. 对用户进行签名校验
5. 将用户请求 通过Redis 传递给 业务核心系统  进行处理
6. 接收业务核心系统的处理结构 返回给用户
```

#### 2.  秒杀核心系统

```go
负责进行真正的秒杀逻辑判断  1.依次处理Redis队列中的用户请求 
			   2.限制用户的购买次数
			   3.并对获得抢购资格的用户生成对应的资格token


业务流程
1. 从Zookeeper中加载 秒杀活动数据 到内存中
2. 处理Redis队列中秒杀业务系统 传递过来的请求
3. 限制用户的购买次数
4. 对商品的抢购频次进行限制
5. 对合法的请求给予生成抢购资格的 Token 并通过Redis 传递给秒杀业务系统
```

#### 3. 秒杀管理系统

```go
服务于秒杀活动管理人员 进行 活动信息 和 秒杀商品信息 的管理

业务流程
1. 配置并管理商品数据 提供对商品数据增加和查询接口
2. 配置并管理秒杀活动数据 提供对秒杀活动的增加和查询接口
3. 将秒杀活动数据 同步到Zookeeper
4. 将秒杀活动数据 持久化到数据库
```



### 3. 整个微服务脚手架

#### 1. 服务注册和发现

pkg/discover    通过Consul作为服务发现与注册中心组件  各个核心业务服务都注册到Consul并查询要通信的服务实例信息

##### 1. 服务实例相关的抽象接口和结构体

1.服务实例

```go
属性: 主机ip  HTTP网路服务的端口号 负载均衡  RPC服务的端口号

type ServiceInstance struct{
    Host   	  string
    Port   	     int
    Weight 	     int
    CurWeight        int
    GrpcPort         int
}
```

2.服务注册与发现客户端

```go
三个方法  Register DeRegister  DiscoverServices

type DiscoveryClient interface{
    Register()
    DeRegister()
    DiscoverServices()
}
```

上述接口的接收器 和一些额外属性

```go
type DiscoveryClientInstance struct{
    Host  string
    Port  int
    // 连接Consul的配置
    config *api.Config
    client  consul.Client
    mutex   sync.Mutex
    // 服务实例缓存字段
    instancesMap  sync.Map
}
```

##### 2. 注册服务 注销服务  查询服务 方法的实现

...  详见Service Discovery即可

##### 3. 后续操作

该组件为后续 load balance组件会有对其的使用

load balance 组件使用它获取服务实例列表 根据一定策略 进行负载均衡



#### 2. 负载均衡策略

##### 1.负载均衡策略的接口

```go
// 负载均衡器
type LoadBalance interface {
    SelectService(service []*common.ServiceInstance)(*common.ServiceInstance, error)
}
```

##### 2.选取策略

带权重的平滑轮询策略  对于负载均衡器的相应接收器

```go
type WeightRoundRobinLoadBalance struct{}

// 权重平滑负载均衡
func(loadbalance *WeightRoundRobinLoadBalance) SelectService()(){
    // 相应实现
}   

累加所有 weight权重值为total  curWeight +=weight -total 更新相应curWeight 选取最优解即可
实现相应的负载均匀
```

#### 3.  RPC 客户端装饰器

构建 RPC客户端装饰器组件,  用于统一封装业务提供的RPC接口服务端

##### 1. 以鉴权系统为例

 Auth Client 示例

```go
type OAuthClient interface{
    CheckToken()
}
```

OAuthClientImpl结构体 定义了客户端管理器 服务名称 负载均衡策略和链路追踪系统

```go
type OAuthClientImpl struct{
    manager      ClientManager
    serviceName  string
    loadbalance  loadbalance
    tracer       opentracing.Tracer
}
```

OAuthClientImpl实现Check Token方法 

而使用该RPC客户端的业务服务即可初始化相应的client即像调本地方法一样

##### 2. client的装饰器方法

还是以鉴权系统相应的为例 因CheckToken 调用了ClientManager的DecoratorInvoke方法

并把RPC请求路径 方法名称 链路追踪 请求上下文 以及请求与响应传递到方法中

```go
DecoratorInvoke   // 1.链路追踪 openTracing.tracer  回调等等
				// 2. Hystrix.Do 实现相应的断路器保护  即熔断机制  达到限流效果
			    // 3. 服务发现
			    // 4. 负载均衡
			    // 5. RPC端口调用远程请求 获取响应值
			    // 6. after的回调函数
```

#### 4. 限流

##### 1. 漏桶算法

请求先进入到漏桶 漏桶以一定速率出水     																		限定流出速率

##### 2. 令牌桶算法

一个存放固定容量令牌的漏桶  请求获取到令牌 直接处理   											  限定流入速率

​												获取不到令牌 要么被限流 要么被丢弃 要么被丢缓冲区

##### 3. 项目使用的限流算法

```go
使用标准库的限流组件				golang.org/x/time/rate   	基于令牌桶限流算法实现的

// 后续补上  以waitgroup和channel 来实现令牌桶算法
```

#### 5. Go Redis

重点说下 list 列表类型 作为本项目的使用的Redis数据结构

```go
因其支持LPush
BRPop操作   移除并获取最后一个元素
		  若列表没有元素  会阻塞列表直到 等到超时 或 发现可弹出元素 即可
		  实现生产者和消费者队列模式
```

#### 6. Zoo keeper集成

​		分布式服务框架

解决分布式应用中经常遇到的 数据管理问题

##### 1.项目使用Zoo keeper

本秒杀项目 1.将 秒杀活动 和 秒杀商品 的信息存储在Zoo keeper中  其他服务可以加载  

​			 	  2. 使用watch机制 实时更新信息

##### 2. Zoo Keeper 连接

```go
func initZk(){
    1. 初始化ZK
    2. 连接 获取ZK的Conn
    3. 设置活动数据 或者商品信息时
    4. Conn.Exists 判断相应zkPath是否存在值
    Set 和 Create 来对相应路径设置值和判断
    
    5. zk.WithEventCallback 来实现相应的watch机制 进行回调
}
```



### 4.秒杀核心逻辑

#### 1.秒杀业务系统

##### 1. 通用数据结构

```go
秒杀活动 	 Activity  
秒杀商品信息 ProductInfo
秒杀请求	 SecRequest			用户信息 用户权限码
秒杀响应     SecResponse		秒杀结果 秒杀成功的购买Token
通过protobuf展示
```

##### 2.秒杀业务系统开发

```go
主要为前端和移动端 1.提供 秒杀活动查询  秒杀的HTTP端口
2.处理有关用户和IP黑白名单的流量限制的逻辑
3.通过Redis 将合法的秒杀请求 发送给 秒杀核心业务
4.把秒杀业务的处理结果 返回给移动端或前端

业务流程
1. 从Zookeeper中加载 秒杀活动数据 到内存中
2. 从Redis中加载 黑名单数据 到内存中
3. 对用户进行  黑名单限制 限流
4. 对用户进行签名校验
5. 将用户请求 通过Redis 传递给 业务核心系统  进行处理
6. 接收业务核心系统的处理结构 返回给用户
```

###### 1.初始化秒杀数据

```go
skApp/setup/zk.go
1. 启动时从Zk中加载秒杀活动数据到内存中  即secProductInfo
// 初始化ZK
func InitZk() {
	var hosts = []string{"39.98.179.73:2181"}
	//option := zk.WithEventCallback(waitSecProductEvent)
	conn, _, err := zk.Connect(hosts, time.Second*5)
	if err != nil {
		fmt.Println(err)
		return
	}

	conf.Zk.ZkConn = conn
	conf.Zk.SecProductKey = "/product"
	loadSecConf(conn)  // 加载秒杀商品信息
}


// 加载秒杀商品信息
func loadSecConf(conn *zk.Conn) {
	log.Printf("Connect zk sucess %s", conf.Zk.SecProductKey)
	v, _, err := conn.Get(conf.Zk.SecProductKey) 
	if err != nil {
		log.Printf("get product info failed, err : %v", err)
		return
	}
	log.Printf("get product info ")
	var secProductInfo []*conf.SecProductInfoConf
	err1 := json.Unmarshal(v, &secProductInfo)
	if err1 != nil {
		log.Printf("Unmsharl second product info failed, err : %v", err1)
	}
	updateSecProductInfo(secProductInfo) // 更新配置信息
}
```

###### 2.建立 Redis 连接并加载黑名单信息

```go
1. 初始化Redis 建立连接
//初始化Redis
func InitRedis() {
	log.Printf("init redis %s", conf.Redis.Password)
	client := redis.NewClient(&redis.Options{
		Addr:     conf.Redis.Host,
		Password: conf.Redis.Password,
		DB:       conf.Redis.Db,
	})

	_, err := client.Ping().Result()
	if err != nil {
		log.Printf("Connect redis failed. Error : %v", err)
	}
	log.Printf("init redis success")
	conf.Redis.RedisConn = client

	loadBlackList(client)  // 加载黑名单信息
	initRedisProcess()
}


2. 加载黑名单列表
//加载黑名单列表
func loadBlackList(conn *redis.Client) {
	conf.SecKill.IPBlackMap = make(map[string]bool, 10000)
	conf.SecKill.IDBlackMap = make(map[int]bool, 10000)

	//用户Id
	idList, err := conn.HGetAll(conf.Redis.IdBlackListHash).Result()

	if err != nil {
		log.Printf("hget all failed. Error : %v", err)
		return
	}

	for _, v := range idList {
		id, err := com.StrTo(v).Int()
		if err != nil {
			log.Printf("invalid user id [%v]", id)
			continue
		}
		conf.SecKill.IDBlackMap[id] = true
	}

	//用户Ip
	ipList, err := conn.HGetAll(conf.Redis.IpBlackListHash).Result()
	if err != nil {
		log.Printf("hget all failed. Error : %v", err)
		return
	}

	for _, v := range ipList {
		conf.SecKill.IPBlackMap[v] = true
	}

	//go syncIpBlackList(conn)
	//go syncIdBlackList(conn)
	return
}

syncIp/IdBlacklist 无限循环地使用Redis的BRPop方法阻塞获取队列中的数据 然后更新或者新增IDBlackMap中的数据
//同步用户ID黑名单
func syncIdBlackList(conn *redis.Client) {
	for {
		idArr, err := conn.BRPop(time.Minute, conf.Redis.IdBlackListQueue).Result()
		if err != nil {
			log.Printf("brpop id failed, err : %v", err)
			continue
		}
		id, _ := com.StrTo(idArr[1]).Int()
		conf.SecKill.RWBlackLock.Lock()
		{
			conf.SecKill.IDBlackMap[id] = true  // 更新IdBlackMap数据
		}
		conf.SecKill.RWBlackLock.Unlock()
	}
}


3. 初始化Redis进程
//初始化redis进程
func initRedisProcess() {
	log.Printf("initRedisProcess %d %d", conf.SecKill.AppWriteToHandleGoroutineNum, conf.SecKill.AppReadFromHandleGoroutineNum)
	for i := 0; i < conf.SecKill.AppWriteToHandleGoroutineNum; i++ {
        // sk-app/service/srv_redis/redis_proc.go
		go srv_redis.WriteHandle()  // 写数据到redis
	}

	for i := 0; i < conf.SecKill.AppReadFromHandleGoroutineNum; i++ {
		go srv_redis.ReadHandle()  // 从redis读取数据
	}
}
```



交互核心

```go
//redis配置
type RedisConf struct {
	RedisConn            *redis.Client //链接
	Proxy2layerQueueName string        //队列名称
	Layer2proxyQueueName string        //队列名称
	IdBlackListHash      string        //用户黑名单hash表
	IpBlackListHash      string        //IP黑名单Hash表
	IdBlackListQueue     string        //用户黑名单队列
	IpBlackListQueue     string        //IP黑名单队列
	Host                 string
	Password             string
	Db                   int
}

LPush BRPop 队列 操作

秒杀业务系统和秒杀核心系统 通过Redis队列交互

SerReqChan---------> Proxy2layerQueueName --------->  Read2HandleChan----
														      Handler
resultChan<---------Layer2proxyQueueName<------------ Handle2WriteChan--

sk-app				Redis								sk-core
```

###### 3.启动HTTP服务

业务层启动的最后一步是初始化服务

transport层

```go
路由 分配 /sec/list  /sec/info /sec/kill
```

endpoint层对应

```go
GetSecInfoListEndpoint  GetSecInfoEndpoint  SecKillEndPoint
```

service层逻辑处理

```go
主要说Service层的 Seckill 秒杀逻辑

	  黑名单校验
		|
	   流量限制
		|
	 商品秒杀信息校验
		|
推入到SecReqChan  (redis)  传递给秒杀核心系统
		|
	根据情况处理
	|    |    	   |   
超时处理 报错处理 结果返回处理
(select 语句  针对不同情况)      

主要用途: 并未进行真正的 秒杀核心逻辑 的处理, 而是将 合法的秒杀请求 通过Redis交给 秒杀核心系统 处理
```

#### 3.秒杀核心系统

业务流程

```go
负责进行真正的秒杀逻辑判断  1.依次处理Redis队列中的用户请求 
					    2.限制用户的购买次数
					    3.并对获得抢购资格的用户生成对应的资格token


业务流程
1. 从Zookeeper中加载 秒杀活动数据 到内存中
2. 处理Redis队列中秒杀业务系统 传递过来的请求
3. 限制用户的购买次数
4. 对商品的抢购频次进行限制
5. 对合法的请求给予生成抢购资格的 Token 并通过Redis 传递给秒杀业务系统
```

main.go

```go
func main() {

	setup.InitZk()
	setup.InitRedis()
	setup.RunService()

}
```

##### 1. Redis相关初始化

```go
1. 启动redis
func RunProcess() {
	for i := 0; i < conf.SecKill.CoreReadRedisGoroutineNum; i++ {
		go HandleReader()
	}

	for i := 0; i < conf.SecKill.CoreWriteRedisGoroutineNum; i++ {
		go HandleWrite()
	}

	for i := 0; i < conf.SecKill.CoreHandleGoroutineNum; i++ {
		go HandleUser()
	}

	log.Printf("all process goroutine started")
	return
}

```

##### 2. HandleReader

```go
1.将Redis的App2CoreQueue队列中的数据转换成业务层能处理的数据 
2.并推入到Read2HandleChan，
3.同时进行超时判断  设置超时时间和超时回调
```

```go
func HandleReader() {
	log.Printf("read goroutine running %v", conf.Redis.Proxy2layerQueueName)
	for {
		conn := conf.Redis.RedisConn
		for {
			//从Redis队列中取出数据
			data, err := conn.BRPop(time.Second, conf.Redis.Proxy2layerQueueName).Result()
			if err != nil {
				continue
			}
			log.Printf("brpop from proxy to layer queue, data : %s\n", data)

			//转换数据结构
			var req config.SecRequest
			err = json.Unmarshal([]byte(data[1]), &req)
			if err != nil {
				log.Printf("unmarshal to secrequest failed, err : %v", err)
				continue
			}

			//判断是否超时
			nowTime := time.Now().Unix()
			//int64(config.SecLayerCtx.SecLayerConf.MaxRequestWaitTimeout)
			fmt.Println(nowTime, " ", req.SecTime, " ", 100)
			if nowTime-req.SecTime >= int64(conf.SecKill.MaxRequestWaitTimeout) {
				log.Printf("req[%v] is expire", req)
				continue
			}

			//设置超时时间
			timer := time.NewTicker(time.Millisecond * time.Duration(conf.SecKill.CoreWaitResultTimeout))
			select {
			case config.SecLayerCtx.Read2HandleChan <- &req:
			case <-timer.C:
				log.Printf("send to handle chan timeout, req : %v", req)
				break
			}
		}
	}
}
```

##### 3. HandleUser

```go
1.会从Read2HandleChan中读取请求， 
2.然后调用 Seckill函数 对用户请求进行秒杀处理 
3.将返回结果推入到Handle2WriteChan中等待结果写入Redis
4.将上述操作 进行设置超时时间和超时回调
```

```go
func HandleUser() {
	log.Println("handle user running")
	for req := range config.SecLayerCtx.Read2HandleChan {
		log.Printf("begin process request : %v", req)
		res, err := HandleSeckill(req)
		if err != nil {
			log.Printf("process request %v failed, err : %v", err)
			res = &config.SecResult{
				Code: srv_err.ErrServiceBusy,
			}
		}
		fmt.Println("处理中~~ ", res)
		timer := time.NewTicker(time.Millisecond * time.Duration(conf.SecKill.SendToWriteChanTimeout))
		select {
		case config.SecLayerCtx.Handle2WriteChan <- res:
		case <-timer.C:
			log.Printf("send to response chan timeout, res : %v", res)
			break
		}
	}
	return
}
```

##### 4. HandleSeckill

```go
1.限制用户购买次数  对商品抢购频次 频率限制
2.对合法的请求基于生成抢购资格Token令牌
```

```go
func HandleSeckill(req *config.SecRequest) (res *config.SecResult, err error) {
	config.SecLayerCtx.RWSecProductLock.RLock()
	defer config.SecLayerCtx.RWSecProductLock.RUnlock()

	res = &config.SecResult{}
	res.ProductId = req.ProductId
	res.UserId = req.UserId

	product, ok := conf.SecKill.SecProductInfoMap[req.ProductId]
	if !ok {
		log.Printf("not found product : %v", req.ProductId)
		res.Code = srv_err.ErrNotFoundProduct
		return
	}

	if product.Status == srv_err.ProductStatusSoldout {
		res.Code = srv_err.ErrSoldout
		return
	}
	nowTime := time.Now().Unix()

	config.SecLayerCtx.HistoryMapLock.Lock()
	userHistory, ok := config.SecLayerCtx.HistoryMap[req.UserId]
	if !ok {
		userHistory = &srv_user.UserBuyHistory{
			History: make(map[int]int, 16),
		}
		config.SecLayerCtx.HistoryMap[req.UserId] = userHistory
	}
	historyCount := userHistory.GetProductBuyCount(req.ProductId)
	config.SecLayerCtx.HistoryMapLock.Unlock()

	if historyCount >= product.OnePersonBuyLimit {
		res.Code = srv_err.ErrAlreadyBuy
		return
	}

	curSoldCount := config.SecLayerCtx.ProductCountMgr.Count(req.ProductId)

	if curSoldCount >= product.Total {
		res.Code = srv_err.ErrSoldout
		product.Status = srv_err.ProductStatusSoldout
		return
	}

	//curRate := rand.Float64()
	curRate := 0.1
	fmt.Println(curRate, product.BuyRate)
	if curRate > product.BuyRate {
		res.Code = srv_err.ErrRetry
		return
	}

	userHistory.Add(req.ProductId, 1)
	config.SecLayerCtx.ProductCountMgr.Add(req.ProductId, 1)

	//用户Id、商品id、当前时间、密钥

	res.Code = srv_err.ErrSecKillSucc
	tokenData := fmt.Sprintf("userId=%d&productId=%d&timestamp=%d&security=%s", req.UserId, req.ProductId, nowTime, conf.SecKill.TokenPassWd)
	res.Token = fmt.Sprintf("%x", md5.Sum([]byte(tokenData))) //MD5加密
	res.TokenTime = nowTime

	return
}
```

##### 5.HandleWrite

```go
1.将HandleUser写入Handle2WriteChan处理数据拉取出来
2.调用sendtoRedis发松到Redis的Core2AppQueue队列中  等待秒杀业务系统会将其拉取
```

```go
 func HandleWrite() {
	log.Println("handle write running")

	for res := range config.SecLayerCtx.Handle2WriteChan {
		fmt.Println("===", res)
		err := sendToRedis(res)
		if err != nil {
			log.Printf("send to redis, err : %v, res : %v", err, res)
			continue
		}
	}
}
```

#### 4.秒杀管理系统

业务流程

```go
1. 将秒杀活动信息和商品信息存储在Mysql 
2. 同时将秒杀活动信息和商品信息 同步到ZooKeeper中
```

系统层次

```go
与秒杀业务层类似
1.通过Go-kit的transport层来提供HTTP服务接口
2.endpoint层将HTTP请求转发给service层对应方法
```

##### 1.CreateActivity 

实现信息存储到Mysql 并调用SyncToZk方法同步到Zookeeper中

```go
func (p ActivityServiceImpl) CreateActivity(activity *model.Activity) error {
	log.Printf("CreateActivity")
	//写入到数据库
	activityEntity := model.NewActivityModel()
	err := activityEntity.CreateActivity(activity)
	if err != nil {
		log.Printf("ActivityModel.CreateActivity, err : %v", err)
		return err
	}

	log.Printf("syncToZk")
	//写入到Zk
	err = p.syncToZk(activity)
	if err != nil {
		log.Printf("activity product info sync to etcd failed, err : %v", err)
		return err
	}
	return nil
}

func (p *ActivityModel) CreateActivity(activity *Activity) error {
	conn := mysql.DB()
	_, err := conn.Table(p.getTableName()).Data(
		map[string]interface{}{
			"activity_name": activity.ActivityName,
			"product_id":    activity.ProductId,
			"start_time":    activity.StartTime,
			"end_time":      activity.EndTime,
			"total":         activity.Total,
			"sec_speed":     activity.Speed,
			"buy_limit":     activity.BuyLimit,
			"buy_rate":      activity.BuyRate,
		},
	).Insert()
	if err != nil {
		return err
	}
	return nil
}
```

##### 2.SyncToZk方法

```go
将新创建的Activity数据同步到Zookeeper中,

1. 首先会从Zookeeper中拉取存储数据, 如果数据为空  转为secProductInfoList 
	即观察响应zk 路径是否存在数据 Exists ————> Set/Update
2. 然后将新创建的Activity添加到列表中 更新Zookeeper
```

```go
func (p ActivityServiceImpl) syncToZk(activity *model.Activity) error {

	zkPath := conf.Zk.SecProductKey
	secProductInfoList, err := p.loadProductFromZk(zkPath)
	if err != nil {
		secProductInfoList = []*model.SecProductInfoConf{}
	}

	var secProductInfo = &model.SecProductInfoConf{}
	secProductInfo.EndTime = activity.EndTime
	secProductInfo.OnePersonBuyLimit = activity.BuyLimit
	secProductInfo.ProductId = activity.ProductId
	secProductInfo.SoldMaxLimit = activity.Speed
	secProductInfo.StartTime = activity.StartTime
	secProductInfo.Status = activity.Status
	secProductInfo.Total = activity.Total
	secProductInfo.BuyRate = activity.BuyRate
	secProductInfoList = append(secProductInfoList, secProductInfo)

	data, err := json.Marshal(secProductInfoList)
	if err != nil {
		log.Printf("json marshal failed, err : %v", err)
		return err
	}

	conn := conf.Zk.ZkConn

	var byteData = []byte(string(data))
	var flags int32 = 0
	// permission
	var acls = zk.WorldACL(zk.PermAll)

	// create or update
	exisits, _, _ := conn.Exists(zkPath)
	if exisits {
		_, err_set := conn.Set(zkPath, byteData, flags)
		if err_set != nil {
			fmt.Println(err_set)
		}
	} else {
		_, err_create := conn.Create(zkPath, byteData, flags, acls)
		if err_create != nil {
			fmt.Println(err_create)
		}
	}

	log.Printf("put to zk success, data = [%v]", string(data))
	return nil
}
```













