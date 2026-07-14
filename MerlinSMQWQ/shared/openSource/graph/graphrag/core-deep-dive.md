# GraphRAG 核心机制深入 · 总览

> 配套文档：[`architecture.md`](./architecture.md)（总体架构）
> 版本：`graphrag` 3.1.0 · 分析路径：`/Users/momo/Projects/openSource/graphrag`
> 所有 `file:line` 引用均相对 `packages/graphrag/graphrag/`。

本系列把 GraphRAG 的核心**算法链**拆成五篇深入讲解，每篇一个环节、各自配图与关键默认值：

| # | 篇目 | 一句话 | 配图 |
|---|---|---|---|
| ① | [LLM 图抽取](./core-01-graph-extraction.md) | 文本 → 实体+关系 → 合并 → 描述摘要 | `svg/08-graph-extraction-gleaning.svg` |
| ② | [层次化 Leiden 聚类](./core-02-hierarchical-leiden.md) | 关系图 → 多层社区树 | `svg/09-hierarchical-leiden.svg` |
| ③ | [社区报告 ★核心创新](./core-03-community-reports.md) | 自底向上递归摘要，超预算时用子报告替换原始数据 | `svg/05-community-report-substitution.svg` |
| ④ | [Global / Local 检索](./core-04-query-global-local.md) | 社区报告 Map-Reduce（全局）+ 混合上下文（局部） | `svg/06-global-mapreduce.svg` |
| ⑤ | [DRIFT 检索](./core-05-drift-search.md) | 全局定调 + 图结构化的迭代局部下钻 | `svg/07-drift-reasoning-tree.svg` |

---

## 为什么挑这五块作深入

`architecture.md` 已经把**工程骨架**（配置聚合根、Factory + 可插拔后端、Storage/Table 抽象、中间件管道、CLI/API）讲清楚了——那些是"任何一个成熟数据管道都会有"的部分，换个项目也大同小异。

真正让 GraphRAG **区别于普通向量 RAG**、也是它论文核心贡献的，是下面这条**算法链**。普通 RAG 是"切块 → 嵌入 → 相似度检索 → 塞进 prompt"；GraphRAG 在中间插入了一整套"**用 LLM 把文本炼成知识图谱，再把图谱层层压缩成可被检索的自然语言摘要**"的机制：

```
① LLM 图抽取         文本 →（实体 + 关系 + 声明）→ 合并去重 → 描述摘要
       ↓
② 层次化 Leiden 聚类  关系图 → 多层社区树（level 0 粗 … level N 细）
       ↓
③ 社区报告（★核心创新）  自底向上递归摘要：叶子社区→父社区，超预算时用子报告替换原始数据
       ↓
─────────── 以上是 Indexing，产出 parquet + 向量库 ───────────
       ↓
④ Global / Local 检索  对社区报告做 Map-Reduce（全局）；对实体做混合上下文（局部）
⑤ DRIFT 检索          全局定调 + 图结构化的迭代局部下钻
```

其中 **③ 社区报告的自底向上递归摘要**是整个系统的"皇冠明珠"——它让 GraphRAG 能回答"这个语料的主要主题是什么"这类**普通向量 RAG 从原理上无法回答**的全局性问题。

---

## 一条端到端的心智模型

把五块串起来看一次数据的完整旅程：

```
原始文档
  │  分块（text_units，1200 token/块）
  ▼
① 每块 → LLM 抽实体+关系（+gleaning 补抽）→ 跨块合并（权重累加）→ 描述滚动摘要
  │        产出：entities / relationships（每个都有唯一规范描述 + 向量）
  ▼
② 关系图 → 层次化 Leiden → 多层社区树（叶子 ≤10 实体，确定性可复现）
  │        产出：communities（parent/children/entity_ids/...）
  ▼
③ 自底向上：叶子社区从原始实体摘要 → 父社区超预算时用子报告替换原始数据
  │        产出：community_reports（title/summary/findings/rating + 向量）★
  ▼
④/⑤ 查询：
     Basic  → 纯向量 RAG，忽略图（基线）
     Local  → query→top-k 实体→混合上下文（报告+实体+关系+原文），单次 LLM
     Global → 社区报告 Map-Reduce（分批打分→排序截断→合成），全局主题
     DRIFT  → primer 定调 + 迭代 LocalSearch 下钻，多跳推理树
```

**一句话抓住本质**：GraphRAG 的全部价值，在于用 LLM 把非结构化文本**离线**炼成"实体图 → 社区树 → 层层递归的自然语言摘要"这三级结构，从而让在线查询既能做普通 RAG 的局部检索（Local/Basic），又能做普通 RAG 做不到的**全局综合**（Global）和**多跳推理**（DRIFT）。而③的"超预算时以子报告替换原始数据"的递归压缩，是让"全局综合"在**任意规模语料**上都可行的关键工程。

---

## 关键默认值速查（全链）

| 环节 | 参数 | 默认 | 出处 |
|---|---|---|---|
| ① 图抽取 | `max_gleanings` | 1 | `defaults.py:150` |
| ① 图抽取 | 定界符 tuple/record/complete | `<\|>` / `##` / `<\|COMPLETE\|>` | `graph_extractor.py:31-33` |
| ① 图抽取 | 默认实体类型 | organization/person/geo/event | `defaults.py:147-149` |
| ① 描述摘要 | `max_length` / `max_input_tokens` | 500 词 / 4000 token | `defaults.py:319-320` |
| ② 聚类 | `max_cluster_size` / `use_lcc` / `seed` | 10 / True / `0xDEADBEEF` | `defaults.py:71-73` |
| ② 聚类 | Leiden resolution / randomness / use_modularity | 1.0 / 0.001 / True | `hierarchical_leiden.py:22-24` |
| ③ 社区报告 | `max_length` / `max_input_length` | 2000 词 / 8000 token | `defaults.py:82-83` |
| ④ Global | `max_context_tokens`（批分）/ `data_max_tokens`（reduce） | 12000 / 12000 | `defaults.py:200-201` |
| ④ Global | `map_max_length` / `reduce_max_length` | 1000 / 2000 词 | `defaults.py:202-203` |
| ④ Global | map score 范围 | 0–100 整数 | map 提示词 |
| ④ 动态选择 | threshold / keep_parent / max_level | 1 / False / 2 | `dynamic_community_selection.py` |
| ④ Local | `text_unit_prop` / `community_prop`（local=剩余） | 0.5 / 0.15（→0.35） | `defaults.py:264-265` |
| ④ Local | `max_context_tokens` / `top_k_entities` / `top_k_relationships` | 12000 / 10 / 10 | `defaults.py:267-269` |
| ⑤ DRIFT | `n_depth` / `drift_k_followups` / `primer_folds` | 3 / 20 / 5 | `defaults.py:99-102` |
| ⑤ DRIFT | 内部 local `text_unit_prop` / `community_prop` | 0.9 / 0.1 | `defaults.py:103-104` |

---

*本系列聚焦 GraphRAG 3.1.0 的算法核心，与 `architecture.md`（工程骨架）互补。所有机制均基于源码实读，`file:line` 引用相对 `packages/graphrag/graphrag/`。*
