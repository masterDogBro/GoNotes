# 单机并发缓存

## 实现内容

LRU方法的并发支持，缓存值（entry,Value）数据结构实现，GROUP（负责用户交互，缓存系统主体逻辑控制）数据结构实现

## 并发支持如何实现？

sync.Mutex 是 Go 语言标准库提供的一个互斥锁，当一个协程(goroutine)获得了这个锁的拥有权后，其它请求锁的协程(goroutine) 就会阻塞在 Lock() 方法的调用上，直到调用 Unlock() 锁被释放。

## 单机并发实现

### 缓存值抽象ByteView实现

**实际缓存值ByteView为何需要只读？**

可能实际目的只是为了保护原始值，防止意料之外的修改。

```go
// ByteView 缓存值结构体，只读
type ByteView struct {
	b []byte // 缓存的真实值，为支持任意类型的数据类型的存储采用byte类型
}
```

不过这个ByteView.b虽然有小写限制的，但go的切片变量其实是类似指针的存在，虽然限制了b本身不可变，但b包含的地址写的数据是可能被改变的。

只读的实现：深拷贝。

```go
// ByteSlice ByteView只读，为防止修改只返回拷贝值
func (bv ByteView) ByteSlice() []byte {
	return cloneBytes(bv.b)
}

// cloneBytes 深复制，返回b的拷贝
func cloneBytes(b []byte) []byte {
	c := make([]byte, len(b))
	copy(c, b)
	return c
}
```

### 并发控制实现/LRU封装

**延迟初始化的意义**

在 LRU 的封装， cache 本身并没有初始化函数，不能在自己初始化时初始化 lru 。这里的实现方法是延迟初始化(Lazy Initialization)。

```go
// cache 将lru.Cache的方法封装为单机可并行的（可对外部开放的主要是Add和Get两个方法）
type cache struct {
	mu         sync.Mutex
	lru        *lru.Cache
	cacheBytes int64
}

func (c *cache) add(key string, value ByteView) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.lru == nil { // cache本身没有
		c.lru = lru.New(c.cacheBytes, nil)
	}
	c.lru.Add(key, value)
}
```

在 add 方法中，判断了 c.lru 是否为 nil，如果等于 nil 再创建实例。这种方法称之为延迟初始化(Lazy Initialization)，一个对象的延迟初始化意味着该对象的创建将会延迟至第一次使用该对象时。主要用于提高性能，并减少程序内存要求。

### 系统控制逻辑

系统主体逻辑（从接收到key开始）

![group系统主体逻辑](https://tuchuang-1318639513.cos.ap-beijing.myqcloud.com/images/202310161441272.png)

分布式缓存系统代码结构（原始）

```
geecache/
    |--lru/
        |--lru.go  // lru 缓存淘汰策略
    |--byteview.go // 缓存值的抽象与封装
    |--cache.go    // 并发控制
    |--geecache.go // 负责与外部交互，控制缓存存储和获取的主流程
```

### 回调函数的意义与实现

当缓存不存在，从数据源中获取数据并添加到缓存中。

但现有数据源种类太多，每个都实现非常繁琐，还限制了未来出现新数据源之后缓存系统的扩展性。所以我们应该定义一个接口，让实现了接口的函数或结构体能使用我们的回调函数。

实际的接口实现：

```go
// Getter Getter接口，含义一个方法：回调函数Get
type Getter interface {
	Get(key string) ([]byte, err)
}

// GetterFunc 函数类型，定义其参数和返回值格式
type GetterFunc func(key string) ([]byte, error)

// Get GetterFunc类型的方法Get
func (f GetterFunc) Get(key string) ([]byte, error) {
	return f(key)
}
```

如此实现接口Getter是为了接口使用的便利性

```go
func GetFromSource(getter Getter, key string) []byte {
   buf, err := getter.Get(key)
   if err == nil {
      return buf
   }
   return nil
}
```

比如以上函数，getter既可以传入GetterFunc类型的匿名函数或者可以转化为GetterFunc类型的普通函数，还可以传入实现了Get方法的结构体。

### 核心数据结构Group

```go
type Group struct {
	name      string // 唯一名称
	getter    Getter // 缓存未命中时调用的回调接口
	mainCache cache  // 并发缓存
}

var (
	mu     sync.RWMutex              // 全局/系统读写锁
	groups = make(map[string]*Group) // group字典，保存系统所有Group地址
)
```

Group是缓存系统中某一个缓存的命名空间/抽象，整个缓存系统的缓存地址存储在一个字典中，用Group.name查找。

Group的Get方法封装了缓存命中和缓存未命中单机处理的两种情况，与远程节点的交互后续会实现。

```go
// Get 从缓存系统中获取缓存值，并对缓存命中，未命中情况处理（目前未命中只有单机实现）
func (g *Group) Get(key string) (ByteView, error) {
	if key == "" {
		return ByteView{}, fmt.Errorf("nil key")
	}

	// 缓存命中
	if v, ok := g.mainCache.get(key); ok {
		log.Printf("key: %s hit cache", key)
		return v, nil
	}

	// 缓存未命中
	return g.load(key)
}
```

