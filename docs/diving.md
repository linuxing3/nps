# 深入代码学习NPS

## 总体架构

- 客户端 client (server, flow, mode)
- 服务端 server (mode as Service, bridge as NetBridge, task as Tunnel)
- 服务 service (connection, bridge, tunnel, flow, data)

## 程序入口

### cmd命令板块

在终端窗口执行命令，可以进入我们的程序

### 服务端命令行入口

在服务器上，执行nps命令，将调用以下文件的功能

[`nps.go`](/cmd/nps/nps.go) 包含服务端

执行顺序如下

- 定义一个文件对象，加载一个网络服务器的隧道，命名为task
- 使用beego的应用配置获取桥接端口
- 使用服务器的StartNewServer方法，将端口，任务和服务器配置等作为参数，执行启动

```go
package main
import	"ehang.io/nps/server"

type nps struct {
	exit chan struct{}
}

func run() {
	task := &file.Tunnel{
		Mode: "webServer",
	}
	bridgePort, err := beego.AppConfig.Int("bridge_port")
  // 开启一个web服务器，作为前端，并允许添加新的服务 
	go server.StartNewServer(bridgePort, task, beego.AppConfig.String("bridge_type"), timeout)
}
```

### 客户端命令入口
[`/cmd/npc/npc.go`](/cmd/npc/npc.go)包含客户端的`main`入口


## Server服务器板块

[`/server/server.go`][/server/server.go]服务器定义

服务器的启动顺序如下
- 调用网桥端口
- 配置文件隧道
- 确定网桥类型
- 必要时断开网桥

```go

package server // 定义为server包

import (
	/nps/bridge
	/nps/lib/common
	/nps/lib/file
	/nps/server/proxy
	/nps/server/tool
)
// 主要函数, 启动服务器，处理客户端请求，获取流，写入到通道
func StartNewServer(
    bridgePort int,  // 网桥
    cnf *file.Tunnel,  // 通道
    bridgeType string,  // 网桥类型
    bridgeDisconnect int // 断开
    )

// 网桥创建通道
Bridge := bridge.NewTunnel(
    bridgePort,
    bridgeType,
    ip,
    RunList,
    bridgeDisconnect
    )

// 启动通道
Bridge.StartTunnel()

// 执行任务
	go DealBridgeTask()

// 处理客户流，流里包含网络【请求】数据
	go dealClientFlow()   ===> dealClientData()

// 按模式创建服务器并启动, c.Mode 是模式"tcp","socks5"
	svr := NewMode(Bridge *bridge.Bridge, c *file.Tunnel)
  svr.Start()

// 运行列表
  RunList.Store(cnf.Id, svr)
    
```

#### 对应不同的网络协议，启动不同的服务器模式，包括`tcp`, `udp`, `sock5`等等

[`/server/server.go`][/server/server.go]服务器模式的切换

```go
//new a server by mode name
func NewMode(Bridge *bridge.Bridge, c *file.Tunnel) proxy.Service {
	var service proxy.Service
	switch c.Mode {
	case "tcp", "file":
		service = proxy.NewTunnelModeServer(proxy.ProcessTunnel, Bridge, c)
	case "socks5":
		service = proxy.NewSock5ModeServer(Bridge, c)
	case "httpProxy":
		service = proxy.NewTunnelModeServer(proxy.ProcessHttp, Bridge, c)
	case "tcpTrans":
		service = proxy.NewTunnelModeServer(proxy.HandleTrans, Bridge, c)
	case "udp":
		service = proxy.NewUdpModeServer(Bridge, c)
	case "webServer":
		InitFromCsv()
		t := &file.Tunnel{
			Port:   0,
			Mode:   "httpHostServer",
			Status: true,
		}
		AddTask(t)
		service = proxy.NewWebServer(Bridge)
	case "httpHostServer":
		httpPort, _ := beego.AppConfig.Int("http_proxy_port")
		httpsPort, _ := beego.AppConfig.Int("https_proxy_port")
		useCache, _ := beego.AppConfig.Bool("http_cache")
		cacheLen, _ := beego.AppConfig.Int("http_cache_length")
		addOrigin, _ := beego.AppConfig.Bool("http_add_origin_header")
		service = proxy.NewHttp(Bridge, c, httpPort, httpsPort, useCache, cacheLen, addOrigin)
	}
	return service
}
```

## Proxy代理板块

这里是代理功能的真正实现，主要是利用了`net.Listener`监听端口。

对收到的客户端数据进行分析，确定对象隧道，并进行数据的来回拷贝。

### 基础服务器

[`/server/proxy/base.go`](/server/proxy/base.go)中定义了服务借口，基础服务器接口

BaseServer的最主要方法
- DealClient 处理客户端请求
- FlowAddHost 数据流添加主机
- FlowAdd  数据流扩容

