# 数据库（gdb）

## 适用场景
- 统一连接池、超时、事务与查询封装。

## 初始化配置
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
```

## 常规查询
```go
ctxQ, cancel := context.WithTimeout(ctx, time.Second)
defer cancel()

var user *User
err := g.DB().Model("user").Where("id", id).Scan(&user)
if err != nil {
    return nil, err
}
```

## 事务
```go
err := g.DB().Transaction(ctx, func(ctx context.Context, tx gdb.TX) error {
    if _, err := tx.Model("user").Data(g.Map{"name": name}).Insert(); err != nil {
        return err
    }
    if _, err := tx.Model("audit").Data(g.Map{"action": "create"}).Insert(); err != nil {
        return err
    }
    return nil
})
```

## 分页
```go
m := g.DB().Model("user").Where("status", 1)
list, err := m.Page(page, pageSize).All()
if err != nil {
    return nil, err
}
```
