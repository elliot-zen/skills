# Go 编码规范

根据用户提供的 `golang-patterns` 文档整理，原文来源标记为 `ECC`。用于 Go 代码编写、审查和重构时的类型、接口、并发及内存使用决策。

项目目录、配置和生命周期以 [Go 工程模式](go-patterns.md)为准；错误、日志、API 和数据库分别遵守对应专题。本文件提供通用编码约定，不引入另一套项目架构、日志实现或工具配置。

## 1. 表达与类型设计

- 优先直接的控制流，先处理错误并提前返回，避免无必要的嵌套、立即执行闭包和隐式控制流。
- 长函数避免裸返回，明确返回值；不使用行数阈值判断函数是否合理。
- 不依赖外部资源的类型尽量让零值可用，例如计数器、缓冲区。需要数据库、配置等必需依赖的类型使用构造函数，不为零值可用而隐藏依赖或自动连接外部系统。
- 修改对象状态或包含锁的类型使用指针接收者，避免复制锁；其余类型按值语义决定，不机械禁止合理的值接收者与指针接收者组合。
- 组合优先使用命名字段；只有确实要将嵌入类型的方法作为外层 API 时才使用 embedding，不为少写转发代码而暴露整个依赖。

零值可用的计数器示例：

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}
```

## 2. 接口与包组织

- 在使用方定义实际需要的最小接口，优先复用 `io.Reader`、`io.Writer` 等标准接口；不为每个具体类型配一份接口。
- 构造函数默认返回具体类型。已有抽象契约需要隐藏实现时可以返回接口，“接受接口、返回结构体”不是硬性签名规则。
- 可选能力使用带 `ok` 的类型断言，例如可选 Flush；调用后仍需处理错误，不能将可选能力检测当作错误检查。
- 包名简短、小写，不使用下划线或重复的角色后缀。项目布局遵守 Go 工程模式，不照搬固定的 handler/service/repository 分层，也不预建 `pkg/`。
- 业务依赖显式传入，不通过 `init()` 读取配置、连接数据库或设置可变全局状态。

```go
// The consumer declares only the behavior it needs.
type UserStore interface {
    Find(ctx context.Context, id string) (User, error)
}

type UserService struct {
    store UserStore
}

func NewUserService(store UserStore) *UserService {
    return &UserService{store: store}
}
```

参数少时直接使用构造参数；可选项较多且需要稳定的构造 API 时，再使用 functional options。必需依赖仍通过显式参数提供。应用的默认配置来自 `configs/config.yaml`，不在 options 或构造函数中重复定义。

## 3. 错误与日志

错误包装、哨兵、具体错误类型、`errors.Is/As` 和 panic 处理遵守 [错误处理](error.md)。底层返回错误，最终处理边界记录；不在通用辅助函数中使用 `log.Fatal` 或退出进程。

诊断信息使用项目的 [结构化日志](logging.md)，不复制原文中 `log.Printf`、`log.Println` 或嵌入自制 Logger 的示例。资源关闭错误按操作语义处理，不能以“其他地方会记录”为由直接忽略。

## 4. 并发与取消

- 有明确并发收益时才创建 goroutine；所有者负责取消和等待结束。可取消操作接收并传递首参 `context.Context`，不将请求上下文长期保存在结构体中。
- 无需错误传播时使用 `sync.WaitGroup`；需要首个错误传播和取消同组任务时使用 `errgroup.WithContext`，通过 `SetLimit` 或固定 worker 数量限制并发。
- worker pool 必须覆盖等待任务、执行任务和交付结果三个阶段的取消。只有确认所有发送者退出后才能关闭结果 channel；明确任务失败是取消整组还是作为单条结果返回。
- 缓冲 channel 可以避免一次结果发送被无人接收阻塞，但不能替代操作本身的取消。异步结果同时携带值与错误，并在完成后关闭 channel。
- 共享结果切片可按固定索引分配给各任务，等待所有任务退出后再读取；不并发 append 或调整同一切片。
- Go 1.22 及以后的循环局部变量通常无需 `i := i`；仍按模块版本和变量实际声明位置判断，不机械添加捕获副本。

下面的单次异步调用将错误交付给调用方，底层函数必须响应上下文取消。调用方负责调用 cancel 并等待结果结束；没有异步消费需求时直接同步调用即可。

```go
type FetchResult struct {
    Data []byte
    Err  error
}

func FetchAsync(
    ctx context.Context,
    fetch func(context.Context) ([]byte, error),
) <-chan FetchResult {
    results := make(chan FetchResult, 1)
    go func() {
        defer close(results)
        if err := ctx.Err(); err != nil {
            results <- FetchResult{Err: err}
            return
        }
        data, err := fetch(ctx)
        results <- FetchResult{Data: data, Err: err}
    }()
    return results
}
```

HTTP 调用使用注入的客户端和请求上下文，超时从项目配置或调用方期限取得。响应体读取与关闭完成前，不提前取消请求上下文。停机使用项目统一关闭总时限，不在辅助函数里另设固定期限。

## 5. 内存与性能

- 已知最终数量时直接分配切片长度并按索引填充；数量仅为上界时可预分配容量，但不能仅为减少分配而无界占用内存。
- 拼接已有字符串列表优先 `strings.Join`；增量构建字符串使用 `strings.Builder`。
- `sync.Pool` 只用于测量确认有收益的临时对象复用，不作为持久缓存。归还前清理状态，归还后不得继续访问对象或其底层内存。
- `bytes.Buffer.Bytes()` 返回的切片共享缓冲区内存。归还缓冲区前复制返回数据，或让调用方提供 `io.Writer`；避免数据被下次复用覆盖。

```go
// Copy before returning the buffer to its pool.
data := bytes.Clone(buf.Bytes())
buf.Reset()
pool.Put(buf)
return data
```

性能调整用 benchmark 和分配数据证明收益；涉及复用和并发时，同时验证数据所有权，不以微小分配收益交换错误或竞态。

## 6. 工具与验证

命令入口统一使用 [SKILL.md](../SKILL.md#3-工程命令)中的 Makefile 目标，不因本规范自动安装 staticcheck、golangci-lint 或改写检查配置。

- 并发修改验证取消、失败交付、channel 关闭及等待退出，并运行适用的 race 检查。
- 对象池或缓冲区复用验证返回数据在后续复用后仍然有效。
- 接口和构造方式调整验证实际调用契约，不为模式本身新增无用途抽象。

原文的 Go 命令清单和旧版 linter 配置不作为项目要求；工具选择、版本和执行时机由技能入口及工程根 Makefile 统一维护。
