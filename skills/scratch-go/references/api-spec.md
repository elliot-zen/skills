# API 规范

## 1. 适用范围

本规范定义 HTTP API 的通用响应、字段命名及列表分页契约。新建接口默认遵守；已有接口优先保持兼容，改变已有响应结构或字段名称时，通过明确迁移或新版本接口实施，并同步实现、接口文档与测试。

应用错误模型、错误转换、请求解码、请求 ID 生成和响应完成状态遵守 [scratch-go 第 2.2、4.1 节](../SKILL.md)。本规范维护对外 JSON 结构，接口文档只补充实际业务字段、Reason 和例外。

## 2. 命名

| 范围 | 约定 | 示例 |
| --- | --- | --- |
| JSON 请求与响应字段 | 小写下划线 `snake_case` | `user_id`、`created_at` |
| Query 参数 | `snake_case` | `page_size`、`order_status` |
| 路由参数名称 | `snake_case` | `:order_id` |
| URL 静态路径 | 小写，多个单词用连字符 | `/api/v1/order-items` |
| 业务结果标识 `reason` | 大写下划线 | `OK`、`ORDER_ALREADY_PAID` |
| Go 字段 | Go 惯用命名，通过显式 JSON tag 映射 | `OrderID` → `json:"order_id"` |

JSON 命名规则递归适用于所有层级，包括 `data` 中的对象、嵌套对象和数组元素中的对象，以及 `metadata` 的固定键。缩写同样小写，如 `user_id`、`api_url`，不使用 `userId`、`userID` 或 `UserID` 作为 JSON 字段名。字段值遵循各自业务契约，不因字段命名规则而改写。

动态字典中由业务数据决定的键（如资源 ID）、第三方协议规定的字段及明确声明为原始透传的数据，保留自身契约；接口文档必须注明例外范围和原因。适配为本服务业务对象的第三方数据仍使用 `snake_case`。

## 3. 通用 JSON 响应

### 3.1 字段与语义

| 字段 | JSON 类型 | 约定 |
| --- | --- | --- |
| `code` | integer | 与实际 HTTP 状态码一致，成功使用对应 2xx，失败使用 4xx/5xx |
| `reason` | string | 稳定的机器可读标识；成功固定为 `OK`，失败使用通用或业务 Reason |
| `message` | string | 非空的可读描述；成功默认使用“成功”，调用方不得用它判断程序分支 |
| `data` | 任意 JSON 值 | 成功时承载业务数据，无数据或失败时为 `null` |
| `request_id` | string | 请求标识，与响应头 `X-Request-ID` 一致 |
| `metadata` | object | 可选，仅失败时提供非空的补充信息，键和值均为字符串 |

前五个字段必须存在，不得因零值或 `null` 被省略。业务数据统一放在 `data` 内，具体类型和字段由各接口声明；不得在外层增加业务字段。成功响应省略 `metadata`，错误响应在 Metadata 为空时省略该字段。

JSON 响应使用 `Content-Type: application/json`，允许 charset。不使用 HTTP 200 承载失败，也不使用 `code: 0` 等独立于 HTTP 状态的成功码。调用方按 HTTP 状态处理通用行为，按 `reason` 判断业务分支。内部 cause、堆栈和错误包装上下文不进入响应。

### 3.2 成功响应

HTTP 200：

```json
{
  "code": 200,
  "reason": "OK",
  "message": "成功",
  "data": {
    "order_id": "123",
    "buyer_info": {
      "user_id": "456"
    },
    "order_items": [
      {
        "product_id": "789",
        "unit_price": 100
      }
    ]
  },
  "request_id": "abc123"
}
```

创建成功使用 HTTP 201，正文 `code` 同为 201、`reason` 仍为 `OK`。同步删除成功且无业务数据时，默认使用 HTTP 200 和 `data: null`。其他成功状态按实际接口语义声明，正文 `code` 始终与状态一致。

新建服务的 `GET /api/v1/ping` 使用 HTTP 200：

```json
{
  "code": 200,
  "reason": "OK",
  "message": "成功",
  "data": {
    "message": "pong"
  },
  "request_id": "abc123"
}
```

### 3.3 失败响应

HTTP 409：

```json
{
  "code": 409,
  "reason": "ORDER_ALREADY_PAID",
  "message": "订单已支付，不能重复付款",
  "data": null,
  "request_id": "abc123"
}
```

参数校验失败时，HTTP 400：

```json
{
  "code": 400,
  "reason": "INVALID_ARGUMENT",
  "message": "购买数量必须在 1 到 100 之间",
  "data": null,
  "request_id": "abc123",
  "metadata": {
    "location": "body",
    "field": "quantity",
    "rule": "range",
    "min": "1",
    "max": "100"
  }
}
```

状态与通用 Reason 的映射、未知错误兜底及 Metadata 的具体含义遵守 scratch-go 第 4.1.1、4.1.2 节。失败时不通过 `data` 返回部分业务结果。

## 4. 列表与分页

列表统一放在 `data.items`，空列表返回 `[]`，不得返回 `null` 或省略。非分页列表仅包含 `items`；页码分页增加 `pagination`：

```json
{
  "code": 200,
  "reason": "OK",
  "message": "成功",
  "data": {
    "items": [
      {
        "order_id": "123",
        "order_status": "paid"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 1
    }
  },
  "request_id": "abc123"
}
```

- Query 参数使用 `page`、`page_size`；`page` 从 1 开始，`page_size` 为正整数。默认值及最大容量由接口声明并校验，不静默修正非法值。
- `pagination` 的三个字段必须存在，均为整数；`page`、`page_size` 表示本次生效的页码和每页容量，`total` 为符合查询条件的总条数，不能用当前页条数代替。
- 无匹配数据时 `items: []`、`total: 0`；页码超出已有数据范围时返回空列表，保留本次页码及实际总条数。
- 不重复返回可由上述字段计算的 `total_pages`。
- 游标分页有实际需求时单独声明请求参数与响应结构，不与页码字段混用，也不伪造 `total`。

## 5. 协议边界与验证

HTTP 204、304 和 HEAD 响应按 HTTP 语义不返回正文。文件下载、SSE、WebSocket、`/metrics` 等使用各自协议，不套 JSON 外层。客户端取消及响应已提交后发生失败时，按 scratch-go 第 4.1.4 节处理，不为补齐格式追加 JSON。

接口文档声明业务字段的类型、必填性及 `null`、零值、空字符串和空集合的含义；本规范不统一强制业务 ID 或时间字段的值类型。

测试通过实际 HTTP 出口验证成功与失败的完整结构、HTTP 状态与 `code` 一致、请求 ID 头体一致、`data` 的空值行为及 `metadata` 的省略规则。命名验证覆盖嵌套对象和数组元素；存在列表与分页时验证空列表、总数、页码边界及非法分页参数；协议例外按各自契约验证。
