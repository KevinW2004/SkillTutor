# 04 · Skill 存放位置 & 跨 Agent 共享（.claude vs .agents vs 软链接）

> 回答："`npx skills add` 为什么装到 `.agents`？各家 agent 路径不一样怎么办？
> 能不能用软链接让多个 agent 共用一份 skill？"

## TL;DR

- **不同产品用不同的 skill 目录**，这是当前的真实状况（碎片化痛点确实存在）。
- **Claude Code 只读 `.claude/skills/`（及全局/插件/企业），不读 `.agents/skills/`。**
- `.agents/skills/` 是**别的 agent**（Codex、Cursor 等）共享的社区约定目录。
- **软链接是官方支持的正解** —— Claude Code 会跟随符号链接。可以让一份 skill 被多个 agent 共用。
- 你用的 `npx skills`（vercel-labs/skills）**本身就能自动做这件事**（symlink 到一份 canonical 副本）。

---

## 一、各家 agent 的 skill 目录（为什么会有 `.agents`）

`npx skills` = **vercel-labs/skills**，一个"跨 agent 的 skill 包管理器"（号称 npm for agent skills），
支持 Claude Code / Codex / Cursor / OpenCode 等 70+ agent。它按**每个 agent 各自的约定**把 skill 放到对应目录：

| Agent | 项目级 skill 目录 |
|-------|-------------------|
| **Claude Code** | `.claude/skills/<name>/SKILL.md` |
| **Codex / Cursor（及其他多家）** | `.agents/skills/<name>/SKILL.md` ← 共享约定 |
| OpenCode 等 | 各自约定 |

所以你观察到的现象成立：`npx skills add` 把 skill 放进了 `.agents/skills/`，那是 **Codex/Cursor 那一派共享的目录**，
而 **Claude Code 不认这个目录**——这就是为什么你把 `skill-creator` 挪到 `.claude/skills/` 后 Claude Code 才读得到。

> 换产品是否就要换路径？**是的**，目前每家路径不同。这正是 `npx skills` 和"软链接方案"要解决的碎片化问题。

---

## 二、Claude Code 原生到底读哪些位置（权威）

来源：官方文档 code.claude.com/docs/en/skills。Claude Code 扫描：

| 作用域 | 路径 | 生效范围 |
|--------|------|----------|
| 企业（托管） | 由 managed settings 下发 | 组织所有人 |
| **个人/全局** | `~/.claude/skills/<name>/SKILL.md` | 你的所有项目 |
| **项目** | `.claude/skills/<name>/SKILL.md` | 当前项目 |
| 插件 | `<plugin>/skills/<name>/SKILL.md` | 启用该插件处 |

补充规则：
- **向上查找**：项目 skill 会从**当前目录到仓库根**的每一级 `.claude/skills/` 加载（子目录启动也能读到根目录的 skill）。
- **按需嵌套**：处理子目录文件时，会按需发现更深层的 `.claude/skills/`（如 `packages/frontend/.claude/skills/`）。
- **`--add-dir` / `/add-dir`**：会额外加载被加目录里的 `.claude/skills/`。（注意：`settings.json` 里的 `permissions.additionalDirectories` 只给文件访问权，**不会加载 skill**。）
- **优先级**（同名时）：企业 > 个人 > 项目 > 内置(bundled)。插件 skill 用 `plugin-name:skill-name` 命名空间，不冲突。
- **没有** settings.json 配置项可以添加任意 skill 搜索目录（除了 `--add-dir`、软链接、插件、托管这几种途径）。
- ❌ **Claude Code 不扫描 `.agents/` 或 `.agents/skills/`。**

结构要求：每个 skill 必须是 `<skills-dir>/<skill-name>/SKILL.md`；**目录名**决定 `/` 后面输入的命令名。

---

## 三、软链接：官方支持，跨 agent 共享的正解 ⭐

官方文档原文（"Where skills live"）明确说明：

> 企业/个人/项目位置下的一个 `<skill-name>` 条目**可以是指向磁盘别处目录的软链接**。
> Claude Code 会**跟随软链接**读取目标目录里的 `SKILL.md`；如果同一目标能从多个位置到达，**只加载一次**（自动去重）。

