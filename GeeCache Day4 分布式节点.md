# 分布式节点

## 实现内容

实现节点注册逻辑，并入之前实现的一致性哈希缓存节点访问机制

实现HTTP客户端，与远程节点的服务端通信

## 与远程节点交互的逻辑

![group系统主体逻辑](https://tuchuang-1318639513.cos.ap-beijing.myqcloud.com/images/202310161441272.png)

与远程节点交互这个逻辑，按照现有的HTTP通信形式和一致性哈希节点选择机制可以进行细化。

![远程节点交互逻辑 (1)](C:/Users/MasterDogBro/Downloads/远程节点交互逻辑 (1).png)

```
// Overall flow char										     requsets					        local
// gee := createGroup() --------> /api Service : 9999 ------------------------------> gee.Get(key) ------> g.mainCache.Get(key)
// 						|											^					|
// 						|											|					|remote
// 						v											|					v
// 				cache Service : 800x								|			g.peers.PickPeer(key)
// 						|create hash ring & init peerGetter			|					|
// 						|registry peers write in g.peer				|					|p.httpGetters[p.hashRing(key)]
// 						v											|					|
//			httpPool.Set(otherAddrs...)								|					v
// 		g.peers = gee.RegisterPeers(httpPool)						|			g.getFromPeer(peerGetter, key)
// 						|											|					|
// 						|											|					|
// 						v											|					v
// 		http.ListenAndServe("localhost:800x", httpPool)<------------+--------------peerGetter.Get(key)
// 						|											|
// 						|requsets									|
// 						v											|
// 					p.ServeHttp(w, r)								|
// 						|											|
// 						|url.parse()								|
// 						|--------------------------------------------
```

## PeerPicker的实现

```
// PeerPicker PickPeer方法基于key选择对应缓存节点peer，并返回其PeerGetter
type PeerPicker interface {
	PickPeer(key string) (peer PeerGetter, ok bool)
}

// PeerGetter Get方法从对应Group查找key的缓存值并返回
type PeerGetter interface {
	Get(group string, key string) ([]byte, error)
}
```

PeerPicker和PeerGetter都是作为接口是实现的，这是后续通信形式扩展做好准备。

```
// HTTPPool 为 HTTP 对等体池实现了 PeerPicker。
// 基于一致性hash来实现缓存节点选择
type HTTPPool struct {
	self        string                 // 主机地址/IP:端口号
	bashPath    string                 // 节点间通讯地址的前缀默认值采用defaultBasePath
	mu          sync.Mutex             // 互斥锁的守护对象是peers和httpGetters
	peers       *consistenthash.Map    // 一致性哈希算法实体
	httpGetters map[string]*httpGetter // 保存httpGetter的字典，其key为缓存节点地址如："http://10.0.0.2:8008"
}

// PickPeer 根据所寻找缓存的key，来确定缓存节点并返回其PeerGetter。如果没有找到或者找到的是自己，则返回nil
func (hp *HTTPPool) PickPeer(key string) (PeerGetter, bool) {
    hp.mu.Lock()
    defer hp.mu.Unlock()
    if peer := hp.peers.Get(key); peer != "" && peer != hp.self { // 确保了不会返回自己而陷入循环
       hp.Log("Pick peer %s", peer)
       return hp.httpGetters[peer], true
    }
    return nil, false
}
```

HTTPPool实现了PeerPikcer接口。

```
type httpGetter struct {
	baseURL string
}

func (h *httpGetter) Get(group string, key string) ([]byte, error) {
	getUrl := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(group),
		url.QueryEscape(key),
	)
	res, errG := http.Get(getUrl)
	if errG != nil {
		return nil, errG
	}

	// GoLand提示我“未封装错误”，小改一下
	defer func(Body io.ReadCloser) {
		errC := Body.Close()
		if errC != nil {
			return
		}
	}(res.Body)

	// 处理服务器返回错误
	if res.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("server returned: %v", res.Status)
	}

	// 处理Response读取错误
	bytes, errR := io.ReadAll(res.Body)
	if errR != nil {
		return nil, fmt.Errorf("reading response body: %v", errR)
	}

	return bytes, nil
}
```

httpGetter实现了PeerGetter方法

## 分布式缓存系统的构建与启动

我们需要两类服务器，一类是缓存节点服务器，另外一类是注册中心/用户API服务器。

### 用户API服务器

用户API服务器响应用户请求。在测试中，实际上用户API服务器和某一个缓存节点服务器逻辑上部署在同一个服务器实体上。故所有用户请求都是该服务器实体响应的。

```
// startAPIServer 用来启动一个 API 服务，与用户进行交互
func startAPIServer(apiAddr string, gee *geecache.Group) {
    http.Handle("/api", http.HandlerFunc(
       func(w http.ResponseWriter, r *http.Request) {
          key := r.URL.Query().Get("key")
          view, err := gee.Get(key)
          if err != nil {
             http.Error(w, err.Error(), http.StatusInternalServerError)
             return
          }
          w.Header().Set("Content-Type", "application/octet-stream")
          w.Write(view.ByteSlice())

       }))
    log.Println("fontend server is running at", apiAddr)
    log.Fatal(http.ListenAndServe(apiAddr[7:], nil))
}
```

但用户API服务器并没有完成缓存节点的注册功能（HTTPPool不完善），虽然Group有相应的peers增减接口，但注册中心没有写得很完善，Group对应的缓存节点peers初始化完成之后就不再变动。

TODO: 分布式缓存系统注册中心的完善。

### 缓存节点服务器

```
// startCacheServer 用来启动缓存服务器：创建 HTTPPool，添加节点信息，注册到缓存空间中，启动 HTTP 服务
func startCacheServer(addr string, addrs []string, gee *geecache.Group) {
    peers := geecache.NewHTTPPool(addr)
    peers.Set(addrs...)
    gee.RegisterPeers(peers)
    log.Println("geecache is running at", addr)
    log.Fatal(http.ListenAndServe(addr[7:], peers))
}
```
