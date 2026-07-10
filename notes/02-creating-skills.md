# 02 · 如何创建 / 编写一个好的 Skill

整合自 Claude 官方博客 + 本地 `skill-creator` skill（`.agents/skills/skill-creator/SKILL.md`）。

---

## 一、Skill 的结构（Anatomy）

```
skill-name/
├── SKILL.md (必需)
│   ├── YAML frontmatter (name, description 必需)
│   └── Markdown 指令
└── 捆绑资源 (可选)
    ├── scripts/    - 确定性/重复任务的可执行代码
    ├── references/ - 按需载入上下文的文档
    └── assets/     - 输出中用到的文件（模板、图标、字体）
```

**多领域组织**：当一个 skill 支持多个框架/领域时，按变体拆分参考文件，模型只读相关的那个：

```
cloud-deploy/
├── SKILL.md          # 工作流 + 如何选择
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

**渐进式披露的实操规则**：
- SKILL.md 保持在 **500 行以内**；接近上限就再加一层目录层级，并清楚指明"接下来该去读哪个文件"。
- 从 SKILL.md 里**明确引用**子文件，并说明**何时**该读它。
- 大的参考文件（>300 行）加个目录（table of contents）。

---

## 二、写 `description`：这是最重要的字段

`description` 是**决定 skill 是否被调用的首要机制**。它是写给**模型**看的触发规范，不是给人看的摘要。

要点：
- **同时写清"做什么" + "什么时候用"**。所有"何时使用"的信息都放这里，不要放正文。
- **包含触发关键词**（比如 skill 名里的动词 `babysit`）。
- **要"推"一点**：Claude 目前倾向于**欠触发**（该用的时候不用）。为对抗这点，description 要写得略微强势。

对比示例：

```
❌ 太温吞：
"How to build a simple fast dashboard to display internal Anthropic data."

✅ 够"推"：
"How to build a simple fast dashboard to display internal Anthropic data.
Make sure to use this skill whenever the user mentions dashboards, data
visualization, internal metrics, or wants to display any kind of company
data, even if they don't explicitly ask for a 'dashboard.'"
```

> `skill-creator` 自己的 description 就是范例（注意它罗列了大量触发场景）：
> `"Create new skills, modify and improve existing skills... Use when users want to
> create a skill from scratch, edit, or optimize an existing skill, run evals..."`

---

## 三、编写最佳实践（来自实战）

### 1. 不要说废话（Don't State the Obvious）
Claude 本来就会写代码、会读代码库。**复述默认行为只会增加上下文却不增加价值。**
知识型 skill 要聚焦于"能把模型推离其默认思维模式"的信息。

### 2. 建一个 Gotchas（坑）小节 ⭐
> **"任何 skill 里信噪比最高的内容，就是 Gotchas 小节。"**
这些坑应该从"Claude 常犯的失败点"中不断积累，随时间往里加。具体例子：
- `subscriptions` 表是 append-only 的——用 version 最高的那行，别用 `created_at` 最新的。
- API 网关里的 `@request_id` 和计费服务里的 `trace_id` 是同一个值。
- Staging 环境即使 Stripe webhook 没处理成功也会返回 200；真实状态要查 `payment_events`。

### 3. 别把 Claude 关进轨道（Avoid Railroading）
Claude 通常会照着指令走。因为 skill 高度可复用，**写得太具体反而会坑自己**。
> "给 Claude 它需要的信息，但也给它随机应变的灵活度。"

### 4. 解释"为什么"，少用生硬的 MUST/ALWAYS/NEVER ⭐
> 如果你发现自己在写全大写的 ALWAYS 或 NEVER、或用极其死板的结构，那是**黄灯信号**。
今天的 LLM 很聪明，有良好的 theory of mind。**尽量解释每件事背后的原因**，
让模型理解"为什么这么做重要"——这比机械指令更人性化、更强大、更有效。

### 5. 把脚本给它，让它组合而非重造轮子 ⭐
> "你能给 Claude 的最强工具之一，就是代码。"
提供脚本和库，让模型把回合花在**组合**上而非重写样板。
**信号**：如果读测试运行的 transcript 发现三个测试用例都各自写了一份类似的 `create_docx.py`，
那就是"该把这个脚本捆进 skill 的 `scripts/`"的强烈信号——写一次，以后每次调用都省了。

### 6. 用文件系统做渐进式披露
最简单的形式：在 SKILL.md 里指向别的 markdown（比如详细函数签名放 `references/api.md`）；
输出模板放 `assets/`；参考、脚本、示例分门别类放子目录。

### 7. 想清楚"配置/初始化"（Setup）
有些 skill 需要用户上下文（比如站会汇报该发到哪个 Slack 频道）。
- 把配置存进 skill 目录里的 `config.json`。
- 配置缺失时，智能体可以主动问用户。
- 需要结构化的多选提问时，指示 Claude 用 **AskUserQuestion** 工具。

### 8. 帮 Claude 记住（跨会话记忆）
Skill 可以存数据来实现跨会话记忆——简单如 append-only 文本日志 / JSON 文件，复杂到 SQLite 数据库都行。
例：`standup-post` 维护一个 `standups.log`，下次 Claude 读自己的历史、识别出有什么变化。
用 **`${CLAUDE_PLUGIN_DATA}`** 环境变量拿到一个稳定的数据目录。

