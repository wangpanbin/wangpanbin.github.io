---
title: REST API 错误响应设计：让前端少写一半判断
description: 错误码、错误消息、字段级错误的统一格式——附一段 Go 实战代码
date: 2026-08-13
slug: rest-api-error-design
categories:
    - 后端
tags:
    - REST
    - API 设计
    - Go
toc: true
draft: false
---

API 错误响应不一致是后端长期债。一种常见且实用的格式：

```json
{
  "error": {
    "code": "validation_failed",
    "message": "请求参数校验失败",
    "details": [
      { "field": "email", "rule": "required", "message": "邮箱不能为空" },
      { "field": "password", "rule": "min_length", "message": "密码至少 8 位" }
    ],
    "request_id": "req_01abc"
  }
}
```

## 关键点

- `code`：**机器读**。稳定字符串，前端按它映射文案、做埋点、写测试
- `message`：**人读**。可以国际化，但默认给开发者一句能定位问题的话
- `details`：字段级错误。表单校验时必填，让前端能精确定位到 input
- `request_id`：日志关联。前端报错时把这个 ID 报给用户，客服 / 工程师一查日志就定位

## HTTP 状态码 vs 业务码

不要把 HTTP 状态码当成业务码。两套分开：

- **HTTP 状态码**：网关、CDN、监控、SLA 用。403 表示"你这个请求没权限解析"，不是"你这个用户没权限"
- **业务码**：`code` 字段，给前端用。哪怕 HTTP 是 200，业务上"余额不足"也算失败

实际工程里：

| 场景 | HTTP | 业务 code |
| --- | --- | --- |
| 字段校验失败 | 400 | `validation_failed` |
| 未登录 | 401 | `unauthenticated` |
| 已登录但权限不够 | 403 | `forbidden` |
| 资源不存在 | 404 | `not_found` |
| 余额不足 | 200 | `insufficient_balance` |
| 服务挂了 | 500 | `internal_error` + `request_id` |

## Go 实战

```go
type APIError struct {
    Code      string          `json:"code"`
    Message   string          `json:"message"`
    Details   []FieldError    `json:"details,omitempty"`
    RequestID string          `json:"request_id"`
}

type FieldError struct {
    Field   string `json:"field"`
    Rule    string `json:"rule"`
    Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, e APIError) {
    if e.RequestID == "" {
        e.RequestID = uuid.NewString()
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(map[string]APIError{"error": e})
}
```

使用：

```go
writeError(w, http.StatusBadRequest, APIError{
    Code:    "validation_failed",
    Message: "请求参数校验失败",
    Details: []FieldError{
        {Field: "email", Rule: "required", Message: "邮箱不能为空"},
    },
})
```

中间件层把任何 panic 转成 `internal_error` 并带上 `request_id`：

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("panic: %v, request_id=%s", r, reqID)
        writeError(w, http.StatusInternalServerError, APIError{
            Code: "internal_error", Message: "服务暂不可用", RequestID: reqID,
        })
    }
}()
```

## 几个原则

1. **永远带 `request_id`**，无论成功失败
2. **不要在 `message` 里泄露内部信息**（SQL、堆栈、绝对路径），那种信息打到日志
3. **`details` 用数组**而不是嵌套对象，前端用一个 for 循环就能渲染
4. **文档化每个 `code`**，OpenAPI 里给每个业务码写一句解释

## 总结

错误响应不是"返回个 500 完事"。格式统一后，前端能用一个 `error.code` switch 走完整套错误提示，运维能用 `request_id` 拉日志，最后是工程师排障时不用一个端点一个端点扒源码。
