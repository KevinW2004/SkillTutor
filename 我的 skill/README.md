# 我的 skill

## LCC Paper Writing

独立的论文写作与修稿工作流，当前版本 **v0.9.1**。以作者选定的问题、机制与证据组织全文，用直白语言讲清具体机制，并由作者决定新增概念、主张、结果解释和局限性表述。

- [技能主入口](lcc-paper-writing/SKILL.md)：通用规则和按任务读取的入口。
- [ZIP 下载](lcc-paper-writing.zip)：解压后得到完整技能目录。
- [Skill 分发包](lcc-paper-writing.skill)：与 ZIP 内容一致。

| 内置指引 | 内容 |
| --- | --- |
| [摘要与引言](lcc-paper-writing/reference/abstract-introduction.md) | 全文主线、摘要压缩、代表性结果与贡献分工 |
| [实验写作](lcc-paper-writing/reference/experiment-writing.md) | 问题、选择理由、设计能力与获准的 insights |
| [图表与图注](lcc-paper-writing/reference/figures-and-captions.md) | 选择性强调、正文与附录的信息取舍、折线视觉层级、Teaser 与图文一致 |
| [章节写作与修订](lcc-paper-writing/reference/manuscript-writing.md) | 提纲、标题层级、方法、公式、相关工作、结论与润色 |
| [LaTeX 编辑与核查](lcc-paper-writing/reference/latex-editing.md) | 顶部浮动默认偏好、聚焦编辑、保护表格与引用、源文件和版式核查 |
| [写作边界示例](lcc-paper-writing/reference/writing-examples.md) | 概念授权、机制关系与实验解释的通用例子 |

## WKW Paper Writing

独立的论文表达与修订技能，当前版本 **v0.1.0**。结合论文写作与 AI 审稿研究，让已有贡献、机制和证据容易被读懂，提供可选的读后复述检查。

- [技能主入口](wkw-paper-writing/SKILL.md)：贡献表达、证据解释与作者原意。
- [读后复述检查](wkw-paper-writing/reference/reader-check.md)：诊断读者是否准确理解论文主线。
- [研究依据](wkw-paper-writing/reference/research-notes.md)：论文出处与关键比较结果。
- [ZIP 下载](wkw-paper-writing.zip) / [Skill 分发包](wkw-paper-writing.skill)：包含完整技能目录，二者内容一致。

## 使用

将选用的 `lcc-paper-writing/` 或 `wkw-paper-writing/` 整个目录复制到目标工具的技能目录，例如项目的 `.agents/skills/` 或 `.claude/skills/`。支持技能名称调用的工具可使用 `$lcc-paper-writing` 或 `$wkw-paper-writing`。复制目录时保留 `reference/` 和 `agents/`。

两个技能均包含各自所需的写作指引与参考，可以独立使用。编译、联网查证和绘图使用目标宿主或论文项目已有的工具与模板。
