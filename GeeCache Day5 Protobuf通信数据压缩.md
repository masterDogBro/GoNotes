# Protobuf通信/数据压缩

## 实现内容

在使用HTTP通信形式的接触上，对缓存返回值封装protobuf格式

## Protocol Buffers

window平台安装方案

[go protoc-gen-go 安装详解](https://www.cnblogs.com/shiding/p/16608117.html)

## Protobuf数据封装实现

### proto文件定义

```protobuf
syntax = "proto3";

package cachepb;
option go_package = "./cachepb";

message Request {
  string group = 1;
  string key = 2;
}

message Response {
  bytes value = 1;
}

service GroupCache {
  rpc Get(Request) returns (Response);
}
```

### PeerGetter的方法变更

```go
// PeerGetter Get方法从对应Group查找key的缓存值并返回
type PeerGetter interface {
    // Get 第一种实现 Get(group string, key string) ([]byte, error)
    Get(in *pb.Request, out *pb.Response) error
}
```

HTTP服务器HTTPPool.ServeHTTP方法变更

```go
func (hp *HTTPPool) ServeHTTP(writer http.ResponseWriter, r *http.Request) {
	// ...

    // 未发生任何错误，返回缓存值

    // http通信实现形式
    //writer.Header().Set("Content-Type", "application/octet-stream")
    //_, errW := writer.Write(view.ByteSlice())
    //if errW != nil {
    // hp.Log("%s", errW)
    //}

    // protobuf通信实现形式
    // 编码HTTP响应
    body, errM := proto.Marshal(&pb.Response{Value: view.ByteSlice()})
    if errM != nil {
       http.Error(writer, errM.Error(), http.StatusInternalServerError)
       return
    }
    writer.Header().Set("Content-Type", "application/octet-stream")
    _, errW := writer.Write(body)
    if errW != nil {
       hp.Log("%s", errW)
    }
}
```

HTTP客户端httpGetter.Get方法变更

```go
func (h *httpGetter) Get(in *pb.Request, out *pb.Response) error {
    getUrl := fmt.Sprintf(
       "%v%v/%v",
       h.baseURL,
       url.QueryEscape(in.GetGroup()),
       url.QueryEscape(in.GetKey()),
    )
    res, errG := http.Get(getUrl)
    if errG != nil {
       return errG
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
       return fmt.Errorf("server returned: %v", res.Status)
    }

    // 处理Response读取错误
    bytes, errR := io.ReadAll(res.Body)
    if errR != nil {
       return fmt.Errorf("reading response body: %v", errR)
    }

    // bytes解码出错
    if errUm := proto.Unmarshal(bytes, out); errUm != nil {
       return fmt.Errorf("decoding response body: %v", errUm)
    }

    return nil
}
```

## 问题

现在protobuf仅仅作为http通信中缓存值的数据风格格式，而由于http的限制，节点之间的通信效率并不会有太大提升。接下来会实现基于protobuf和新PeerGetter接口的gRPC的通信形式。



