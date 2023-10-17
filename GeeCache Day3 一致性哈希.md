# 一致性哈希

## 实现内容

一致性哈希算法/分布式缓存节点访问机制

## 一致性哈希的原理

一致性哈希算法将 key 映射到 2^32 的空间中，将这个数字首尾相连，形成一个环。

- 如何划分节点对应缓存Key：

  计算节点/机器(通常使用节点的名称、编号和 IP 地址)的哈希值，放置在环上。

- 如何通过Key确定该访问的缓存节点：

  计算 key 的哈希值，放置在环上，顺时针寻找到的第一个节点，就是应选取的节点/机器。

- 如何完成节点增删：

  重新确定节点对应缓存Key哈希区间，并将一部分涉及到的缓存迁移

![一致性哈希添加节点 consistent hashing add peer](https://tuchuang-1318639513.cos.ap-beijing.myqcloud.com/images/202310171455254.jpeg)

- 如何解决数据倾斜问题：

  数据倾斜/Key倾斜的根本原因是缓存节点过少。

  引入虚拟节点，一个真实节点对应多个虚拟节点，先将缓存落到虚拟节点上再实际落到其对应的真实节点上，并一个字典(map)维护真实节点与虚拟节点的映射关系。

## 简单Hash为什么不能胜任分布式缓存节点访问机制

简单哈希（如散列）可以解决该访问哪个缓存节点的问题，但当缓存节点出现了变化如新增或者删除时，由于哈希函数本身的重要参数发生了变化，所有缓存都可能被波及/所有的缓存值都失效了。节点在接收到对应的请求时，均需要重新去数据源获取数据，容易引起缓存雪崩。

## Hash环的实现

缓存节点添加

```go
// Map 一致性哈希算法的主数据结构
type Map struct {
	hash         Hash           // 一致性哈希所用哈希函数，默认使用crc32.ChecksumIEEE
	copyMultiple int            // 虚拟节点倍数即一个真实缓存节点对应多少虚拟几点
	keys         []int          // 哈希环，有序的（加入先后有序？没太理解这里有序的含义），保存虚拟节点的哈希值
	hashMap      map[int]string // 字典，存储真实节点与虚拟节点的映射关系
}

// 添加节点
func (m *Map) Add(keys ...string) {
	m.Lock()
	defer m.Unlock()
	newValues := m.loadValues()
	for _, key := range keys {
		// 对每个 key(节点) 创建 m.replicas 个虚拟节点
		for i := 0; i < m.replicas; i++ {
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			newValues.keys = append(newValues.keys, hash)
			newValues.hashMap[hash] = key
		}
	}
	sort.Ints(newValues.keys)
	m.values.Store(newValues)
}
```

哈希环m.keys实际上只存储了虚拟节点的哈希值，作者在注释中强调了keys的有序性。~~但我并不能理解其有序的意义，实际实现上可没有体现。~~有序性实际上依赖特殊的二分查找函数实现。

缓存节点选择

```
// Get 根据key值选出需要访问的缓存节点
func (m *Map) Get(key string) string {
	if len(m.keys) == 0 {
		return ""
	}
	hashValue := int(m.hash([]byte(key)))
	idx := sort.Search(len(m.keys), func(i int) bool {
		return m.keys[i] > hashValue
	})
	return m.hashMap[m.keys[idx%len(m.keys)]]

}
```

因为 sort.Search 使用的比较函数是 keys[i] >= hashValue ，使用二分查找我们能找到第一个"m.keys[i] >= hashValue"的 i 。当这个 i 不存在时，idx 获得的值为 len(m.keys) ，这就需要取余了，因为从某个虚拟节点逆时针一侧(不跨越哈希环起终点时，哈希值小于它本身)开始直到下一个虚拟节点结束的哈希环才是它的对应缓存Key的哈希值范围。