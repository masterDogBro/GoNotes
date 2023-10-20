# gRPC通信

## 实现内容

提供与HTTP通信对等的gRPC通信形式

## protoc-gen-go-grpc

之前为了实现Protobuf通信格式，安装了protoc-gen-go，为了gRPC使用还需要安装protoc-gen-go-grpc。

```bash
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

生成cachepb.pb.go和cachepb_grpc.pb.go

```bash
cd tinygroupcache/cachepd
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative cachepb.proto
```

## gRPC服务器端和客户端

gRPC服务器端和客户端PeerPicker和PeerGetter实现与HTTP的类似。

需要注意的是GrpcPool因为向前继承pb.UnimplementedGroupCacheServer，需要实现的是Get方法，与之对应的是HTTPPool.ServeHTTP方法。

```go
// Get 实现GrpcPool的Get方法，类似ServeHTTP的逻辑，但其使用被封装在cachepb_grpc.pb.go里了
func (p *GrpcPool) Get(ctx context.Context, in *pb.Request) (*pb.Response, error) {
	p.Log("%s %s", in.Group, in.Key)
	response := &pb.Response{}

	group := GetGroup(in.Group)
	if group == nil {
		p.Log("no such group %v", in.Group)
		return response, fmt.Errorf("no such group %v", in.Group)
	}
	value, err := group.Get(in.Key)
	if err != nil {
		p.Log("get key %v error %v", in.Key, err)
		return response, err
	}

	response.Value = value.ByteSlice()
	return response, nil
}
```

