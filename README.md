# /ai-startup — AI 原生创业全流程 Claude Code Skill

一个陪伴创始人走完 **创意 → MVP → 发布 → 规模化** 四个阶段的 Claude Code skill。

每个阶段的关键文档自动持久化到本地 `startup/` 文件夹，随时可中断、续接。

---

## 安装

```bash
mkdir -p ~/.claude/skills/ai-startup
curl -o ~/.claude/skills/ai-startup/SKILL.md \
  https://raw.githubusercontent.com/giraffe-tree/ai-startup/main/SKILL.md
```

或者克隆后软链接：

```bash
git clone https://github.com/giraffe-tree/ai-startup.git
mkdir -p ~/.claude/skills
ln -s $(pwd)/ai-startup ~/.claude/skills/ai-startup
```

## 使用

在任意项目目录下，对 Claude Code 说：

- "开始创业"
- "我有个创业想法"
- "继续创业"
- "帮我验证想法"

或者直接 `/ai-startup`。

---

## 四个阶段

| 阶段 | 目标 | 关键产出 |
|------|------|---------|
| **idea** | 验证真实问题存在，再写第一行代码 | `problem.md` / `market.md` / `interviews.md` / `solution.md` |
| **mvp** | 构建真实用户会使用的可运行产品 | `CLAUDE.md` / `scope.md` / `metrics.md` |
| **launch** | 将早期牵引力转化为可重复增长 | `tech-debt.md` / `operations.md` / `compliance.md` |
| **scale** | 构建可持续扩张的组织与增长系统 | `channels.md` / `scale-plan.md` |

---

## 本地文档结构

```
startup/
├── state.md              ← 每次 session 必读/必写
├── idea/
│   ├── problem.md
│   ├── market.md
│   ├── interviews.md
│   └── solution.md
├── mvp/
│   ├── CLAUDE.md
│   ├── scope.md
│   ├── metrics.md
│   └── session-log.md
├── launch/
│   ├── tech-debt.md
│   ├── operations.md
│   └── compliance.md
└── scale/
    ├── channels.md
    └── scale-plan.md
```

`startup/` 是你的项目数据，建议加入 `.gitignore` 或单独管理。

---

## 核心设计原则

1. **先验证再构建** — 原型是对话道具，不是验证证据
2. **文档驱动续接** — 每次 session 结束前更新 `state.md`
3. **对抗确认偏误** — 始终让 Claude 找反驳证据
4. **退出标准驱动** — 每个阶段有明确退出条件，不靠感觉判断
5. **CLAUDE.md 是命脉** — 架构上下文是 AI 协作的持久记忆

---

## 示例

见 [`examples/`](./examples/) 目录，包含各阶段文档的填写示例。

---

## License

MIT
