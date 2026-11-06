# 日志规范

适用于业务、HTTP 请求、外部依赖及程序生命周期日志。配置加载、错误传播和 HTTP 完成边界遵守 [scratch-go](../SKILL.md)。

## 1. 输出与公共字段

新项目使用 `slog.JSONHandler` 输出到 stdout，一条事件一行 JSON；已有项目沿用满足本规范的结构化日志方案。HTTP 服务器内部日志接入同一输出。

容器收集 stdout/stderr 并设置日志轮转，例如 `json-file` 的 `max-size: 10m`、`max-file: 3`。

| 字段 | 要求 |
| --- | --- |
| `time`、`level`、`msg` | 时间、日志级别、简洁的事件描述；不另设 `log_level` 字段 |
| `service`、`version` | 基础记录器统一附加；入口定义构建可赋值的 version，开发允许 `dev`，发布使用可追溯标识 |
| `logger` | 稳定模块名，如 `order.service`、`order.repository`、`http.access` |
| `request_id` | 请求上下文存在时自动附加，贯穿业务与依赖调用 |

使用应用配置 `log.level` / `LOG_LEVEL`，默认 `info`，支持 `debug/info/warn/error`。字段按事件需要添加，空的可选字段省略；日志不输出 `outcome`。

上下文取值由统一日志封装或 Handler 实现；`slog.*Context` 本身不会附加请求 ID。日志封装必须保留实际调用位置。

## 2. 业务错误与故障定位

错误分类与传播遵守 [scratch-go 第 2.2 节](../SKILL.md)。在掌握诊断上下文的业务位置记录 ERROR，允许记录后继续返回错误。同一故障的完整详情只记录一次，传播层补充错误上下文，不逐层重复打印；HTTP、任务或启动边界负责补记尚未记录的故障。

统一日志包提供 `RecordError(ctx, logger, err, msg, attrs...) error`：未记录的故障输出详情后，返回携带已记录标记的错误包装；已记录则原样返回。包装的 `Error()` 委托原错误，`Unwrap()` 保留错误链，单条包装链通过 `errors.As` 检查标记。业务调用方必须传播返回的错误，边界使用同一函数补记。标记只表示该故障已记录，不使用请求级布尔值，不修改共享哨兵；聚合错误逐分支识别和补记，不能因其中一个分支已记录而跳过其他故障。

| 情形 | 级别与内容 |
| --- | --- |
| 非预期失败、最终依赖故障 | ERROR，记录具体失败动作、`error`、相关业务 ID、已知的 `dependency` |
| 预期业务拒绝 | 按需记录 INFO，不因返回 error 自动升级为 ERROR |
| 可恢复但需关注的异常 | WARN，说明实际影响与处理结果 |
| panic | ERROR，记录 panic 类型、脱敏后的值和恢复点捕获的 `stack`，包含实际故障函数、源码文件及行号 |
| 正常关闭、单纯客户端取消 | 不作为故障 ERROR；同时发生的独立故障仍记录 |

ERROR 附 `source`，包含实际日志调用处的文件、行号及函数；它不代表底层错误最初产生的位置。边界补记时 source 指向边界调用处，业务定位信息由错误包装上下文和业务字段提供。`error` 保留脱敏后的错误链和依赖原因，不能统一替换为 `internal error`。普通错误默认不打印完整堆栈。

`logger` 与 `msg` 已能描述动作时不再要求 `operation`。稳定业务错误码确有诊断价值时附 `error_code`，与响应中的错误码保持一致，不机械复制响应的全部字段。任务按需附 `job_id`，业务标识不得携带秘密。

重试中间失败由交互日志记录；重试或明确降级后成功，不记整体失败 ERROR。最终失败的详情保留尝试次数和最终原因。

业务 ERROR 与 HTTP 完成摘要可以同时存在，通过 `request_id` 关联；完成摘要不重复业务上下文和堆栈。

## 3. 请求与连接日志

### 3.1 HTTP 请求完成

每个请求在统一完成边界生成一条日志，`logger=http.access`、`msg="Request Completed"`。默认 INFO 下记录所有请求，包括 200、201、204、304、HEAD、404/405 和运维端点；不按成功状态过滤或采样。提高应用日志级别后按级别过滤。

