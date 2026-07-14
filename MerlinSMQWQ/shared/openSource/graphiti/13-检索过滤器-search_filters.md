# Graphiti 检索过滤器：SearchFilters 与双时态过滤下推

> 「查 2024 上半年生效的事实」「只要 WORKS_AT 类型的关系」「重现系统在 3 月时以为的真相」——这些精细过滤怎么做到的？Graphiti 用 `SearchFilters` 把过滤条件**翻译成参数化 Cypher WHERE 子句下推到数据库**。本文讲清它的结构、双时态过滤如何支撑「时间旅行查询」。
>
> 代码位置：`graphiti_core/search/search_filters.py`、`search/search_utils.py`

> 📖 关联阅读：[检索管道](05-检索管道-search.md)（filter 作为参数贯穿四路 scope）· [双时态数据模型](03-双时态数据模型-nodes与edges.md)（四个时间字段的语义）。

---

## 一、SearchFilters 有哪些字段

`SearchFilters` 是一个纯声明式的过滤描述：

| 字段 | 类型 | 过滤什么 |
|------|------|---------|
| `node_labels` | `list[str]` | 按**实体类型**（节点标签）过滤 |
| `edge_types` | `list[str]` | 按**关系类型**（边的 name，如 `WORKS_AT`） |
| `valid_at` | `list[list[DateFilter]]` | 事实**开始生效**时间 🟢事件轴 |
| `invalid_at` | `list[list[DateFilter]]` | 事实**失效**时间 🟢事件轴 |
| `created_at` | `list[list[DateFilter]]` | 记录**写入图**时间 🔵系统轴 |
| `expired_at` | `list[list[DateFilter]]` | 记录被**取代/过期**时间 🔵系统轴 |
| `edge_uuids` | `list[str]` | 按具体边 UUID 白名单 |
| `property_filters` | `list[PropertyFilter]` | ⚠️ **已定义但未接线**（图查询里不生效，预留给 Neptune/OpenSearch） |

> 四个日期字段都是 `RELATES_TO` **边**上的属性，所以只在边搜索里处理。它们正是[双时态数据模型](03-双时态数据模型-nodes与edges.md)讲的两条时间轴。

> **防注入细节**：`node_labels` 有 Pydantic 校验器，用正则 `^[A-Za-z_][A-Za-z0-9_]*$` 检查每个标签——因为标签是**直接字符串拼接**进 Cypher 的（不是参数化），必须防注入。

---

## 二、DateFilter 与布尔组合（OR-of-AND）

### ComparisonOperator：枚举值就是 Cypher 运算符

```python
equals='='  not_equals='<>'  greater_than='>'  less_than='<'
greater_than_equal='>='  less_than_equal='<='
is_null='IS NULL'  is_not_null='IS NOT NULL'
```

枚举值直接是 Cypher 运算符字符串，构造 WHERE 时可直接拼接。

### DateFilter

```python
class DateFilter(BaseModel):
    date: datetime | None                   # 比较的时间点（is_null 时可为 None）
    comparison_operator: ComparisonOperator
```

### 关键：`list[list[DateFilter]]` = OR-of-AND（析取范式）

嵌套 list 就是布尔组合的表达方式：

- **外层 list = OR**（各元素之间用 `OR` 连接）
- **内层 list = AND**（同一子列表里的多个 DateFilter 用 `AND` 连接）

**例**：表达 `valid_at > 2024-01-01 AND valid_at < 2024-06-01`：

```python
SearchFilters(valid_at=[[
    DateFilter(date=datetime(2024,1,1), comparison_operator=ComparisonOperator.greater_than),
    DateFilter(date=datetime(2024,6,1), comparison_operator=ComparisonOperator.less_than),
]])   # 两个 DateFilter 在同一内层 list → 用 AND 连
```

想表达"上半年 **或** 空值"，就再加一个内层 list（外层多一个元素 → OR）。

---

## 三、过滤怎么「下推」到数据库

### 构造器：filter → (Cypher 片段列表, 参数字典)

`node_search_filter_query_constructor` 和 `edge_search_filter_query_constructor` 把 `SearchFilters` 翻译成 `(list[str], dict)`——**Cypher WHERE 片段列表 + 参数字典**。

**简单字段直接参数化**（防注入）：
```python
filter_queries.append('e.name in $edge_types')     # edge_types
filter_params['edge_types'] = edge_types            # 值走参数，不拼进字符串
filter_queries.append('e.uuid in $edge_uuids')     # edge_uuids
```

**日期字段逐段拼接**：`date_filter_query_constructor` 把单个 DateFilter 变成带括号的片段：
```python
# 普通比较 → "(e.valid_at > $valid_at_0)"
# is_null   → "(e.valid_at IS NULL)"（不生成参数）
```
同一内层 list 内用 ` AND ` 连、各内层 list 间用 ` OR ` 连、整体外包括号。

第二节的例子最终生成：
```cypher
((e.valid_at > $valid_at_0) AND (e.valid_at < $valid_at_1))
```
参数 `{'valid_at_0': 2024-01-01, 'valid_at_1': 2024-06-01}`。

### 拼进完整 WHERE

每个搜索函数模式一致：
```python
filter_queries, filter_params = edge_search_filter_query_constructor(search_filter, driver.provider)
if group_ids is not None:                          # group_id 也追加
    filter_queries.append('e.group_id IN $group_ids')
filter_query = ' WHERE ' + ' AND '.join(filter_queries)   # 所有片段用 AND 连
# 插到 MATCH 和 RETURN 之间，**filter_params 展开传给 execute_query
```

