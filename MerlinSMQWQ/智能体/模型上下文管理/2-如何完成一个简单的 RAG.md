---
tags:
  - rag构建步骤
  - 数据预处理/json
  - embedding调用
  - 向量数据库方案
  - dashscope集成
  - rag-基础设施
  - 向量数据库
---

# RAG 搭建步骤

## 项目概览

本项目演示如何从零搭建一个 RAG（Retrieval-Augmented Generation）系统，涵盖三个向量数据库方案：

| 方案 | 文件 | 特点 |
|------|------|------|
| ChromaDB | `2-how-to-build-a-chromadb.py` | 内置 embedding 支持，开箱即用 |
| FAISS | `3-how-to-use-faiss.py` | Facebook 开源，高性能，适合大规模数据 |
| SQLite-vec | `4-how-to-use-sqlite.py` | 基于 SQLite 扩展，轻量且便于集成 |

三个方案共享相同的数据格式和 embedding 模型，仅在向量存储层有差异。

## 环境准备

项目使用 `uv` 管理依赖，Python >= 3.13：

```bash
uv sync
```

核心依赖（见 `pyproject.toml`）：
- `openai` — 调用 DashScope 兼容接口
- `dashscope` — 阿里云百炼 SDK
- `chromadb` / `faiss-cpu` / `sqlite-vec` — 三种向量数据库
- `dotenv` — 环境变量管理

在项目根目录创建 `.env` 文件，写入你的 DashScope API Key：

```
DASHSCOPE_API_KEY=sk-xxx
```

## 准备数据

RAG 的第一步是获取并格式化数据。本项目的数据源为 JSON 文件（`data/dataset.json`），结构为：

```json
{
  "条目名称1": "条目内容1",
  "条目名称2": "条目内容2"
}
```

如果你的原始数据是文本格式（如问答对），可以用正则表达式清洗后转为 JSON：

```python
import re
import json

def process_dataset(input_path, output_path):
    with open(input_path, 'r', encoding='utf-8') as file:
        content = file.read()

    qa_pairs = re.split(r'\n\n(?=\(\d+\))', content)

    dataset = {}
    for qa in qa_pairs:
        qa_match = re.match(r'\((\d+)\)(.*?)\n答：(.*)', qa, re.DOTALL)
        if qa_match:
            _, question, answer = qa_match.groups()
            dataset[question.strip()] = answer.strip()

    with open(output_path, 'w', encoding='utf-8') as out:
        json.dump(dataset, out, ensure_ascii=False, indent=2)

process_dataset('your/file/path.txt', 'data/dataset.json')
```

## Embedding 方案

本项目不使用本地 embedding 模型，而是通过 DashScope 的 `text-embedding-v4` 接口进行向量化。这样做的好处是无需下载大模型、不占用本地显存。

统一的 embedding 调用封装：

```python
import json
from openai import OpenAI

def embedding(client: OpenAI, input: str) -> list:
    completion = client.embeddings.create(model="text-embedding-v4", input=input)
    result = json.loads(completion.model_dump_json()).get("data")[0].get("embedding")
    return result
```

三个向量数据库方案都复用这个函数，保证 embedding 一致性。

## 方案一：ChromaDB

ChromaDB 是一个轻量级向量数据库，内置 embedding 管理，适合小规模数据快速上手。

```bash
pip install chromadb
```

核心实现（完整代码见 `src/2-how-to-build-a-chromadb.py`）：

