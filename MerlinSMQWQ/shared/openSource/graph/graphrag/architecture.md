# Microsoft GraphRAG — 架构分析

> 版本：`graphrag` 3.1.0 · Python 3.11–3.13 · 仓库：<https://github.com/microsoft/graphrag>
> 分析路径：`/Users/momo/Projects/openSource/graphrag`
> 📖 进阶：算法核心逐环节深挖见 [`core-deep-dive.md`](./core-deep-dive.md)（图抽取 / 层次化 Leiden / 社区报告 / Global·Local / DRIFT 五篇）。

GraphRAG 是一套**数据管道 + 转换套件**，利用 LLM 从非结构化文本中抽取出结构化的知识图谱（实体、关系、社区、声明），并借助该图谱增强 LLM 的问答/推理能力。核心思路：先"索引"（indexing）把文档转成知识图谱制品（parquet 表 + 向量库），再"查询"（query）用四种检索策略对图谱提问。

配套架构图（本目录 `svg/`）：

- `01-overview.svg` — 总体分层架构
- `02-indexing-pipeline.svg` — 索引管道
- `03-query-methods.svg` — 四种检索策略
- `04-llm-middleware.svg` — LLM 中间件管道

---

## 目录

1. [总体架构](#1-总体架构)
2. [Monorepo 包布局](#2-monorepo-包布局)
3. [配置系统](#3-配置系统)
4. [数据模型（Domain Model）](#4-数据模型domain-model)
5. [索引子系统（Indexing）](#5-索引子系统indexing)
6. [查询子系统（Query）](#6-查询子系统query)
7. [基础设施包（可插拔后端）](#7-基础设施包可插拔后端)
8. [CLI 与 API 表面](#8-cli-与-api-表面)
9. [关键设计模式总结](#9-关键设计模式总结)

---

## 1. 总体架构

```
                       ┌─────────────────────────────────────────────┐
   settings.yaml ─────▶│              GraphRagConfig                  │
   .env / env vars     │       (pydantic, 单一配置聚合根)              │
                       └───────────────────┬─────────────────────────┘
                                           │
        ┌──────────────────────────────────┼──────────────────────────────────┐
        ▼                                   ▼                                   ▼
 ┌─────────────┐                    ┌───────────────┐                   ┌──────────────┐
 │  CLI (Typer)│───────────────────▶│  graphrag.api  │◀──────────────────│  程序化调用   │
 │  graphrag   │  init/index/       │ (公共入口层)    │                   │  (unstable)  │
 │             │  update/query/tune │                │                   └──────────────┘
 └─────────────┘                    └───┬───────┬────┘
                                        │       │
                          build_index() │       │ *_search()
                                        ▼       ▼
        ┌───────────────────────────────────┐  ┌───────────────────────────────────┐
        │        INDEXING 子系统             │  │          QUERY 子系统              │
        │  Pipeline(有序 workflow 列表)      │  │  local / global / drift / basic    │
        │  workflow → operations → graphs    │  │  ContextBuilder + Search 引擎      │
        └──────────────┬────────────────────┘  └──────────────┬────────────────────┘
                       │  读/写                                 │  读
                       ▼                                        ▼
        ┌───────────────────────────────────────────────────────────────────────┐
        │        制品层：parquet 表 + 向量库（entities / relationships /          │
        │        communities / community_reports / text_units / covariates ...）  │
        └───────────────────────────────────────────────────────────────────────┘
                       ▲
                       │  统一抽象 + Factory 选择后端
        ┌──────────────┴────────────────────────────────────────────────────────┐
        │  基础设施包：graphrag-storage / -vectors / -cache / -llm / -chunking /   │
        │              -input / -common                                          │
        └───────────────────────────────────────────────────────────────────────┘
```

**两条主流程**：

- **Indexing**：`文档 → 分块(text_units) → LLM 抽取实体/关系/声明 → Leiden 社区聚类 → LLM 生成社区报告 → 向量嵌入 → parquet 制品`
- **Query**：`问题 → ContextBuilder 从制品组装 token 预算内的上下文 → Search 引擎调用 LLM → 有据可依的答案`

技术栈：NetworkX + graspologic（层次化 Leiden 聚类）、spaCy/NLTK/TextBlob（NLP 抽取）、pandas/pyarrow（parquet）、pydantic v2（配置/模型）、litellm（LLM 抽象）、Typer（CLI）、LanceDB（默认向量库）。

---

## 2. Monorepo 包布局

采用 **uv workspace**（`[tool.uv.workspace] members = ["packages/*"]`），核心包 `graphrag` 依赖 7 个独立可安装的基础设施包（均锁定同版本 `3.1.0`）。

| 包 | import 名 | 职责 |
|---|---|---|
| **graphrag** | `graphrag` | 核心编排：索引管道、查询引擎、配置、数据模型、CLI、API |
| graphrag-common | `graphrag_common` | 共享原语：泛型 `Factory`、`hash_data`、配置加载器 |
| graphrag-llm | `graphrag_llm` | LLM 抽象（基于 litellm）：chat/embedding、中间件管道、metrics、retry、限流、tokenizer |
| graphrag-cache | `graphrag_cache` | K/V 缓存抽象：`JsonCache`/`MemoryCache`/`NoopCache` |
| graphrag-storage | `graphrag_storage` | 字节/字符串 blob 存储 + 表格存储：`File`/`Memory`/`AzureBlob`/`AzureCosmos` |
| graphrag-vectors | `graphrag_vectors` | 向量库抽象：`LanceDB`/`AzureAISearch`/`CosmosDB` |
| graphrag-chunking | `graphrag_chunking` | 文本分块：`TokenChunker`/`SentenceChunker` |
| graphrag-input | `graphrag_input` | 输入读取：`csv`/`text`/`json`/`jsonl`/`markitdown`/`parquet` |

> 另有 `unified-search-app/`（Streamlit 演示，非官方支持）、`docs/`（mkdocs 文档站）、`tests/`、`scripts/`。

**核心包 `graphrag` 内部模块**：

```
graphrag/
├── api/          # 公共入口：index / query / prompt_tune
├── cli/          # Typer 命令行（main.py 为 entrypoint）
├── config/       # GraphRagConfig + models/ 子配置 + defaults + enums
├── data_model/   # 领域实体（Entity/Relationship/Community/...）
├── index/        # 索引子系统：workflows / operations / run / update
├── query/        # 查询子系统：structured_search / context_builder / ...
├── graphs/       # 图算法：hierarchical_leiden / connected_components / modularity ...
├── prompts/      # 默认 index/query 提示词
├── prompt_tune/  # 自动提示词调优（domain/persona/entity_types 生成器）
├── callbacks/    # workflow / query 回调（进度、LLM token 流）
├── cache/  logger/  tokenizer/  utils/
```

---

## 3. 配置系统

### 加载流程（`config/load_config.py` → `graphrag_common.config.load_config`）

1. **文件发现**：在 `root_dir` 找 `settings.yaml` / `.yml` / `.json`（按序）。
2. **加载 .env**：用 `python-dotenv` 把 `.env` 注入 `os.environ`。
3. **环境变量替换**：原始文本经 `string.Template(text).substitute(os.environ)`，展开 `${GRAPHRAG_API_KEY}` 之类占位符（缺失 → `ConfigParsingError`）。
4. **解析**：YAML/JSON。
5. **CLI 覆盖**：`cli_overrides` 嵌套字典经 `_recursive_merge_dicts` 深合并。
6. **实例化**：chdir 到配置文件目录后构造 `GraphRagConfig(**data)`。

### `GraphRagConfig`（`config/models/graph_rag_config.py`）

pydantic 聚合根，**每个字段都有来自 `graphrag_config_defaults` 的默认值**，因此最小 `settings.yaml` 即可运行。主要子配置：

- **语言模型**：`completion_models: dict[str, ModelConfig]`、`embedding_models: dict[str, ModelConfig]`（按 ID 键控，其他段用 `completion_model_id`/`embedding_model_id` 引用）。全局：`concurrent_requests=25`、`async_mode`。
- **输入/存储**：`input`、`input_storage`、`output_storage`、`update_output_storage`、`table_provider`（默认 parquet 落盘）。
- **分块/嵌入**：`chunking`（size=1200/overlap=100/`o200k_base`）、`embed_text`。
- **图构建**：`extract_graph`（LLM）、`extract_graph_nlp`（NLP 名词短语）、`summarize_descriptions`、`prune_graph`、`cluster_graph`（Leiden：`max_cluster_size`/`use_lcc`/`seed`）、`extract_claims`（默认关闭）。
- **社区报告**：`community_reports`。
- **搜索**：`local_search` / `global_search` / `drift_search` / `basic_search`。
- **后端**：`vector_store`（默认 LanceDB `output/lancedb`）、`cache`（默认 JSON，`cache/`）、`reporting`（默认文件，`logs/`）。
- **其他**：`snapshots`、`workflows: list[str] | None`（显式指定后**完全覆盖**内置管道顺序）。

构造后 `@model_validator(mode="after")` 校验各存储 base_dir，并为每个必需 embedding 注入默认 `IndexSchema`、解析 LanceDB `db_uri`。

### 默认值架构（`config/defaults.py`）

单一真相源：一组 `@dataclass`（`ChunkingDefaults`、`ClusterGraphDefaults`、`LocalSearchDefaults`…）聚合为 `graphrag_config_defaults` 单例，各子配置的 `Field(default=...)` 均引用它（DRY）。关键常量：`DEFAULT_COMPLETION_MODEL="gpt-4.1"`、`DEFAULT_EMBEDDING_MODEL="text-embedding-3-large"`、`ENCODING_MODEL="o200k_base"`、`DEFAULT_ENTITY_TYPES=["organization","person","geo","event"]`。

关键枚举（`config/enums.py`）：`SearchMethod`(LOCAL/GLOBAL/DRIFT/BASIC)、`IndexingMethod`(Standard/Fast/StandardUpdate/FastUpdate)、`NounPhraseExtractorType`、`ModularityMetric`、`AsyncType`、`ReportingType`。三个命名嵌入（`config/embeddings.py`）：`entity_description`、`community_full_content`、`text_unit_text`。

---

## 4. 数据模型（Domain Model）

`data_model/` 中的纯 `@dataclass`，均可通过 `from_dict(...)`（带可配置列映射）从 parquet 列反序列化。继承链：**`Identified` → `Named` → 领域类型**。

| 类 | 文件 | 继承 | 关键字段 | 用途 |
|---|---|---|---|---|
| `Identified` | identified.py | — | `id`, `short_id` | 有稳定 ID 者的基类 |
| `Named` | named.py | `Identified` | `+ title` | 额外带名称/标题 |
| **`Entity`** | entity.py | `Named` | `type, description, description_embedding, community_ids, text_unit_ids, rank, attributes` | 图节点（组织/人物/地点/事件）；`rank` 映射自 `degree` |
| **`Relationship`** | relationship.py | `Identified` | `source, target, weight, description, text_unit_ids, rank` | 两实体间有向加权边（无 title 故继承 `Identified`） |
| **`Community`** | community.py | `Named` | `level, parent, children, entity_ids, relationship_ids, text_unit_ids, covariate_ids, size, period` | 层次化 Leiden 社区树节点 |
| **`CommunityReport`** | community_report.py | `Named` | `community_id, summary, full_content, rank, full_content_embedding, size, period` | 社区的 LLM 摘要报告；**global search 的主要输入单元** |
| **`Covariate`** | covariate.py | `Identified` | `subject_id, subject_type="entity", covariate_type="claim", text_unit_ids` | 附着在主体上的结构化声明 |
| **`TextUnit`** | text_unit.py | `Identified` | `text, entity_ids, relationship_ids, covariate_ids, n_tokens, document_id` | 源文本块，回链其抽取出的实体/关系 |
| **`Document`** | document.py | `Named` | `type="text", text_unit_ids, text` | 源文档，被分解为 text units |

辅助：`schemas.py`（各 parquet 输出的列名常量与 `*_FINAL_COLUMNS` 顺序）、`data_reader.py`/`row_transformers.py`/`dfs.py`（DataFrame ↔ model 转换）。

---

## 5. 索引子系统（Indexing）

### 5.1 管道概览

管道在 `index/workflows/factory.py` 中声明为**有序 workflow 名称列表**，注册到 `PipelineFactory`，按 `IndexingMethod` 键控。每条管道前置一个文档加载 workflow（新建用 `load_input_documents`，更新用 `load_update_documents`）。

**标准管道**（`IndexingMethod.Standard`）：

| # | Workflow | 作用 |
|---|---|---|
| 1 | `create_base_text_units` | 文档按 token 分块为 text units |
| 2 | `create_final_documents` | 终态化 documents 表（含 text-unit id 映射） |
| 3 | `extract_graph` | LLM 抽取实体+关系，再 LLM 摘要描述 |
| 4 | `finalize_graph` | 计算节点度数，终态化实体/关系表（可选 GraphML 快照） |
| 5 | `extract_covariates` | LLM 抽取声明（claims）→ covariates 表 |
| 6 | `create_communities` | 层次化 Leiden 聚类为社区树 |
| 7 | `create_final_text_units` | 用实体/关系/声明引用富化 text units |
| 8 | `create_community_reports` | LLM 为每个社区生成报告 |
| 9 | `generate_text_embeddings` | 将选定字段嵌入向量库 |

**Fast / NLP 管道**（`IndexingMethod.Fast`）：形态相同，但用 `extract_graph_nlp`（名词短语图，**无 LLM**）替代 LLM 抽取、增加 `prune_graph`、去掉 `extract_covariates`、用 `create_community_reports_text`（基于 text unit 而非图）。

**更新管道**（`StandardUpdate`/`FastUpdate`）：对新文档增量跑完整标准/fast 列表，再追加合并链 `_update_workflows`：`update_final_documents → update_entities_relationships → update_text_units → update_covariates → update_communities → update_community_reports → update_text_embeddings → update_clean_state`，将新增量与备份的旧索引合并。

`api/index.py::build_index` 选择方法（`is_update_run` 时追加 `-update`），经 `PipelineFactory.create_pipeline` 构建，`run_pipeline` 流式产出结果。`config.workflows` 若设置则**完全覆盖**默认列表。

### 5.2 Workflow 抽象

- **定义**（`index/typing/workflow.py`）：`Workflow = tuple[str, WorkflowFunction]`，其中 `WorkflowFunction = Callable[[GraphRagConfig, PipelineRunContext], Awaitable[WorkflowFunctionOutput]]`。每个 workflow 模块导出 `async def run_workflow(config, context)`，把产物写入 storage/tables，返回 `WorkflowFunctionOutput(result, stop=False)`（`result` 仅用于日志；`stop=True` 终止管道）。
- **注册**（`factory.py`）：`PipelineFactory` 持类级 `workflows: dict` 与 `pipelines: dict`；`register`/`register_pipeline`/`create_pipeline` 完成名称→函数解析。
- **运行器**（`index/run/run_pipeline.py`）：构建 input/output `Storage`、`TableProvider`、`Cache`，加载持久化 `context.json`，经 `create_run_context` 建 `PipelineRunContext`，逐个 workflow 执行（`WorkflowProfiler` 计时、触发 `workflow_start/end` 回调），产出 `PipelineRunResult`；每步后写 `stats.json`/`context.json`，异常被捕获并作为带 `error` 的结果回传。
- **状态流转**（`index/typing/context.py`）：`PipelineRunContext` 携带 `stats`、`input_storage`、`output_storage`、`output_table_provider`、`previous_table_provider`（仅更新时）、`cache`、`callbacks`、`state`（`PipelineState=dict`，跨运行经 `context.json` 持久化）。**Workflow 间不靠返回值通信，而是通过共享 `output_table_provider` 读写命名表**（如 `DataReader(context.output_table_provider)`）；LLM cache 按模型经 `context.cache.child(...)` 命名空间隔离。
- **更新接线**：更新模式下运行器建带时间戳的子存储，含 `delta/` 与 `previous/`，把现有输出拷入 `previous/`，把 context 的 `output_*` 指向 delta、`previous_table_provider` 指向备份，并在 state 存 `update_timestamp`。

### 5.3 Operations 层（`index/operations/`）

可复用、与 workflow 无关的转换（重型 LLM 操作在子包）：

| Operation | 作用 |
|---|---|
| `extract_graph/` | LLM 从文本抽取实体/关系（`graph_extractor.py`） |
| `extract_covariates/` | LLM 抽取声明（`claim_extractor.py`） |
| `summarize_descriptions/` | LLM 把逐次提及的描述摘要成单一规范描述 |
| `summarize_communities/` | 生成社区报告（含 `graph_context/`、`text_unit_context/` 两种上下文构建） |
| `embed_text/` | 把流式表文本嵌入向量库 |
| `build_noun_graph/` | NLP 名词短语图构建（fast 管道，含 `np_extractors/`） |
| `cluster_graph.py` | 关系图的层次化 Leiden 聚类为分层社区 |
| `finalize_entities.py` / `finalize_relationships.py` | 流式富化实体/关系行（度数、id） |
| `compute_edge_combined_degree.py` | 每条边的 source+target 合并度数 |
| `prune_graph.py` | 剪除低价值节点/边（fast 管道） |
| `snapshot_graphml.py` | 按配置导出 GraphML 快照 |

底层分块见 `index/text_splitting/text_splitting.py`（`TokenTextSplitter`，默认 8191/100 重叠）；标准路径实际用 `graphrag_chunking` 包的 `create_chunker`。图算法在 `graphs/`：`hierarchical_leiden`、`connected_components`、`stable_lcc`、`modularity`、`edge_weights`、`compute_degree`。

### 5.4 产出制品

全部经 `output_table_provider` 写为 parquet：

| 表 | 由谁产出 |
|---|---|
| `documents` | `load_input_documents` → `create_final_documents` |
| `text_units` | `create_base_text_units` → `create_final_text_units` |
| `entities` | `extract_graph`（含摘要描述）→ `finalize_graph` |
| `relationships` | `extract_graph` → `finalize_graph` |
| `covariates` | `extract_covariates`（仅标准管道） |
| `communities` | `create_communities`（层次化 Leiden 树） |
| `community_reports` | `create_community_reports`（标准）/ `create_community_reports_text`（fast） |
| 向量嵌入 | `generate_text_embeddings`（默认字段：`text_unit.text`、`entity.description`、`community.full_content`） |
| `raw_entities`/`raw_relationships` | `extract_graph`（当 `config.snapshots.raw_graph`） |
| `graph`(GraphML) | `finalize_graph`（当 `config.snapshots.graphml`） |

另写运行元数据：`stats.json`（逐 workflow 计时）、`context.json`（持久化 state）。

---

## 6. 查询子系统（Query）

对已索引的知识图谱提问。提供**四种检索-应答策略**，每种由一个 *ContextBuilder*（把图数据组装成 token 预算内的提示上下文）+ 一个 *Search 引擎*（调 LLM 产出有据答案）构成，统一暴露流式/非流式接口，经 `query/factory.py` 从 `GraphRagConfig` 构建，由 `api/query.py` 调用。

### 6.1 四种搜索方法

| 方法 | 引擎 / 上下文构建器 | 机制 | 适用问题 |
|---|---|---|---|
| **Local** | `LocalSearch` / `LocalSearchMixedContext` | 把 query 嵌入并映射到 top-k 最近实体（`map_query_to_entities` 对实体描述向量库），组装**混合上下文**（社区报告 + 实体/关系/声明表 + 关联原文 text units），按 `community_prop`/`text_unit_prop` 分配 `max_context_tokens`。**单次** LLM 调用 | 具体的、实体局部的问题 |
| **Global** | `GlobalSearch` / `GlobalCommunityContext` | 把**社区报告摘要**分批（可选 `DynamicCommunitySelection` 用 LLM 打分选社区）。**Map-Reduce**：`_map_response_single_batch` 并行（`asyncio.Semaphore` 限流）对每批出 JSON 要点+重要度分；`_reduce_response` 滤掉 0 分、按分排序、截断到 `max_data_tokens`、终局合成 | 全局性、数据集级主题问题 |
| **DRIFT** | `DRIFTSearch` / `DRIFTSearchContextBuilder` | "Dynamic Reasoning and Inference with Flexible Traversal"：全局定调 + 迭代局部跟进。`PrimerQueryProcessor` 用 HyDE 扩展 query→cos 排序选 top-k 报告；`DRIFTPrimer.decompose_query` 出初始答案+跟进问题。主循环维护 `QueryState`（`networkx.MultiDiGraph` 的 `DriftAction`），每轮（至多 `n_depth`）执行 top `drift_k_followups`（内部用 `LocalSearch`），把新跟进加回图，最后 reduce | 复杂多跳问题 |
| **Basic** | `BasicSearch` / `BasicSearchContext` | 纯向量 RAG，**完全忽略图**：`similarity_search_by_text` 取 top-k 文本块，打包为 CSV 上下文，单次 LLM 调用 | 直接的段落检索 / 基线 |

### 6.2 搜索抽象（`structured_search/base.py`）

`BaseSearch(ABC, Generic[T])`（T 为四种 ContextBuilder 之一）持 `model`（`LLMCompletion`）、`context_builder`、`tokenizer`、`model_params`。两个抽象协程：`search(...) -> SearchResult`（非流式）与 `stream_search(...) -> AsyncGenerator[str, None]`（流式 token）。

`SearchResult` 数据类携带 `response`、`context_data`（DataFrame 字典）、`context_text`、`completion_time` 及遥测（`llm_calls`/`prompt_tokens`/`output_tokens` 及分类明细）。`GlobalSearch` 用 `GlobalSearchResult` 扩展（含 `map_responses`）。流式产出 `chunk.choices[0].delta.content` 并触发 `QueryCallbacks`（`on_llm_new_token`、`on_context`、global 专属 `on_map_response_start/end`）；DRIFT 的流式内部先跑非流式 `reduce=False` 再仅流式化 reduce 步。

### 6.3 上下文构建器（`context_builder/`）

`builders.py` 定义抽象 `GlobalContextBuilder`/`LocalContextBuilder`/`DRIFTContextBuilder`/`BasicContextBuilder` 及共享 `ContextBuilderResult`。图表→文本组装由辅助模块完成：`community_context.build_community_context`（渲染社区报告为带 weight/rank 列的定界表并排序、按 `max_context_tokens` 打包）、`local_context.py`（`build_entity_context`/`build_relationship_context`/`build_covariates_context`）、`source_context.build_text_unit_context`、`entity_extraction.map_query_to_entities`。

**Token 预算**为按比例顺序分配：`LocalSearchMixedContext.build_context` 先扣除对话历史成本，再按 `community_prop`/`text_unit_prop`/剩余 local-prop 切分（断言 `community_prop + text_unit_prop <= 1`），逐段构建，溢出即回退。

**对话历史**（`conversation_history.py`）：`ConversationHistory` 持 `ConversationTurn`（system/user/assistant），`get_user_turns` 取最近 N 个用户问题（拼到 query 前再做实体映射），`build_context` 渲染为 token 受限的 CSV 段（可 `recency_bias`）。

### 6.4 API 层（`api/query.py`）

四族搜索各含非流式协程 + `_streaming` 变体：`global_search`/`_streaming`、`local_search`/`_streaming`、`drift_search`/`_streaming`、`basic_search`/`_streaming`，均 `@validate_call`。接线：非流式版挂 `NoopQueryCallbacks`（`on_context` 捕获 `context_data`），迭代其 `_streaming` 版累积为 `full_response`，返回 `(full_response, context_data)`。`_streaming` 版做真正组装：`init_loggers` → `get_embedding_store` 取所需向量库 → 经 `read_indexer_*` 适配器把 DataFrame 转为领域对象 → `load_search_prompt` 载提示词 → `query/factory.py` 的 `get_*_search_engine` 建引擎 → 返回 `stream_search`。

> `question_gen/`（`LocalQuestionGen` 生成后续候选问题）是旁支能力，**未接入** `api/query.py`。

---

## 7. 基础设施包（可插拔后端）

7 个基础设施包遵循**同一设计范式**：抽象基类 + 若干可插拔后端实现 + 由配置 `StrEnum` 选择的 Factory（继承共享 `graphrag_common.factory.Factory`）。惰性注册使 Azure SDK / LanceDB / markitdown / NLTK 等重依赖延迟到真正请求时才导入。

### 7.1 Factory + 可插拔后端范式

泛型机制（`graphrag_common/factory/factory.py`）：`Factory[T]` 是单例泛型 ABC，按策略名键控。

- `register(strategy, initializer, scope="transient")` 记录 `_ServiceDescriptor`。
- `create(strategy, init_args)` 查描述符，剥除 `None` 值参数（让后端保留自身默认），`**init_args` 调用。
- `scope`：`"transient"`（每次新建）或 `"singleton"`（按 `hash_data({strategy, init_args})` 记忆化，相同配置→同一对象）。

每包在其上加一层薄封装：子类（如 `StorageFactory(Factory[Storage])`）+ 模块级单例 + `register_*` + `create_*(config)`：读 `config.type` 枚举 → 未注册则 `match` 惰性导入并注册 → `factory.create(...)`。

```python
# graphrag_storage/storage_factory.py —— 典型示例
class StorageFactory(Factory[Storage]): ...
storage_factory = StorageFactory()

def create_storage(config: StorageConfig) -> Storage:
    if config.type not in storage_factory:
        match config.type:
            case StorageType.File:        register_storage(StorageType.File, FileStorage)   # 惰性导入
            case StorageType.Memory:      register_storage(StorageType.Memory, MemoryStorage)
            case StorageType.AzureBlob:   register_storage(StorageType.AzureBlob, AzureBlobStorage)
            case StorageType.AzureCosmos: register_storage(StorageType.AzureCosmos, AzureCosmosStorage)
            case _: raise ValueError(...)
    return storage_factory.create(config.type, config.model_dump())
```

同一模板复现于 `cache_factory`、`table_provider_factory`、`chunker_factory`、`input_reader_factory`、`vector_store_factory`。Factory 亦可注入依赖：`create_cache`/`create_table_provider`/`create_input_reader` 会把 `Storage` 实例注入 `config_model["storage"]`。

### 7.2 LLM 中间件管道（graphrag-llm）

`with_middleware_pipeline.py` 在基础 LLM 可调用（`LLMCompletion` / embedding，构造时带 `ModelConfig`、`Tokenizer`、`MetricsStore` 等）外层层包裹横切关注点。每个 `with_*` 接受 sync+async 中间件并返回新的 `(sync, async)` 元组；**后应用者在最外层（最先运行）**，且除 logging 外均条件启用。

请求流向（最外→最内）：

```
with_logging          # 边界记录 请求/响应
→ with_request_count  # 计数（若有 metrics_processor）
→ with_cache          # 命中即短路返回；成功即写缓存（跳过流式/mock）
→ with_retries        # 重试（置于限流之外，每次重试重新排队限流）
→ with_rate_limiting  # 用 Tokenizer 估 token 成本节流
→ with_metrics        # 逐请求计时/token 指标
→ with_errors_for_testing  # 测试注入故障（failure_rate_for_testing>0）
→ 基础 model_fn       # litellm 实际调用
```

**缓存先于重试/限流** → 缓存命中不消耗限流预算、不触发重试。响应沿同栈冒泡（记指标、写缓存、增计数、记日志）。同一中间件栈同时包裹 chat 与 embedding（用 `request_type: Literal["chat","embedding"]` 区分）。

### 7.3 Storage vs Tables（graphrag-storage）

两个独立抽象：

- **`Storage`**（`storage.py`）：字节/字符串 K/V blob 存储。异步面：`get`/`set`/`has`/`delete`/`clear`，同步 `find(re.Pattern)`/`keys()`/`child(name)`（命名空间子存储）/`get_creation_date`。实现：`File`/`Memory`/`AzureBlob`/`AzureCosmos`。**低层基底**——cache、input readers、文件型 table provider 都消费 `Storage`。
- **`TableProvider`**（`tables/table_provider.py`）：管道 DataFrame 的高层表格抽象。异步：`read_dataframe`/`write_dataframe`/`has`，同步 `list()`/`child(name)`（更新管道用其隔离 delta/previous 表集）/`open(...) -> Table`（流式）。后端：`Parquet`/`CSV`/`Cosmos`。
- **`Table`**（`tables/table.py`）：`open()` 返回的行流式句柄，异步上下文管理器，支持 `async for row`、`length()`、`has`、`write`；可选 `RowTransformer`（如 Pydantic 模型）把原始行映射为类型化对象，实现大表的逐行内存高效处理。

一句话：`Storage`=无类型 blob 持久化 + 共享基底；`TableProvider`/`Table`=类型化表格持久化（整 DataFrame 或流式行），各有 file 与 database 两类后端。

---

## 8. CLI 与 API 表面

### CLI（Typer，`cli/main.py` 的 `app`；`python -m graphrag` 亦可）

| 命令 | 实现 | 说明 |
|---|---|---|
| `init` | `cli/initialize.py` | 脚手架：写默认 `settings.yaml` + `.env`。`--root/--model/--embedding/--force` |
| `index` | `cli/index.py::index_cli` | 从输入文档构建知识图谱索引。`--method/--verbose/--dry-run/--cache/--skip-validation` |
| `update` | `cli/index.py::update_cli` | 增量更新既有索引（写入 update_output） |
| `prompt-tune` | `cli/prompt_tune.py`（async） | 从自有数据自动生成领域定制的抽取/报告提示词。`--domain/--selection-method/--language/--discover-entity-types/...` |
| `query` | `cli/query.py` | 查询索引，按 `--method`（`SearchMethod`，默认 `GLOBAL`）分派。`--data/--community-level/--dynamic-community-selection/--response-type/--streaming` |

### API（`graphrag.api`，标记为 unstable，无向后兼容保证）

`__all__` 为权威公共表面：

- **索引**：`build_index(...)`（async）
- **查询**：`global_search`/`_streaming`、`local_search`/`_streaming`、`drift_search`/`_streaming`、`basic_search`/`_streaming`
- **提示词调优**：`generate_indexing_prompts(...)` + `DocSelectionType`

CLI 本质上是"载配置 → 调 API"的薄封装。

---

## 9. 关键设计模式总结

1. **Factory + 可插拔后端 + 惰性导入**：所有 I/O 后端（storage/vectors/cache/chunking/input/table）统一走 `Factory[T]`，由配置枚举选实现，重依赖延迟导入。
2. **配置驱动 + DRY 默认**：`GraphRagConfig` 单一聚合根，默认值集中于 `graphrag_config_defaults` 单例，`.env` + `${VAR}` 替换支撑密钥外置。
3. **有序 Workflow 管道 + 共享表存储通信**：索引是命名 workflow 的有序列表，workflow 间**不靠返回值、而靠读写共享 `TableProvider` 命名表**通信；`PipelineFactory` 使管道可组合、可完全自定义（`config.workflows`）。
4. **中间件管道**：LLM 调用被 cache/retry/ratelimit/metrics/logging 层层包裹，顺序精心设计（缓存先于重试/限流）。
5. **策略化检索**：四种搜索（local/global/drift/basic）共享 `BaseSearch` 抽象与统一流式接口，各配专属 ContextBuilder，覆盖从"实体局部"到"数据集全局"到"多跳推理"到"纯向量 RAG"的问答谱系。
6. **图谱制品作为契约**：parquet 表（+向量库）是索引与查询之间的稳定契约；`data_model` 的 dataclass 经 `read_indexer_*` 适配器双向桥接 DataFrame 与领域对象。
7. **增量更新**：`*Update` 管道用 delta/previous 双表集 + 合并 workflow 链，实现文档增量而非全量重建。

---

*本文档由对源码的自动化架构分析生成，反映 graphrag 3.1.0 快照状态。*
