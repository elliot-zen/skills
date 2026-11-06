# 错误处理

采用 samber 的单次处理原则：错误要么向调用方返回，要么在最终处理边界记录，不同时做两者。内部错误用于诊断，对外文案由协议适配层单独生成。日志字段见 [日志规范](logging.md)，HTTP 状态与响应结构见 [API 规范](api-spec.md)。

## 1. 错误定义

业务函数使用标准 `error`，通常作为最后一个返回值。调用方检查返回错误后再使用结果，不用 `_` 静默丢弃错误。内部错误文本使用小写英文、无结尾标点，描述失败动作，不添加重复的包名、类型名或笼统的 `error occurred` 前缀。

| 调用方需要 | 定义方式 |
| --- | --- |
| 识别某个预期条件 | 在所属业务包 `errors.go` 中定义 `ErrXxx` 哨兵，使用 `errors.New` 一次创建 |
| 读取字段、规则或其他结构化信息 | 在所属包定义具体错误类型，实现 `Error()`；包装其他错误时实现 `Unwrap()` |
| 仅诊断失败，无需匹配 | `errors.New` 或 `fmt.Errorf` |
| 在生产中携带堆栈、用户/租户上下文或结构化属性 | 使用 `github.com/samber/oops`，固定依赖版本；简单错误无需引入 |

错误类型表达业务语义，不依赖 Gin，不携带 HTTP 状态、JSON 响应或日志是否已记录等状态。不预建统一错误基类或集中罗列未使用的错误；只导出调用方确实需要识别的值和类型。

```go
// Package order owns its business errors.
var ErrAlreadyPaid = errors.New("order already paid")

type QuantityError struct {
    Min int
    Max int
}

func (e *QuantityError) Error() string {
    return fmt.Sprintf("quantity outside range [%d, %d]", e.Min, e.Max)
}
```

`Error()` 是内部诊断文本，不承诺可直接公开。共享哨兵不保存请求数据，具体错误实例按次创建；返回 `error` 的成功路径返回真正的 `nil`，不返回装有 nil 指针的错误接口。

## 2. 包装、识别与转换

内部传播使用 `fmt.Errorf("operation: %w", err)` 保留错误链，在职责边界补充有意义的操作上下文；不重复添加相同描述。使用 `errors.Is` 匹配哨兵、`errors.As` 提取具体类型；所用 Go 版本支持时可使用 `errors.AsType`。不依赖错误字符串、直接相等比较或裸类型断言识别包装后的错误。

```go
func (s *Service) Pay(ctx context.Context, id string) error {
    if err := s.repo.MarkPaid(ctx, id); err != nil {
        return fmt.Errorf("mark order paid: %w", err)
    }
    return nil
}
```

SDK 或数据库错误由适配层识别。只有确认记录不存在、特定唯一约束冲突等语义时，才转换为所属业务包的对应错误；连接失败、超时、第三方凭证失效不能伪装为业务拒绝。技术故障保留底层原因与操作上下文；有意转换为领域错误时，调用方依赖领域语义，不依赖驱动类型。

公开 Go 库/API 边界需要隐藏实现类型时，可使用 `%v` 截断解包关系，先明确对调用方承诺的错误语义；内部调用链不随意截断。`%v` 仍包含底层文本，不能用来脱敏或生成安全的 HTTP 响应。

独立失败需要一起返回时使用 `errors.Join`，例如主要操作和清理同时失败，避免丢失任一原因。日志责任仍归最终处理方，汇总为一次故障事件；不逐分支记录后再返回聚合错误。聚合技术故障与业务拒绝时，不得因匹配到其中一个业务哨兵就把整体错误映射成 4xx；无法确定公开语义时使用内部错误兜底。

## 3. 单次处理与日志责任

返回错误表示把处理责任交给调用方，此时不记录同一故障。最终处理边界消费错误、决定响应或退出，并使用带上下文的 slog 记录一次。不得通过错误包装标记、请求级已记录布尔值或错误去重注册表协调日志位置。

| 场景 | 最终处理位置 |
| --- | --- |
| 普通 HTTP 请求 | 错误出口保存结果，统一请求完成边界记录；业务层、依赖层和路由处理器不另记故障 |
| Worker 单次任务 | 决定最终成功、重试耗尽或失败处置的任务执行边界；返回给 supervisor 处理的错误由 supervisor 记录 |
| 启动、CLI 或进程级失败 | `run` 返回错误，`main` 记录并设置退出码；`run` 不先记录 |
| WebSocket 或接管连接 | 协议适配层的统一关闭边界，读写循环只交付错误 |

