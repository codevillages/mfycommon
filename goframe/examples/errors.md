# 错误处理与约定

## 适用场景
- 统一业务错误码、参数错误、依赖超时等。

## gerror/gcode 示例
```go
func validateName(name string) error {
    if name == "" {
        return gerror.NewCode(gcode.CodeInvalidParameter, "name required")
    }
    return nil
}
```

## 统一响应中返回错误
```go
func (c *Controller) PostUsers(ctx context.Context, req *CreateReq) (*CreateRes, error) {
    if err := validateName(req.Name); err != nil {
        return nil, err
    }
    // do work
    return &CreateRes{ID: 1}, nil
}
```

## 依赖超时映射
```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
err := callDownstream(ctx)
if errors.Is(err, context.DeadlineExceeded) {
    return gerror.NewCode(gcode.CodeTimeout, "downstream timeout")
}
```

## 重试建议
- 仅对幂等操作重试，最多 3 次，总耗时 < 10s。
- 业务冲突（如幂等写）不要盲目重试。