| 字段 | 取值 |
| --- | --- |
| `request_id` | 当前请求 ID |
| `method` | 实际请求方法 |
| `path` | 实际 URL path，不含 query；敏感路径段按第 5 节处理 |
| `status` | 实际已提交状态；中断且未提交时为 `null`，不得填框架预置的 200 |
| `duration_ms` | 请求进入到 [scratch-go 第 4.2 节](../SKILL.md)规定的完成边界的单调计时，单位毫秒 |
| `remote_addr` | 按可信代理配置解析的客户端地址，不盲信转发头 |
| `size` | 应用向响应写入器写入正文时累计返回的字节数，不包含响应头，不代表网络实际发送量或客户端接收量；HEAD、204、304 及未写正文时为 0 |

正常响应及预期 4xx 使用 INFO；5xx、非预期失败和 panic 使用 ERROR。即使已提交 200，后续失败仍使用 ERROR，并保留实际 status。客户端取消本身使用 INFO，附 `error="request canceled"`；有独立故障时按故障级别记录。

失败时按需附脱敏后的简短 `error`，有诊断价值时附 `error_code`；无内部错误对象的 5xx 使用状态描述，不编造原因。故障详情按第 2 节处理。

不默认记录 query、headers、body、referer，也不要求 `route`、`handler` 或重复 status 的 `code`。路由模板及 `unmatched` 仅用于指标；未匹配请求仍记录实际 path。

### 3.2 连接接管与断开

实际使用 WebSocket 或连接接管时，HTTP 完成日志记录握手结果和耗时，连接关闭另记一条 `msg="Connection Closed"` 日志。协议适配层统一负责关闭记录，避免读写循环、关闭回调重复打印。

断开日志包含 `connection_id`、建立连接时的 `request_id` 和连接持续时间 `duration_ms`；协议提供关闭码时附 `close_code`，有关闭原因时附 `close_reason`。正常断开或服务正常关闭使用 INFO；异常断开、业务故障或 panic 导致的关闭使用 ERROR，附简短 `error`，故障详情与 panic 堆栈按第 2 节记录，已有详情不重复输出。

## 4. 第三方 HTTP 交互

复用统一 HTTP 适配与日志能力，由传输捕获结合响应体读取、解析及第三方业务契约确定完成结果。不能仅凭响应头或 HTTP 200 判定成功。

| 有效级别 | 每次实际尝试的日志 |
| --- | --- |
| DEBUG | `dependency.request`、`dependency.response` 两条 DEBUG 事件，记录请求与响应头、普通文本报文及完成信息；不额外输出 INFO 摘要 |
| INFO | 结束时一条 `dependency.call` INFO 摘要 |
| WARN / ERROR | 不捕获常规交互报文；最终故障仍按第 2 节记录 |

按记录器的 `Enabled` 选择路径，一次尝试内保持一致；未启用的级别不捕获或格式化报文。不增加独立报文日志开关。

| 字段 | 约定 |
| --- | --- |
| `dependency`、`operation` | 第三方系统名称及稳定操作名 |
| `call_id`、`attempt` | 同一逻辑调用共用 call_id，各次尝试从 1 递增；上游 request_id 按第 1 节附加 |
| `method`、`url` | 请求方式及脱敏后的目标地址，查询参数按第 5 节处理 |
| `request`、`response` | 可安全记录的文本报文，DEBUG 完整记录，INFO 按下述上限截断 |
| `status` | 实际取得的 HTTP 状态；无响应时省略 |
| `duration_ms` | 本次尝试开始到响应体处理结束或失败的单调计时，单位毫秒 |
| `error` | 传输、读取、解析或第三方业务失败的脱敏原因；成功时省略 |
| `request_truncated`、`response_truncated` | 对应摘要是否超过长度上限 |
| `response_complete` | 响应体是否完整取得；未取得响应、读取失败、取消或提前关闭时为 false |

INFO 的 request、response 各保留前 1024 个 Unicode 码点，不切断 UTF-8 字符；超过上限标记 truncated。摘要作为 JSON 字符串输出，不要求其内容仍是合法 JSON。先处理敏感字段再截断；只捕获有限前缀及必要状态，不为摘要完整缓存报文。

