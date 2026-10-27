# skills

面向 AI Agent 的技能库。每个技能是 `skills/` 下的一个自包含目录，以 `SKILL.md` 为入口，可附带 `references/` 与 `templates/`。

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

指导 Go 后端项目的初始化、开发、修复、重构与审查。手册按工作流程、通用工程规范、配置与外部依赖、HTTP 服务、初始化与部署、验证与交付六章组织，配套提供 Makefile 模板。

- 开发指南：[skills/scratch-go/SKILL.md](skills/scratch-go/SKILL.md)
- 工程 Makefile 模板：[skills/scratch-go/Makefile](skills/scratch-go/Makefile)

本技能由 `elliot-zen/scratch-go` 迁入（源提交 `b6a70e1da2379580708916aa49c87ba53eddb18d`），技能名称为 `scratch-go`。
