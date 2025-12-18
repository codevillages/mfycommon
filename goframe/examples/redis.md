# Redis（gredis）

## 适用场景
- 缓存、分布式锁、计数、队列。

## 初始化配置
```yaml
# config/config.yaml
redis:
  default:
    address: "127.0.0.1:6379"
    db: 0
    maxIdle: 10
    maxActive: 100
    dialTimeout: "1s"
    readTimeout: "1s"
    writeTimeout: "1s"
```

## 常规操作
```go
ctx, cancel := context.WithTimeout(ctx, time.Second)
defer cancel()

rds := g.Redis()
_, err := rds.Do(ctx, "SET", "k", "v", "EX", 60)
if err != nil {
    return err
}
val, err := rds.Do(ctx, "GET", "k")
if err != nil {
    return err
}
_ = val.String()
```

## 使用连接（批量/流水线）
```go
conn, err := g.Redis().Conn(ctx)
if err != nil {
    return err
}
defer conn.Close(ctx)

_, _ = conn.Do(ctx, "INCR", "counter")
_, _ = conn.Do(ctx, "EXPIRE", "counter", 60)
```

## 未命中处理
- 结果为空或 nil 视为未命中，不当作系统错误。
- 业务层转换为 NotFound/空值。