```python
import json
import os
import chromadb
from pprint import pprint
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

openai_client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)


class ChromaDataBase:
    def __init__(self, database_name: str, persist_dir: str, data_path: str):
        self.db_name = database_name
        self.db_dir = persist_dir
        self.source_data_path = data_path
        self.collection: None | chromadb.Collection = None

    @staticmethod
    def embedding(client: OpenAI, input: str) -> list:
        completion = client.embeddings.create(model="text-embedding-v4", input=input)
        result = (
            json.loads(completion.model_dump_json()).get("data")[0].get("embedding")
        )

        return result

    def create_chroma_database(self):
        with open(self.source_data_path, "r", encoding="utf-8") as file:
            content = file.read()
            dataset = json.loads(content)

        dataset_keys = list(dataset.keys())
        dataset_values = list(dataset.values())

        client = chromadb.PersistentClient(path=self.db_dir)
        self.collection = client.get_or_create_collection(name=self.db_name)
        if self.collection.count() > 0:
            print("collection 中已经有数据了，跳过写入")
        else:
            self.collection.add(
                ids=[f"id{i}" for i in range(1, len(dataset_keys) + 1)],
                embeddings=[
                    ChromaDataBase.embedding(openai_client, dataset_value)
                    for dataset_value in dataset_values
                ],
                metadatas=[
                    {"source": self.source_data_path} for _ in range(len(dataset))
                ],
                documents=[dataset_value for dataset_value in dataset_values],
            )
            print("collection 已创建")

    def query_result(self, input: str):
        if isinstance(self.collection, chromadb.Collection):
            return self.collection.query(
                query_embeddings=[ChromaDataBase.embedding(openai_client, input)],
                n_results=1,
                include=["metadatas", "documents", "distances"],
            )
        else:
            return None


if __name__ == "__main__":
    database = ChromaDataBase(
        "my_chromadb_collection",
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/chroma-collection",
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/dataset.json",
    )

    database.create_chroma_database()
    result = database.query_result(input="登封窑陶瓷烧制技艺")
    pprint(result)

```

## 方案二：FAISS

FAISS 是 Facebook 开源的向量检索库，性能优异，适合大规模向量检索场景。

```bash
pip install faiss-cpu
```

核心实现：

```python
import json
import os
import faiss
import numpy as np
from pprint import pprint
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

openai_client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)


class FAISSDataBase:
    def __init__(self, index_path: str, data_path: str):
        self.index_path = index_path
        self.source_data_path = data_path
        self.index: faiss.Index | None = None
        self.documents: list[str] = []
        self.ids: list[str] = []

    @staticmethod
    def embedding(client: OpenAI, input: str) -> list:
        completion = client.embeddings.create(model="text-embedding-v4", input=input)
        result = (
            json.loads(completion.model_dump_json()).get("data")[0].get("embedding")
        )
        return result

    def create_faiss_index(self):
        with open(self.source_data_path, "r", encoding="utf-8") as file:
            content = file.read()
            dataset = json.loads(content)

        dataset_keys = list(dataset.keys())
        dataset_values = list(dataset.values())

        if os.path.exists(f"{self.index_path}.index"):
            self.index = faiss.read_index(f"{self.index_path}.index")
            with open(f"{self.index_path}.docs", "r", encoding="utf-8") as f:
                saved_data = json.load(f)
                self.documents = saved_data["documents"]
                self.ids = saved_data["ids"]
            print("索引已从磁盘加载")
        else:
            embeddings = [
                FAISSDataBase.embedding(openai_client, value)
                for value in dataset_values
            ]

            dimension = len(embeddings[0])
            self.index = faiss.IndexFlatL2(dimension)
            self.index.add(np.array(embeddings, dtype=np.float32))  # type: ignore

            self.ids = [f"id{i}" for i in range(1, len(dataset_keys) + 1)]
            self.documents = dataset_values

            faiss.write_index(self.index, f"{self.index_path}.index")
            with open(f"{self.index_path}.docs", "w", encoding="utf-8") as f:
                json.dump({"ids": self.ids, "documents": self.documents}, f)
            print("索引已创建并保存到磁盘")

    def query_result(self, input: str, n_results: int = 1):
        if self.index is None:
            return None

        query_embedding = np.array(
            [FAISSDataBase.embedding(openai_client, input)], dtype=np.float32
        )
        distances, indices = self.index.search(query_embedding, n_results)  # type: ignore[reportCallIssue]

        results = []
        for i, idx in enumerate(indices[0]):
            if idx < len(self.documents):
                results.append(
                    {
                        "id": self.ids[idx],
                        "document": self.documents[idx],
                        "distance": float(distances[0][i]),
                    }
                )
        return results


if __name__ == "__main__":
    database = FAISSDataBase(
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/faiss-index",
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/dataset.json",
    )

    database.create_faiss_index()
    result = database.query_result(input="登封窑陶瓷烧制技艺")
    pprint(result)

```

注意：FAISS 本身只存储向量，文档内容需要单独保存（本项目用 `.docs` JSON 文件）。

## 方案三：SQLite-vec

SQLite-vec 是 SQLite 的向量搜索扩展，适合希望用单一数据库管理结构化数据和向量的场景。

