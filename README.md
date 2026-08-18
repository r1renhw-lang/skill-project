# Skill Project

本项目是一个 **Agent Skills 知识库**，用于维护当前项目可用的 Codex / Cursor Skills。每个 Skill 以 `SKILL.md` 为核心，定义 AI Agent 在特定场景下的工作流、输出规范与使用约束。

## 项目结构

```text
skill-project/
├── README.md
├── readme.txt
└── skills/
    ├── project-default/                         # 项目默认工作规范
    ├── markdown-to-html/                        # Markdown 转 HTML 工具
    ├── MagAI-Agent-Requirement-Analysis-Assistant/
    ├── MagAI-Agent-Workflow-Design-Assistant/
    ├── MagAI-Daily-Tech-Tracker/
    ├── MagAI-Engineer-Resume-Interview-Assistant/
    ├── MagAI-Knowledge-Accelerator-Skill/
    ├── MagAI-Meeting-Intelligence-Assistant/
    ├── MagAI-Model-Comparison-Assistant/
    ├── MagAI-Personal-Email-Assistant/
    ├── MagAI-Personal-Meeting-Assistant/
    ├── MagAI-Personal-Todo-Assistant/
    ├── MagAI-Project-Release-Timeline/
    ├── MagAI-Prompt-Architect-Assistant/
    ├── MagAI-Requirement-Estimator/
    ├── MagAI-Skill-Architect-Skill/
    └── MagAI-Tech-Trend-Radar/
```

单个 Skill 的标准目录结构：

```text
MagAI-XXX/
├── SKILL.md       # Skill 定义（元数据 + 工作流指令）
├── assets/        # 资源文件（模板、示例等）
├── agents/        # Agent 配置
└── scripts/       # 辅助脚本
```

## Skills 总览

共 **17** 个 Skills，按场景分为五类：

| 类别 | 数量 | 说明 |
| --- | --- | --- |
| 元能力 | 2 | 项目规范与 Skill 设计 |
| 智能体建设 | 4 | 从需求分析到 Prompt / 流程 / 工时评估 |
| 技术研判与学习 | 5 | 趋势研判、日报追踪、知识速通、模型对比、Markdown 转 HTML |
| 业务辅助 | 5 | 招聘面试、会议整理、会议智能、企业微信待办、个人邮件 |
| 项目管理 | 1 | 项目发布记录与可视化时间线 |

---

## 元能力

| Skill 名称 | 目录 | 用途 |
| --- | --- | --- |
| `project-default` | `skills/project-default/` | 项目默认工作规范。在本仓库内工作时，指导 Agent 读取代码、保持改动范围、遵循已有模式、选择验证命令，并维护项目上下文。 |
| `mag-ai-skill-architect` | `skills/MagAI-Skill-Architect-Skill/` | Skill 设计助手。根据业务需求完成 Skill 需求分析、架构设计、Prompt 设计、Workflow 规划和 `SKILL.md` 生成；支持 Prompt / Knowledge / Tool / Workflow / Multi-Agent 等类型，并对已有 Skill 进行评审和优化。 |

---

## 智能体建设

| Skill 名称 | 目录 | 用途 |
| --- | --- | --- |
| `mag-ai-agent-requirement-analysis-assistant` | `skills/MagAI-Agent-Requirement-Analysis-Assistant/` | 智能体需求分析。面向 AI 智能体建设场景，将业务方原始需求结构化，识别需求目标、功能范围、AI 实现价值、技术可行性、风险及实施建议。 |
| `mag-ai-agent-workflow-design-assistant` | `skills/MagAI-Agent-Workflow-Design-Assistant/` | Agent 流程设计。根据业务需求设计 Agent 执行流程、任务拆解、Agent 角色划分、工具调用流程和异常处理机制。 |
| `mag-ai-prompt-architect-assistant` | `skills/MagAI-Prompt-Architect-Assistant/` | Prompt 架构设计。根据业务需求设计高质量、结构化、可维护的 Prompt，并输出设计思路、优化建议及适配不同模型的版本。 |
| `mag-ai-requirement-estimator` | `skills/MagAI-Requirement-Estimator/` | AI 需求评估与工时估算。基于业务需求拆解开发工作、评估开发复杂度、估算开发工时、识别技术风险，并输出可解释的工时评估结果。 |

---

## 技术研判与学习

