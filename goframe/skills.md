# @mfycommon/goframe v2.9.0 — GoFrame Skills 指南

## 标题/目的/适用场景
- 库名+版本：`@mfycommon/goframe`（GoFrame v2.9.0），用于构建一体化 HTTP 服务（路由、配置、日志、校验、错误、DB/Redis）。
- 推荐用在：中小型/中大型业务 HTTP 服务、后台管理、对内 API、需要统一配置/日志/错误码的团队工程。
- 替代方案：轻量 API 用 `net/http`/`@mfycommon/gin`；强约束/生成式框架可用 go-zero/kratos；纯 RPC 场景用 `@mfycommon/grpc-go`。
- 不适用：极简脚本/CLI；仅需高性能路由而不需要全家桶时；只做 gRPC 且无 HTTP 需求时。

## 所需输入（配置/参数/环境）
- 配置源：默认搜索 `config/`、`manifest/config/` 等目录；可用 `gf.gcfg.file`（配置文件）与 `gf.gcfg.path`（配置目录）覆盖。
- HTTP Server：
  - `server.address` 默认 `:0`（随机端口，生产禁用），推荐 `:8080` 或业务端口。
  - `server.readTimeout` 默认 `60s`，推荐 `5-10s`。
  - `server.writeTimeout` 默认 `0`（无限），推荐 `10s`。
  - `server.idleTimeout` 默认 `60s`，推荐 `60-120s`。
  - `server.maxHeaderBytes` 默认 `10240`（10KB），推荐 `1-2MB` 视场景调整。
  - `server.clientMaxBodySize` 默认 `8MB`，上传类接口按需提升。
- 日志：
  - `server.logLevel` 默认 `all`，推荐 `info`/`warn`。
  - `server.logStdout` 默认 `true`，生产推荐写文件并接入采集。
  - `server.accessLogEnabled` 默认 `false`，网关/高价值接口建议开启。
- 数据库（可选，gdb）：`database.default` 中配置 `type/host/port/user/pass/name/maxIdle/maxOpen/queryTimeout/execTimeout`。
- Redis（可选，gredis）：`redis.default` 中配置 `address/db/user/pass/maxIdle/maxActive/dialTimeout/readTimeout/writeTimeout`。

## 流程/工作流程
1. **加载配置**：使用 `g.Cfg()` 读取配置，禁止散落 `os.Getenv`；环境覆盖用 `gf.gcfg.file`/`gf.gcfg.path`。
2. **初始化日志**：使用 `g.Log()` 获取全局 logger；设定 `LogLevel`/`LogPath`，统一格式并做脱敏。
3. **构建 Server**：`s := g.Server()`，配置 `Read/Write/IdleTimeout`、`MaxHeaderBytes`、`ClientMaxBodySize`。
4. **中间件顺序**：RequestID -> 访问日志 -> `ghttp.MiddlewareHandlerResponse` -> 鉴权/限流；避免重复写响应。
5. **路由/分组**：`s.Group("/api", ...)` + `group.Bind/BindHandler`，使用 `g.Meta` 定义 `path/method`.
6. **错误处理**：业务错误用 `gerror.NewCode(gcode.*)` 返回；统一由 `MiddlewareHandlerResponse` 输出 `code/message/data`。
7. **超时与上下文**：所有下游调用使用 `ctx := r.Context()` 派生超时；需要异步任务时用 `r.GetNeverDoneCtx()`。
8. **优雅退出**：`s.Shutdown()` 或 `s.SetGraceful(true)`；退出前 flush 日志/关闭连接池。

## 何时使用该技能
- 需要标准化的 HTTP 服务（路由+配置+日志+错误+校验）。
- 需要自动化 OpenAPI/Swagger、路由约定、统一返回结构时。
- 需要统一接入 DB/Redis 配置、超时与可观测性时。

## 输出格式
- HTTP 统一响应（`ghttp.MiddlewareHandlerResponse` 默认）：
  - `{"code":0,"message":"OK","data":{...}}`
  - 错误时 `code` 来自 `gcode`，`message` 来自 `gerror`。
- 日志字段建议：`ts`、`level`、`request_id`、`method`、`path`、`status`、`latency_ms`、`client_ip`、`err`。
- 数据/缓存：
  - DB 未命中：`err` 为 `sql.ErrNoRows` 或为空结果，业务转换为 NotFound，不当作系统错误。
  - Redis 未命中：结果为空或 nil 即视为未命中，不当作系统错误；业务层转换为 NotFound/空值。

