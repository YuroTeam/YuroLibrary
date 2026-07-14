# Graphiti LLM 提示词设计：`prompts/` 模块详解

> Graphiti 的「智能」全靠 LLM，而 LLM 的表现全靠提示词（prompt）。这份文档讲 `prompts/` 模块的**架构设计**（怎么组织、怎么调用）和**提示词工程手法**（怎么把 prompt 写得让 LLM 稳定听话）。
>
> 代码位置：`graphiti_core/prompts/`

> 📖 关联阅读：[写入管道](04-数据写入管道-add_episode.md)（每步的 LLM 调用点）· [去重算法](06-去重算法-dedup.md)（dedupe_nodes 提示词）· [社区构建](08-社区构建-communities.md)（summarize_nodes 提示词）。

---

## 一、为什么要专门设计一套提示词系统？

写入管道里每一步（抽实体、抽边、去重、抽时间、生成摘要）都要调 LLM，且要求 LLM 返回**严格结构化**的结果（能直接解析成对象）。如果 prompt 散落各处、格式随意，会有两个大问题：

1. **不稳定**：LLM 一会儿多抽、一会儿漏抽、一会儿返回格式不对。
2. **难维护**：想调优某个 prompt，找不到、改一处漏一处。

所以 Graphiti 把所有提示词**集中管理**在 `prompts/` 目录，配一套统一的调用/版本/结构化输出机制。

---

## 二、架构：三层 + 一个全局单例

### 核心数据类型（`models.py`）

```python
class Message(BaseModel):      # 一条消息
    role: str                  # 'system' 或 'user'
    content: str

PromptFunction = Callable[[dict], list[Message]]   # 提示词函数：吃 context 字典，吐消息列表
```

**每个提示词本质上就是一个函数**：输入一个 `context` 字典（要填进模板的数据），输出一个 `list[Message]`（通常是一条 system + 一条 user）。

### 每个 prompt 文件的固定三件套

以 `extract_nodes.py` 为例，每个文件都导出：

| 导出物 | 类型 | 作用 |
|--------|------|------|
| `Prompt` | Protocol | 声明本任务有哪些「版本」，纯给 IDE 做类型提示 |
| `Versions` | TypedDict | 同上，运行时的版本注册表类型 |
| `versions` | dict | **真正的注册表**：`{版本名: 提示词函数}` |

比如 `extract_nodes.py` 的 `versions` 里有 `extract_message`、`extract_json`、`extract_text`、`classify_nodes`、`extract_attributes`、`extract_summary`……——**一个任务下有多个「版本」**（针对不同输入类型或子任务）。

### 全局单例 `prompt_library`（`lib.py`）

`lib.py` 把所有文件的 `versions` 汇总，包成一个可以**点号访问**的全局对象：

```python
# 调用方式：prompt_library.<任务>.<版本>(context)
messages = prompt_library.extract_nodes.extract_message(context)
messages = prompt_library.dedupe_nodes.nodes(context)
messages = prompt_library.extract_edges.edge(context)
```

包装层次：`PromptLibraryWrapper` → `PromptTypeWrapper`（每个任务）→ `VersionWrapper`（每个版本）。

> **一个巧妙的细节**：`VersionWrapper` 在每次调用后，会自动给 **system 消息**追加一句 `"Do not escape unicode characters."`（`DO_NOT_ESCAPE_UNICODE`）。这样中文/日文/韩文不会被转义成 `\uXXXX`，在 LLM 日志里可读、模型理解也更好。所有提示词统一享受这个待遇，不用各自去写。

---

## 三、结构化输出：Pydantic 模型即「输出契约」

这是 Graphiti 提示词最关键的设计。每个 prompt 文件顶部都定义了一批 **Pydantic `BaseModel`**，作为 LLM 的**输出格式契约**（`response_model`）。

