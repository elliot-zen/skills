# skills

[English](./README.md) | [中文](./docs/README.zh-CN.md)

A collection of agent skills. Each skill is a self-contained directory under
`skills/` with a `SKILL.md` entry point plus optional `references/` and
`templates/`.

## Quick Start

Install a skill with the [Skills CLI](https://skills.sh):

```bash
# Interactive picker: choose skills and target agents
npx skills@latest add elliot-zen/skills

# Install a specific skill
npx skills@latest add elliot-zen/skills --skill write-spec

# Install globally for a specific agent
npx skills@latest add elliot-zen/skills --skill write-spec -g -a claude-code
```

Installed skills
are picked up automatically by your agent; restart the agent if it was
already running.


## Available skills

### write-spec

Write and maintain reproducible specifications for AI agents. The goal is that a
new agent with no chat history or author context can reproduce semantically
equivalent behavior from the repository and the spec alone.

1. Read the project documentation guide, then the existing spec for the
   affected `<id>` before touching code.
2. Keep three documents per feature under `docs/biz/<id>/`: `product.md`
   (external behavior), `tech.md` (implementation constraints), and
   `history.md` (why the spec changed). Cross-cutting contracts shared by
   multiple features (error handling, HTTP conventions, auth, logging, …) are
   extracted to `docs/common/<id>/` once a second consumer appears, and the
   feature specs link to them instead of restating them.
3. Only start implementation after the spec is updated and validated.

See `skills/write-spec/SKILL.md` for the full workflow,
`skills/write-spec/references/` for the rules, and
`skills/write-spec/templates/` for the document templates.
