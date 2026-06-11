---
tags:
  - api调用
  - 环境变量
  - 项目环境管理
  - 代码重构
  - dotenv
  - dashscope集成
  - 开发工具/自动化
---

# 我们要做什么

现在市面上的大模型非常的多，作为用户，我们有权随意调用任意一个商用的模型，但是很多同学并不清楚我们是如何调用 LLM API 的，这一次我们将会学习如何通过代码调用大模型的 API。

我们选择的语言是 Python，而这一次练习使用的 API 一个是 DeepSeek 的官方 API，另一个是阿里百炼的 API，DeepSeek 最近发布了 DeepSeek-V4 系列非常令人兴奋，这也是我最常使用的一个模型，而阿里的千问则是国内产品最全的模型系列，这两个模型的调用方式足够覆盖日常生活和一般开发中的所有情况，OpenAI 格式和 Anthropic 格式是大模型 API 最常见的调用格式，这一次 Qwen 的调用会使用 OpenAI 格式，DeepSeek 的调用将使用 Anthropic 格式。

# 开始之前

好的再开始之前我们应该先安装一个非常重要的工具，之前同学们可能用过 pip 或者 pipx 对 Python 环境下的包进行管理，但是如果一个 Python 环境想用于多个项目很容易产生冲突，所以我们最好是按照项目对环境进行隔离，而 uv 就是一个这样的工具。

这里是 uv 的安装方式：[uv 的官方文档](https://uv.doczh.com/getting-started/installation)

此外值得推荐的工具也有很多，比如 pixi，如果非常喜欢 conda 生态的人一定会很喜欢它，但是这部分内容自由探索。

# 给模型打招呼

每一个模型的供应商都会提供 API 的文档，实际上我们要做的就是根据这些文档写代码，这非常的简单，甚至不需要我们动手，大家也可以用 AI 写代码，但是我的建议是好好练习，熟悉过程。

这里提供阿里百炼的文档和 DeepSeek 的文档：

- [阿里百炼](https://bailian.console.aliyun.com/cn-beijing/?spm=5176.swas-next_server-detail.0.0.4fb84ad8PiYx2a&tab=api#/api/?type=model&url=2712576)
- [DeepSeek](https://api-docs.deepseek.com/zh-cn/guides/anthropic_api)

调用 API 的前提是账户里有可用的资金，确保有资金以后再开始下一步。

现在打开阿里百炼的 API 文档，我们将使用 OpenAI-chat 的格式，这是最简单的一种调用形式：

![[Pasted image 20260512002852.png]]

然后我们就可以看到官方提供的 Python 代码示例，复制先来，粘贴到代码编辑器里面。

![[Pasted image 20260512002924.png]]

但我们得到了一个报错：

![[Pasted image 20260512003118.png]]

我们的环境中并没有安装名为 openai 的包，解决这个报错的办法很简单，我们只需要在终端使用 `uv add openai` 把这个包安装上即可。

![[Pasted image 20260512003333.png]]

报错就消失了，但是此时还是不能运行代码，我们可以使用命令 `uv run src/1-how-to-use-api.py`，就会得到这样的结果，得到了一个 OpenAIError，这个 Error 是 OpenAI 这个类报出来的，报错信息是 OpenAIError里面的文字 "Missing credentials. Please pass an `api_key`, `workload_identity`, `admin_api_key`, or set the `OPENAI_API_KEY` or `OPENAI_ADMIN_KEY` environment variable."：

![[Pasted image 20260512003746.png]]

这是说没有 API，现在来分析代码（使用 pdb 对代码进行调试，先 `tbreak 6`，然后 `c`，然后 `whatis os.getenv("DASHSCOPE_API_KEY")`，结果是 None，说明没有实际 API）：

```python
# 导入会使用的包
import os
from openai import OpenAI


client = OpenAI(
    # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx"
    api_key=os.getenv("DASHSCOPE_API_KEY"), # 获取环境变量
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

completion = client.chat.completions.create(
    # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
    model="qwen-plus",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "你是谁？"},
    ]
)
print(completion.model_dump_json())
```

那现在要做的就是配置争取的 API，我们常用的办法是使用 dotenv 这个包，可以使用 `uv add dotenv` 添加：

![[Pasted image 20260512010147.png]]

添加以后，我们创建一个新的文件，叫做 .env，把我们的 API 放到这个文件里去就好了：

![[Pasted image 20260512010333.png]]

![[Pasted image 20260512010942.png]]

接下来需要修改代码，因为使用到了 dotenv，所以要导入一下：

```python
# 导入会使用的包
import os
from openai import OpenAI
from dotenv import load_dotenv # 使用 dotenv 的 load_dotenv

load_dotenv() # 这会加载 .env 中的环境变量

client = OpenAI(
    # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx"
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

completion = client.chat.completions.create(
    # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
    model="qwen-plus",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "你是谁？"},
    ],
)
print(completion.model_dump_json())

```

得到结果：

![[Pasted image 20260512012131.png]]

同理，大家可以尝试调用 DeepSeek 的 API。

接下来我们可以对代码进行重构，封装为一个简单的函数：

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()


def chat(user_prompt: str) -> str:
    client = OpenAI(
        # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx"
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    )

    completion = client.chat.completions.create(
        # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        model="qwen-plus",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": user_prompt},
        ],
    )

    return completion.model_dump_json()


if __name__ == "__main__":
    response = chat("你好")
    print(response)
```

我们还可以更进一步，使用装饰器给这个函数加上一个日志：

```python
import logging
import time
from functools import wraps
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

logging.basicConfig(
    level=logging.INFO,
    format="{asctime} [{levelname}] {message}",
    style="{",
    force=True,
)

logger = logging.getLogger("chat")


def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logger.info(f"Calling {func.__name__} args={args} kwargs={kwargs}")
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start

        logger.info(f"{func.__name__} returned in {elapsed:.2f}s")
        return result

    return wrapper


@log_call
def chat(user_prompt: str) -> str:
    client = OpenAI(
        # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx"
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    )

    completion = client.chat.completions.create(
        # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        model="qwen-plus",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": user_prompt},
        ],
    )

    return completion.model_dump_json()


if __name__ == "__main__":
    response = chat("你好")
    print(response)

```