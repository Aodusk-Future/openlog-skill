# OpenLog

> A lightweight per-module development journal for [Claude Code](https://claude.com/claude-code) — keep architecture, dev log, and current-stage notes for every module of a large project, so any future session or agent can pick up the work in under 1k tokens.
>
> 一个轻量级的"按模块开发档案"工具，用于 [Claude Code](https://claude.com/claude-code)——为大型项目的每个模块沉淀架构说明、开发日志、当前断点，让未来任何会话 / agent 都能在 1k token 以内接手工作。

[中文版说明 ↓](#中文)

---

## Why OpenLog?

When a project grows past a handful of modules, three things start to hurt:

1. **High handoff cost** — a new session (or a new agent) has to re-derive "where did we stop?" and "why is it built this way?"
2. **Design / implementation drift** — architectural decisions live in chat history and slowly disappear.
3. **Wasted tokens on re-exploration** — every fresh context has to grep / read the same files to rebuild understanding.

OpenLog fixes this by writing the three pieces of context that matter — **architecture, dev log, current breakpoint** — into three fixed files per module, then standardising how Claude reads and updates them.

## Quick Start

```bash
# Clone directly into Claude Code's user skills directory
git clone https://github.com/aodusk/openlog-skill ~/.claude/skills/openlog
```

Then in any Claude Code session, type:

```
/openlog
```

…or just say something like "let's work on the Combat module" / "start the Combat module" — Claude will recognise the trigger and run the OpenLog workflow.

## How It Works

### Data layout (lives in your project root)

```
OpenLog/
├── INDEX.md                          # Minimal table of all modules + one-liner + status
├── <ModuleA>/
│   ├── ARCHITECTURE.md               # Module architecture — kept up to date
│   ├── DEV_LOG.md                    # Dev log, newest entries on top
│   ├── DEV_LOG.archive/              # Auto-archive when DEV_LOG > 500 lines
│   │   └── YYYY-MM.md
│   └── CURRENT.md                    # Current breakpoint, <30 lines
└── <ModuleB>/...
```

The `OpenLog/` directory is committed to your project's git, so multiple worktrees / multiple agents / multiple humans share the same journal.

### Two-phase workflow

**Startup phase** (when you invoke the skill):
1. Read `OpenLog/INDEX.md` (the only full read).
2. Try to fuzzy-match the module name from what you just said.
3. If matched → read ARCHITECTURE (full), DEV_LOG (first ~40 lines = newest ~5 entries), CURRENT (full). Output a 2-3 sentence handoff summary.
4. If not matched → ask which module, or create a new one.

**Wrap-up phase** (only triggered by explicit user signal — "done", "wrap it up", "next task", `/openlog commit`, etc.):
1. Prepend a new entry to DEV_LOG (timestamp + summary + key files + link to CURRENT items).
2. Update CURRENT (move "in progress" items, log new blockers, refresh timestamp).
3. Update ARCHITECTURE **only** if interfaces / dependencies / key files changed.
4. Auto-archive old DEV_LOG entries when the file exceeds 500 lines.

### Designed for low token cost

- Resuming a module: typically **700-1000 tokens** of reads, no grep, no whole-codebase exploration.
- Creating a module: **~300 tokens**.
- Wrap-up: **~150-400 tokens** (ARCHITECTURE updates are skipped most of the time).

## Customisation

The skill is a single markdown file at [`SKILL.md`](SKILL.md). Edit the rules, file templates, or wrap-up triggers to match your team's conventions.

A reference English-language version of the skill body is provided at [`SKILL.en.md`](SKILL.en.md). To use it, replace the YAML body of `SKILL.md` with the English content, or rename it directly.

## License

[MIT](LICENSE) — do whatever you want with it; attribution appreciated but not required.

---

<a id="中文"></a>
# 中文

## 为什么需要 OpenLog？

当项目模块数超过几个，就会遇到三个痛点：

1. **接手成本高**——新会话或新 agent 要重新摸清"上次做到哪""为什么这么设计"
2. **设计与实现分裂**——架构决策散落在对话里，没有沉淀
3. **重复探索浪费 token**——每次新上下文都要 grep / read 同一批文件重建理解

OpenLog 把"架构说明 + 开发日志 + 当前断点"沉淀到每个模块固定的三个文件里，并标准化 Claude 读写它们的方式。

## 快速上手

```bash
# 直接 clone 到 Claude Code 的用户 skills 目录
git clone https://github.com/aodusk/openlog-skill ~/.claude/skills/openlog
```

之后在任何 Claude Code 会话里输入：

```
/openlog
```

…或者直接说"开始开发 Combat 模块" / "接手战斗模块"等，Claude 会识别触发条件并走 OpenLog 工作流。

## 工作原理

### 数据布局（位于项目根目录）

```
OpenLog/
├── INDEX.md                          # 模块清单（极简表格 + 一句话定位 + 状态）
├── <模块A>/
│   ├── ARCHITECTURE.md               # 模块架构（保持最新）
│   ├── DEV_LOG.md                    # 开发日志（倒序，最新在最上）
│   ├── DEV_LOG.archive/              # DEV_LOG > 500 行时自动归档
│   │   └── YYYY-MM.md
│   └── CURRENT.md                    # 当前断点（< 30 行）
└── <模块B>/...
```

`OpenLog/` 目录跟随项目 git 仓库提交，团队 / 多 worktree / 多 agent 共享同一份档案。

### 两阶段工作流

**启动阶段**（唤起 skill 时）：
1. Read `OpenLog/INDEX.md`（唯一允许的全量读）
2. 尝试从你的指令里模糊匹配模块名
3. 匹配到 → 读 ARCHITECTURE（全文）+ DEV_LOG（前 40 行 ≈ 最近 5 条）+ CURRENT（全文），输出 2-3 句话接手摘要
4. 未匹配 → 询问要开发哪个模块，或新建

**收尾阶段**（仅在用户明确发信号时触发——"做完了""收尾""下一个任务""`/openlog 收尾`"等）：
1. 在 DEV_LOG 顶部插一条新记录（时间戳 + 摘要 + 关键文件 + 关联 CURRENT 条目）
2. 更新 CURRENT（"进行中"移走 / 标记新阻塞 / 刷新时间戳）
3. **仅当**接口 / 依赖 / 关键文件变化时才改 ARCHITECTURE
4. DEV_LOG > 500 行时自动归档老条目

### 为低 token 设计

- 接手已有模块：**~700-1000 tokens**，无需 grep，无需全代码库探索
- 新建模块：**~300 tokens**
- 收尾：**~150-400 tokens**（多数情况下跳过 ARCHITECTURE 更新）

## 自定义

整个 skill 就是一个 markdown 文件 [`SKILL.md`](SKILL.md)，规则、模板、触发口令都可以按团队习惯改。

如需英文版 skill body，参考 [`SKILL.en.md`](SKILL.en.md)。把 `SKILL.md` 的正文替换为英文内容，或者直接改名使用即可。

## 许可证

[MIT](LICENSE)——随便用，署名鼓励但不强制。
