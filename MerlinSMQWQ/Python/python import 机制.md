---
tags:
  - python-导入机制
  - 模块与包
  - 绝对导入
  - 相对导入
  - sys-path
  - 脚本运行方式
  - 项目布局
  - 可编辑安装
  - 导入错误
  - 循环导入
---

# Python 导入机制

> 一句话总结：**相对导入只能在「包内」用；脚本入口用绝对导入；好项目通过把自己安装成包（`pip install -e .`），让 `import` 不再依赖运行目录。**

---

## 目录

1. [模块、包、`__init__.py`](#1-模块包__init__py)
2. [绝对导入 vs 相对导入](#2-绝对导入-vs-相对导入)
3. [Python 如何找模块（sys.path）](#3-python-如何找模块sys-path)
4. [`if __name__ == "__main__"` 与 `-m` 运行方式](#4-if-__name__--__main__-与--m-运行方式)
5. [常见报错及成因](#5-常见报错及成因)
6. [项目布局：flat layout vs src layout](#6-项目布局flat-layout-vs-src-layout)
7. [推荐姿势：editable install](#7-推荐姿势editable-install)
8. [速查表（Cheat Sheet）](#8-速查表cheat-sheet)

---

## 1. 模块、包、`__init__.py`

- **模块（module）**：一个 `.py` 文件就是一个模块。
  例如 `src/how_to_use_api.py` 是名为 `how_to_use_api` 的模块。
- **包（package）**：一个包含 `__init__.py` 的目录就是一个包，可以容纳模块和子包。
- `__init__.py` 可以**完全为空**，作用只是告诉 Python「这个目录是一个包」。
  - Python 3.3+ 没有它也能用（叫 namespace package），但**显式写一个更清楚、能避免一些坑**。
  - 项目**根目录**通常**不需要** `__init__.py`，根本身不是包。

```
项目根/
├── main.py            ← 入口脚本，不放进包里
├── pyproject.toml
└── src/
    ├── __init__.py    ← 有这个文件 → src 是一个包
    └── how_to_use_api.py
```

---

## 2. 绝对导入 vs 相对导入

### 绝对导入

从「项目根」开始写完整路径，明确、不依赖当前文件位置。

```python
# main.py
from src.how_to_use_api import chat
```

### 相对导入

用点号 `.` 表示包的层级。

```python
from .how_to_use_api import chat      # 同一个包
from ..other_pkg.utils import foo     # 上一层包里的另一个包
```

| 写法 | 含义 |
|------|------|
| `from .x import y` | 当前包里的 `x` 模块 |
| `from ..x import y` | 上一层包里的 `x` 模块 |
| `from ...x import y` | 上上层包，以此类推 |

### ⚠️ 关键限制

**相对导入只能用在「被作为包内模块导入」的文件里，不能用在直接当脚本运行的文件里。**

```bash
python src/foo.py    # foo.py 里若有 from .xxx → 报错
python -m src.foo    # 这样跑就 OK
```

报错信息：`ImportError: attempted relative import with no known parent package`。这是新手最常见的坑。

---

## 3. Python 如何找模块（sys.path）

执行 `import xxx` 时，Python 按顺序在 `sys.path` 列表里查找：

1. **当前脚本所在目录**（`python main.py` → `main.py` 所在目录被自动加入）
2. `PYTHONPATH` 环境变量里的目录
3. 已安装的第三方包（`site-packages`）

### 实例

```bash
cd /Users/a1234/WorkingSpace/python/AIEngineering
python main.py
```

- 项目根目录被加进 `sys.path`
- 所以 `from src.how_to_use_api import chat` 能找到 `src/` 这个包

```bash
cd /tmp
python /Users/a1234/.../main.py
```

- 加进 `sys.path` 的是 `main.py` 所在目录（项目根），所以**也能跑**
- 但如果你把 `main.py` 移到 `src/` 里，再 `python src/main.py`，加进去的就是 `src/`，那时候 `from src.xxx` 就找不到了

> **结论**：脚本运行依赖「执行目录」，不稳定。这就是为什么大项目要走 editable install（第 7 节）。

---

## 4. `if __name__ == "__main__"` 与 `-m` 运行方式

```python
# how_to_use_api.py
def chat(prompt: str) -> str:
    ...

if __name__ == "__main__":
    print(chat("你好"))
```

- 当文件被 `import` 时，`__name__` 是模块名（如 `"src.how_to_use_api"`），`if` 块**不执行**。
- 当文件被**直接运行**时，`__name__` 是 `"__main__"`，`if` 块**执行**。
- 这个模式让一个文件既能当库被 `import`，又能单独跑测试。

### 两种运行方式

| 命令 | 加入 sys.path | 相对导入能用吗 |
|------|---------------|----------------|
| `python src/foo.py` | `src/` 这个目录 | ❌ 不能，相对导入会报错 |
| `python -m src.foo` | **项目根目录** | ✅ 可以，foo 被识别为 `src` 包的成员 |

**有相对导入的文件，一律用 `-m` 跑。**

---

## 5. 常见报错及成因

### `ModuleNotFoundError: No module named 'src'`

- 当前工作目录不是项目根 → `sys.path` 里没有项目根 → 找不到 `src/`
- 修复：`cd` 到项目根再跑，或者用 editable install。

### `ImportError: attempted relative import with no known parent package`

- 直接 `python somefile.py` 运行了一个含相对导入的文件
- 修复：改用 `python -m package.somefile`；或者改成绝对导入。

### `ImportError: attempted relative import beyond top-level package`

- 相对导入点号太多，超出了顶层包
- 修复：减少 `.` 数量，或者改用绝对导入。

### 循环导入（A 导 B，B 又导 A）

- 模块层就 `import` 对方时容易触发
- 修复：把 `import` 推迟到函数内部；或者把共用代码抽出到第三个模块。

---

## 6. 项目布局：flat layout vs src layout

### flat layout（小项目常见）

```
myproject/
├── pyproject.toml
├── myproject/          ← 包目录直接在根下
│   ├── __init__.py
│   ├── api.py
│   └── db.py
└── tests/
```

### src layout（主流推荐）⭐

```
myproject/
├── pyproject.toml
├── src/
│   └── myproject/      ← 多套了一层 src/
│       ├── __init__.py
│       ├── api.py
│       └── db.py
└── tests/
```

### 为什么推荐 src layout？

- **防止误 import 到源码目录**：flat layout 里，从项目根跑 python 会优先 import 到 `myproject/` 源码，而不是已安装的版本。src layout 把源码挪到 `src/` 里，就不会被 `sys.path` 自动捡到，强制走「已安装」的路径，**和用户实际使用方式一致**。
- **测试更可靠**：测试代码必须经过「安装」才能 import，不会出现「测试能跑、发布完用户跑不起来」。

---

## 7. 推荐姿势：editable install

把项目本身当作一个包安装到当前 Python 环境，但代码改动会立刻生效（不需要重新安装）。

### 配置 `pyproject.toml`

```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    # 你的依赖
]

[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]    # src layout 用这个；flat layout 用 ["."]
```

### 安装

```bash
uv pip install -e .
# 或
pip install -e .
```

### 之后随处可用

```python
from myproject.api import chat
from myproject.db import ChromaDataBase
```

### 这样做的好处

1. **不再依赖运行目录**：无论在哪运行 Python，包都能被找到。
2. **包内部可以放心用相对导入** `from .api import chat`，未来重命名包只改 `pyproject.toml`。
3. **测试 / 外部使用方式一致**：测试代码 `from myproject.api import chat`，跟最终用户一样。
4. **`-e` 是 editable**：改完代码不用重装。

---

## 8. 速查表（Cheat Sheet）

### 文件结构记忆点

| 场景 | 怎么做 |
|------|--------|
| 一个目录想被 `import` | 在里面放 `__init__.py`（可以空） |
| 项目根目录 | **不要**放 `__init__.py` |
| 想把项目装成包用 | `src/包名/` + `pyproject.toml` + `pip install -e .` |

### 导入语法

| 写法 | 用在哪 |
|------|--------|
| `from pkg.mod import x` | 任何地方（绝对导入） |
| `from .mod import x` | 仅包内文件，且不能被当脚本直接跑 |
| `from ..pkg import x` | 仅包内文件，跨包用 |

### 运行命令

| 命令 | 什么时候用 |
|------|------------|
| `python main.py` | 入口脚本，里面只有绝对导入 |
| `python -m pkg.mod` | 要跑包里的某个模块（含相对导入也 OK） |
| `pip install -e .` | 项目装成包，import 摆脱目录依赖 |

### 排错口诀

- `ModuleNotFoundError` → 大概率是**运行目录不对**或**包没装**
- `attempted relative import` → 大概率是**当脚本跑了带相对导入的文件**，改用 `-m`
- 跨文件改名后到处坏 → 大概率是**到处写绝对路径**，没装成包

---

## 附：本项目（AIEngineering）的现状与升级路径

### 当前状态

```
AIEngineering/
├── __init__.py          ← ⚠️ 多余，可删
├── main.py
├── pyproject.toml       ← 没配置 build-system 和 packages
├── src/
│   ├── __init__.py
│   ├── how_to_use_api.py
│   ├── how_to_build_a_chromadb.py
│   └── ...
```

`main.py` 里 `from src.how_to_use_api import chat` 能跑，**但前提是从项目根目录运行**。

### 升级到 src layout（可选）

1. 把 `src/*.py` 挪进新建的 `src/aiengineering/` 目录。
2. `pyproject.toml` 加：
   ```toml
   [build-system]
   requires = ["setuptools"]
   build-backend = "setuptools.build_meta"

   [tool.setuptools.packages.find]
   where = ["src"]
   ```
3. `uv pip install -e .`
4. `main.py` 改成：
   ```python
   from aiengineering.how_to_use_api import chat
   from aiengineering.how_to_build_a_chromadb import ChromaDataBase
   ```

不升级也完全 OK——只要记住「永远从项目根运行 `python main.py`」就行。
