# 中间件

## 适用场景
- 统一 RequestID/日志/鉴权/CORS/限流等链路能力。

## RequestID
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
```

## 访问日志
```go
func accessLogMiddleware(r *ghttp.Request) {
    start := gtime.Now()
    r.Middleware.Next()
    cost := gtime.Now().Sub(start)
    g.Log().Infof(r.Context(), "method=%s path=%s status=%d cost=%s rid=%s",
        r.Method, r.URL.Path, r.Response.Status, cost, r.GetCtxVar("request_id"))
}
```

## CORS
```go
func corsMiddleware(r *ghttp.Request) {
    r.Response.CORSDefault()
    r.Middleware.Next()
}
```

## 鉴权
```go
func authMiddleware(r *ghttp.Request) {
    token := r.Header.Get("Authorization")
    if token == "" {
        r.SetError(gerror.NewCode(gcode.CodeNotAuthorized, "missing token"))
        return
    }
    r.Middleware.Next()
}
```

## 限流（模板）
```go
func rateLimitMiddleware(r *ghttp.Request) {
    // 这里接入团队限流组件或网关限流，不建议手写全局锁限流
    r.Middleware.Next()
}
```
