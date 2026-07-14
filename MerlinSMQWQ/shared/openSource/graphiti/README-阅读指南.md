# Graphiti 源码学习 · 阅读指南

> 这是整个 `graphiti/` 笔记库的**总入口**。这里的文档是按「循序渐进」编排的学习路径——从零基础的图数据库入门，一路到核心算法和对外服务。每篇都配了 SVG 结构图。
>
> 不知道从哪读起？**按下面的顺序读就对了。**

---

## 🗺️ 一张图看懂全貌

```
                          ┌─────────────────────────┐
                          │   architecture.md 总览    │  ← 随时回来查的地图
                          └────────────┬─────────────┘
                                       │
        ┌──────────────┬───────────────┼───────────────┬──────────────┐
        ▼              ▼               ▼               ▼              ▼
   ① 打基础        ② 数据地基        ③ 两条主管道      ④ 核心算法      ⑤ 对外封装
   Neo4j 入门      双时态模型        写入 / 检索       去重/社区/       Provider/
   Quickstart                       (+批量)          重排/提示词      服务/批量
```

Graphiti 一句话：**面向 AI Agent 的双时态知识图谱框架**——把一段段文本增量地织成一张「带时间」的实体关系网，供检索。

---

## 📚 推荐阅读顺序

难度：🟢 入门 ｜ 🟡 进阶 ｜ 🔴 深入

### 第 ① 梯队 · 打基础（不懂图数据库先看这里）

| 顺序 | 文档 | 难度 | 讲什么 |
|:----:|------|:----:|--------|
| 1 | [01-Neo4jDriver简介.md](01-Neo4jDriver简介.md) | 🟢 | 图数据库 Neo4j 是什么、graphiti 怎么连它。**零基础从这开始** |
| 2 | [02-quickstart-walkthrough.md](02-quickstart-walkthrough.md) | 🟢 | 官方示例逐行拆解，一条主线跑通「配置→写入→检索」，建立整体手感 |

> 📌 已经懂图数据库？可跳过第①梯队，但强烈建议看 quickstart 建立全局印象。

### 第 ② 梯队 · 数据地基（先搞懂"图里存什么"）

| 顺序 | 文档 | 难度 | 讲什么 |
|:----:|------|:----:|--------|
| 3 | [03-双时态数据模型-nodes与edges.md](03-双时态数据模型-nodes与edges.md) | 🟡 | 3 类节点 + 3 类边的所有字段；**双时态**（事件时间轴 vs 系统时间轴）——这是 Graphiti 的灵魂 |

> ⭐ 这篇是**理解后面一切的前提**。两条时间轴的概念务必吃透，后面写入的"时序失效"、检索的"时间点回溯"都建立在它上面。

### 第 ③ 梯队 · 两条主管道（框架的主干）

| 顺序 | 文档 | 难度 | 讲什么 |
|:----:|------|:----:|--------|
| 4 | [04-数据写入管道-add_episode.md](04-数据写入管道-add_episode.md) | 🟡 | **写**：一段文本 → 抽实体 → 去重 → 抽关系 → 时序失效 → 落库（7 阶段） |
| 5 | [05-检索管道-search.md](05-检索管道-search.md) | 🟡 | **读**：四路 scope 并行 → 三路召回（向量+BM25+图遍历）→ 重排融合 |

> 这两篇是 Graphiti 的「主动脉」，一写一读。读完就掌握了框架 80% 的运作方式。

### 第 ④ 梯队 · 核心算法（挑感兴趣的深挖）

| 顺序 | 文档 | 难度 | 讲什么 | 前置 |
|:----:|------|:----:|--------|------|
| 6 | [06-去重算法-dedup.md](06-去重算法-dedup.md) | 🔴 | 写入里最有技术含量的一步：MinHash + LSH + LLM 三级漏斗 | 文档 4 |
| 7 | [07-重排算法-RRF与MMR.md](07-重排算法-RRF与MMR.md) | 🔴 | 检索重排的公式与逐行实现，带数字算例 | 文档 5 |
| 8 | [08-社区构建-communities.md](08-社区构建-communities.md) | 🟡 | 标签传播聚类 + 二叉树式层次化摘要 | 文档 3 |
| 9 | [09-LLM提示词设计-prompts.md](09-LLM提示词设计-prompts.md) | 🟡 | 提示词架构 + 8 大提示词工程手法 + 结构化输出 | 文档 4 |

> 这一梯队各篇**相对独立**，可按兴趣挑读。想懂"张三不会重复入图"看 6；想懂"检索怎么排序"看 7；想懂"高层视图"看 8；想懂"怎么让 LLM 听话"看 9。

### 第 ⑤ 梯队 · 对外封装与工程（生产落地）

