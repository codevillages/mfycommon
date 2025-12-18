# 可观测性

## 适用场景
- 接入 AccessLog/ErrorLog、pprof、Tracing、指标埋点。

## 访问日志与错误日志
```yaml
server:
  logLevel: "info"
  logStdout: true
  errorLogEnabled: true
  errorLogPattern: "error-{Ymd}.log"
  accessLogEnabled: true
  accessLogPattern: "access-{Ymd}.log"
```

## pprof
```yaml
server:
  pprofEnabled: true
  pprofPattern: "/debug/pprof"
```

## Tracing
- GoFrame HTTP server 默认启用 OTel server tracing。
- 确保下游调用使用 `r.Context()` 传递 trace。

## 指标埋点建议
- http_qps/http_latency/http_error_rate：按 `method/path/status` 标签。
- db_latency/redis_latency：按 `op` 和 `status` 标签。
- 慢请求阈值建议 300ms，超出打 `warn` 日志。