## 示例（最小可运行 + 高频用例）
1. **初始化（配置+日志+路由+优雅启动）**
```go
func main() {
    ctx := gctx.GetInitCtx()
    log := g.Log()
    log.SetLevel("info")
    log.SetStdout(true)

    s := g.Server()
    s.SetConfigWithMap(g.Map{
        "address":        g.Cfg().MustGet(ctx, "server.address", ":8080").String(),
        "readTimeout":    "5s",
        "writeTimeout":   "10s",
        "idleTimeout":    "60s",
        "maxHeaderBytes": "1m",
    })
    s.Use(requestIDMiddleware, accessLogMiddleware, ghttp.MiddlewareHandlerResponse)
    s.Group("/api", func(group *ghttp.RouterGroup) {
        group.Bind(new(controller))
    })
    s.Run()
}
```

2. **常规操作（结构体路由+校验）**
```go
type CreateReq struct {
    g.Meta `path:"/users" method:"post" summary:"Create user"`
    Name   string `json:"name" v:"required|min-length:1|max-length:50"`
}
type CreateRes struct {
    ID int64 `json:"id"`
}
type controller struct{}

func (c *controller) PostUsers(ctx context.Context, req *CreateReq) (*CreateRes, error) {
    if req.Name == "" {
        return nil, gerror.NewCode(gcode.CodeInvalidParameter, "name required")
    }
    return &CreateRes{ID: 1}, nil
}
```

3. **错误/重试（下游调用）**
```go
func callWithRetry(ctx context.Context, fn func(context.Context) error) error {
    var lastErr error
    for i := 0; i < 3; i++ {
        if err := fn(ctx); err == nil {
            return nil
        } else {
            lastErr = err
        }
        time.Sleep(time.Duration(i+1) * 100 * time.Millisecond)
    }
    return gerror.WrapCode(gcode.CodeTimeout, lastErr, "downstream timeout")
}
```

4. **并发/连接管理（后台任务 + 池配置示例）**
```go
func asyncTask(r *ghttp.Request) {
    ctx := r.GetNeverDoneCtx()
    go func() {
        g.Log().Info(ctx, "async start")
        // do work...
    }()
}
```
```yaml
# config/config.yaml
database:
  default:
    type: "mysql"
    host: "127.0.0.1"
    port: "3306"
    user: "app"
    pass: "${DB_PASS}"
    name: "appdb"
    maxIdle: 20
    maxOpen: 80
    queryTimeout: "1s"
    execTimeout: "3s"
redis:
  default:
    address: "127.0.0.1:6379"
    db: 0
    maxIdle: 10
    maxActive: 100
    dialTimeout: "1s"
```

5. **收尾清理（优雅关闭）**
```go
func waitShutdown(s *ghttp.Server) {
    ch := make(chan os.Signal, 1)
    signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
    <-ch
    s.Shutdown()
}
```

## 限制条件与安全规则
- 禁止生产使用 `server.address=":0"`；禁止未配置超时的下游调用。
- 统一响应必须走 `ghttp.MiddlewareHandlerResponse`，避免手写 JSON 导致格式不一。
- 性能/资源红线：单实例 QPS >5k 必须压测；DB `maxOpen` 建议 ≤200，Redis `maxActive` 建议 ≤200（按压测调整）。
- 敏感信息必须脱敏（密码、Token、身份证、手机号）；禁止日志记录完整请求体。
- 重试规则：仅对幂等操作重试，最多 3 次，总耗时 < 10s；超时必须显式设置。

## 常见坑/FAQ（按严重度）
- 高：忘记设置 `Read/Write` 超时导致连接长期占用。
- 高：中间件重复 `Write` 响应导致 `MiddlewareHandlerResponse` 输出异常。
- 中：未使用 `g.Meta` 路由标签导致路由注册失败或不可预期路径。
- 中：`server.dumpRouterMap=true` 在生产泄露路由信息，需关闭。
- 低：默认 `clientMaxBodySize=8MB` 导致上传失败，需按场景调整。

## 可观测性/诊断
- AccessLog/ErrorLog：通过 `server.accessLogEnabled`/`server.errorLogEnabled` 开关，字段含 `status/latency/path`.
- Tracing：GoFrame 默认接入 OTel server tracing，中间件自动创建 span。
- 慢请求：>300ms 记录 `warn` 日志，包含 `request_id/method/path/status/duration_ms`.

## 版本与依赖
- GoFrame v2.9.0；Go 1.22（见 `gf/go.mod`）。
- 依赖：`ghttp/glog/gcfg/gerror/gcode`；可选 `gdb`（MySQL/PG 等）、`gredis`（Redis）。
- 内部封装路径：`@mfycommon/goframe`。

## 更新记录/Owner
- 最后更新时间：2025-xx-xx（首版）。
- Owner：框架维护人（@架构/中间件负责人），评审人：服务线负责人；变更需双人 Review。
