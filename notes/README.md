# Skills 学习笔记 — 索引

这个文件夹是关于 **Agent Skills** 的知识整理，方便随时查阅。内容去重整合自两个权威来源，加上针对"如何在自研智能体框架中支持 skill"的实现指南。

## 文件导航

| 文件                                                                          | 内容                                                                                                                               | 适合什么时候看                                         |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| [01-what-are-skills.md](01-what-are-skills.md)                                 | Skill 是什么、渐进式披露、什么时候该用、九大类别                                                                                   | 入门、建立心智模型                                     |
| [02-creating-skills.md](02-creating-skills.md)                                 | 如何创建/编写一个好 skill、最佳实践、常见坑、评测迭代流程                                                                          | 自己动手写 skill 时                                    |
| [03-implementing-skill-support.md](03-implementing-skill-support.md)           | **如何让你自研的智能体框架完美支持 skill**（发现、触发、加载、Hook、数据持久化）                                             | 给你自己的 agent 框架加 skill 能力时                   |
| [04-skill-locations-and-cross-agent.md](04-skill-locations-and-cross-agent.md) | **skill 存放位置**（.claude vs .agents）、Claude Code 到底读哪些目录、**软链接跨 agent 共享**、`npx skills` 与锁文件 | 排查"skill 装了却不生效"、想多 agent 共用一份 skill 时 |

## 来源

1. **Claude 官方博客** — *Lessons from building Claude Code: How we use skills*
   https://claude.com/blog/lessons-from-building-claude-code-how-we-use-skills
   （已抓取整理，见各笔记文件）
2. **本地 `skill-creator` skill** — `.agents/skills/skill-creator/`
   Anthropic 官方开源的"用来创建 skill 的 skill"，是权威的一手规范。

## 一句话总览

> Skill = 一个**文件夹**（不只是一个 markdown），里面装着指令、脚本和资源，
> 让智能体在特定任务上更准确。核心机制是**渐进式披露**：平时只加载简短的
> name+description，命中时才加载正文，需要时才读取子文件/执行脚本。
