---
tags:
  - 缰绳工程
  - 编程原语
  - 上下文窗口
  - 反馈循环
  - ai编程框架
  - 上下文-工程
  - 人工智能/智能体
  - 工程效率
  - 开发工具/自动化
  - 智能体-架构
---

\#知识星球

# 所有 AI 编程工具，底层都是这 5 种积木

------------------------------------------------------------------------

## 第一层：底层原理（一切的起点）

在深入任何框架之前，必须先理解一个基础事实：

**大语言模型本身是无状态的函数：输入文本 → 输出文本**

所有"AI 编程工具"、"Agent 框架"、"Skills"、"Harness"------本质上都在回答同一个问题：

> 在 LLM 推理的那一刻，上下文窗口里应该有什么文本？

这个问题有三个维度：

| 维度     | 决策内容                                           |
|----------|----------------------------------------------------|
| **内容** | 放什么指令、知识、代码、历史？                     |
| **时机** | 什么时候放进去，什么时候移除？                     |
| **方式** | 用确定性代码强制执行，还是用概率性 Markdown 指令？ |

所有后面要讨论的框架都是在这三个维度上做不同的权衡。

------------------------------------------------------------------------

## 第二层：五种底层原语（所有框架的共同语言）

不管框架叫什么名字、有多少命令、多少 Skills，拆开后底层只有 5 种原语。

### ① AGENTS.md / CLAUDE.md --- 宪法层

- **本质**：项目根目录的 Markdown 文件
- **机制**：启动时自动注入到系统提示词
- **特点**：始终占用上下文，Agent 每次推理都能看到
- **最佳实践**：保持在 60 行以内，放"无论做什么任务都必须遵守的最小规则集"
- **反模式**：塞进所有规范和指南，变成"千页手册"

### ② Skills --- 渐进式披露层

- **本质**：`skills/X/SKILL.md` 文件
- **机制**：Agent 根据任务上下文判断后按需加载
- **特点**：不用时不占上下文空间
- **关键字段**：每个 SKILL.md 开头有 `description`，Agent 用它来决定何时激活
- **代表用法**：Superpowers 的 TDD、debugging、code-review 等

### ③ Commands --- 触发式提示词模板

- **本质**：`.claude/commands/X.md` 文件
- **机制**：用户输入 `/command-name` 时，对应 Markdown 被注入
- **特点**：预写好的提示词模板，省去重复打字
- **代表用法**：GSD 的 `/gsd:plan-phase`、OpenSpec 的 `/opsx:propose`

### ④ Rules --- 始终执行的约束层

- **本质**：`.claude/rules/X.md` 文件
- **机制**：和 AGENTS.md 一样始终注入，但按文件分类
- **特点**：便于按语言、按主题管理（security.md、typescript.md）
- **代表用法**：ECC 的 34+ 条 rules

### ⑤ Hooks --- 确定性代码层

- **本质**：`.js` / `.sh` 脚本
- **机制**：在生命周期事件（session-start、session-end、pre-tool-use、post-tool-use 等）触发
- **特点**：**唯一不是 Markdown 的原语**------是真正的代码
- **为什么重要**：前 4 种原语都是"请求 Agent 做某事"（概率性），Hooks 是"强制 Agent 做某事"（确定性）
- **代表用法**：ECC 的 session 记忆持久化、pre-commit 类型检查

> **核心洞察**：所有框架的差异，本质上都是这 5 种原语的不同排列组合和不同使用比例。

------------------------------------------------------------------------

## 第三层：统一理论（Harness Engineering）

2026 年 2 月由 OpenAI 提出、Mitchell Hashimoto 普及的新学科。它是**描述前两层的统一理论**。

### 核心定义

> **Harness Engineering** 是设计环境、约束和反馈循环以使 AI 编程 Agent 可靠工作的新工程学科。模型是马，harness 是缰绳。

### 关键公式

    Coding Agent = AI 模型 + Harness

