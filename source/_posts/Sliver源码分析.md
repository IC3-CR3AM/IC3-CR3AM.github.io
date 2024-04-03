---
title: Sliver源码分析
date: 2024-04-03 18:26:07
tags:
---
<!-- # 流水账跟读分析 -->

## 服务端

server/main进去实现了一个cobra，自定义了参数，例如grpc开放的端口和host

### 初始化

1. 创建隐藏配置文件的文件夹
2. 初始化ca和wg的key等等密码学相关参数
3. 初始化服务器启动相关参数，有就从文件里读
4. 接着启动c2和console的持久化job
5. 最后启动daemon或者console的grpc服务

### c2 job揭秘

该job是为了创建Listener设立，根据配置文件打开mlts、wg、DNS和Http监听（总共只有这四种类型的监听器）

其核心为创建一个job结构体

```go
type Job struct {
    ID           int       // The unique identifier for the job
    Name         string    // The name of the job
    Description  string    // A detailed description of what the job does
    Protocol     string    // The protocol that the job uses
    Port         uint16    // The port that the job uses
    Domains      []string  // An array of domains that the job covers
    JobCtrl      chan bool // A control channel for the job
    PersistentID string    // A persistent identifier for the job
}
```

其中id为唯一递增序列号

jobCtrl为控制job是否关闭的chan

并在最后把Job放入一个全局结构体中

吐槽：只有四种监听器，不够灵活，不过结构好直接自定义也行，不知msf如何写一个万能监听器的。

### console-admin job揭秘

这个job是为了客户端接入服务的，打开给客户端连接的grpc接口，采用的和上面的是一套job结构。

同样，它的grpc一套也是给和daemon的grpc公用的

### 服务端GRPC API解析 TODO

## 客户端

同样是cobra起手，初始化日志记录，根据读取的配置文件连接服务器，建立mtls的grpc连接

使用grumble建立交互式控制台

1. 初始化`SliverConsoleClient` 对象
    
    ```go
    type SliverConsoleClient struct {
    	App                      *grumble.App
    	Rpc                      rpcpb.SliverRPCClient
    	ActiveTarget             *ActiveTarget
    	EventListeners           *sync.Map
    	BeaconTaskCallbacks      map[string]BeaconTaskCallback
    	BeaconTaskCallbacksMutex *sync.Mutex
    	IsServer                 bool
    	Settings                 *assets.ClientSettings
    }
    ```
    
2. 绑定命令
    - 从文件加载Reaction事件
    - 从`alias.json`中加载别名到sliver shell中
    - 从`extension.json`中解析json并加载扩展
    - 将`Aliases`、`Armory`、`Update`、`Jobs`、`Operators`、`Reconfig`、`Sessions`、`Close`、`Tasks`、`Use`、`Settings`、`Info`、`Shell`、`Shellcode Encoders`、`Exec`、`Generate`、`Filesystem`、`Network`、`Privileges`、`Websites`、`Screenshot`、`Backdoor`、`Beacons`、`Environment`、`Licenses`、`Registry`、`Reverse Port Forwarding`、`Pivots`、`WireGuard`、`Portfwd`、`Socks`、`Monitor`、`Loot`、`Hosts`、`Reactions`、`DLL Hijack`、`Get Privs`、`Extensions`、`Prelude's Operator`、`Curse Commands`、`Builders`等加入命令
3. 增加`Observers`当session发生变化时提醒
4. 创新线程调用event的grpc stream
5. 根据事件类型触发相应的Reaction
6. 解析传入的隧道消息，并将其分发到会话/隧道对象中
7. 跑命令行，循环读取键盘输入

## 局部分析

### Tunnel代码分析

**client**

```go
type TunnelIO struct {
	ID        uint64
	SessionID string

	Send chan []byte
	Recv chan []byte

	isOpen bool
	mutex  *sync.RWMutex
}
```

整体上是一个client stream的grpc调用，其底层是TunnelIO ，上面由tunnels管理，再由tunnel负责调用。调用会触发server端rpc-tunnel，其实现在server/core/tunnels中

**server**

```go
type Tunnel struct {
	ID        uint64
	SessionID string

	ToImplant         chan []byte
	ToImplantSequence uint64

	FromImplant         chan *sliverpb.TunnelData
	FromImplantSequence uint64

	Client rpcpb.SliverRPC_TunnelDataServer

	mutex               *sync.RWMutex
	lastDataMessageTime time.Time
}
```

本身这个tunnel是全双工，上层不需要操心，但是sliver只把这个作为将implant返回消息转发给client的通道，其他调用都是有其他rpc实现

**implant**