```python
# extract_edges.py
class Edge(BaseModel):
    source_entity_name: str = Field(..., description='The name of the source entity from the ENTITIES list')
    target_entity_name: str = Field(..., description='...')
    relation_type: str = Field(..., description='...in SCREAMING_SNAKE_CASE (e.g., WORKS_AT, LIVES_IN)')
    fact: str = Field(..., description='A natural language description...')
    valid_at: str | None = Field(None, description='...when the relationship became true...ISO 8601...')
    invalid_at: str | None = Field(None, description='...when it stopped being true...')
```

调用时把这个模型作为 `response_model` 传给 LLM 客户端，LLM **被强制返回符合该 schema 的 JSON**，直接解析成对象——不用手写正则、不用容错解析。

> **关键洞察**：`Field(description=...)` 里的描述**本身就是提示词的一部分**！模型 schema 会连同描述一起发给 LLM，告诉它每个字段该填什么、什么格式。比如 `relation_type` 的描述直接给了 `WORKS_AT` 这样的例子。**所以「写模型」和「写 prompt」是一体的。**

---

## 四、提示词工程手法（真正的「手艺」）🎨

这部分是最值得学的——Graphiti 用了哪些技巧让 LLM 稳定听话。以实体抽取 `extract_message` 为范本。

### 1. System 消息 = 人设 + 一条硬约束

```
You are an entity extraction specialist for conversational messages.
NEVER extract abstract concepts, feelings, or generic words.
```

给 LLM 一个明确身份（"抽取专家"），再叠加一条最重要的**大写强调**禁令。

### 2. XML 风格分隔符划分区块

```
<ENTITY TYPES> ... </ENTITY TYPES>
<PREVIOUS MESSAGES> ... </PREVIOUS MESSAGES>
<CURRENT MESSAGE> ... </CURRENT MESSAGE>
```

用标签把「类型定义」「历史上下文」「当前要处理的内容」清晰隔开，LLM 不会混淆哪些是要抽取的、哪些只是背景。

### 3. 超详尽的「排除清单」（负面指令）

抽实体的 prompt 花了大量篇幅列举 **NEVER extract** 的东西：代词、抽象概念/情绪、泛化名词、裸关系称谓（"dad" 要写成 "Nisha's dad"）、句子片段、形容词……

> **为什么**：LLM 天然倾向于「多抽」，抽出一堆 "happiness"、"day"、"things" 这种垃圾实体污染图谱。与其正面说「抽有用的」，不如反面把常见错误一条条堵死——**负面指令往往比正面指令更有效**。

### 4. 大量 Good/Bad few-shot 示例

```
<EXAMPLE>
Message: "Nisha: My dad is visiting next week. He loves walking his dogs in Riverside Park."
Good extractions: "Nisha" (speaker), "Nisha's dad" (Person), "Riverside Park" (Location)
Do NOT extract: "dad" (裸称谓→限定为"Nisha's dad"), "dogs" (裸动物词), "next week" (时间)
</EXAMPLE>
```

每个示例都**同时给出该抽的和不该抽的**，还标注原因。抽实体的 prompt 里有 5+ 个这样的例子，覆盖各种边界情况。去重 prompt 也有 `Sam=Sam`、`NYC=New York City`、`Java语言≠Java岛` 这类精准对照。

### 5. 编号规则 + 内联 BAD/GOOD 对比

规则用 `1. 2. 3.` 编号带子弹点，关键处直接内联对比：

```
BAD: "Alice feels happy" (模糊的单实体状态)
GOOD: "Alice feels happy about Bob's promotion" → Alice -FEELS_HAPPY_ABOUT-> Bob's promotion
```

### 6. 时态分离——各司其职

- 抽实体 prompt 明确说：**"Do NOT extract dates, times — these will be handled separately"**。
- 抽边 prompt 才负责时间：用 `REFERENCE_TIME` 解析「上周」这类相对时间，多 episode 时优先用各自的时间戳。

