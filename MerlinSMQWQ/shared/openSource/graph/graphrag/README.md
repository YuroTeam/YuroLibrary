# Microsoft GraphRAG · 架构 & 核心机制文档

> 对 [`microsoft/graphrag`](https://github.com/microsoft/graphrag) **v3.1.0** 的源码级架构分析。
> 分析路径：`/Users/momo/Projects/openSource/graphrag` · 所有 `file:line` 引用相对 `packages/graphrag/graphrag/`。

GraphRAG 用 LLM 把非结构化文本**离线**炼成"实体图 → 社区树 → 层层递归的自然语言摘要"三级结构，从而让在线查询既能做普通 RAG 的局部检索，又能做普通 RAG 做不到的**全局综合**与**多跳推理**。

## 🖼️ 只想看图？从这两个入口进

- **[总览大图 `svg/00-overview-map.svg`](./svg/00-overview-map.svg)** —— 一屏看全：索引建库 → 制品层 → 四种检索。
- **[图集画廊 `gallery.html`](./gallery.html)** —— 所有 10 张结构图的缩略图墙，点开看大图，**不用读任何文档**。用浏览器打开即可。

> 每张图都用同一个示例（Alice / Acme Corp / Berlin）串起来，且能独立看懂。下面的文字文档是可选的配文。

---

## 🔰 初学者：`svg/` 目录该按什么顺序看

`svg/` 目录里有 10 张图。**文件名的数字不是阅读顺序**——`05`~`09` 是按"图属于骨架 / 核心"归类的编号，不是算法真实的执行顺序。按下表顺序看，才是跟着 GraphRAG"文本进、答案出"的真实处理链路走一遍（也是 [`gallery.html`](./gallery.html) 图集画廊的分组顺序）：

| 顺序 | 文件 | 讲什么 |
|---|---|---|
| 1 | [`00-overview-map.svg`](./svg/00-overview-map.svg) | **先看这张**：一屏看全链路——索引建库 → 制品层 → 四种检索 |
| 2 | [`08-graph-extraction-gleaning.svg`](./svg/08-graph-extraction-gleaning.svg) | 索引①：文本 → LLM 抽取实体/关系 → gleaning 补抽 → 跨块合并 → 描述滚动摘要 |
| 3 | [`09-hierarchical-leiden.svg`](./svg/09-hierarchical-leiden.svg) | 索引②：关系图 → 层次化 Leiden 聚类 → 递归细分成多层社区树 |
| 4 | [`05-community-report-substitution.svg`](./svg/05-community-report-substitution.svg) ★ | 索引③（核心创新）：自底向上递归生成社区报告，超预算时用子报告替换原始数据 |
| 5 | [`06-global-mapreduce.svg`](./svg/06-global-mapreduce.svg) | 查询①：Global Search —— 对社区报告做 Map-Reduce |
| 6 | [`07-drift-reasoning-tree.svg`](./svg/07-drift-reasoning-tree.svg) | 查询②：DRIFT 检索 —— 全局定调 + 图结构化的多跳推理树 |
| 7 | [`01-overview.svg`](./svg/01-overview.svg) | 回头看工程骨架：总体分层架构（配置聚合根 / CLI-API / 可插拔后端） |
| 8 | [`02-indexing-pipeline.svg`](./svg/02-indexing-pipeline.svg) | 工程骨架：索引管道（有序 workflow 列表 + 共享表通信） |
| 9 | [`03-query-methods.svg`](./svg/03-query-methods.svg) | 工程骨架：四种检索策略（Local / Global / DRIFT / Basic）一览 |
| 10 | [`04-llm-middleware.svg`](./svg/04-llm-middleware.svg) | 工程骨架：LLM 中间件管道（cache / retry / 限流 / 日志层层包裹） |

**为什么这么排：**
- **1 → 6**：跟着"一段文本怎么变成一次有据问答"的真实链路走——先总览，再索引三步（抽取→聚类→报告），再查询两法（Global 的 Map-Reduce、DRIFT 的推理树）。
- **7 → 10**：链路看懂之后，再补"系统是怎么搭起来的"工程细节。这四张是骨架/参考图，彼此没有先后依赖，可以乱序看，甚至先跳过，需要时再回来查。

**只想看图、不想读文字？** 直接浏览器打开 [`gallery.html`](./gallery.html)——缩略图墙已按上表顺序分组排好，点开看大图即可，图上自带文字说明，不用另外读 `.md`。

**想连文字文档一起深入？** 见下面「📚 阅读路径」，每张图都配了对应的 `.md` 文章，可以顺着图找到对应源码位置。

---

本套文档分两层：**工程骨架**（architecture）讲"系统怎么搭"，**算法核心**（core-deep-dive）逐环节讲"GraphRAG 凭什么与众不同"。

---

## 📚 阅读路径

### 1. 先看总体架构

**[`architecture.md`](./architecture.md)** — 工程骨架全景

> 配置聚合根 · Monorepo 包布局 · 数据模型 · 索引管道 · 查询子系统 · Factory + 可插拔后端 · LLM 中间件 · CLI/API · 关键设计模式

适合：想快速建立"这个项目由哪些部分组成、怎么协作"的整体认知。

### 2. 再钻算法核心

**[`core-deep-dive.md`](./core-deep-dive.md)** — 核心机制深入 · 总览

真正让 GraphRAG 区别于普通向量 RAG 的**算法链**，拆成五篇逐环节深挖（每篇自含配图 + `file:line` + 关键默认值 + 反直觉点）：

| # | 篇目 | 一句话 | 配图 |
|---|---|---|---|
| ① | [LLM 图抽取](./core-01-graph-extraction.md) | 文本 → 实体+关系 → 合并 → 描述摘要（含 gleaning 补抽） | [`08`](./svg/08-graph-extraction-gleaning.svg) |
| ② | [层次化 Leiden 聚类](./core-02-hierarchical-leiden.md) | 关系图 → 多层社区树（大社区递归细分） | [`09`](./svg/09-hierarchical-leiden.svg) |
| ③ | [社区报告 ★核心创新](./core-03-community-reports.md) | 自底向上递归摘要，超预算时用子报告替换原始数据 | [`05`](./svg/05-community-report-substitution.svg) |
| ④ | [Global / Local 检索](./core-04-query-global-local.md) | 社区报告 Map-Reduce（全局）+ 混合上下文（局部） | [`06`](./svg/06-global-mapreduce.svg) |
| ⑤ | [DRIFT 检索](./core-05-drift-search.md) | 全局定调 + 图结构化的迭代局部下钻 | [`07`](./svg/07-drift-reasoning-tree.svg) |

> ★ **③ 社区报告的自底向上递归摘要**是整套系统的"皇冠明珠"——它让 GraphRAG 能回答"这个语料的主要主题是什么"这类普通向量 RAG 从原理上无法回答的全局性问题。

---

## 🗺️ 一图看懂算法链

```
① LLM 图抽取         文本 →（实体 + 关系 + 声明）→ 合并去重 → 描述摘要
       ↓
② 层次化 Leiden 聚类  关系图 → 多层社区树（level 0 粗 … level N 细）
       ↓
③ 社区报告（★核心创新）  自底向上递归摘要：叶子→父，超预算时用子报告替换原始数据
       ↓
─────────── 以上是 Indexing，产出 parquet + 向量库 ───────────
       ↓
④ Global / Local 检索  对社区报告 Map-Reduce（全局）；对实体做混合上下文（局部）
⑤ DRIFT 检索          全局定调 + 图结构化的迭代局部下钻
```

---

## 🖼️ 架构图索引（`svg/`）

工程骨架图（配 `architecture.md`）：

- [`01-overview.svg`](./svg/01-overview.svg) — 总体分层架构
- [`02-indexing-pipeline.svg`](./svg/02-indexing-pipeline.svg) — 索引管道
- [`03-query-methods.svg`](./svg/03-query-methods.svg) — 四种检索策略
- [`04-llm-middleware.svg`](./svg/04-llm-middleware.svg) — LLM 中间件管道

算法核心图（配 `core-*` 五篇）：

- [`05-community-report-substitution.svg`](./svg/05-community-report-substitution.svg) — ③ 社区报告自底向上递归摘要 + 子报告替换 ★
- [`06-global-mapreduce.svg`](./svg/06-global-mapreduce.svg) — ④ Global Search 的 Map-Reduce
- [`07-drift-reasoning-tree.svg`](./svg/07-drift-reasoning-tree.svg) — ⑤ DRIFT 推理树
- [`08-graph-extraction-gleaning.svg`](./svg/08-graph-extraction-gleaning.svg) — ① 图抽取 / gleaning / 合并 / 描述摘要
- [`09-hierarchical-leiden.svg`](./svg/09-hierarchical-leiden.svg) — ② 层次化 Leiden 递归细分

---

## 🧭 文件地图

```
graphrag/
├── README.md                       ← 本文件（总索引）
├── architecture.md                 工程骨架全景
├── core-deep-dive.md               算法核心 · 总览 + 心智模型 + 全链速查表
├── core-01-graph-extraction.md     ① LLM 图抽取
├── core-02-hierarchical-leiden.md  ② 层次化 Leiden 聚类
├── core-03-community-reports.md    ③ 社区报告 ★核心创新
├── core-04-query-global-local.md   ④ Global / Local 检索
├── core-05-drift-search.md         ⑤ DRIFT 检索
└── svg/                            01–04 骨架图 · 05–09 核心机制图
```

导航关系：`architecture.md ⇄ core-deep-dive.md`（总览）⇄ `core-01…05`（五篇相邻互链，各自可回总览）。

---

*基于 GraphRAG 3.1.0 源码实读。工程骨架与算法核心互补，`file:line` 引用相对 `packages/graphrag/graphrag/`。*