### 9. 用按需 Hook（On-Demand Hooks）
Skill 可以带**只在该 skill 被调用时才激活、且只在本次会话有效**的 hook。
适合那些"你不想一直开着、但在特定场景极其有用"的强意见 hook：
- `/careful` — 通过 Bash 的 PreToolUse matcher 拦截破坏性命令（`rm -rf`、`DROP TABLE`、强推、`kubectl delete`）。只在动生产时才需要。
- `/freeze` — 拦截某目录之外的一切 Edit/Write。调试时防止"顺手修好"了不相关的代码。

### 10. 安全原则（Principle of Lack of Surprise）
Skill 不得含恶意软件/漏洞利用代码。其内容不应与其描述的意图相悖。
不要制作误导性的、或旨在协助未授权访问/数据外泄的 skill。（"扮演某某角色"这类是 OK 的。）

---

## 四、写作风格与格式模式

- 指令用**祈使句**（imperative）。
- **定义输出格式**：
  ```markdown
  ## Report structure
  ALWAYS use this exact template:
  # [Title]
  ## Executive summary
  ## Key findings
  ## Recommendations
  ```
- **示例模式**：
  ```markdown
  ## Commit message format
  **Example 1:**
  Input: Added user authentication with JWT tokens
  Output: feat(auth): implement JWT-based authentication
  ```
- 先写草稿，再用**新鲜的眼光**重看一遍并改进。用 theory of mind，让 skill 通用而不是绑死在具体例子上。

---

## 五、创建流程（skill-creator 的方法论）

高层流程（顺序灵活，看用户处在哪一步就从哪切入）：

1. **捕捉意图（Capture Intent）** — 想清楚 4 个问题：
   1) 这个 skill 让 Claude 能做什么？
   2) 什么时候该触发？（哪些用户措辞/场景）
   3) 期望输出格式是什么？
   4) 要不要建测试用例？（输出可客观验证的——文件转换、数据抽取、代码生成、固定步骤——适合；主观输出——写作风格、艺术——通常不需要。）
2. **访谈 & 调研** — 主动问边角情况、输入输出格式、示例文件、成功标准、依赖。
3. **写 SKILL.md** — 填 `name` / `description` / (可选 `compatibility`) / 正文。
4. **写 2-3 个真实测试用例** — 像真实用户会说的话，存到 `evals/evals.json`（先只写 prompt，assertion 后面补）。
5. **跑测试并评测** — 对每个用例同时起两个子智能体：一个带 skill、一个不带（baseline）。
6. **看结果 + 迭代** — 定性（人看输出）+ 定量（assertion 通过率、耗时、token）。据反馈改 skill，重复。
7. **优化 description** — 用 `scripts/run_loop.py` 自动优化触发准确率（见下）。
8. **打包** — `python -m scripts.package_skill <folder>` 产出 `.skill` 文件。

### 评测的关键纪律（如果你要做严肃评测）
- **同一回合**里把 with-skill 和 baseline 全部起出来，别先跑一批再补另一批。
- 好的 assertion 是**可客观验证**且**名字有描述性**的。主观 skill（写作、设计）靠人评，别硬套 assertion。
- 子任务完成通知里带 `total_tokens` 和 `duration_ms`——**只有这一次机会拿到**，立刻存进 `timing.json`。
- 用 `eval-viewer/generate_review.py` 生成审阅界面给人看，**先让人看到样例再自己改**。

### 迭代时如何思考改进（skill-creator 的精华）
1. **从反馈中泛化**——你们只在几个例子上反复迭代是为了快，但如果 skill 只对这几个例子有效就没用了。
   遇到顽固问题，别塞过拟合的补丁或压迫性的 MUST，试试换个比喻/换种工作模式。
2. **保持精简**——去掉不出力的内容。读 transcript（不只看最终输出），如果 skill 让模型浪费时间做无用功，就删掉那部分再看效果。
3. **解释 why**——见上面第 4 条。
4. **找跨用例的重复劳动**——见上面第 5 条（重复写的脚本→捆进 skill）。

---

## 六、`skill-creator` 里有什么现成工具

本地 `.agents/skills/skill-creator/` 是一手规范，包含可直接用的脚本：

| 路径 | 作用 |
|------|------|
| `scripts/package_skill.py` | 把 skill 文件夹打包成 `.skill` 文件 |
| `scripts/quick_validate.py` | 快速校验 skill 结构 |
| `scripts/run_loop.py` | description 优化循环（自动迭代提升触发准确率） |
| `scripts/run_eval.py` | 运行触发评测 |
| `scripts/aggregate_benchmark.py` | 聚合评测结果成 benchmark.json/.md |
| `scripts/improve_description.py` | 优化 description |
| `eval-viewer/generate_review.py` | 生成人工审阅的 HTML 界面 |
| `references/schemas.md` | evals.json / grading.json / benchmark.json 等的完整 JSON schema |
| `agents/grader.md` | 如何用 assertion 给输出打分 |
| `agents/comparator.md` | 如何做盲评 A/B 对比 |
| `agents/analyzer.md` | 如何分析"为什么一个版本赢了" |

> 想动手时，最快的路径就是让我（或你自己）**调用这个 skill-creator**，它会带着你走完整个创建→评测→迭代→打包流程。