> 把「抽实体」「抽关系」「抽时间」拆成不同 prompt，每个只干一件事，比一个巨型 prompt 干所有事更稳定、更好调优。这呼应了[写入管道](04-数据写入管道-add_episode.md)的分步设计。

### 7. 防幻觉 + 拒绝泛化的硬约束

- **防编造**：`"Using names not in the list will cause the edge to be rejected"`（边的两端必须来自给定实体列表）；`"Do not hallucinate or infer temporal bounds"`。
- **拒绝泛化**：`"NEVER generalize 'Gamecube' to 'gaming console', 'Ford Mustang' to 'car'"`——强制保留原文的品牌名、型号、数量、颜色等具体细节。这保证了知识图谱的信息密度。

### 8. 可复用的提示片段（`snippets.py`）

公共指令抽成片段复用，比如 `summary_instructions`（摘要规范：信息密集、少于 `MAX_SUMMARY_CHARS`、别用 "mentioned/described" 填充词、直接陈述事实……）被多个摘要类 prompt 共享。

---

## 五、提示词全家福（`prompts/` 目录）

| 文件 | 任务 | 主要版本 | 对应管道步骤 |
|------|------|---------|-------------|
| `extract_nodes.py` | 抽实体 | `extract_message`/`_json`/`_text`（按 EpisodeType）、`classify_nodes`、`extract_attributes`、`extract_summary` | 写入②③⑤ |
| `extract_edges.py` | 抽关系 | `edge`、`extract_timestamps`、`extract_attributes` | 写入④ |
| `dedupe_nodes.py` | 实体去重 | `nodes` | 写入③（[去重](06-去重算法-dedup.md)第 3 级） |
| `dedupe_edges.py` | 边去重/消解 | 判断重复与矛盾 | 写入④ |
| `summarize_nodes.py` | 摘要 | `summarize_pair`、`summary_description` | [社区](08-社区构建-communities.md)、属性水合 |
| `summarize_sagas.py` | Saga 增量摘要 | | 长会话摘要 |
| `extract_nodes_and_edges.py` | 一次性抽点+边 | | 批量场景 |
| `eval.py` | 评估 | | 端到端评测 |

> **命名规律**：`extract_*`（从文本抽结构）、`dedupe_*`（去重消解）、`summarize_*`（浓缩摘要）——正好对应写入管道的三大动作。

---

## 六、一次完整调用长什么样

以抽边为例，把前面串起来：

```python
# 1. 准备 context（要填进模板的数据）
context = {
    'previous_episodes': [...],       # 历史上下文
    'episode_content': "...",         # 当前要处理的文本
    'nodes': [...],                   # 已抽取的实体（边的两端必须来自这里）
    'reference_time': "2026-07-03...",# 解析相对时间的基准
    'edge_types': {...},              # 自定义边类型
    'custom_extraction_instructions': "",
}

# 2. 取提示词函数，生成 messages（system + user）
messages = prompt_library.extract_edges.edge(context)

# 3. 配结构化输出模型，调 LLM
result = await llm_client.generate_response(messages, response_model=ExtractedEdges)

# 4. result 直接是符合 schema 的对象，无需解析
for edge in result.edges: ...
```

---

## 七、一句话总结

> Graphiti 把所有 LLM 提示词集中在 `prompts/`，用**「函数(context)→list[Message]」+ Pydantic 结构化输出模型**的统一机制组织，通过 `prompt_library.<任务>.<版本>()` 调用。提示词工程手法密集：**人设+硬约束、XML 分隔、超长排除清单、Good/Bad few-shot、时态分离、防幻觉、拒绝泛化**——核心目标是让 LLM 的输出**稳定、结构化、信息密集、不污染图谱**。而 Pydantic 模型的 `Field 描述本身就是提示词**，写模型和写 prompt 是一体的。

> 配套结构图见：![提示词架构图](svg/09-prompts-architecture.svg)
