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
- 需要统一中间件链路（RequestID/访问日志/鉴权/限流）与默认响应封装。
- 需要 gRPC 服务或客户端，并希望复用 GoFrame 的配置/日志/注册发现能力（`contrib/rpc/grpcx`）。
- 需要插件化扩展（`ghttp.Plugin`）或以配置驱动开启/关闭服务能力。
- 需要 OpenAPI/Swagger、pprof、AccessLog 等运维能力。
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
示例较多，已拆分到子目录，按需加载：
- `mfycommon/goframe/examples/quickstart.md`：最小可运行（配置+日志+路由+优雅启动）。
- `mfycommon/goframe/examples/config.md`：配置读取、环境覆盖、动态刷新建议。
- `mfycommon/goframe/examples/routing.md`：路由分组、REST 绑定、OpenAPI/Swagger、静态/文件上传。
- `mfycommon/goframe/examples/middleware.md`：RequestID、访问日志、CORS、鉴权、限流模板。
- `mfycommon/goframe/examples/response.md`：统一响应、文件下载、流式/重定向。
- `mfycommon/goframe/examples/errors.md`：gerror/gcode 约定、错误映射、重试策略。
- `mfycommon/goframe/examples/db.md`：gdb 初始化、查询、事务、超时。
- `mfycommon/goframe/examples/redis.md`：gredis 初始化、常规操作、超时/重试。
- `mfycommon/goframe/examples/grpcx.md`：grpcx 服务端/客户端、配置、拦截器。
- `mfycommon/goframe/examples/plugin.md`：ghttp.Plugin 示例与生命周期。
- `mfycommon/goframe/examples/observability.md`：AccessLog、pprof、Tracing、指标埋点建议。

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