| 顺序 | 文档 | 难度 | 讲什么 | 前置 |
|:----:|------|:----:|--------|------|
| 10 | [10-Provider抽象层-llm与embedder.md](10-Provider抽象层-llm与embedder.md) | 🟡 | 怎么统一 OpenAI/Anthropic/Gemini… 多家厂商 | 文档 9 |
| 11 | [11-批量导入-add_episode_bulk.md](11-批量导入-add_episode_bulk.md) | 🔴 | 批量导入为何更快、又牺牲了什么 | 文档 4、6 |
| 12 | [12-服务封装-server与mcp.md](12-服务封装-server与mcp.md) | 🟡 | REST 服务 + MCP 服务怎么把核心库包成对外接口 | 文档 4、5 |

### 第 ⑥ 梯队 · 工程细节与可观测（进一步深挖）

| 顺序 | 文档 | 难度 | 讲什么 | 前置 |
|:----:|------|:----:|--------|------|
| 13 | [13-检索过滤器-search_filters.md](13-检索过滤器-search_filters.md) | 🟡 | 日期/类型过滤如何下推数据库、双时态时间旅行查询 | 文档 3、5 |
| 14 | [14-分布式追踪-opentelemetry.md](14-分布式追踪-opentelemetry.md) | 🟡 | Tracer 抽象「不配置零开销、配置了全可观测」 | 文档 4、5 |
| 15 | [15-评估体系-evals.md](15-评估体系-evals.md) | 🟡 | LLM-as-judge 成对比较量化抽取质量 | 文档 4 |

---

## 🎯 按目标选读（不想全看的话）

| 我想…… | 读这些 |
|--------|--------|
| 快速跑起来体验 | 1 → 2 |
| 理解框架整体设计 | 2 → 3 → 4 → 5（+ architecture.md） |
| 深挖某个算法 | 6（去重）/ 7（重排）/ 8（社区）任选 |
| 学 LLM 应用工程 | 9（提示词）→ 10（多厂商抽象） |
| 生产部署 / 集成 | 11（批量）→ 12（服务） |
| 精细检索 / 时间旅行查询 | 13（过滤器） |
| 可观测 / 质量保障 | 14（追踪）→ 15（评估） |
| 当参考手册随时查 | [architecture.md](architecture.md)（总览，每节都链到深入版） |

---

## 📁 文档清单（全 15 篇 + 18 张图）

**总览**
- [architecture.md](architecture.md) — 核心库分层架构总览，每节 📖 链到对应深入文档

**深入专题**（每篇都在 `svg/` 下有配套结构图）

| 文档 | 配套 SVG |
|------|---------|
| Neo4jDriver简介 | [svg/01-neo4jdriver-architecture.svg](svg/01-neo4jdriver-architecture.svg) |
| quickstart-walkthrough | [svg/02-quickstart-flow.svg](svg/02-quickstart-flow.svg) |
| 双时态数据模型 | [svg/03-bitemporal-data-model.svg](svg/03-bitemporal-data-model.svg) |
| 数据写入管道 | [svg/04-add-episode-pipeline.svg](svg/04-add-episode-pipeline.svg) |
| 检索管道 | [svg/05-search-pipeline.svg](svg/05-search-pipeline.svg) |
| 去重算法 | [svg/06-dedup-funnel.svg](svg/06-dedup-funnel.svg) |
| 重排算法 RRF/MMR | [svg/07-rerank-rrf-mmr.svg](svg/07-rerank-rrf-mmr.svg) |
| 社区构建 | [svg/08-community-building.svg](svg/08-community-building.svg) |
| LLM 提示词设计 | [svg/09-prompts-architecture.svg](svg/09-prompts-architecture.svg) |
| Provider 抽象层 | [svg/10-provider-abstraction.svg](svg/10-provider-abstraction.svg) |
| 批量导入 | [svg/11-bulk-import.svg](svg/11-bulk-import.svg) |
| 服务封装 | [svg/12-service-layers.svg](svg/12-service-layers.svg) |
| 检索过滤器 | [svg/13-search-filters.svg](svg/13-search-filters.svg) |
| 分布式追踪 | [svg/14-tracing-spans.svg](svg/14-tracing-spans.svg) |
| 评估体系 | [svg/15-eval-system.svg](svg/15-eval-system.svg) |

---

## 💡 阅读小贴士

- **SVG 看图**：用 Typora / VS Code 预览 / Obsidian 等本地阅读器能直接在文中看到结构图。（GitHub 网页出于安全不渲染 `![]()` 引的 SVG，会显示成链接。）
- **交叉链接**：每篇文档开头都有「📖 关联阅读」，末尾有「一句话总结」+ 配套图路径。文档之间用 `[[]]` 概念互相打通，跟着链接跳即可。
- **代码对照**：每篇都标了对应的源码文件路径（`graphiti_core/...`），想对着源码看随时能定位。
- **不用背**：先建立"地图感"（读完 ②③ 梯队），细节需要时再回来查。architecture.md 就是那张随时可查的地图。

---

## 🧭 一句话学习路线

> **入门**（1→2 跑通）→ **地基**（3 吃透双时态）→ **主干**（4→5 掌握读写）→ **算法**（6/7/8/9 挑兴趣）→ **工程**（10/11/12 落地）→ **细节**（13/14/15 过滤/追踪/评估）。
>
> 卡住了就回 [architecture.md](architecture.md) 看全局，它是这一切的地图。