同一个模型配不同的 harness，表现天差地别。LangChain 的 coding agent 不换模型只换 harness，在 Terminal Bench 2.0 上从排名 30 外跳到 Top 5。

### 三大支柱（OpenAI 框架）

1.  **Context Engineering（上下文工程）**

- 把所有关键知识推入仓库，使代码库成为唯一事实来源
- Agent 看不到的东西等于不存在

2.  **Architectural Constraints（架构约束）**

- 用自定义 linter、CI、pre-commit hooks 强制执行架构边界
- 把"黄金原则"编码到仓库中，由确定性工具执行

3.  **Feedback Loops（反馈循环）**

- Agent 失败 → 识别缺失的能力 → 改进 harness → 让 Agent 再也不犯同样的错
- Agent 自审查、交叉审查、垃圾回收 Agent、可观测性追踪

### 五条实践原则

| 原则                 | 核心思想                                     |
|----------------------|----------------------------------------------|
| 先可检测，再给自主权 | 能验证输出后才能给 Agent 更多自主权          |
| 每个错误都是改进信号 | Mitchell Hashimoto 的核心洞察                |
| 给地图，不给千页手册 | 渐进式披露 \> 一次性灌满                     |
| 构建可拆卸的 harness | 模型进步后能移除过时的"聪明"逻辑             |
| 复合效应是超能力     | 今天的 harness 改进适用于未来每次 Agent 运行 |

> **记忆锚点**：前面的五种原语是"原子"，Harness Engineering 是"物理定律"。懂了物理定律，才知道怎么用原子搭分子。

------------------------------------------------------------------------

## 第四层：具体框架（五种原语的不同配方）

2026 年主流的 AI 编程框架都是五种原语的组合。按照它们**解决的核心问题**分为四个维度。

### 4.1 维度对比表

| 框架 | 核心定位 | 主要原语 | 解决的核心问题 |
|----|----|----|----|
| **OpenSpec** | 规格管理系统 | Commands + AGENTS.md | 规格散落在聊天记录中 |
| **Superpowers** | 执行纪律框架 | Skills + Commands | Agent 跳过测试和代码审查 |
| **GSD** | 端到端编排 | Commands + Agents + Hooks | 上下文腐烂 + 生命周期管理 |
| **ECC** | 基础设施工具箱 | 全部 5 种原语 | Agent 行为不一致 + 平台碎片化 |

### 4.2 详细对比

#### OpenSpec（27k stars）

    技术构成：3 个 Commands + 1 个 AGENTS.md
    真相来源：openspec/specs/ 目录下的 Markdown 文件
    核心流程：propose → apply → archive
    独占优势：持久化的规格库、Brownfield 优先

#### Superpowers（104k stars）

    技术构成：14 个 Skills + 3 个 Commands
    真相来源：docs/plans/ + 运行时的 Skills 激活
    核心流程：brainstorm → write-plan → subagent TDD → review
    独占优势：强制 TDD 纪律、两阶段代码审查

#### GSD（39.8 k stars）

    技术构成：40+ Commands + 6 Agents + Hooks
    真相来源：.planning/ 目录下的 PROJECT.md、STATE.md、PLAN.md
    核心流程：new-project → discuss → plan → execute → verify → ship
    独占优势：波次并行执行、里程碑管理、模型成本配置

#### ECC（114 k+ stars）

    技术构成：38 Agents + 156 Skills + 72 Commands + 8 Hooks + 34 Rules
    真相来源：分散在 rules/ + skills/ + commands/ 中，按语言分组
    核心流程：无强制流程，提供工具箱让用户组合
    独占优势：确定性 hooks、持续学习（Instincts）、安全扫描、12 种语言支持

### 4.3 本质差异的一句话总结

- **OpenSpec** 回答「系统是什么样」的问题 → 知识管理
- **Superpowers** 回答「怎么做得对」的问题 → 工程纪律
- **GSD** 回答「按什么流程做」的问题 → 流程编排
- **ECC** 回答「Agent 带什么装备」的问题 → 基础设施