| Skill 名称 | 目录 | 用途 |
| --- | --- | --- |
| `mag-ai-tech-trend-radar` | `skills/MagAI-Tech-Trend-Radar/` | 技术趋势研判。对目标 AI 技术进行结构化研判，通过联网搜索采集多源信号，从成熟度、商业价值、团队适配度、风险成本四个维度评估，给出技术雷达定位（Adopt / Trial / Assess / Hold）与行动建议。 |
| `mag-ai-daily-tech-tracker` | `skills/MagAI-Daily-Tech-Tracker/` | AI 技术动态追踪。汇总全球 AI 技术动态，重点关注大模型、Agent、AI 开发框架、开源项目、论文、产品更新及企业动态，并从 AI 技术经理视角分析技术价值与落地价值。 |
| `mag-ai-knowledge-accelerator` | `skills/MagAI-Knowledge-Accelerator-Skill/` | AI 知识速通。通过结构化拆解、通俗解释、技术定位、实践总结和关联学习路径，帮助快速掌握 AI 技术概念。 |
| `mag-ai-model-comparison-assistant` | `skills/MagAI-Model-Comparison-Assistant/` | AI 模型对比。帮助 AI 技术人员快速理解、分析和比较大语言模型，辅助企业模型选型。 |
| `markdown-to-html` | `skills/markdown-to-html/` | Markdown 转 HTML。将 Markdown 文本转换为带内嵌 CSS 的自包含 HTML，适用于文档、报告、邮件模板和 Newsletter。 |

---

## 业务辅助

| Skill 名称 | 目录 | 用途 |
| --- | --- | --- |
| `mag-ai-engineer-resume-interview-assistant` | `skills/MagAI-Engineer-Resume-Interview-Assistant/` | AI 工程师简历分析与面试助手。根据 AI 工程师简历，分析候选人的技术能力、项目经验、优势不足、岗位匹配度，并生成针对性的面试问题和招聘建议。 |
| `mag-ai-meeting-intelligence-assistant` | `skills/MagAI-Meeting-Intelligence-Assistant/` | 会议智能分析助手。支持会前材料分析与问题准备、会中讨论分析与决策辅助、会后概要会议总结，帮助沉淀会议认知。 |
| `mag-ai-personal-meeting-assistant` | `skills/MagAI-Personal-Meeting-Assistant/` | 个人会议助手。将会议、讨论、AI 对话和学习笔记整理为会议纪要和知识资产，提炼重点、待办事项、个人思考和知识卡片，并支持导出 RTF 文档。 |
| `mag-ai-personal-todo-assistant` | `skills/MagAI-Personal-Todo-Assistant/` | 企业微信待办管理助手。通过自然语言创建、修改、查看、删除企业微信待办，基于 `wecom-cli` 完成待办操作。 |
| `mag-ai-personal-email-assistant` | `skills/MagAI-Personal-Email-Assistant/` | 网易邮箱个人邮件管理助手。通过网易邮箱连接器实现邮件收发、搜索、汇总和管理，适用于日常个人邮件处理场景。 |

---

## 项目管理

| Skill 名称 | 目录 | 用途 |
| --- | --- | --- |
| `mag-ai-project-release-timeline` | `skills/MagAI-Project-Release-Timeline/` | 项目发布时间线。记录项目发布信息（版本号、发布内容、负责人、发布周期等），并生成可视化 HTML 时间线页面，适用于版本迭代记录、发布复盘和多项目发布追踪。 |

---

## 使用方式

在 Cursor / Codex 对话中描述相关任务，Agent 会根据 Skill 的 `name` 和 `description` 自动匹配并读取对应的 `SKILL.md`。

示例：

- “分析这段智能体需求” → `mag-ai-agent-requirement-analysis-assistant`
- “帮我设计一个 Agent 执行流程” → `mag-ai-agent-workflow-design-assistant`
- “对比 Claude 和 GPT 在这个场景下的表现” → `mag-ai-model-comparison-assistant`
- “生成今天的 AI 技术日报” → `mag-ai-daily-tech-tracker`
- “把这段 Markdown 转成 HTML” → `markdown-to-html`
- “帮我整理这场会议纪要” → `mag-ai-personal-meeting-assistant`
- “帮我记录项目发布版本” → `mag-ai-project-release-timeline`
- “帮我设计一个新的 Skill” → `mag-ai-skill-architect`

部分 Skill 配置了 `agents/openai.yaml`，可在 Codex 界面中通过 `$skill-name` 直接引用。
