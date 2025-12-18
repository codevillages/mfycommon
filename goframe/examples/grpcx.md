# gRPC（grpcx）

## 适用场景
- 需要 gRPC 服务/客户端，并复用 GoFrame 的配置、日志与注册发现。

## 配置示例
```yaml
# config/config.yaml
grpc:
  name: "user-service"
  address: ":9000"
  logger:
    level: "info"
    stdout: true
```

## 服务端
```go
func main() {
    s := grpcx.Server.New()

    // 注册服务（示意）
    // pb.RegisterGreeterServer(s.Server, &Greeter{})

    s.Run()
}
```

## 客户端
```go
conn, err := grpcx.Client.NewGrpcClientConn("user-service")
if err != nil {
    return err
}
defer conn.Close()

// client := pb.NewGreeterClient(conn)
// resp, err := client.SayHello(ctx, &pb.HelloReq{Name: "hi"})
```

## 拦截器链
- Server 默认链：Tracing -> Logger -> Recover -> AllowNilRes -> Error
- Client 默认链：Tracing -> Error
- 如需自定义，传入 `grpcx.Server.New(&GrpcServerConfig{Options: []grpc.ServerOption{...}})`。