```go
// base 任何一个服务借口必须可以启动和关闭
type Service interface {
	Start() error
	Close() error
}
// 任何网桥可以发送连接信息，包括客户id，连接，隧道等等，并返回一个连接
type NetBridge interface {
	SendLinkInfo(clientId int, link *conn.Link, t *file.Tunnel) (target net.Conn, err error)
}

//BaseServer struct 基础服务器必须包括网桥和隧道，并且容错，同步加锁
type BaseServer struct {
	id           int
	bridge       NetBridge // 网桥创建通道
	task         *file.Tunnel // [任务]指向[通道]的指针
	errorContent []byte
	sync.Mutex  // 同步锁
}

// 创建连接并开始拷贝字节, conn是在/lib/conn/conn.go和link.go中定义
// create a new connection and start bytes copying
func (s *BaseServer) DealClient(c *conn.Conn, client *file.Client, addr string, rb []byte, tp string, f func(), flow *file.Flow, localProxy bool) error {
  // 1. 获取连接
	link := conn.NewLink(tp, addr, client.Cnf.Crypt, client.Cnf.Compress, c.Conn.RemoteAddr().String(), localProxy)
  // 2. 通过客户端id和连接，获取net.Conn对象
	if target, err := s.bridge.SendLinkInfo(client.Id, link, s.task); err != nil {
		logs.Warn("get connection from client id %d  error %s", client.Id, err.Error())
		c.Close()
		return err
	} else {
		if f != nil {
			f()
		}
    // 开始拷贝数据
		conn.CopyWaitGroup(target, c.Conn, link.Crypt, link.Compress, client.Rate, flow, true, rb)
	}
	return nil
}
```

### tcp服务器
[`/nps/server/proxy/tcp.go`](/nps/server/proxy/tcp.go)定义了tcp服务器的内容

隧道模式服务器，可以处理和监听进程及通道

```go
type TunnelModeServer struct {
	BaseServer
	process  process // 处理进程, 主要是对通道进行操作
	listener net.Listener // 监听器
}

//tcp|http|host
func NewTunnelModeServer(process process, bridge NetBridge, task *file.Tunnel) *TunnelModeServer {
	s := new(TunnelModeServer)
	s.bridge = bridge
	s.process = process
	s.task = task
	return s
}

// 启动新的Tcp监听器并粗粒素具
func (s *TunnelModeServer) Start() error {
	return conn.NewTcpListenerAndProcess(s.task.ServerIp+":"+strconv.Itoa(s.task.Port), func(c net.Conn) {
		if err := s.CheckFlowAndConnNum(s.task.Client); err != nil {
			logs.Warn("client id %d, task id %d,error %s, when tcp connection", s.task.Client.Id, s.task.Id, err.Error())
			c.Close()
			return
		}
		logs.Trace("new tcp connection,local port %d,client %d,remote address %s", s.task.Port, s.task.Client.Id, c.RemoteAddr())
		s.process(conn.NewConn(c), s)
		s.task.Client.AddConn()
	}, &s.listener)
}

type process func(c *conn.Conn, s *TunnelModeServer) error // 处理进程

func ProcessTunnel(c *conn.Conn, s *TunnelModeServer) error {
	targetAddr, err := s.task.Target.GetRandomTarget()
	if err != nil {
		c.Close()
		logs.Warn("tcp port %d ,client id %d,task id %d connect error %s", s.task.Port, s.task.Client.Id, s.task.Id, err.Error())
		return err
	}
  // 让[通道模式服务器]处理客户端请求
	return s.DealClient(c, s.task.Client, targetAddr, nil, common.CONN_TCP, nil, s.task.Flow, s.task.Target.LocalProxy)
}
func ProcessHttp(c *conn.Conn, s *TunnelModeServer) error 
```


## Bridge网桥板块

## Client客户端板块

### 理解一些基础的定义

[`/lib/file/obj.go`](/lib/file/obj.go)中定义了通道、客户端、数据流等

```go
type Flow struct {
	ExportFlow int64
	InletFlow  int64
	FlowLimit  int64
	sync.RWMutex
}

type Client struct {
	Cnf             *Config
	Id              int        //id
	VerifyKey       string     //verify key
	Addr            string     //the ip of client
	Remark          string     //remark
	Status          bool       //is allow connect
	IsConnect       bool       //is the client connect
	RateLimit       int        //rate /kb
	Flow            *Flow      //flow setting
	Rate            *rate.Rate //rate limit
	NoStore         bool       //no store to file
	NoDisplay       bool       //no display on web
	MaxConn         int        //the max connection num of client allow
	NowConn         int32      //the connection num of now
	WebUserName     string     //the username of web login
	WebPassword     string     //the password of web login
	ConfigConnAllow bool       //is allow connected by config file
	MaxTunnelNum    int
	Version         string
	sync.RWMutex
}

type Tunnel struct {
	Id           int
	Port         int
	ServerIp     string
	Mode         string  // tcp, socks5 etc
	Status       bool
	RunStatus    bool
	Client       *Client // 客户端
	Ports        string
	Flow         *Flow   // 流
	Password     string
	Remark       string
	TargetAddr   string
	NoStore      bool
	LocalPath    string
	StripPre     string
	Target       *Target
	MultiAccount *MultiAccount
	Health
	sync.RWMutex
}

```
## Gui用户界面板块

## Web网络界面板块

