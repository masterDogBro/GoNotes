# 防止缓存击穿

## 实现内容

使用 singleflight 防止缓存击穿，实现与测试

## BigKey缓存失效

在单机并发缓存的部分，已经出现了类似的问题。

```
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

在缓存未命中的情况下，Group.load被调用。但如果针对同一个key的两个或更多并发Get，同时出现了缓存未命中的情况，load也会返回两次。两次返回值的一致性可能因为用户自定义的Group.getter的不同而无法保证。而且除了一致性之外，两次重复的直接向数据源获取缓存值的回调也可能会增加数据源的压力，造成性能下降。

##  singleflight的作用

```
// call 进行中或已经结束的缓存请求
type call struct {
	wg  sync.WaitGroup // WaitGroup并发等待组锁，避免重入
	val interface{}
	err error
}

// CallMap singelefight的主要结构，负责管理不同key的call
type CallMap struct {
	mu sync.Mutex
	m  map[string]*call
}

func (cm *CallMap) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	cm.mu.Lock()
	// CallMap没有New函数，采用延迟初始化
	if cm.m == nil {
		cm.m = make(map[string]*call)
	}

	// 当cm.m[key]已经存在时，意味着已经有对相同key的call请求完成了步骤（1），但还没有完成步骤（2）
	// 此时相同的请求再次到达，只需等待之前相同key的call完成，并返回其结果
	if c, ok := cm.m[key]; ok {
		cm.mu.Unlock()
		c.wg.Wait() // 等待，直至WaitGroup锁为0
		return c.val, c.err
	}

	// 当cm.m[key]不存在时，（1）创建新call并将其加入CallMap，并将WaitGroup锁+1
	c := new(call)
	c.wg.Add(1)
	cm.m[key] = c
	cm.mu.Unlock()

	// （2）调用fn函数，发起请求。再调用完成后，将WaitGroup锁-1
	c.val, c.err = fn()
	c.wg.Done()

	// （3）更新CallMap，删除已完成的call
	cm.mu.Lock()
	delete(cm.m, key)
	cm.mu.Unlock()

	// （4）返回结果
	return c.val, c.err

}
```

CallMap将对某一个Key的Call存入字典中， 并将这个字典用并发锁保护起来。

当某一个key缓存请求进入Do方法，首先会在CallMap中确定已经有对相同key的Call请求。

（0）如果已经有，只需等待之前相同key的call完成（等待到该Call的WaitGroup锁为0），并返回其结果

（1）如果没有，则创建新call并将其加入CallMap，并将WaitGroup锁+1

（2）call调用fn函数，发起请求。在调用完成后，将WaitGroup锁-1

（3）更新CallMap，删除已完成的call

（4）返回结果
