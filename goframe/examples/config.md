# 配置（gcfg）

## 适用场景
- 统一读取配置文件、环境覆盖、按环境拆分配置。

## 基础读取
```go
ctx := gctx.GetInitCtx()
addr := g.Cfg().MustGet(ctx, "server.address", ":8080").String()
logLevel := g.Cfg().MustGet(ctx, "server.logLevel", "info").String()
```

## 目录/文件覆盖
- 默认搜索：`config/`、`manifest/config/`。
- 覆盖目录：`gf.gcfg.path=/path/to/config`。
- 覆盖文件：`gf.gcfg.file=/path/to/config.yaml`。

## 配置文件示例
```yaml
# config/config.yaml
server:
  address: ":8080"
  readTimeout: "5s"
  writeTimeout: "10s"
  idleTimeout: "60s"
  maxHeaderBytes: "1m"
  clientMaxBodySize: "16m"
  dumpRouterMap: false
  pprofEnabled: false
  pprofPattern: "/debug/pprof"

logger:
  path: "./logs"
  level: "info"

```

## 动态刷新建议
- 关键参数（端口、超时、池大小）不建议在线变更；如需变更，建议灰度发布。
- 运行时动态值（开关、白名单）可使用 `g.Cfg().Get(ctx, key)` 读取。