若当前层实施了有效恢复或降级，可按需记录一次 WARN，并返回实际恢复后的结果；不能为避免返回错误而吞掉失败或伪造成功。预期业务拒绝按需 INFO，非预期失败 ERROR，正常关闭或单纯客户端取消不记故障 ERROR。

重试尝试日志用于观察交互，不是最终故障处理；最终失败由拥有处理责任的边界记录，成功恢复后不再记整体失败。HTTP 故障详情合并到一次请求完成事件，避免另起一条内容重复的 ERROR。

稳定的日志消息用于聚合，ID、依赖名、路径和计数放入结构化字段。业务位置独有的上下文通过具体错误字段或 oops 属性向上携带，而不是提前记录。使用 oops 时在边界显式对接 slog，验证字段、原因链和堆栈能按预期输出，不假定任意 Handler 会自动展开属性；属性和堆栈也必须脱敏。

```go
// The caller handles the error; this function does not log it.
if err := chargeCard(ctx, order); err != nil {
    return fmt.Errorf("charge card: %w", err)
}

// A task boundary consumes the failure and ends this attempt.
if err := service.Pay(ctx, orderID); err != nil {
    logger.ErrorContext(ctx, "payment failed", "order_id", orderID, "error", err)
    return // No error is propagated to another logging boundary.
}
```

## 4. 对外错误映射

HTTP 层使用 `errors.Is/As` 显式识别已声明的业务错误，映射状态、稳定错误码和安全文案。没有明确映射的错误一律使用通用 500；不直接返回 `err.Error()`，不序列化原始错误或 oops 属性。

例如 `order.ErrAlreadyPaid` 映射到 `409 / ORDER_ALREADY_PAID / 订单已支付，不能重复付款`。`order.QuantityError` 只公开接口约定的范围字段，并生成 400 文案；SQL、依赖 URL、凭证、内部包装上下文仅用于脱敏后的内部诊断。

映射函数只转换，不记录日志；统一错误出口验证映射有效性，保存原始失败供完成边界记录。具体输出字段、无效映射兜底与参数校验规则由 [API 规范](api-spec.md#4-错误与参数校验)定义。

## 5. panic 与 recover

网络失败、参数错误、资源不存在等预期失败返回 `error`，不 panic。panic 仅用于程序错误、不可能状态或损坏的不变量；不能作为普通控制流程。

在能够隔离失败的 goroutine 边界通过 defer/recover 处理 panic，捕获类型、值及 `debug.Stack()`，执行必要清理并明确标记失败，不能恢复后伪装为成功。recover 只作用于当前 goroutine；子 goroutine 的失败需要交给其所有者处理。如果转成错误返回，则当前层不记录；如果边界消费失败，则记录一次。无法保证进程状态安全时，应终止进程，而不是无限恢复。

HTTP 的恢复位置、响应提交后的连接中断及接管连接清理由 [Go 工程模式第 3.2 节](go-patterns.md#32-请求处理与完成边界)定义。普通 panic 保存详情，在完成边界记录。`http.ErrAbortHandler` 是传输中断信号，原样继续抛出，不当作新的业务 panic，不追加响应或重复堆栈。

## 6. 验证

- 经实际业务调用链验证多层 `%w` 包装后 `errors.Is/As` 仍有效，具体错误字段可提取，技术原因保留；有意截断的公开 Go API 边界符合声明的错误契约。
- 验证错误检查和清理路径：主要失败与清理失败同时存在时均可识别；成功返回 nil；SDK 故障不误转为业务拒绝。
- 捕获生产日志验证下层返回但不记录，最终边界一次记录；多层包装、重试耗尽及独立错误聚合不重复输出，恢复成功不误记最终失败。
- 通过实际 HTTP 错误出口验证已知错误显式映射，未知错误及混合技术故障兜底 500；即使错误链包含内部文本，也不会出现在响应的 message 或 metadata。
- panic 的堆栈能定位实际故障；恢复后资源清理、失败结果和后续请求行为正确。HTTP 中断语义按 Go 工程模式验证。
- 实际使用 oops 时验证属性和堆栈提取、`errors.Is/As` 兼容以及敏感信息不泄漏。

参考：[samber 错误处理](https://github.com/samber/cc-skills-golang/blob/bb1d43d98a3f8ab639a0042f90d0ec524e39a477/skills/golang-error-handling/SKILL.md)。本规范采用其标准错误、上下文包装、单次处理和边界安全转换原则，并明确本项目 HTTP 完成边界的集成方式。
