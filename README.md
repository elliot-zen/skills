# skills

面向 AI Agent 的技能库。每个技能是 `skills/` 下的一个自包含目录，以 `SKILL.md` 为入口，可附带 `references/` 与 `assets/`。

## 安装

```bash
npx skills@latest add elliot-zen/skills
```

安装后重启 Agent 即可自动生效。

## 技能列表

### write-spec

编写和维护面向 AI Agent 的可复现规格文档（Spec）。核心理念：一个没有聊天记录的新 Agent，仅凭仓库和 Spec 就能复现语义等价的功能。

每个功能在 `docs/biz/<id>/` 下维护三份文档：

- `product.md`：系统对外表现
- `tech.md`：内部实现约束
- `history.md`：重要规格变化及原因

跨业务共享的契约（错误处理、HTTP 约定、认证、日志等）提取到 `docs/biz/common/<id>/`，业务 Spec 只链接不复述。

完整流程见 `skills/write-spec/SKILL.md`。

### scratch-go

指导 Go 后端项目的初始化、开发、修复、重构与审查。初始化时询问 Docker 构建 Go 版本，默认选项为 `1.27.3`；本地 Go 版本允许不同，但均需满足 `go.mod` 的最低要求。Dockerfile 默认使用 Alpine 构建、scratch 运行的多阶段结构，运行身份统一为 `UID/GID 1000:1000`。`SKILL.md` 定义任务导航、Makefile 命令用法与验收流程；各专题按需读取。错误处理采用下层传播、最终边界记录，HTTP 层显式映射安全响应。

- 开发指南：[skills/scratch-go/SKILL.md](skills/scratch-go/SKILL.md)
- Go 版本与容器构建：[skills/scratch-go/references/build.md](skills/scratch-go/references/build.md)
- Go 工程模式：[skills/scratch-go/references/go-patterns.md](skills/scratch-go/references/go-patterns.md)
- Go 编码规范：[skills/scratch-go/references/go-idioms.md](skills/scratch-go/references/go-idioms.md)，根据用户提供的 ECC `golang-patterns` 整理
- 错误处理：[skills/scratch-go/references/error.md](skills/scratch-go/references/error.md)
- API 规范：[skills/scratch-go/references/api-spec.md](skills/scratch-go/references/api-spec.md)
- 数据库规范：[skills/scratch-go/references/database-spec.md](skills/scratch-go/references/database-spec.md)
- MySQL/MariaDB 模式：[skills/scratch-go/references/mysql-patterns.md](skills/scratch-go/references/mysql-patterns.md)，根据用户提供的 ECC `mysql-patterns` 整理
- 日志规范：[skills/scratch-go/references/logging.md](skills/scratch-go/references/logging.md)
- 工程 Makefile 模板：[skills/scratch-go/assets/Makefile](skills/scratch-go/assets/Makefile)
- 容器构建模板：[Dockerfile](skills/scratch-go/assets/Dockerfile)、[.dockerignore](skills/scratch-go/assets/.dockerignore)

本技能由 `elliot-zen/scratch-go` 迁入（源提交 `b6a70e1da2379580708916aa49c87ba53eddb18d`），技能名称为 `scratch-go`。