> **关键认知**：ECC 和前三者不在同一层级。ECC 是基础设施，其他三个是运行在基础设施上的方法论。它们可以叠加使用。

------------------------------------------------------------------------

## 第五层：如何组合（决策脉络）

### 5.1 选型决策树

    你是什么场景？
    │
    ├── 个人开发者 / 独立项目
    │ └── GSD + ECC
    │ 理由：GSD 提供端到端流程，ECC 补齐基础设施和持续学习
    │
    ├── 多人团队 / 长周期企业项目
    │ └── OpenSpec + Superpowers + ECC
    │ 理由：OpenSpec 管长期知识，SP 管代码质量，ECC 管底层约束
    │
    ├── 只想用一个
    │ └── ECC
    │ 理由：156 Skills 覆盖面最广，Hooks 确定性最强
    │
    └── 追求最简单
    └── 一个精简的 AGENTS.md + pre-commit hooks
    理由：Harness Engineering 的第一原则——从简单开始

### 5.2 组合策略：三层责任分离

    ┌─────────────────────────────────────────────────┐
    │ 方法论层（Method Layer） │
    │ OpenSpec + Superpowers + GSD │
    │ 职责：定义工作流、执行纪律、规格管理 │
    └─────────────────────────────────────────────────┘
    ↓ 运行在
    ┌─────────────────────────────────────────────────┐
    │ 基础设施层（Infrastructure Layer） │
    │ ECC（Rules + Hooks + 跨平台适配 + 安全扫描） │
    │ 职责：确定性约束、会话持久化、跨平台一致性 │
    └─────────────────────────────────────────────────┘
    ↓ 基于
    ┌─────────────────────────────────────────────────┐
    │ 原语层（Primitive Layer） │
    │ AGENTS.md + Skills + Commands + Rules + Hooks │
    │ 职责：Claude Code 等运行时提供的底层机制 │
    └─────────────────────────────────────────────────┘
    ↓ 由...驱动
    ┌─────────────────────────────────────────────────┐
    │ 模型层（Model Layer） │
    │ Claude Opus / Sonnet / GPT-5 / Qwen / ... │
    │ 职责：推理、生成、工具调用 │
    └─────────────────────────────────────────────────┘

### 5.3 重叠消解规则

当组合使用多个框架时，必须明确"每层只有一个真相来源"：

| 职责     | 胜出者            | 理由                        |
|----------|-------------------|-----------------------------|
| 规格库   | OpenSpec          | 唯一有持久化 spec 管理的    |
| 需求讨论 | GSD discuss-phase | 支持 assumptions 模式       |
| 技术设计 | GSD plan-phase    | 自带研究和验证 Agent        |
| 执行编排 | GSD execute-phase | 波次并行                    |
| TDD 纪律 | Superpowers       | 强制 RED/GREEN/REFACTOR     |
| 代码审查 | Superpowers       | GSD 原生缺失                |
| 调试     | Superpowers       | systematic-debugging        |
| 验证/UAT | GSD verify-work   | 有人工验收流程              |
| 交付归档 | GSD + OpenSpec    | GSD ship + OpenSpec archive |
| 底层约束 | ECC               | Rules + Hooks               |
| 安全扫描 | ECC               | AgentShield 102 条规则      |
| 会话记忆 | ECC               | session-start/end hooks     |

------------------------------------------------------------------------

## 第六层：实战脉络（从理论到落地）

### 6.1 Harness 构建的四个阶段

    Day 1（1 小时投入）：基础层
    ├── 创建 AGENTS.md（< 60 行）
    ├── 配置 pre-commit hooks
    └── 建立 progress.md 进度文件
    ↓
    Week 1（半天投入）：约束层
    ├── 自定义架构 linter
    ├── 黄金原则文件
    └── 功能列表 JSON
    ↓
    Week 2-4（持续迭代）：执行层
    ├── 子 Agent 上下文防火墙
    ├── 推理三明治模式
    ├── 强制验证循环
    └── 循环检测（防 doom loop）
    ↓
    Month 2+（长期投入）：自愈层
    ├── 垃圾回收 Agent
    ├── 失败模式数据库
    └── Harness 版本化管理

