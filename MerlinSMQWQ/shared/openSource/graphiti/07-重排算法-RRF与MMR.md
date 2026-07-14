# Graphiti 重排算法详解：RRF 与 MMR（附 node_distance / episode_mentions）

> [检索管道](05-检索管道-search.md)里说过「多路召回后要用重排器融合」。这份文档把两个最核心的重排器 **RRF** 和 **MMR** 的数学公式和代码逐行讲透，用具体数字算给你看。
>
> 代码位置：`graphiti_core/search/search_utils.py`（`rrf` / `maximal_marginal_relevance` / `node_distance_reranker` / `episode_mentions_reranker`）

> 📖 关联阅读：[检索管道](05-检索管道-search.md)——本文是它「重排」那一步的深挖。

---

## 一、先回顾：重排器要解决什么问题？

多路召回（BM25 关键词、向量语义、BFS 图遍历）各自返回一个**排好序的列表**。问题是：

1. **怎么把多个列表融合成一个？**——而且它们的「分数」根本没法直接比（BM25 分数和余弦相似度不是一个量纲）。→ **RRF**
2. **怎么避免结果全是「近似重复」？**——前 5 条都在讲同一件事，信息量低。→ **MMR**

RRF 管「融合」，MMR 管「去冗余、要多样」。

---

## 二、RRF（Reciprocal Rank Fusion，倒数排名融合）⭐默认

### 2.1 核心思想

**只看排名，不看分数。** 一个结果在各路召回里排得越靠前，综合分越高。它排在多个列表的前面 → 综合分累加起来就很高。

> 为什么只用排名？因为 BM25 分数（可能是 0~30）和余弦相似度（0~1）**量纲不同、没法直接相加**。但「排名」是通用的——不管哪路召回，第 1 名就是第 1 名。RRF 用排名把不可比的分数变成可比的。

### 2.2 完整代码（就这么短）

```python
def rrf(results: list[list[str]], rank_const=1, min_score: float = 0):
    scores: dict[str, float] = defaultdict(float)
    for result in results:              # results = 多路召回的排名列表
        for i, uuid in enumerate(result):
            scores[uuid] += 1 / (i + rank_const)   # rank_const=1 → 1/(排名+1)
    # 按累加分数降序排序
    scored_uuids = sorted(scores.items(), reverse=True, key=lambda t: t[1])
    return [uuid for uuid, s in scored_uuids if s >= min_score], [...]
```

### 2.3 公式

对每个结果 item，它的 RRF 总分：

```
score(item) = Σ  1 / (rank_in_list + rank_const)
             各列表
```

`rank_const=1`，排名从 0 开始，所以：
- 某列表第 1 名（i=0）贡献 `1/(0+1) = 1.0`
- 第 2 名（i=1）贡献 `1/(1+1) = 0.5`
- 第 3 名（i=2）贡献 `1/(2+1) = 0.333`
- ……越靠后贡献越小，且**衰减很快**

> `rank_const` 的作用：防止第 1 名分数「一家独大」。如果 rank_const 更大（比如常见的 60），头部差距会被压平，更看重"广泛上榜"而非"某处夺冠"。Graphiti 用 1，头部权重较高。

### 2.4 算给你看

假设 3 路召回，找 X 和 Y 两个结果：

| 结果 | BM25 排名 | 向量排名 | BFS 排名 | RRF 总分 |
|------|:--------:|:-------:|:-------:|---------|
| **X** | 第1 (1.0) | 第3 (0.333) | 未上榜 (0) | **1.333** |
| **Y** | 第2 (0.5) | 第1 (1.0) | 第1 (1.0) | **2.5** ✅ |

**Y 赢了**——虽然它在 BM25 里只排第 2，但它**在三路里都名列前茅**，综合分远超只在一路夺冠的 X。这就是 RRF 的精髓：**奖励「多路都认可」的结果**，比单路的冠军更可靠。

---

## 三、MMR（Maximal Marginal Relevance，最大边际相关性）

### 3.1 核心思想

**在「和查询相关」与「和已选结果不重复」之间做权衡。** 一个候选既要相关，又不能和别的候选太像——否则就是冗余。

### 3.2 公式

```
mmr(candidate) = λ · 相关性 − (1−λ) · 冗余度

  相关性 = cos(query, candidate)          ← 和查询有多像
  冗余度 = max cos(candidate, 其他候选)    ← 和最像的另一个候选有多像
  λ = mmr_lambda（默认 0.5）
```

- λ 越大 → 越看重相关性（越接近纯向量检索）
- λ 越小 → 越看重多样性（越强调结果各不相同）
- **默认 λ=0.5**：相关性和多样性五五开

### 3.3 代码要点