```bash
pip install sqlite-vec
```

核心实现：

```python
import json
import os
import sqlite3
import sqlite_vec
from pprint import pprint
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

openai_client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)


class SQLiteVecDatabase:
    def __init__(self, db_path: str, data_path: str):
        self.db_path = db_path
        self.source_data_path = data_path
        self.conn: sqlite3.Connection | None = None

    @staticmethod
    def embedding(client: OpenAI, input: str) -> list:
        completion = client.embeddings.create(model="text-embedding-v4", input=input)
        result = (
            json.loads(completion.model_dump_json()).get("data")[0].get("embedding")
        )
        return result

    def _connect(self):
        self.conn = sqlite3.connect(self.db_path)
        self.conn.enable_load_extension(True)
        sqlite_vec.load(self.conn)
        self.conn.enable_load_extension(False)

    def create_database(self):
        with open(self.source_data_path, "r", encoding="utf-8") as file:
            dataset = json.loads(file.read())

        dataset_values = list(dataset.values())

        self._connect()
        assert self.conn is not None

        # 获取 embedding 维度
        sample_embedding = SQLiteVecDatabase.embedding(openai_client, dataset_values[0])
        dimension = len(sample_embedding)

        # 先建表
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS documents (
                id TEXT PRIMARY KEY,
                content TEXT NOT NULL
            )
        """)

        self.conn.execute(f"""
            CREATE VIRTUAL TABLE IF NOT EXISTS vec_documents USING vec0(
                id TEXT PRIMARY KEY,
                embedding float[{dimension}]
            )
        """)

        # 检查是否已有数据
        count = self.conn.execute("SELECT COUNT(*) FROM documents").fetchone()[0]
        if count > 0:
            print("数据库中已经有数据了，跳过写入")
            return

        # 插入数据
        for i, value in enumerate(dataset_values):
            doc_id = f"id{i + 1}"
            emb = SQLiteVecDatabase.embedding(openai_client, value)

            self.conn.execute(
                "INSERT INTO documents (id, content) VALUES (?, ?)",
                (doc_id, value),
            )
            self.conn.execute(
                "INSERT INTO vec_documents (id, embedding) VALUES (?, ?)",
                (doc_id, json.dumps(emb)),
            )

        self.conn.commit()
        print(f"数据库已创建，共写入 {len(dataset_values)} 条数据")

    def query_result(self, input: str, n_results: int = 1):
        if self.conn is None:
            self._connect()

        assert self.conn is not None
        query_embedding = SQLiteVecDatabase.embedding(openai_client, input)

        rows = self.conn.execute(
            """
            SELECT
                d.id,
                d.content,
                vec_distance_cosine(v.embedding, ?) AS distance
            FROM vec_documents v
            JOIN documents d ON d.id = v.id
            ORDER BY distance
            LIMIT ?
            """,
            (json.dumps(query_embedding), n_results),
        ).fetchall()

        results = []
        for row in rows:
            results.append({
                "id": row[0],
                "document": row[1],
                "distance": row[2],
            })
        return results

    def close(self):
        if self.conn:
            self.conn.close()
            self.conn = None


if __name__ == "__main__":
    database = SQLiteVecDatabase(
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/sqlite-vec.db",
        "/Users/a1234/WorkingSpace/python/AIEngineering/data/dataset.json",
    )

    database.create_database()
    result = database.query_result(input="登封窑陶瓷烧制技艺")
    pprint(result)
    database.close()

```

## 接入大模型完成 RAG 闭环

以上三个方案都完成了「向量存储 + 语义检索」的部分。将检索结果拼入 prompt 交给大模型，就构成了完整的 RAG 链路：

```python
def build_prompt(question: str, context: str) -> str:
    return f"""使用以下上下文来回答用户的问题。如果你不知道答案，就说你不知道。总是使用中文回答。
问题: {question}
可参考的上下文：
···
{context}
···
如果给定的上下文无法让你做出回答，请回答你不知道。
有用的回答:"""

# 以 ChromaDB 为例
results = collection.query(
    query_embeddings=[embedding(openai_client, "你的问题")],
    n_results=3,
    include=["documents"],
)
context = "\n".join(results["documents"][0])
prompt = build_prompt("你的问题", context)

# 调用大模型
from src import chat  # 参见 1-how-to-use-api.py
response = chat(prompt)
```
