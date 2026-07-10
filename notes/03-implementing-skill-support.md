# 03 · 如何让你自研的智能体框架支持 Skill

> 目标：让你的 agent 像 Claude Code / claude.ai 一样"完美支持 skill"。
> 本文把 skill 机制拆成可实现的组件，逐个说明该怎么在框架里落地。

Skill 之所以能工作，本质上只依赖几个简单能力：**扫描目录 → 注入元数据 → 模型选择 → 按需加载正文/资源 → 能读文件、能跑脚本**。
下面按实现顺序拆解。

---

## 组件全景（一张图）

```
启动时
  ├─ 1. 发现（Discovery）        扫描约定目录，找到所有 SKILL.md
  ├─ 2. 解析（Parse）           读 YAML frontmatter：name + description（必需）
  └─ 3. 注入（Inject）          把 [name + description] 列表放进系统提示 → available_skills

运行时（每个用户请求）
  ├─ 4. 触发（Trigger）         模型看 available_skills，判断是否需要某个 skill
  ├─ 5. 加载正文（Load body）    把选中 skill 的 SKILL.md 正文读进上下文
  ├─ 6. 按需读资源             模型自己 Read references/、执行 scripts/
  ├─ 7. Hook（可选）           激活该 skill 携带的、仅本会话有效的 hook
  └─ 8. 数据/记忆（可选）       给 skill 一个稳定的数据目录做跨会话记忆
```

---

## 1. 发现（Discovery）

启动时扫描一组**约定目录**，找出每个含 `SKILL.md` 的文件夹。Claude Code 的约定：
- 项目级：`./.claude/skills/<name>/SKILL.md`
- 插件/市场安装的 skill
- 你的本仓库里已经用的是 `./.agents/skills/<name>/SKILL.md`（见 `skills-lock.json`）

**实现建议**：
- 支持多个搜索根（项目级、用户级 `~/.your-agent/skills`、插件目录），有优先级/去重规则。
- 每个 skill 是一个目录，目录名即 skill 标识（应与 frontmatter 的 `name` 一致）。

**版本锁定（可选但推荐）**：你仓库里的 `skills-lock.json` 就是范例——记录每个 skill 的来源
（如 GitHub `anthropics/skills`）、路径、内容 hash。这让 skill 可以像依赖一样被固定版本、可复现：

```json
{
  "version": 1,
  "skills": {
    "skill-creator": {
      "source": "anthropics/skills",
      "sourceType": "github",
      "skillPath": "skills/skill-creator/SKILL.md",
      "computedHash": "5ea13a6d..."
    }
  }
}
```

---

## 2. 解析（Parse）

读每个 `SKILL.md` 的 **YAML frontmatter**。最小契约：

```yaml
---
name: skill-creator          # 必需，标识符
description: ...             # 必需，触发规范（写给模型看）
compatibility: ...          # 可选，声明所需工具/依赖，很少用
---
```

- `name` + `description` 是**必需**字段。其余（compatibility 等）可选。
- frontmatter 之后是 Markdown 正文（第 5 步才加载）。
- 建议做**校验**：缺 name/description 应报错（可参考 `skill-creator/scripts/quick_validate.py`）。

---

## 3. 注入（Inject）→ 形成 `available_skills`

把所有已发现 skill 的 **`name + description`** 拼成一个列表，放进**系统提示**（或等价的常驻上下文）。
这就是模型看到的 `available_skills`。

**这是渐进式披露的第 1 层**：无论装了多少 skill，平时上下文里只有每个 skill ~100 词的元数据，
**正文和资源都不加载**。这是 skill 能规模化的根本原因。

实现要点：
- 每个条目至少含 `name` 和 `description`，可选加"如何调用"的提示。
- 保持简短。这层是"目录"，不是"内容"。

---

## 4. 触发（Trigger）— 让模型来选

模型读 `available_skills`，**自己判断**当前任务是否需要某个 skill。你要做的是给它一个"选择/加载 skill"的途径。两种主流实现：

**方式 A（推荐，Claude Code 的做法）：提供一个 `Skill` 工具**
- 暴露一个工具（如 `Skill(name, args)`），模型调用它来"启用"某个 skill。
- 工具的作用就是把该 skill 的 SKILL.md 正文读进上下文（见第 5 步）。
- 好处：显式、可观测、可加权限控制。

**方式 B：注入式**
- 在系统提示里说明"当某 skill 相关时，先 Read 它的 SKILL.md 再继续"。
- 更简单，但可控性和可观测性弱。

**触发行为的两个关键事实**（做评测和调 prompt 时必须知道）：
1. 模型**只在自己搞不定的任务**上才查 skill。"读一下这个 PDF"这种一步搞定的请求，
   即使 description 完美匹配也常常**不触发**——因为模型用基础工具直接就做了。
2. 因此 description 要写得**略"推"一点**来对抗欠触发（见 `02-creating-skills.md` 第二节）。

---

## 5. 加载正文（Load SKILL.md body）

