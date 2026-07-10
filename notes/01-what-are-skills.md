# 01 · Skill 是什么

## 核心定义

**Skill 是一个文件夹**，里面装着指令（instructions）、脚本（scripts）和资源（resources），
让智能体（agent）能发现并使用它们，从而更准确地完成任务。

最常见的误解是"skill 只是一个 markdown 文件"——**不是**。它是一个目录，可以包含脚本、
素材、数据等，还支持很多配置选项（比如动态注册 hook）。

一个最小 skill 的结构：

```
skill-name/
├── SKILL.md          # 必需：YAML frontmatter (name, description) + Markdown 指令
└── (可选的资源)
    ├── scripts/      # 确定性/重复性任务的可执行代码
    ├── references/   # 按需加载进上下文的文档
    └── assets/       # 输出中用到的文件（模板、图标、字体等）
```

---

## 最重要的机制：渐进式披露（Progressive Disclosure）

这是理解 skill 的关键。Skill 用**三层加载系统**，避免把所有内容一次性塞进上下文：

| 层级 | 内容 | 何时进入上下文 | 大致体量 |
|------|------|----------------|----------|
| 1. Metadata | `name` + `description` | **始终在上下文里**（所有已安装的 skill 都是） | ~100 词 |
| 2. SKILL.md 正文 | Markdown 指令主体 | **仅当该 skill 被触发时** | 理想 < 500 行 |
| 3. 捆绑资源 | references/ scripts/ assets/ | **需要时才读**（脚本甚至可以直接执行而不载入上下文） | 无限制 |

> 把**整个文件系统都当成一种上下文工程和渐进式披露的手段**。
> 你只要告诉模型 skill 里有哪些文件，它就会在合适的时机去读。

**为什么重要**：上下文是稀缺资源。这套机制让你可以捆绑海量参考资料/脚本，
却只在真正需要时才付出上下文代价。

---

## Skill 什么时候会被触发？

（对做自研框架的你尤其重要，实现细节见 `03-implementing-skill-support.md`）

- 所有 skill 会以 `name + description` 出现在模型的 **`available_skills` 列表**里。
- 模型**根据 description 判断**这个 skill 是否匹配当前请求。
- **关键**：模型只在**自己搞不定的任务**上才会去查 skill。
  像"读一下这个 PDF"这种一步就能完成的简单请求，即使 description 完全匹配也可能不触发——
  因为模型用基础工具就能直接做。
  **复杂、多步、专业化**的请求，只要 description 匹配就会可靠触发。

推论：
- `description` 是**触发规范**，不是给人看的摘要 → 写法见 `02-creating-skills.md`。
- 评测 skill 触发时，测试用例要**足够有分量**，简单单步请求是糟糕的测试用例。

---

## Skill 的九大类别

Anthropic 内部盘点自家 skill，发现它们聚成九类。**最好的 skill 都干净利落地只属于其中一类**；
想身兼多职的 skill 会让智能体犯迷糊。

1. **库 & API 参考**（Library & API Reference）
   教正确使用某个库/CLI/SDK。常附一堆参考代码片段和"坑"清单。
   例：`billing-lib`（内部计费库的边角情况）、`internal-platform-cli`（内部 CLI 全部子命令）。

2. **产品验证**（Product Verification）
   描述如何测试/验证代码是否正常，常配合 Playwright、tmux 等。
   **内部经验：这类对 Claude 输出质量的可衡量影响最大。** 建议派一名工程师花整整一周把验证 skill 打磨到极致。
   例：`signup-flow-driver`（无头浏览器跑注册→邮箱验证→引导）、`checkout-verifier`。

3. **数据获取 & 分析**（Data Fetching & Analysis）
   连接数据和监控栈：带凭证的取数库、dashboard ID、常见工作流。
   例：`grafana`（datasource UID、集群名、"问题→dashboard"查找表）、`datadog`（字段映射）。

4. **业务流程 & 团队自动化**（Business Process & Team Automation）
   把重复工作流收敛成一条命令。把历史结果存进日志文件能帮模型保持一致、复盘过往执行。
   例：`standup-post`（聚合工单+GitHub+Slack 生成站会汇报）、`weekly-recap`。

5. **代码脚手架 & 模板**（Code Scaffolding & Templates）
   为特定代码库生成框架样板。当脚手架有"代码本身表达不了的自然语言要求"时尤其有用。
   例：`new-migration`（迁移文件模板+常见陷阱）、`create-app`（预接好鉴权/日志/部署）。

6. **代码质量 & 评审**（Code Quality & Review）
   强制代码规范、辅助 code review。可含确定性脚本以求稳健；可经 hook 或 GitHub Actions 自动运行。
   例：`adversarial-review`（起一个全新子智能体来批评→改→迭代直到只剩鸡毛蒜皮）、`code-style`。

7. **CI/CD & 部署**（CI/CD & Deployment）
   帮忙取代码、推代码、部署。可引用别的 skill 来取数据。
   例：`babysit-pr`（盯 PR、重试 flaky CI、解冲突、开自动合并）、`deploy-<service>`（灰度+自动回滚）。

8. **运行手册**（Runbooks）
   拿到一个症状（Slack 线程、告警、错误签名），走一遍多工具排查，产出结构化报告。
   例：`<service>-debugging`（症状→工具→查询模式）、`log-correlator`（给个 request ID 拉全链路日志）。

9. **基础设施运维**（Infrastructure Operations）
   执行例行维护和运维流程，**包括那些需要护栏的破坏性操作**。
   例：`<resource>-orphans`（找孤儿 pod/卷→发 Slack→冷静期→用户确认→级联清理）、`cost-investigation`。

---

## 关键心态

> **"我们大多数最好的 skill，一开始都只是几行字加一个坑（gotcha），
> 然后随着 Claude 不断撞上新的边角情况，人们持续往里加东西，它才慢慢变好。"**

→ **从小处起步，边用边迭代**。别一开始就想写个大而全的 skill。
