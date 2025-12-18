# 响应与输出

## 适用场景
- 统一返回结构、文件下载、流式/重定向响应。

## 默认统一响应
```go
s.Use(ghttp.MiddlewareHandlerResponse)

s.BindHandler("/ping", func(r *ghttp.Request) {
    r.Response.WriteJson(g.Map{"pong": true})
})
```

## 基于 HandlerResponse 的统一结构
```go
type ListReq struct {
    g.Meta `path:"/items" method:"get"`
}

type ListRes struct {
    Items []string `json:"items"`
}

func (c *Controller) GetItems(ctx context.Context, req *ListReq) (*ListRes, error) {
    return &ListRes{Items: []string{"a", "b"}}, nil
}
```

## 文件下载
```go
s.BindHandler("/download", func(r *ghttp.Request) {
    r.Response.ServeFileDownload("./files/demo.txt")
})
```

## 流式响应（SSE 示例）
```go
s.BindHandler("/stream", func(r *ghttp.Request) {
    r.Response.Header().Set("Content-Type", "text/event-stream")
    r.Response.Write("data: hello\n\n")
    r.Response.Flush()
})
```

## 重定向
```go
s.BindHandler("/jump", func(r *ghttp.Request) {
    r.Response.RedirectTo("/target", 302)
})
```
