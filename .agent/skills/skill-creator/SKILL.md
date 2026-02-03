---
name: skill-creator
description: 专门用于创建新的 Agent Skill。当你需要标准化地定义新能力、规范或工作流时，请使用此技能。
---

# Skill Creator (技能生成器)

此技能用于指导创建一个新的 Agent Skill。Skill 是扩展 Agent 能力的标准方式。

## 何时使用

- 当用户要求“创建一个 skill”或“教你一个新规则”时。
- 当你发现某个任务需要重复执行，且有一套固定的最佳实践时。
- 需要将某个文档（如代码规范、部署流程）固化为 Agent 可执行的指令时。

## 目录结构规范

Skill 必须遵循以下目录结构。通常存放在 `.agent/skills/` 目录下。

```text
.agent/skills/<skill-name>/
├── SKILL.md       # 核心指令文件（必须）
├── scripts/       # 辅助脚本（可选，如 python/bash 脚本）
├── examples/      # 参考示例（可选）
└── resources/     # 模板或其他静态资源（可选）
```

## 创建步骤

1.  **确定名称**：使用小写字母和连字符（kebab-case），例如 `code-review` 或 `deploy-prod`。
2.  **创建目录**：在 `.agent/skills/` 下创建对应名称的文件夹。
3.  **编写 SKILL.md**：这是核心文件，必须包含 YAML frontmatter 和 Markdown 指令正文。

## SKILL.md 模板

创建新 skill 时，请严格使用以下模板：

```markdown
---
name: <skill-name>
description: <一句话描述这个技能做什么，以及何时应该被触发>
---

# <Skill Title>

这里写关于该技能的详细说明。

## When to use this skill (何时使用)

- 场景 1...
- 场景 2...

## How to use it (如何执行)

详细的步骤指南、代码规范或思维链路。

1. 第一步...
2. 第二步...

## Best Practices (最佳实践)

- 规则 1
- 规则 2
```

## 编写建议

1.  **Description 至关重要**：Agent 依靠 frontmatter 中的 `description` 来决定是否调用此 skill。描述必须清晰、包含关键词，并使用第三人称（例如：“生成 Python 单元测试...”）。
2.  **指令清晰**：正文应包含 Agent 执行任务所需的具体规则、模板和约束。
3.  **单一指责**：不要创建一个“万能 skill”，根据功能拆分为多个专注的 skill。
