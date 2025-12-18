# 插件（ghttp.Plugin）

## 适用场景
- 在服务启动前统一安装功能（如路由、指标、白名单等），并在退出时清理。

## 插件实现
```go
type MetricsPlugin struct{}

func (p *MetricsPlugin) Name() string        { return "metrics" }
func (p *MetricsPlugin) Author() string      { return "platform" }
func (p *MetricsPlugin) Version() string     { return "v1.0.0" }
func (p *MetricsPlugin) Description() string { return "metrics plugin" }

func (p *MetricsPlugin) Install(s *ghttp.Server) error {
    s.BindHandler("/metrics", func(r *ghttp.Request) {
        r.Response.Write("# metrics placeholder\n")
    })
    return nil
}

func (p *MetricsPlugin) Remove() error {
    return nil
}
```

## 挂载插件
```go
s := g.Server()
s.Plugin(new(MetricsPlugin))
```