```python
def maximal_marginal_relevance(query_vector, candidates, mmr_lambda=0.5, min_score=-2.0):
    # 1. 所有候选嵌入做 L2 归一化（这样点积 = 余弦相似度）
    candidate_arrays = {uuid: normalize_l2(emb) for uuid, emb in candidates.items()}

    # 2. 建候选两两之间的相似度矩阵（对称，对角线为 0）
    for i, uuid_1:
        for j in range(i):              # 只算下三角，再镜像
            similarity_matrix[i][j] = similarity_matrix[j][i] = dot(cand[i], cand[j])

    # 3. 每个候选算 MMR 分
    for i, uuid:
        max_sim = np.max(similarity_matrix[i, :])   # 和其他候选的最大相似度 = 冗余度
        mmr = mmr_lambda * dot(query, cand[uuid]) + (mmr_lambda - 1) * max_sim
        #     = λ·相关性 − (1−λ)·冗余度     （因为 mmr_lambda-1 = -(1-λ)）
    # 按 mmr 分降序排序
```

> **代码细节**：相似度矩阵**对角线保持 0**（只填 i≠j 的格子），所以 `max_sim` 取的是「和**其他**候选的最大相似度」，不会被自己和自己的相似度(=1)干扰。这是正确实现冗余度的关键。

### 3.4 算给你看

λ=0.5，三个候选，看 MMR 怎么惩罚冗余：

| 候选 | 和查询相关性 | 和其他候选最大相似度（冗余） | MMR 分 = 0.5·相关 − 0.5·冗余 |
|------|:-----------:|:------------------------:|---------|
| **A** | 0.80（很相关） | 0.90（和 B 几乎重复） | 0.5·0.8 − 0.5·0.9 = **−0.05** |
| **B** | 0.78 | 0.90（和 A 几乎重复） | 0.5·0.78 − 0.5·0.9 = **−0.06** |
| **C** | 0.70（略低） | 0.30（独一无二） | 0.5·0.7 − 0.5·0.3 = **+0.20** ✅ |

**C 排到了第一**——尽管它和查询的原始相关性(0.70)低于 A(0.80)，但 A 和 B 是近似重复（互相冗余度 0.90），被狠狠扣分。MMR 宁可选一个稍不相关但**信息独特**的 C，也不想让结果列表被 A、B 这对「双胞胎」占满。

> 这就是 MMR 的价值：纯向量检索会把 A、B 都排在前面（都很相关），但用户看到两条几乎一样的结果没意义。MMR 保证结果**既相关又多样**。

---

## 四、RRF vs MMR 对比

| | RRF | MMR |
|--|-----|-----|
| **解决什么** | 融合多路召回的排名 | 结果去冗余、保多样性 |
| **输入** | 多个排名列表（只要排名） | 查询向量 + 候选向量 |
| **需要嵌入?** | ❌ 不需要 | ✅ 需要（要算相似度） |
| **成本** | 极低（纯累加排序） | 较高（要建 N×N 相似度矩阵，O(N²)） |
| **超参** | rank_const=1 | λ=0.5（相关 vs 多样） |
| **何时用** | 默认；多路召回融合 | 结果容易雷同、要多样性时 |

---

## 五、另外两个重排器（附带）

### node_distance_reranker —— 按图上距离重排

给定一个**中心节点** `center_node_uuid`，让**离它近的结果排前面**（"以某实体为中心"的检索）。

- Cypher 查 `(center)-[:RELATES_TO]-(n)`：直接邻居距离得分 1，非邻居得 `inf`。
- 按距离**升序**排（近的在前），返回的分数是 `1/距离`。
- 中心节点自己给 0.1 特殊分，放最前。

### episode_mentions_reranker —— 按被提及次数

- 先用 RRF 做初步排序。
- 再查每个实体被多少条 episode `MENTIONS`：`count(*)` of `(Episodic)-[:MENTIONS]->(n)`。
- 按提及次数重排（越常被提到的实体越"重要"）。

> 注：边（edge）版的 episode_mentions 更直接——`reranked_edges.sort(reverse=True, key=lambda e: len(e.episodes))`，按边关联的 episode 数降序。

---

## 六、cross_encoder 呢？

第 5 种重排器 cross_encoder 不在 `search_utils.py`——它调 [Provider 抽象层](10-Provider抽象层-llm与embedder.md)的 `CrossEncoderClient.rank()`，把 (query, 每条候选文本) 成对喂给重排模型精打分，**最准但最慢**。详见 Provider 文档。

---

## 七、一句话总结

> **RRF** 只用排名把多路召回融合成一个列表——`score = Σ 1/(排名+1)`，奖励「多路都靠前」的结果，妙在无需可比的分数量纲。**MMR** 用 `λ·相关性 − (1−λ)·冗余度`（默认 λ=0.5）在相关和多样之间权衡，宁可选稍不相关但独特的结果，也不让列表被近似重复占满。两者一个管「融合」、一个管「去冗余」，是 Graphiti 混合检索「召回保全、重排保准」里"保准"的核心。

> 配套结构图见：![RRF/MMR 重排算法图](svg/07-rerank-rrf-mmr.svg)
