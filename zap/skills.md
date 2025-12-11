# zap v1.27.0 使用手册

## 标题 / 目的 / 适用场景
- 目的：使用 `go.uber.org/zap` v1.27.0 输出高性能结构化日志，统一字段、输出与关闭流程。
- 适用：Go 服务需要结构化日志并写入本地文件/ELK/Stdout，要求低分配、可配置级别与动态字段。
- 替代：轻量场景可用 stdlib `log`；需要复杂格式化或 Sentry/Axiom 可考虑 zerolog/logrus/otel-logger。
- 不适用：需要直接写 JSONL 以外格式（如 CSV/XML）或需自带切割的场景（需配合 lumberjack/云采集端切割）。

## 所需输入（配置与推荐值）
- 日志级别：`DEBUG/INFO/WARN/ERROR`；默认 `INFO`，线上推荐 `INFO`，排障可临时提到 `DEBUG`。
- 输出：默认 `stdout`；生产建议 `stdout` 由容器/agent 收集，或 `lumberjack` 滚动到文件（大小 100MB、备份 7、压缩）。
- 编码：`json`（推荐）或 `console`（本地调试）；字段采用小写蛇形。
- 公共字段：`service`（必填）、`env`、`version`；请求级字段 `request_id/trace_id/user_id/client_ip` 由中间件注入。
- 性能参数：开启 `zap.AddCaller()` 需在 QPS <5w 时使用；`AddCallerSkip` 适配封装层。
- 关闭：所有初始化返回的 `*zap.Logger` 需在进程退出前调用 `Sync()`。

## 何时使用
- 需要结构化日志并写入本地/ELK/云采集的服务端日志。
- 需要在高 QPS 场景减少分配（比 logrus 更轻）。
- 需要统一字段、按请求注入 `request_id/trace_id`、或需要 error 栈时。
- 需要自定义采样、写多路输出（core tee）或扩展字段编码时。

## 流程 / 步骤
1. 配置加载：读取 level/encoding/output/切割/采样/公共字段（service/env/version）；禁止在代码中硬编码敏感路径或密钥。
2. 构建 Encoder：`encCfg := zap.NewProductionEncoderConfig()`，设置 `TimeKey="ts"`、`EncodeTime=zapcore.RFC3339NanoTimeEncoder`、`EncodeLevel=zapcore.LowercaseLevelEncoder`，线上使用 `zapcore.NewJSONEncoder(encCfg)`。
3. 构建输出：`zapcore.AddSync(os.Stdout)` 或 `lumberjack.Logger`（文件滚动），如需多路输出使用 `zapcore.NewTee`。
4. 级别控制：`atomicLevel := zap.NewAtomicLevelAt(level)`；如需动态调整，在 HTTP/管理命令中调用 `atomicLevel.SetLevel(...)`。
5. 构建 Core：`core := zapcore.NewCore(encoder, writeSyncer, atomicLevel)`；高 QPS 可包一层 `zapcore.NewSamplerWithOptions(core, time.Second, 100, 10)`。
6. 创建 Logger：`logger := zap.New(core, zap.AddCaller(), zap.AddCallerSkip(skip))`，并追加公共字段 `logger = logger.With(zap.String("service", svc), zap.String("env", env), zap.String("version", ver))`。
7. 全局注入：如需统一使用 `zap.ReplaceGlobals(logger)` + `zap.RedirectStdLog(logger)`，或将 logger 存入自研封装（如 `pkg/log`）并提供 `GetLogger(ctx)`。
8. 请求上下文：在 HTTP/gRPC 中间件内 `ctx = context.WithValue(ctx, loggerKey, logger.With(zap.String("request_id", rid), zap.String("trace_id", tid)))`，handler 通过封装取出。
9. 业务落日志：使用强类型字段 `zap.String/Int/Duration/Error`，记录耗时/输入关键参数（已脱敏）。
10. flush：在 `main` 退出钩子（signal/errgroup）统一 `defer logger.Sync()`；不要在业务路径调用 Sync。

## 输出格式 / 错误处理约定
- 字段规范：`ts`(RFC3339Nano)、`level`(小写)、`msg`、`caller`、`request_id`、`trace_id`、`service`、`env`、`version`、`elapsed_ms`、`err`。
- 错误：使用 `zap.Error(err)` 输出；可同时输出分级字段 `err_kind`（io/net/timeout/biz）与 `stack`（zap.Stack）用于定位。
- 采样：高频 INFO 日志建议开启 `zapcore.NewSamplerWithOptions`，避免 IO 放大；ERROR 默认不采样。
- 敏感信息：禁止记录密钥/口令/完整手机号，必要时脱敏（中间位用 `***`）。

