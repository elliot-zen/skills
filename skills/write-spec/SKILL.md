---
name: write-spec
description: 编写和维护面向 AI Agent 的可复现规格文档。在创建或修改功能、行为或缺陷修复前使用.
---

# Write Spec

## 目标

Spec 描述系统当前成立的事实，不描述开发过程。

唯一验收标准：

> 一个未参与前序讨论的新 Agent，只拥有仓库和 Spec，能否复现语义等价的功能？

语义等价指产品行为、状态、API、错误、数据约束、关键技术约束一致，不要求代码相同。

## 文档结构

```text
docs/
├── documentation-guide.md   # 项目级文档规范（若存在）
├── common/                  # 跨业务共享的契约与约定
│   └── <id>/
│       ├── product.md
│       ├── tech.md
│       ├── history.md
│       └── references/      # 本 Spec 所依赖的外部接口快照（按需）
└── biz/
    └── <id>/
        ├── product.md       # 系统对外表现
        ├── tech.md          # 内部实现约束
        ├── history.md       # 重要规格变化及原因
        └── references/      # 本 Spec 所依赖的外部接口快照（按需）
```

`biz/` 与 `common/` 使用相同的三文档结构。管理跨业务复用的公共事实（统一错误处理、HTTP 约定、认证、日志等）见 `references/document-model.md`。

## 工作流程

1. 读取 `docs/documentation-guide.md`（若存在），确定业务 `<id>` 与 Scope。
2. 读取现有 `product.md`、`tech.md`、`history.md`（含相关 `docs/common/` 公共 Spec），以及仓库中相关的设计文档。若实现依赖外部接口文档，提取复现所需的接口内容，保存到该 Spec 目录的 `references/`，并记录来源 URL、版本或获取日期及定位信息。
3. 将收集到的事实分类：Product Facts / Technical Facts / History / Unknown。
4. 识别公共事实：已存在于公共 Spec 的改为链接；未存在但被复用或属于项目级约定的提取到 `docs/common/`。
5. 更新 `product.md`。
6. 更新 `tech.md`。
7. 仅在存在重要规格变化时更新 `history.md`。
8. 按 `references/review-checklist.md` 做可复现性检查。
9. 确认 Spec 与最终实现一致。

未确认现有 Spec（含公共 Spec）是否已拥有该行为前，不要新建 Spec。

## 参考文件

| 时机 | 文件 |
| --- | --- |
| 开始前 | `references/principles.md`、`references/document-model.md` |
| 判断具体写法 | `references/writing-rules.md` |
| 完成后 | `references/review-checklist.md` |
| 新建文档 | `templates/product.md`、`templates/tech.md`、`templates/history.md` |

## 核心约束

- `product.md` 写系统对外是什么，`tech.md` 写内部必须满足什么，`history.md` 写重要变化为什么发生。
- 同一事实只定义一次。跨业务复用的公共契约放入 `docs/common/`，业务 Spec 只链接不复述。
- 只写当前事实。决策过程、方案比较、TODO、开发进度一律不写，完整历史交给 Git。
- 对复现无用的事实不写，缺失会导致猜测的事实必写。
- 外部接口文档是实现依赖时，必须将复现所需的接口定义保存到对应 Spec 的 `references/`，并由 `tech.md` 链接该本地文件。快照只保留所需接口、字段、错误和约束，注明原始 URL、版本或获取日期及相关章节/操作；不能只引用会变化的外部页面。无法确认的接口约束标为 Unknown，不将其推断为确定事实。
- Unknown 不得写成确定事实。
- 代码与 Spec 冲突时，先判断是实现偏离 Spec 还是 Spec 错误，不默认以代码为准。