一旦某个 skill 被选中，把它的 **SKILL.md 正文**读进上下文。**这是渐进式披露的第 2 层**。
- 正文理想 < 500 行。
- 正文里会用相对路径指向子文件（references/、scripts/、assets/），并说明何时去读。

---

## 6. 按需读取资源（第 3 层）

这一层不需要框架做特殊工作，只需要模型**具备两个基础能力**：

1. **文件读取**（Read/Glob/Grep 等）——让模型能读 `references/*.md`、`assets/` 模板等。
2. **执行代码**（Bash/Shell）——让模型能跑 `scripts/*.py`。
   **脚本可以直接执行而不必把源码载入上下文**，这是省 token 的关键。

这就是"**把整个文件系统当作上下文工程**"的含义：你只要在 SKILL.md 里告诉模型有哪些文件、
何时用，它就会在恰当时机自己去读/去跑。

> 换句话说：**如果你的 agent 已经有"读文件 + 跑命令"这两个工具，你就已经具备支持 skill 第 3 层的全部能力了。**

---

## 7. 按需 Hook（On-Demand Hooks，进阶）

Skill 可以携带 **只在它被调用时激活、且只在本会话有效** 的 hook。实现要点：
- 在 skill 目录里声明 hook（如 PreToolUse matcher）。
- 当 skill 被加载（第 5 步）时，把这些 hook 注册进本会话的 hook 系统；会话结束或 skill 停用时移除。
- 典型用途：`/careful`（拦 `rm -rf`、`DROP TABLE`、强推等破坏性命令）、`/freeze`（禁止改某目录外的文件）。

前提：你的框架需要有一个**工具调用拦截层**（PreToolUse/PostToolUse 钩子机制）。
如果暂时没有，这一步可以后置——不影响 skill 的核心功能。

---

## 8. 数据 / 跨会话记忆（进阶）

给每个 skill 一个**稳定的数据目录**，让它能跨会话记忆。
- Claude Code 用环境变量 **`${CLAUDE_PLUGIN_DATA}`** 指向该目录。
- 你可以定义自己的等价变量（如 `${YOURAGENT_SKILL_DATA}`），指向 `~/.your-agent/data/<skill-name>/`。
- Skill 在里面存 append-only 日志 / JSON / SQLite 都行。例：`standup-post` 存 `standups.log`，
  下次读自己的历史来保持一致、识别变化。

**Setup/配置**：skill 可在自己目录放 `config.json` 存用户上下文（如 Slack 频道）。
配置缺失时，让模型主动问用户；结构化多选提问用 AskUserQuestion 类工具。

---

## 9. 分发与管理（团队规模化）

两种分发方式：
1. **签入仓库** — 放 `./.claude/skills`（你这里是 `./.agents/skills`）。适合小团队、仓库少的情况。
   注意：**每个签入的 skill 都会给模型增加常驻上下文**（第 1 层元数据）。
2. **插件 + 市场（marketplace）** — 用户自行上传/安装插件。团队规模变大后更合适，
   让每个人决定装哪些 skill，还能支持 setup 流程。

Anthropic 内部的市场治理很轻：没有中央委员会。有人写了 skill 就传到 GitHub 的 sandbox 文件夹、
在 Slack 分享；某个 skill 有了热度后（作者自行决定），提 PR 把它挪进正式市场。

**组合 skill**：市场暂未内建 skill 间依赖管理，但你可以在一个 skill 里**按名字引用另一个 skill**，
模型若发现它已安装就会去调用它（例：一个"生成 CSV"的 skill 引用一个"上传文件"的 skill）。

**度量 skill**：用 **PreToolUse hook 记录 skill 使用情况**，
借此发现哪些 skill 受欢迎、哪些"该触发却没触发"（undertriggering）。

---

## 10. 最小可行实现（MVP）清单

如果你想在自研框架里最快跑通 skill，按这个顺序做：

- [ ] **1. 目录扫描** — 递归找 `<root>/*/SKILL.md`。
- [ ] **2. frontmatter 解析** — 提取 `name` + `description`（缺失报错）。
- [ ] **3. 系统提示注入** — 把 `name + description` 列表拼进 system prompt。
- [ ] **4. Skill 工具** — 提供一个工具，参数是 skill name，作用是把该 SKILL.md 正文注入上下文。
- [ ] **5. 复用已有的文件读取 + 命令执行工具** — 让模型能读 references/、跑 scripts/。

做完这 5 步，你的 agent 就已经"支持 skill"了。剩下的（hook、数据目录、市场、版本锁、评测）都是增强项，可按需叠加。

---

## 实现时的常见误区

- ❌ 把 SKILL.md 正文也一次性注入系统提示 → 破坏渐进式披露，上下文爆炸。**正文必须按需加载。**
- ❌ 用关键词硬匹配来决定触发 → 应交给模型基于 description 判断（更鲁棒、能处理同义/隐含表达）。
- ❌ description 写成给人看的摘要 → 它是**触发规范**，要含触发场景/关键词、还要略"推"。
- ❌ 用简单单步请求测触发 → 这类请求模型自己就做了，测不出触发质量。测试用例要够复杂。
- ❌ 把脚本源码读进上下文再执行 → 直接执行即可，省 token。
