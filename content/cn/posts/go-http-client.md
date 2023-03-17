---
title: "Go Http Client连接复用问题"
date: 2023-03-12T23:03:00+08:00
categories: ['Golang','http.client']
keywords: ['Golang','http.client','源码阅读']
description: "Golang Http Client 源码阅读"
---

## 背景
发压工具一般都需要提供尽量大的发压能力，发压工具在从Python迁移到Golang的过程中遇到了两个HttpClient连接相关的问题：

{{< highlight go "linenos=table,linenostart=1" >}}
func NewHTTPClient() *http.Client {
	t := &http.Transport{
		DialContext: (&net.Dialer{
			Timeout:   30 * time.Second,
			KeepAlive: 30 * time.Second,
		}).DialContext,
		IdleConnTimeout:       180 * time.Second,
		TLSHandshakeTimeout:   10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
		MaxIdleConns:          800,
		MaxIdleConnsPerHost:   800,
	}
	return &http.Client{
		Transport: t,
		Timeout:   time.Second * 10,
	}
}

func callEndpoint() bool {
	client := NewHTTPClient()
	httpReq, _ := http.NewRequest("GET", "https://google.com", nil)
	res, _ := client.Do(httpReq)
	defer func() {
		_ = res.Body.Close()
	}()
	if res.StatusCode == 200 {
		return true
	}
	return false
}
{{< / highlight >}}
* Q1，上述代码为什么连接没有复用，而是频繁创建？
* Q2，连接池是如何复用的？


## 数据结构
{{< highlight go "linenos=table,linenostart=1" >}}
type Client struct {
    // 具体数据传输实现
	Transport RoundTripper

    // 在重定向前执行，可以用来返回错误终止重定向或者修改Header等
	CheckRedirect func(req *Request, via []*Request) error

    // 相当于浏览器里每个域下的cookie管理
	Jar CookieJar

    // 请求超时时间
	Timeout time.Duration
}

type RoundTripper interface {
	RoundTrip(*Request) (*Response, error)
}

// Transport 是一个RoundTripper的实现
type Transport struct {
	idleMu       sync.Mutex
	idleConn     map[connectMethodKey][]*persistConn // most recently used at end
	idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
	idleLRU      connLRU

	connsPerHostMu   sync.Mutex
	connsPerHost     map[connectMethodKey]int
	connsPerHostWait map[connectMethodKey]wantConnQueue // waiting getConns

    // 支持http、https、socks5代理
	Proxy func(*Request) (*url.URL, error)

    // 禁用长连接
	DisableKeepAlives bool

    // 最大空闲连接数，0代表不限制
	MaxIdleConns int

    // 单一Host的最大空闲连接数，0的话值为2
	MaxIdleConnsPerHost int

    // 单一Host的最大连接数，包含连接中、使用中、空闲状态下的，0代表不限制。
	MaxConnsPerHost int

    // 长连接能Idle多久才关闭，0代表不限制
	IdleConnTimeout time.Duration
}

type persistConn struct {
	t         *Transport
	cacheKey  connectMethodKey
	conn      net.Conn
	br        *bufio.Reader       // from conn
	bw        *bufio.Writer       // to conn
	nwrite    int64               // bytes written
	reqch     chan requestAndChan // written by roundTrip; read by readLoop
	writech   chan writeRequest   // written by roundTrip; read by writeLoop
	closech   chan struct{}       // closed when conn closed
	isProxy   bool
	sawEOF    bool  // whether we've seen EOF from conn; owned by readLoop
	readLimit int64 // bytes allowed to be read; owned by readLoop
    // 如果写入失败，则连接无法进行复用
	writeErrCh chan error

	// Both guarded by Transport.idleMu:
	idleAt    time.Time   // time it last become idle
	idleTimer *time.Timer // holding an AfterFunc to close it

	mu                   sync.Mutex // guards following fields
	closed               error // set non-nil when conn is closed, before closech is closed
	canceledErr          error // set non-nil if conn is canceled
	broken               bool  // an error has happened on this connection; marked broken so it's not reused.
	reused               bool  // whether conn has had successful request/response and is being reused.
}
{{< / highlight >}}

## 源码分析
[-> oneNote](https://1drv.ms/u/s!AlojCX6YGXCNeOX4M4DZcQXHLFk)

## 如何设置连接池参数
核心参数就是MaxIdleConns、MaxIdleConnsPerHost、MaxConnsPerHost和IdleConnTimeout。
如果不使用连接池，短连接不断地建立和关闭，会导致产生非常多的TIME_WAIT，最后本地端口用光服务就不可用了。		
如果只有一个Host，那MaxIdleConnsPerHost=MaxIdleConns，就只需要考虑最大连接数和最大空闲连接数了，具体还是看业务产生多大的并发调用。
比如上面代码，如果不设置MaxConnsPerHost，当并发数远超连接池大小时，依然会创建很多的连接，最后就会因没法放回连接池而频繁关闭。
这里也需要考虑闲时和忙时，如果MaxIdleConnsPerHost很大，忙时自然有优势，但是等到低峰期会存在大量空闲连接，所以这时候就很需要IdleConnTimeout，让连接自动关闭。

基本上database/sql连接池也类似，可参考[database/sql连接池源码分享](https://docs.google.com/presentation/d/1hqpyg88yupIbQg8ZjM9yXVhj6EqdSOEF9hqnYRzlGQw/edit#slide=id.g10894cbdb5b_0_26)


## 其他用法

### ClientTrace
http库也提供了ClientTrace来方便定位问题，ClientTrace是一堆hook方法的集合，比如可以用来分析DNS解析画了多少时间，或者通过PutIdleConn方法知道是否连接复用上有error发生
{{< highlight go "linenos=table,linenostart=1" >}}
type ClientTrace struct {
	GetConn func(hostPort string)

	GotConn func(GotConnInfo)

	PutIdleConn func(err error)

	GotFirstResponseByte func()

	Got100Continue func()

	Got1xxResponse func(code int, header textproto.MIMEHeader) error

	DNSStart func(DNSStartInfo)

	DNSDone func(DNSDoneInfo)

	ConnectStart func(network, addr string)

	ConnectDone func(network, addr string, err error)

	TLSHandshakeStart func()

	TLSHandshakeDone func(tls.ConnectionState, error)

	WroteHeaderField func(key string, value []string)

	WroteHeaders func()

	Wait100Continue func()

	WroteRequest func(WroteRequestInfo)
}
{{< / highlight >}}

### Cookie管理
如果是请求前登陆，然后后续不断地请求，就可以不用单独给每个request设置单独的cookie了，和浏览器的使用一样

### 代理设置
可以支持http、https、socks5代理