### 6.2 关键反模式清单

避免这六个坑就能绕过大部分弯路：

- ❌ **千页手册综合症**：把所有规则塞进 AGENTS.md
- ❌ **工具囤积症**：连接 15 个 MCP 服务器、50 个工具
- ❌ **自动生成 AGENTS.md**：用 AI 扫描代码生成，ETH Zurich 研究证明会降低性能
- ❌ **验证缺失**：Agent 写完代码就标记完成，不做端到端验证
- ❌ **One-shotting 复杂任务**：期望一个会话搞定所有需求
- ❌ **过度工程化 Harness**：花几个月构建复杂框架，模型更新后过时

### 6.3 效果度量

- **首次通过率**：Agent 完成任务后无需人工修改的比例
- **上下文利用率**：主编排 Agent 平均上下文使用率（目标 \< 40%）
- **每功能 token 消耗**：持续优化
- **人工干预频率**：每 10 会话的介入次数
- **失败模式重现率**：已记录的失败再次出现的频率

------------------------------------------------------------------------

## 第七层：思考锚点（记忆体系）

如果只记住三件事，记这三个：

### 锚点 1：所有框架都是 5 种原语的不同配方

AGENTS.md + Skills + Commands + Rules + Hooks

这是"元素周期表"。理解了就不会被花哨的命名迷惑。

### 锚点 2：概率性 vs 确定性的边界

- 前 4 种原语是**概率性的**（Markdown 指令，Agent 大概率遵守）
- Hooks 是**确定性的**（代码执行，100% 执行）
- 重要约束放 Hooks，不要只靠 Markdown 提示词

### 锚点 3：Harness 是无限演化的"免疫系统"

Agent 失败 → 分析缺什么 → 更新 harness → 永远不再犯

这是 Mitchell Hashimoto 的核心洞察。Harness 不是一次性构建的，是随着项目积累的。

------------------------------------------------------------------------

## 附录：核心参考资料

### 原始理论文献

| 来源 | 贡献 | 链接 |
|----|----|----|
| OpenAI | 提出 Harness Engineering 三大支柱 | https://openai.com/index/harness-engineering/ |
| Anthropic | 长时间运行 Agent 的 harness 设计 | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents |
| Mitchell Hashimoto | "每个错误都是改进信号" | https://mitchellh.com/writing/my-ai-adoption-journey |
| Martin Fowler / Birgitta Böckeler | Harness 作为新服务模板 | https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html |
| HumanLayer | 子 Agent 作为上下文防火墙 | https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents |

### 开源框架仓库

| 框架 | 仓库 | Stars |
|----|----|----|
| OpenSpec | https://github.com/Fission-AI/OpenSpec | 27 k+ |
| Superpowers | https://github.com/obra/superpowers | 104 k+ |
| GSD | https://github.com/gsd-build/get-shit-done | 39.8 k+ |
| Everything Claude Code | https://github.com/affaan-m/everything-claude-code | 114 k+ |
| unified-workflow (概念验证) | https://github.com/mattjaikaran/unified-workflow | 早期 |

### 社区实战经验

- Rick Hightower 的框架对比系列（GSD + Superpowers 实战组合）
- OpenSpec Issue \#780 （将 OpenSpec 打包为 Superpowers skill pack 的讨论）
- Phil Schmid: Agent Harness 2026 的展望

**最后的话**：这个领域每周都有新东西。记住原理（五种原语 + Harness 三支柱），忘掉具体框架的命名------它们明年可能就被新的框架取代。原理是复利资产，工具是折旧品。
