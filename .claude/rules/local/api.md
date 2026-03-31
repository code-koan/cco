# API 开发模板

> 定义项目的 API 开发规范和模板。

## API 设计原则

- RESTful 风格
- 统一响应格式
- 完善的错误处理
- 版本控制

---

## 路由命名

| 操作 | HTTP 方法 | 路径 | 示例 |
|-----|----------|------|------|
| 列表 | GET | /resources | GET /users |
| 详情 | GET | /resources/:id | GET /users/123 |
| 创建 | POST | /resources | POST /users |
| 更新 | PUT/PATCH | /resources/:id | PUT /users/123 |
| 删除 | DELETE | /resources/:id | DELETE /users/123 |

---

## 响应格式模板

### 成功响应

```json
{
  "success": true,
  "data": {},
  "meta": {
    "timestamp": "2024-01-01T00:00:00Z"
  }
}
```

### 分页响应

```json
{
  "success": true,
  "data": [],
  "meta": {
    "total": 100,
    "page": 1,
    "pageSize": 20,
    "totalPages": 5
  }
}
```

### 错误响应

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求参数验证失败",
    "details": []
  }
}
```

---

## Controller 模板

```
[待填写：项目特定的 Controller 代码模板]
```

## Service 模板

```
[待填写：项目特定的 Service 代码模板]
```

## 中间件模板

```
[待填写：项目特定的中间件代码模板]
```

---

## 错误码定义

| 错误码 | HTTP 状态码 | 描述 |
|-------|------------|------|
| VALIDATION_ERROR | 400 | 请求参数验证失败 |
| UNAUTHORIZED | 401 | 未授权 |
| FORBIDDEN | 403 | 禁止访问 |
| NOT_FOUND | 404 | 资源不存在 |
| INTERNAL_ERROR | 500 | 服务器内部错误 |

[待填写：项目特定的错误码]