```go
type Tunnel struct {
	ID uint64

	// Reader       io.ReadCloser
	Readers      []io.ReadCloser
	readSequence uint64

	Writer        io.WriteCloser
	writeSequence uint64

	mutex *sync.RWMutex
}
```

implant如此实现的好处很明显，结构化标准化，不需要一次传输大量数据，只要将数据加载进缓存，根据需要发送数据，

那么转发实现的结构是什么呢？

1. implant先封装数据到envelope，再序列化成`TunnelData`，其中包含数据的顺序，因为传输并不会顺序传输
2. implant向server的listener发起mtls/http/……等，建立连接并传输数据
3. server获取各种调用的句柄
    
    ```go
    map[uint32]ServerHandler{
    		// Sessions
    		sliverpb.MsgRegister:    registerSessionHandler,
    		sliverpb.MsgTunnelData:  tunnelDataHandler,
    		sliverpb.MsgTunnelClose: tunnelCloseHandler,
    		sliverpb.MsgPing:        pingHandler,
    		sliverpb.MsgSocksData:   socksDataHandler,
    
    		// Beacons
    		sliverpb.MsgBeaconRegister: beaconRegisterHandler,
    		sliverpb.MsgBeaconTasks:    beaconTasksHandler,
    
    		// Pivots
    		sliverpb.MsgPivotPeerEnvelope: pivotPeerEnvelopeHandler,
    		sliverpb.MsgPivotPeerFailure:  pivotPeerFailureHandler,
    	}
    ```
    
4. server读取conn中的数据，有些需要先解密，后根据其中的type进行调用（sliver规定了一套implant回传数据的结构）
5. 执行tunnelDataHandler，先匹配session id再判断是否要建立反向tunnel，不需要才执行根据id获取tunnel，调用`SendDataFromImplant`加入缓存队列
6. 最后再客户端发起的`TunnelData`将数据通过grpc返回

### 注册的吐槽

```go
&sliverpb.Register{
		Name:              consts.SliverName,
		Hostname:          hostname,
		Uuid:              uuid,
		Username:          currentUser.Username,
		Uid:               currentUser.Uid,
		Gid:               currentUser.Gid,
		Os:                runtime.GOOS,
		Version:           version.GetVersion(),
		Arch:              runtime.GOARCH,
		Pid:               int32(os.Getpid()),
		Filename:          filename,
		ReconnectInterval: int64(transports.GetReconnectInterval()),
		ConfigID:          "{{ .Config.ID }}",
		PeerID:            pivots.MyPeerID,
		Locale:            locale.GetLocale(),
	}
```

从impalnt开始就要实现这个grpc，封装好之后调用服务端grpc进行注册，无非中间的信道可以换

信息获取太多，不够灵活，获取这些信息容易触动edr，需要更精简

最精简的形式，应该是生成一个uuid，然后返回，server可以收到id和外部ip

### 密码系统实现分析 TODO

### Event代码分析

使用的是`PerRPCCredentials`，并且配置为block，只有在连接成功时才继续代码

## Sliver Implant分析

sliver的implant算是比较庞大的一种beacon了，造成这样的原因有三：

1. go编译带上运行时和混淆
2. 用donut转换成.net assembly
3. implant携带了太多的功能

### 流程

1. 初始化
2. 判断执行环境
3. 注册svc服务运行、beacon心跳运行、session直连运行三选一

**Beacon**

开始一个beacon循环根据`.Config.ConnectionStrategy`选择连接策略，生成对应beacon

```go
type Beacon struct {
	Init    BeaconInit
	Start   BeaconStart
	Send    BeaconSend
	Recv    BeaconRecv
	Close   BeaconClose
	Cleanup BeaconCleanup

	ActiveC2 string
	ProxyURL string
}
```

所谓生成beacon就是完成上述实例的初始化，在start内完成相关连接

有了Beacon后进入`beaconMainLoop`

调用对应beacon初始化，确认回连时间、向sliver server发送注册请求

注册结构如下：

```go
 &sliverpb.BeaconRegister{
		ID:          InstanceID,
		Interval:    beacon.Interval(),
		Jitter:      beacon.Jitter(),
		Register:    register,
		NextCheckin: int64(beacon.Duration().Seconds()),
	}
```

注册之后才是跑主程序

先发心跳后接收，反序列化后加锁执行任务，最后返回id和结果

结构非常清晰

# 主体架构

![主体架构.png](struct.jpg)

<aside>
💡 这里有个疑问，在第五步的时候，<-implantConnection.Send并不是缓存队列，而心跳并不会这么快的获取数据，理应在这里阻塞。那么一旦client再次发送请求就会因为这个send还未发出而被丢弃数据

</aside>