含义：
- ✅ **单个 skill 软链接**：`~/.claude/skills/foo -> /canonical/foo` —— 支持。
- ✅ **整个 skills 目录软链接**：`.claude/skills -> ../.agents/skills` —— 文档没明说，但按发现逻辑极可能可行。
  不过**更推荐按 skill 逐个软链**（更细粒度、更安全，避免把不兼容的 skill 全量共享）。

### 方案 A（推荐）：让 `npx skills` 自动做
`npx skills` 安装时支持两种策略：**从各 agent 软链到一份 canonical 副本**（单一真源、更新方便），
或各 agent 独立拷贝（不支持软链时）。直接一条命令装到多家：

```bash
npx skills add anthropics/skills -a claude-code -a codex -a cursor
```

它会把 canonical 副本放好，并在各 agent 目录建软链/拷贝。你不用手搓软链。

### 方案 B（手动 DIY）：canonical 目录 + 逐个软链
选一个**单一真源**目录（比如就用 `.agents/skills/`，它是多 agent 共享约定），然后为 Claude Code 建软链：

```bash
# 在项目根 /home/wukaiwen/Program/SkillTutor 下
mkdir -p .claude/skills
ln -s ../../.agents/skills/skill-creator .claude/skills/skill-creator
#      ^ 相对路径：从 .claude/skills/ 往上两级到项目根，再进 .agents/skills/
```

验证：`ls -l .claude/skills/` 应看到 `skill-creator -> ../../.agents/skills/skill-creator`。
之后 Claude Code、Codex、Cursor 都指向同一份，改一处三家同步。

> 相对软链（`../../...`）比绝对软链更好——项目整体移动/被别人 clone 时不会失效。

---

## 四、`skills-lock.json`：skill 的"锁文件"

你项目里的 `skills-lock.json` 是 `npx skills` 的**锁文件**（类比 `package-lock.json`）：
记录每个 skill 的来源(`source`)、类型(`sourceType`)、内容 hash，**应提交进 git**。作用：
- 团队/CI/外包拿到**完全一致**的 skill 版本（`npx skills install` ≈ `npm ci`，从锁文件还原）。
- 供应链审查：能像 diff `package.json` 一样在 PR 里 diff skill。

（还有个全局锁 `~/.agents/.skill-lock.json` 管全局安装。工具仍处实验阶段，flag 名变动较快，写 CI 前查最新 README。）

⚠️ 注意：你现在 `skills-lock.json` 记录的 `skillPath` 仍是 `skills/skill-creator/SKILL.md`（源仓库内路径），
而你已把本地文件挪到 `.claude/skills/`。若之后跑 `npx skills install/update`，可能与你手动挪动的布局不一致——
建议**要么全交给 `npx skills` 管，要么手动管软链后清楚锁文件的预期**，别两套混用。

---

## 五、对"自研 agent 框架"的启示

你在给自己的 agent 加 skill 支持时（见 `03-implementing-skill-support.md`），关于**目录约定**可以这样设计：

1. **想开箱即用地与生态互通**：直接扫描 **`.agents/skills/`** 这个跨 agent 共享约定目录，
   这样你的 agent 天然能读 Codex/Cursor 已装的 skill。
2. **同时兼容 Claude 生态**：也扫描 `.claude/skills/`（项目）和 `~/.claude/skills/`（全局）。
3. **务必跟随软链接**（follow symlinks）并**对同一真实路径去重**——这是让"一份 skill 多处复用"成立的关键。
4. 支持多个搜索根 + 明确优先级（全局 vs 项目 vs 内置）。

> 一句话：**别再发明第 N 个私有 skill 目录**。读 `.agents/skills/` + `.claude/skills/` + 跟随软链，你的框架就能无缝复用整个生态的 skill。

---

## 参考来源

- Claude Code 官方文档（Skills）：https://code.claude.com/docs/en/skills.md
- vercel-labs/skills（`npx skills`）：https://github.com/vercel-labs/skills
- npx skills 实践指南：https://dev.to/toyama0919/managing-ai-agent-skills-with-npx-skills-a-practical-guide-2an8
- skills-lock.json 说明：https://explainx.ai/blog/skills-lock-json-reproducible-agent-skills-2026
- 官方 skill 目录：https://skills.sh
