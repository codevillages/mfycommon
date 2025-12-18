# GoFrame Quickstart（最小可运行）

## 适用场景
- 新服务起步，快速跑通配置/日志/路由/统一响应。

## 最小可运行
```go
package main

import (
    "context"

    "github.com/gogf/gf/v2/frame/g"
    "github.com/gogf/gf/v2/net/ghttp"
    "github.com/gogf/gf/v2/os/gctx"
)

type PingReq struct {
    g.Meta `path:"/ping" method:"get" summary:"health check"`
}

type PingRes struct {
    OK bool `json:"ok"`
}

type Controller struct{}

func (c *Controller) GetPing(ctx context.Context, req *PingReq) (*PingRes, error) {
    return &PingRes{OK: true}, nil
}

func main() {
    ctx := gctx.GetInitCtx()
    s := g.Server()

    s.SetConfigWithMap(g.Map{
        "address":        g.Cfg().MustGet(ctx, "server.address", ":8080").String(),
        "readTimeout":    "5s",
        "writeTimeout":   "10s",
        "idleTimeout":    "60s",
        "maxHeaderBytes": "1m",
    })

    s.Use(ghttp.MiddlewareHandlerResponse)

    s.Group("/api", func(group *ghttp.RouterGroup) {
        group.Bind(new(Controller))
    })

    s.Run()
}
```

## 参考中间件
```go
func requestIDMiddleware(r *ghttp.Request) {
    rid := r.Header.Get("X-Request-ID")
    if rid == "" {
        rid = guid.S()
    }
    r.SetCtxVar("request_id", rid)
    r.Response.Header().Set("X-Request-ID", rid)
    r.Middleware.Next()
}

func accessLogMiddleware(r *ghttp.Request) {
    start := gtime.Now()
    r.Middleware.Next()
    cost := gtime.Now().Sub(start)
    g.Log().Infof(r.Context(), "method=%s path=%s status=%d cost=%s rid=%s",
        r.Method, r.URL.Path, r.Response.Status, cost, r.GetCtxVar("request_id"))
}
```

## 配置示例
```yaml
# config/config.yaml
server:
  address: ":8080"
  logLevel: "info"
  accessLogEnabled: false
```
