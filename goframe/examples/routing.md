# 路由与分组

## 适用场景
- 版本化 API、REST 风格路由、控制器绑定、静态/上传接口。

## 分组与 REST 绑定
```go
s := g.Server()

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Group("/v1", func(v1 *ghttp.RouterGroup) {
        v1.Bind(new(UserController))
    })
})

type UserController struct{}

type GetUserReq struct {
    g.Meta `path:"/users/{id}" method:"get" summary:"Get user"`
    ID     int64 `path:"id" v:"required|min:1"`
}

type GetUserRes struct {
    ID   int64  `json:"id"`
    Name string `json:"name"`
}

func (c *UserController) GetUsers(ctx context.Context, req *GetUserReq) (*GetUserRes, error) {
    return &GetUserRes{ID: req.ID, Name: "demo"}, nil
}
```

## BindHandler（函数式绑定）
```go
s.BindHandler("/healthz", func(r *ghttp.Request) {
    r.Response.WriteJson(g.Map{"ok": true})
})
```

## Swagger/OpenAPI
```yaml
server:
  swaggerPath: "/swagger"
  openapiPath: "/openapi.json"
```

## 静态资源
```go
s.AddStaticPath("/static", "./public")
```

## 文件上传
```go
s.BindHandler("/upload", func(r *ghttp.Request) {
    file := r.GetUploadFile("file")
    if file == nil {
        r.SetError(gerror.NewCode(gcode.CodeInvalidParameter, "missing file"))
        return
    }
    name, err := file.Save("./uploads", true)
    if err != nil {
        r.SetError(err)
        return
    }
    r.Response.WriteJson(g.Map{"saved": name})
})
```
