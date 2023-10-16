# HTTP 服务端

## 实现内容

使用 Go 语言标准库 http 搭建 HTTP Server

## http.ListenAndServe的实现

```go
package main

import (
	"log"
	"net/http"
)

type server int

func (h *server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	log.Println(r.URL.Path)
	w.Write([]byte("Hello World!"))
}

func main() {
	var s server
	http.ListenAndServe("localhost:9999", &s)
}
```

以上代码用于一个基本HTTP服务器的创建。

```go
http.ListenAndServe("localhost:9999", &s)
```

ListenAndServe的第一个参数用于接收服务器启动地址，第二个参数为Handler。任何实现了 ServeHTTP 方法的对象都可以作为 HTTP 的 Handler。

### ServeHTTP的错误处理

```go
func (hp *HTTPPool) ServeHTTP(writer http.ResponseWriter, r *http.Request) {
	if !strings.HasPrefix(r.URL.Path, hp.bashPath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	hp.Log("%s %s", r.Method, r.URL.Path)
	// url path 格式应为 /<basepath>/<groupname>/<key>
	parts := strings.SplitN(r.URL.Path[len(hp.bashPath):], "/", 2)
	if len(parts) != 2 { // url path 格式错误
		http.Error(writer, "bad request", http.StatusBadRequest)
		return
	}

	groupName := parts[0]
	key := parts[1]

	group := GetGroup(groupName)
	if group == nil { // 缓存对象示例不存在
		http.Error(writer, "no such group: "+groupName, http.StatusNotFound)
		return
	}

	view, err := group.Get(key)
	if err != nil { // 缓存查询报错
		http.Error(writer, err.Error(), http.StatusInternalServerError)
		return
	}

	// 未发生任何错误，返回缓存值
	writer.Header().Set("Content-Type", "application/octet-stream")
	_, errW := writer.Write(view.ByteSlice())
	if errW != nil {
		hp.Log("%s", errW)
	}
}
```

ServeHTTP的错误处理按共四次，按顺序分别为URL Path格式错误、缓存对象不存在错误、缓存查询错误（无论是缓存和数据源都失败了）、ResponseWriter.Write执行错误（最后一个错误为goland提醒我修复的，不知道是否必要）