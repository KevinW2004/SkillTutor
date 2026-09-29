# 我的 skill

## LCC Paper Writing

独立的论文写作与修稿工作流，当前版本 **v0.6.0**。以作者选定的问题、机制与证据组织全文，用直白语言讲清具体机制，并由作者决定新增概念、主张、结果解释和局限性表述。

- [技能主入口](lcc-paper-writing/SKILL.md)：通用规则和按任务读取的入口。
- [分发包](lcc-paper-writing.skill)：包含整个技能目录的 ZIP 包。

| 内置指引 | 内容 |
| --- | --- |
| [摘要与引言](lcc-paper-writing/reference/abstract-introduction.md) | 全文主线、引言四个问题与摘要压缩 |
| [实验写作](lcc-paper-writing/reference/experiment-writing.md) | 问题、选择理由、设计能力与获准的 insights |
| [图表与图注](lcc-paper-writing/reference/figures-and-captions.md) | Teaser、方法图、选图型、面板、图注与数值格式 |
| [章节写作与修订](lcc-paper-writing/reference/manuscript-writing.md) | 新论文规划、方法、公式、相关工作、结论、润色与翻译 |
| [LaTeX 编辑与核查](lcc-paper-writing/reference/latex-editing.md) | 聚焦编辑、保护表格与引用、验证源文件和渲染结果 |
| [写作边界示例](lcc-paper-writing/reference/writing-examples.md) | 概念授权、机制关系与实验解释的通用例子 |

## 使用

将 `lcc-paper-writing/` 整个目录复制到目标工具的技能目录，例如项目的 `.agents/skills/` 或 `.claude/skills/`。支持技能名称调用的工具可使用 `$lcc-paper-writing`。复制目录时保留 `reference/` 和 `agents/`。

核心论文写作与修稿指引均已包含在本技能中，无需安装其他论文写作技能。编译、联网查证和绘图使用目标宿主或论文项目已有的工具与模板。
