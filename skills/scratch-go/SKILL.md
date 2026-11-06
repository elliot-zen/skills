---
name: scratch-go
description: 指导 Go 后端项目的开发、修复、重构与审查，提供项目结构、错误处理、日志、HTTP API、数据库及构建验证约定。
---

# Go 后端工程开发指南

## 1. 适用范围与执行流程

适用于 Go 服务、后台任务程序（Worker）和命令行程序（CLI）。HTTP 和数据库规则仅在项目具有相应能力时适用，不预建无实际用途的模块和依赖。

“必须”“不得”表示硬性要求；未加限定的规范性陈述同样需要落实。“默认”表示无明确项目约定或业务特例时采用的方案；调整时说明原因，并同步实现、契约与测试。“例如”仅作说明，不要求创建示例业务。

开始工作前读取项目的 `AGENTS.md`、相关源码、测试、依赖与 Makefile，确认实际版本、用户要求和未提交修改。保留与当前任务无关的文件和用户修改。实现调整时同步受影响的契约、文档和测试。

按下表读取任务涉及的专题，完成实现和对应行为验证，再执行第 4 节要求的工程检查。每条规则由一个专题维护，其他文件只引用；不默认加载全部参考文档。

## 2. 专题导航

| 任务 | 阅读内容 |
| --- | --- |
| Go 类型、接口、代码表达、并发模式或内存复用的编写与审查 | [Go 编码规范](references/go-idioms.md)的相关章节 |
| 包组织、配置、外部依赖、Context、并发或服务运行 | [Go 工程模式](references/go-patterns.md)的相关章节 |
| 创建、包装、转换或处理错误，以及 panic 恢复 | [错误处理](references/error.md) |
| 日志输出、请求完成记录、第三方交互或脱敏 | [日志规范](references/logging.md)；涉及故障记录责任时同时读错误处理 |
| HTTP API 开发或审查 | [API 规范](references/api-spec.md)及 Go 工程模式第 3 章；涉及失败路径时读错误处理与日志规范 |
| 数据库表结构、迁移或数据访问 | [数据库规范](references/database-spec.md)；外部调用和错误转换按相关专题执行 |

各专题包含自身验证要求；公共命令与执行时机只在本文件定义。

## 3. 工程命令

从工程根目录使用 GNU Make 和 POSIX shell。[assets/Makefile](assets/Makefile)提供命令模板，`SERVICE` 对应主要服务名；实际命令以工程根 Makefile 为准，不自动用模板覆盖。

| 命令 | 用途与行为 |
| --- | --- |
| `make build` | 构建 `cmd/$(SERVICE)` 到 `bin/$(SERVICE)`，注入 `main.version`；发布时指定可追溯的 `VERSION` |
| `make run` | 构建后前台运行，不加载 `.env` |
| `make dev` | 构建后通过 shell 加载根目录 `.env` 并以前台 exec 运行；具体覆盖规则见 Go 工程模式第 2.2 节 |
| `make fmt` | 使用所选 Go 工具链的 gofmt 修复源码格式，跳过 `.git`、`.tools`、`vendor` |
| `make test` | 运行整个模块测试，默认禁用测试结果缓存；也是 Makefile 默认目标 |
| `make lint` | 执行 `go vet ./...` 静态检查，失败返回非零；不修改源码、不安装额外工具 |
| `make coverage` | 运行整个模块测试，生成 `coverage.out`，输出函数级及总覆盖率 |

`build/run/dev` 默认使用 `SERVICE` 指定的服务，可通过 `make build SERVICE=orders-worker` 等方式切换。`VERSION` 开发默认 `dev`，发布示例为 `make build VERSION=v1.2.3`。Go 版本约束只维护在 `go.mod`。

`test` 和 `coverage` 共用 `GO_TEST_FLAGS ?= -count=1 -v -timeout=5m -json`，默认输出详细 JSON 测试事件，每个包超时 5 分钟。可覆盖，例如：

```sh
make test GO_TEST_FLAGS='-count=1 -timeout=5m -run TestCreateOrder'
make test GO_TEST_FLAGS='-count=1 -timeout=5m -race'
HTTP_ADDR=:8081 LOG_LEVEL=debug make run
```

`coverage` 使用 atomic 模式，不设置统一覆盖率门槛；用 `go tool cover -html=coverage.out` 定位未覆盖路径，覆盖率不能替代行为断言。`fmt` 只格式化，`lint` 只静态检查，两者互不代替；`coverage` 已包含测试，同次验收无需再执行等价的 `test`。

默认不额外执行依赖校验、模块一致性检查或安装 golangci-lint；项目已配置的检查继续执行。漏洞扫描由项目发布流程配置。

## 4. 验证与交付

| 变更 | 必需检查 |
| --- | --- |
| Go 开发、修复、重构 | `make fmt`、`make lint` 及相关测试；需要覆盖率时以 `make coverage` 替代等价测试 |
| 错误、日志或并发修改 | 上述检查，加上对应专题的失败、取消、重复记录或资源清理反例验证 |
| 配置或服务生命周期修改 | 适用检查，加上配置覆盖、非法配置启动失败、监听与关闭行为验证 |

缺少工具或环境能力时标明未执行项及原因，不绕过或降级检查。交付记录源码及工具版本、执行命令、实际结果、运行入口和必要偏离；复核职责边界、抽象必要性及诊断信息是否足够。