### 为什么「下推」比「取回来再过滤」高效？

| 好处 | 说明 |
|------|------|
| **减少传输/内存** | 数据库扫描阶段就丢弃不匹配的行，不把全部候选拉回 Python 再循环过滤 |
| **利用索引** | `WHERE e.valid_at > $x` 让图库走索引、提前剪枝遍历 |
| **与打分融合** | 过滤和 `score > $min_score`、`ORDER BY score LIMIT` 同一次查询完成，避免"先取 TopK 再过滤导致结果不足" |
| **参数化** | `$xxx` 让数据库缓存查询计划，且杜绝注入 |

---

## 四、双时态过滤：时间旅行查询 🕰️

这是最有价值的能力。边上四个时间字段两两成对（详见[双时态数据模型](03-双时态数据模型-nodes与edges.md)）：

- **事件时间轴**（现实世界何时为真）：`valid_at` / `invalid_at`
- **系统时间轴**（系统何时记录）：`created_at` / `expired_at`

### 「查询某时间点 T 现实世界的真相」

用 `valid_at <= T AND (invalid_at > T OR invalid_at IS NULL)`：

```python
SearchFilters(
    valid_at=[[DateFilter(date=T, comparison_operator=ComparisonOperator.less_than_equal)]],
    invalid_at=[
        [DateFilter(date=T, comparison_operator=ComparisonOperator.greater_than)],
        [DateFilter(comparison_operator=ComparisonOperator.is_null)],   # 或从未失效
    ],
)
```
生成：`((e.valid_at <= $valid_at_0)) AND ((e.invalid_at > $invalid_at_0) OR (e.invalid_at IS NULL))`

### 一个具体例子

> 用户 3 月说"我住北京"（valid_at=3月），6 月说"搬到上海"。系统把"住北京"这条边的 `invalid_at` 设为 6 月。
> - 查"**5 月**这个人住哪" → `valid_at≤5月 AND invalid_at>5月` → 命中**北京** ✅
> - 查"**7 月**" → 命中**上海** ✅

这就是**时间旅行**——旧事实不删除，靠 `invalid_at` 标记，任意历史时间点都能重建真相。

**其他玩法**：
- **只看当前有效事实**：`invalid_at IS NULL AND expired_at IS NULL`
- **审计"系统在某时刻以为的真相"**：用系统时间轴 `created_at <= T AND (expired_at > T OR expired_at IS NULL)`——能重现"当时数据库的认知"，即使后来发现那时的信息是错的。

---

## 五、类型过滤

- **node_labels**（按实体类型）：`n:Label1|Label2`（Cypher 原生标签 OR 语义）；用于边时要求**两端节点都**匹配（`n:... AND m:...`）。
- **edge_types**（按关系类型）：`e.name in $edge_types`，参数化，各后端一致。
- **edge_uuids**：`e.uuid in $edge_uuids`，常用于"只在给定的一批边里搜"（写入去重时的二次搜索就用它）。

---

## 六、多后端差异

**日期过滤、edge_types、edge_uuids 三类的 Cypher 对所有后端完全相同**（都基于参数化的 `e.valid_at`/`e.name`/`e.uuid`）。差异只集中在两处：

| | Neo4j/FalkorDB/Neptune | Kuzu |
|--|------------------------|------|
| **node_labels** | 原生标签 `n:Label1\|Label2`（字符串拼接） | 标签存成数组属性 → `list_has_all(n.labels, $labels)`（参数化） |
| **边的 MATCH 拓扑** | `(n)-[e:RELATES_TO]->(m)` | 边物化成中间节点：`(n)-[:RELATES_TO]->(e:RelatesToNode_)-[:RELATES_TO]->(m)` |

（Kuzu 向量还需 `CAST($search_vector AS FLOAT[N])`；Neptune 走 OpenSearch 召回再回图库匹配。）

---

## 七、如何贯穿主流程

```
Graphiti.search(..., search_filter=None)   # None → 默认空 SearchFilters()（不加任何 WHERE）
   │  分发同一个 filter 给四路 scope
   ├── edge_search    → edge_fulltext/similarity/bfs_search → edge 构造器
   ├── node_search    → node_fulltext/similarity_search    → node 构造器
   ├── episode_search → ⚠️ 收到 filter 但基本未使用
   └── community_search
```

> 注意：**日期/类型过滤主要作用于 edge 和 node 两个 scope**。episode 全文检索里 filter 参数名带下划线前缀且未使用。

---

## 八、一句话总结

> `SearchFilters` 是声明式过滤描述——字段 + `DateFilter`/`ComparisonOperator` 组成 **OR-of-AND 布尔式**；构造器把它翻译成**参数化 Cypher WHERE 片段 + 参数字典**，各搜索函数用 `WHERE ... AND ...` 拼进查询下推给数据库，在扫描/打分阶段就完成过滤（比取回再过滤高效得多）。四个**双时态**日期字段配合 `<= / > / IS NULL`，即可实现"查询任意时间点的历史真相"这种时间旅行查询。后端差异仅在 `node_labels`（Kuzu 用 `list_has_all`）和边的物化拓扑上。

> 配套结构图见：![检索过滤器结构图](svg/13-search-filters.svg)