文件、二进制和流式内容只记录类型、已处理字节数等描述，不为日志预读或整体缓冲。日志捕获不得改变请求字节、消耗业务响应体，或破坏关闭、取消和重试重放语义。

超时、取消、连接失败、读取或解析失败均产生完成事件；部分响应如实记录完成标记。每次尝试只完成记录一次，不因 EOF、关闭或异常处理重复输出。

## 5. 敏感信息

敏感值统一替换为 `*******`，适用于所有级别，包括 DEBUG。脱敏只作用于日志副本，不修改业务数据、请求或响应。

| 位置 | 处理 |
| --- | --- |
| 配置 | 仅输出需要诊断的配置项；密码、secret、token、API key、私钥等敏感项保留名称，值替换为 `*******` |
| HTTP 头 | 仅选择需要的头；Authorization、Proxy-Authorization、Cookie、Set-Cookie 及依赖自定义凭证头如需出现，整值替换 |
| URL | 屏蔽 userinfo 中的凭证、敏感查询参数值及敏感路径段；入站完成日志不记录查询参数 |
| 报文与业务字段 | 按接口契约显式选取可记录字段，在已知敏感字段处替换；不直接打印含秘密的完整对象 |
| error、msg、panic、close_reason | 仅替换其中的凭证、带密 URL 中的敏感值及敏感业务数据；保留动作、错误类别、关闭原因及安全的诊断上下文，不将整条原因统一替换为掩码 |

在配置加载、依赖适配和业务日志调用处明确敏感字段；共用简单的掩码、配置键及 URL 处理函数，不要求通用递归脱敏框架。普通日志记录器不会自动识别字符串或任意嵌套对象中的秘密。

无法可靠处理的报文省略内容，仅记录类型、字节数或安全字段；此规则优先于 DEBUG 完整报文和 INFO 摘要。不得先截断原文再依靠完整 JSON 解析脱敏，也不得将处理失败的原文作为回退输出。

## 6. 验证

捕获生产记录器的 JSON 输出，验证以下适用行为：

- 正常响应、无正文响应、预期 4xx 各有一条 INFO 完成日志；path、status 和非负 duration_ms 正确，日志与指标计时范围一致。size 累计写入返回的字节数，部分写入只计实际返回的数量，HEAD、204、304 为 0。
- 5xx、panic、响应提交后失败产生 ERROR 完成摘要；保留实际状态，未提交中断不伪造 200；客户端取消不误报故障。
- 实际使用连接接管时，连接保持打开期间即生成握手完成日志，耗时与 HTTP 指标一致，后续关闭不重复生成 HTTP 完成日志。正常断开及正常停机关闭记 INFO，异常断开记 ERROR；同一连接仅有一条关闭日志，能关联握手与业务日志。关闭原因中的安全文本保留，仅敏感值替换为 `*******`。
- 业务 ERROR 包含模块、原因链、关联 ID 和业务调用处 source；panic 的 stack 能定位实际故障位置。已记录错误经包装传播后不重复详情，未记录及同一请求内的独立故障均能补记；记录标记不改变 errors.Is/As，正常关闭不记 ERROR。
- 公共字段完整，级别过滤有效，不输出 outcome；请求 ID 能关联业务、访问及依赖日志。
- 第三方 DEBUG 每次尝试两条交互日志，INFO 一条摘要，WARN/ERROR 不捕获常规报文；重试可通过 call_id、attempt 关联。失败、取消和部分响应仅完成记录一次。
- 覆盖 1024 码点边界、多字节字符及脱敏后的截断，核对 status、error、response_complete；日志不改变请求、响应读取和资源关闭语义。
- 用测试凭证覆盖配置、头、URL、报文及错误文本，确认各级别均无明文泄漏，掩码严格为 `*******`，无法安全处理时不回退原文。

参考实现：[Grafana 请求日志](https://github.com/grafana/grafana/blob/main/pkg/middleware/loggermw/logger.go)、[业务日志](https://github.com/grafana/grafana/blob/main/pkg/services/authn/authnimpl/sync/user_sync.go)、[配置脱敏](https://github.com/grafana/grafana/blob/main/pkg/setting/setting.go)。具体要求以本规范为准。