## 示例（按主题选用）
- 初始化（stdout JSON）：
```go
level := zap.NewAtomicLevelAt(zap.InfoLevel)
encCfg := zap.NewProductionEncoderConfig()
encCfg.TimeKey = "ts"
encCfg.EncodeTime = zapcore.RFC3339NanoTimeEncoder
core := zapcore.NewCore(zapcore.NewJSONEncoder(encCfg), zapcore.AddSync(os.Stdout), level)
logger := zap.New(core, zap.AddCaller())
defer logger.Sync()
logger.Info("logger init ok", zap.String("service", "order"), zap.String("env", "prod"))
```
- 初始化（stdout + 文件切割 + 采样）：
```go
hook := &lumberjack.Logger{Filename: "/var/log/order/app.log", MaxSize: 100, MaxBackups: 7, MaxAge: 7, Compress: true}
ws := zapcore.NewMultiWriteSyncer(zapcore.AddSync(os.Stdout), zapcore.AddSync(hook))
core := zapcore.NewSamplerWithOptions(zapcore.NewCore(
    zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
    ws,
    zap.InfoLevel,
), time.Second, 100, 10)
logger := zap.New(core, zap.AddCaller()).With(zap.String("service", "order"))
defer logger.Sync()
```
- 在中间件注入 request_id：
```go
rid := uuid.NewString()
ctxLogger := logger.With(zap.String("request_id", rid))
// 将 ctxLogger 放入 context，handler 内获取使用
```
- 业务日志（携带耗时与错误）：
```go
start := time.Now()
if err := svc.Do(ctx); err != nil {
    logger.Error("call service failed",
        zap.String("request_id", rid),
        zap.Duration("elapsed_ms", time.Since(start)),
        zap.Error(err),
    )
    return
}
logger.Info("call service ok", zap.String("request_id", rid), zap.Duration("elapsed_ms", time.Since(start)))
```
- 带堆栈的错误日志（仅关键路径开启）：
```go
logger.Error("panic recovered", zap.Any("event", evt), zap.Stack("stack"))
```
- gRPC/HTTP 中间件注入（示例片段）：
```go
func LoggingMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    rid := r.Header.Get("X-Request-ID")
    ctx := context.WithValue(r.Context(), loggerKey, logger.With(zap.String("request_id", rid)))
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}
```
- 动态调整日志级别（如通过管理接口）：
```go
var atomicLevel = zap.NewAtomicLevelAt(zap.InfoLevel)
func setLevel(l string) error {
    var lv zapcore.Level
    if err := lv.UnmarshalText([]byte(strings.ToLower(l))); err != nil {
        return err
    }
    atomicLevel.SetLevel(lv)
    return nil
}
```
- 将 zap 作为标准库日志后端（避免混用 stdout）：
```go
zap.ReplaceGlobals(logger)
zap.RedirectStdLog(logger)
log.Println("this goes to zap core")
```
- 统一清理（signal 退出）：
```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()
go func() { <-ctx.Done(); logger.Sync() }()
```
- Sugared Logger（少用，仅在拼接方便时用）：
```go
s := logger.Sugar()
s.Infof("user %s login from %s", userID, ip)
```
- 并发安全：同一个 `*zap.Logger` 可在多 goroutine 共享，无需锁；仅确保初始化一次。
- 收尾清理：在 `signal.NotifyContext` 退出或 `errgroup.Wait` 前执行 `logger.Sync()`，忽略 `EBADF`。

## 限制条件与安全规则
- 禁止在热路径频繁创建 logger/sugar；使用全局或从 context 传递的实例。
- 禁止每次日志后调用 `Sync()`，仅在进程退出时调用。
- 输出敏感数据必须脱敏；禁止记录密码、token、完整身份证/手机号。
- QPS 红线：若单实例日志吞吐 >5w QPS，必须开启采样或拆分落盘；避免阻塞业务线程。
- 禁用 API：少用 `zap.S()` 在高性能场景；禁止 `zap.NewExample()` 在生产。

## 常见坑 / FAQ
- 忘记 `Sync()`：导致缓冲未刷出；在 `main` 用 `defer logger.Sync()`。
- 未设置 caller/skip：封装层导致 caller 指向封装，使用 `zap.AddCallerSkip(1+)`。
- 生产使用 console 编码：采集失败且体积大，线上请使用 JSON。
- 在高频路径使用 `Sugar()`：增加分配，改用强类型字段。
- 日志字段缺少 request_id/trace_id：在中间件统一注入并从 context 获取 logger。

## 可观测性 / 诊断
- 指标：可在 core 外层包一层 Hook 统计日志计数/级别耗时，暴露到 Prometheus（自定义 zapcore.Core wrapper）。
- 链路追踪：在中间件里把 trace_id 放入字段；如使用 OTEL，可用 `otelzap` 适配器将 span 上下文写入字段。
- 慢日志关键字段：`request_id/trace_id/service/env/version/caller/elapsed_ms/err`；异常需包含输入关键参数但脱敏。

## 版本与依赖
- 版本：`go.uber.org/zap` v1.27.0（兼容 v1.24+）；Go 1.20+。
- 可选依赖：`gopkg.in/natefinch/lumberjack.v2` 用于文件切割；`go.opentelemetry.io/contrib/bridges/otelzap` 用于写入 trace。
- 内部封装：如已有 `pkg/log` 等全局封装，优先调用封装提供的获取/注入方法，禁止自行 new。

## 更新记录 / Owner
- 最后更新时间：2024-06
- 维护人/评审人：请补充日志组件 Owner / 平台组联系人。
