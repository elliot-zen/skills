# API 规范

## 1. 适用范围

本规范定义 HTTP API 的命名、请求校验、响应、错误码及列表分页契约。实现、接口文档与测试必须遵守同一契约；协议例外在第 6 节声明，不由单个处理器自行改变响应约定。

Go 错误模型遵守 [错误处理](error.md)，HTTP 实现与完成边界见 [Go 工程模式第 3 章](go-patterns.md#3-http-服务)。接口文档补充实际业务字段、错误码及例外。

## 2. 命名

| 范围 | 约定 | 示例 |
| --- | --- | --- |
| JSON 请求与响应字段 | 小写下划线 `snake_case` | `user_id`、`created_at` |
| Query 参数 | `snake_case` | `page_size`、`order_status` |
| 路由参数名称 | `snake_case` | `:order_id` |
| URL 静态路径 | 小写，多个单词用连字符 | `/api/v1/order-items` |
| 错误码 `error_code` | 大写下划线 | `INVALID_ARGUMENT`、`ORDER_ALREADY_PAID` |
| Go 字段 | Go 惯用命名，通过显式 JSON tag 映射 | `OrderID` → `json:"order_id"` |

JSON 命名规则递归适用于所有层级，包括 `data` 中的对象、嵌套对象和数组元素中的对象，以及 `metadata` 的固定键。缩写同样小写，如 `user_id`、`api_url`，不使用 `userId`、`userID` 或 `UserID` 作为 JSON 字段名。字段值遵循各自业务契约，不因字段命名规则而改写。

动态字典中由业务数据决定的键（如资源 ID）、第三方协议规定的字段及明确声明为原始透传的数据，保留自身契约；接口文档必须注明例外范围和原因。适配为本服务业务对象的第三方数据仍使用 `snake_case`。

## 3. 通用 JSON 响应

### 3.1 字段与语义

| 字段 | JSON 类型 | 约定 |
| --- | --- | --- |
| `error_code` | string | 稳定、非空的机器可读错误码；失败时必填，成功时省略 |
| `message` | string | 非空的可读描述；成功默认使用“成功”，调用方不得用它判断程序分支 |
| `data` | 任意 JSON 值 | 成功时承载业务数据，无数据或失败时为 `null` |
| `request_id` | string | 请求标识，与响应头 `X-Request-ID` 一致 |
| `metadata` | object | 可选，仅失败时提供非空的补充信息，键和值均为字符串 |

`message`、`data`、`request_id` 必须存在，不得因零值或 `null` 被省略。业务数据统一放在 `data` 内，具体类型和字段由各接口声明；不得在外层增加业务字段。成功响应省略 `error_code` 和 `metadata`，错误响应在 `metadata` 为空时省略该字段。

JSON 响应使用 `Content-Type: application/json`，允许 charset。成功使用对应 2xx，失败使用 4xx/5xx，不使用 HTTP 200 承载失败。响应外层不包含 `code` 或 `reason`。调用方按 HTTP 状态处理通用行为，按 `error_code` 判断具体错误分支。错误对象按第 4 节转换，不直接序列化；底层原因、堆栈和内部诊断不进入响应。

### 3.2 成功响应

HTTP 200：

```json
{
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

创建成功使用 HTTP 201。同步删除成功且无业务数据时，默认使用 HTTP 200 和 `data: null`。其他成功状态按实际接口语义声明。

### 3.3 失败响应

HTTP 409：

```json
{
  "error_code": "ORDER_ALREADY_PAID",
  "message": "订单已支付，不能重复付款",
  "data": null,
  "request_id": "abc123"
}
```

参数校验失败时，HTTP 400：

```json
{
  "error_code": "INVALID_ARGUMENT",
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

状态、错误码和参数错误的 `metadata` 见第 4 节。失败时不通过 `data` 返回部分业务结果。

## 4. 错误与参数校验

### 4.1 状态与错误码

业务专用错误码优先，无专用码时使用下表。仅在实际引入相应功能时实现对应映射和鉴权流程。

| 场景 | HTTP 状态 | 通用 `error_code` |
| --- | --- | --- |
| 请求参数无效 | 400 | `INVALID_ARGUMENT` |
| 缺少或无效的身份凭证 | 401 | `UNAUTHENTICATED` |
| 身份已确认但权限不足 | 403 | `PERMISSION_DENIED` |
| 路由或资源不存在 | 404 | `NOT_FOUND` |
| 方法不支持 | 405 | `METHOD_NOT_ALLOWED` |
| 资源状态冲突 | 409 | `CONFLICT` |
| 请求体媒体类型不受支持 | 415 | `UNSUPPORTED_MEDIA_TYPE` |
| 请求格式及参数有效，但业务语义无法处理 | 422 | `UNPROCESSABLE_CONTENT` |
| 请求受限流策略拒绝 | 429 | `TOO_MANY_REQUESTS` |
| 未识别错误、依赖故障、超时或响应前 panic | 500 | `INTERNAL_ERROR` |

HTTP 适配层通过 `errors.Is/As` 识别已声明的业务错误，显式选择安全文案，动态部分只引用契约允许公开的字段；不得直接返回最外层或底层 `err.Error()`。未识别错误、依赖故障及超时使用 `500 / INTERNAL_ERROR` 和固定文案，例如“服务内部错误”。映射状态不属于 4xx/5xx，或 `error_code`、`message` 为空时，同样按内部错误兜底。

错误码在应用内唯一，业务码带模块前缀，发布后不随重构改名、复用或改变含义。`metadata` 仅包含接口契约声明的有界安全字段，键和值均为字符串。

### 4.2 请求与参数校验

解码或参数校验失败返回 `400 / INVALID_ARGUMENT`；业务语义无法处理使用 422，资源状态冲突使用 409。客户端取消、超时及其他 I/O 故障按 [Go 工程模式第 3.2 节](go-patterns.md#32-请求处理与完成边界)处理，不一律返回 400。

- query/path/header 的字符串按接口声明的类型解析；不支持的 Content-Type 返回 415。
- 必需请求体为空时返回参数错误。接口声明字段的必填性及 null、零值、空字符串的含义，允许的 `0`、`false` 不得当作未提供。
- 除非接口明确约定，不隐式修剪、改写或规范化输入；长度规则注明字节、Unicode 码点等计量单位。

解码失败立即返回；参数校验每次返回一个错误，按接口约定的字段与规则顺序选择，不依赖 map 或校验库的遍历顺序。失败后不执行后续业务操作。

`message` 按错误类别提供可读说明；参数错误的 `metadata` 使用以下字段：

| 键 | 约定 |
| --- | --- |
| `location` | 参数来源：`body`、`query`、`path`、`header` |
| `field` | 能可靠定位时提供对外字段路径，例如 `quantity`、`items[0].quantity`；body 使用 JSON 名称，其他来源使用接口参数名 |
| `rule` | 稳定类别：`syntax`、`required`、`type`、`format`、`range`、`length`；特殊组合规则扩展时同步接口文档 |
| `min`、`max` | range 错误中适用的数值边界 |
| `min_length`、`max_length` | length 错误中适用的长度边界，单位遵循字段契约 |

必需请求体缺失使用 `rule=required`；JSON 语法错误使用 `location=body`、`rule=syntax`，无法定位字段时省略 `field`，文案例如“请求体不是有效的 JSON”。范围错误示例见第 3.3 节。

## 5. 列表与分页

列表统一放在 `data.items`，空列表返回 `[]`，不得返回 `null` 或省略。非分页列表仅包含 `items`；页码分页增加 `pagination`：

```json
{
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

## 6. 协议边界

HTTP 204、304 和 HEAD 响应按 HTTP 语义不返回正文。文件下载、SSE、WebSocket、`/metrics` 等使用各自协议，不套 JSON 外层。客户端取消及响应已提交后发生失败时，按 [Go 工程模式第 3.2 节](go-patterns.md#32-请求处理与完成边界)处理，不为补齐格式追加 JSON。

接口文档声明业务字段的类型、必填性及 `null`、零值、空字符串和空集合的含义；本规范不统一强制业务 ID 或时间字段的值类型。

## 7. 验证

测试通过实际 HTTP 出口验证成功与失败的完整结构、HTTP 状态符合契约、响应外层不含 `code` 和 `reason`、`error_code` 失败时正确且成功时省略、请求 ID 头体一致、`data` 的空值行为及 `metadata` 的省略规则。命名验证覆盖嵌套对象和数组元素；存在列表与分页时验证空列表、总数、页码边界及非法分页参数；协议例外按各自契约验证。

通过实际解码、校验和错误出口验证以下适用行为：

- 无效 JSON、类型错误、必需请求体为空及媒体类型按契约返回 400/415；取消和 I/O 故障不误报为参数错误。
- 必填、边界、格式和组合规则符合契约；缺失、null 与零值按声明区分，允许的 0/false 可通过；多个错误时首个错误稳定，响应文案及 metadata 正确，后续业务操作未执行。
- 实际业务拒绝及 404/405 的状态、错误码和安全文案正确；422 优先使用业务码，无专用码时为 UNPROCESSABLE_CONTENT。
- 多层包装后的业务错误仍能显式映射；未识别错误、混合技术故障及无效映射使用 500 / INTERNAL_ERROR，错误链中的内部文本不进入响应；metadata 不跨请求共享。
