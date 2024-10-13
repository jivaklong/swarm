![Swarm Logo](assets/logo.png)

# Swarm (experimental, educational)

一个探索人体工程学、轻量级多智能体编排的教育框架。

> [!WARNING]
>Swarm目前是一个实验性的示例框架，旨在探索多智能体系统的人机工程学界面。它不打算用于生产，因此没有官方支持。（这也意味着我们不会审查PR或问题！）
>
>Swarm的主要目标是展示[编排代理：切换和例程]中探索的切换和例程模式(https://cookbook.openai.com/examples/orchestrating_agents)烹饪书。它不是一个独立的图书馆，主要用于教育目的。

## Install

需要Python 3.10+

```shell
pip install git+ssh://git@github.com/openai/swarm.git
```

or

```shell
pip install git+https://github.com/openai/swarm.git
```

## Usage

```python
from swarm import Swarm, Agent

client = Swarm()

def transfer_to_agent_b():
    return agent_b


agent_a = Agent(
    name="Agent A",
    instructions="You are a helpful agent.",
    functions=[transfer_to_agent_b],
)

agent_b = Agent(
    name="Agent B",
    instructions="Only speak in Haikus.",
)

response = client.run(
    agent=agent_a,
    messages=[{"role": "user", "content": "I want to talk to agent B."}],
)

print(response.messages[-1]["content"])
```

```
希望之光闪闪发光，
新的路径优雅地汇聚，
我能帮什么忙？
```

## Table of Contents

- [Overview](#overview)
- [Examples](#examples)
- [Documentation](#documentation)
  - [Running Swarm](#running-swarm)
  - [Agents](#agents)
  - [Functions](#functions)
  - [Streaming](#streaming)
- [Evaluations](#evaluations)
- [Utils](#utils)

# Overview

Swarm致力于使代理**协调**和**执行**轻量级、高度可控且易于测试。

它通过两个基本抽象来实现这一点：“代理”和**切换**。“代理”包括“指令”和“工具”，可以在任何时候选择将对话交给另一个“代理”。

这些原语足够强大，可以表达工具和代理网络之间的丰富动态，使您能够构建可扩展的现实世界解决方案，同时避免陡峭的学习曲线。

> [!NOTE]
>Swarm代理与助理API中的助理无关。为了方便起见，它们的名称相似，但在其他方面完全无关。Swarm完全由聊天完成API提供支持，因此在调用之间是无状态的。

##为什么选择Swarm

Swarm探索了轻量级、可扩展和高度可定制的设计模式。类似于Swarm的方法最适合处理大量独立功能和指令的情况，这些功能和指令很难编码到单个提示中。

助理API对于寻找完整托管线程和内置内存管理和检索的开发人员来说是一个很好的选择。然而，Swarm是一个教育资源，供那些想了解多代理编排的开发人员使用。Swarm（几乎）完全在客户端上运行，与Chat Completions API非常相似，不存储调用之间的状态。

#示例

查看“/示例”以获取灵感！在README中了解更多关于每一个的信息。

- [`basic`](examples/basic): 基本原理的简单示例，如设置、函数调用、切换和上下文变量
- [`triage_agent`](examples/triage_agent): 设置基本分诊步骤以移交给正确代理的简单示例
- [`weather_agent`](examples/weather_agent): 函数调用的简单示例
- [`airline`](examples/airline): 在航空公司环境中处理不同客户服务请求的多代理设置。
- [`support_bot`](examples/support_bot): 一个客户服务机器人，包括一个用户界面代理和一个带有多种工具的帮助中心代理
- [`personal_shopper`](examples/personal_shopper): 可以帮助进行销售和退款订单的个人购物代理

# 文档

![Swarm Diagram](assets/swarm_diagram.png)

##运行Swarm

首先实例化一个Swarm客户端（在内部只实例化一个“OpenAI”客户端）。

```python
from swarm import Swarm

client = Swarm()
```

### `client.run()`

Swarm的“run（）”函数类似于chat Complementations API中的“chat.completions.create（）”功能——它接收“消息”并返回“消息”，并且在调用之间不保存状态。然而，重要的是，它还处理代理函数执行、切换、上下文变量引用，并且可以在返回给用户之前进行多次轮换。

Swarm的`client.run（）`实现了以下循环：

1.从当前代理处获取完成信息
2.执行工具调用并附加结果
3.必要时切换代理
4.必要时更新上下文变量
5.如果没有新函数调用，则返回

#### Arguments

| Argument              | Type    | Description                                                                                                                                            | Default        |
| --------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- |
| **agent**             | `Agent` | 要呼叫的（初始）代理。                                                                                                                 | (required)     |
| **messages**          | `List`  | 消息对象列表，与 [Chat Completions `messages`](https://platform.openai.com/docs/api-reference/chat/create#chat-create-messages) | (required)     |
| **context_variables** | `dict`  | 附加上下文变量的字典，可用于函数和代理指令                                                            | `{}`           |
| **max_turns**         | `int`   | 允许的最大会话回合数                                                                                                    | `float("inf")` |
| **model_override**    | `str`   | 一个可选字符串，用于覆盖代理正在使用的模型                                                                                     | `None`         |
| **execute_tools**     | `bool`  | 如果为“False”，则中断执行，并在代理尝试调用函数时立即返回“tool_calls”消息                                   | `True`         |
| **stream**            | `bool`  | 如果为“True”，则启用流式响应
                                                                                                                 | `False`        |
| **debug**             | `bool`  | 如果为True，则启用调试日志记录                                                                                                                       | `False`        |

一旦`client.run（）`完成（在可能多次调用代理和工具之后），它将返回一个包含所有相关更新状态的`Response`。具体来说，新的“消息”、要调用的最后一个“代理”和最新的“context_variables”。您可以将这些值（加上新用户消息）传递给下一次执行`client.run（）`，以继续它停止的交互——就像`chat.coompletions.create（）`一样。（`run_demo_loop`函数在`/swarm/repl/repl.py `中实现了一个完整执行循环的示例。）

#### `Response` Fields

| Field                 | Type    | Description                                                                                                                                                                                                                                                                  |
| --------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **messages**          | `List`  |对话期间生成的消息对象列表。非常类似于 [Chat Completions `messages`](https://platform.openai.com/docs/api-reference/chat/create#chat-create-messages), 但带有一个“发送者”字段，指示消息来自哪个“代理”。 |
| **agent**             | `Agent` | 处理消息的最后一个代理。                                                                                                                                                                                                                                          |
| **context_variables** | `dict`  |与输入变量相同，加上任何更改。                                                                                                                                                                                                                           |

## 代理人

“代理”只是将一组“指令”与一组“函数”（加上下面的一些附加设置）封装在一起，并能够将执行权移交给另一个“代理”。

虽然人们很容易将“代理”拟人化为“做X的人”，但它也可以用来表示由一组“指令”和“函数”定义的非常具体的工作流程或步骤（例如一组步骤、复杂的检索、数据转换的单个步骤等）。这允许将“代理”组成一个由“代理”、“工作流”和“任务”组成的网络，所有这些都由同一个原语表示。

## `Agent` Fields

| Field            | Type                     | Description                                                                   | Default                      |
| ---------------- | ------------------------ | ----------------------------------------------------------------------------- | ---------------------------- |
| **name**         | `str`                    | 代理人的姓名。                                                                 | `"Agent"`                    |
| **model**        | `str`                    | 代理人使用的模型。                                                             | `"gpt-4o"`                   |
| **instructions** | `str` or `func() -> str` | 代理的指令可以是字符串，也可以是返回字符串的可调用指令。                           | `"You are a helpful agent."` |
| **functions**    | `List`                   | 代理可以调用的函数列表。                                                        | `[]`                         |
| **tool_choice**  | `str`                    | 代理的工具选择（如果有的话）。                                                  | `None`                       |

### 使用说明

`代理的“指令”直接转换为对话的“系统”提示（作为第一条消息）。在任何给定时间，只有活动“代理”的“指令”会出现（例如，如果有“代理”切换，“系统”提示会改变，但聊天历史不会改变。）

```python
agent = Agent(
   instructions="You are a helpful agent."
)
```

“指令”可以是常规的“str”，也可以是返回“str”的函数。该函数可以选择接收一个`context_variables`参数，该参数将由传递到`client.run（）`的`context-variables`填充。

```python
def instructions(context_variables):
   user_name = context_variables["user_name"]
   return f"Help the user, {user_name}, do whatever they want."

agent = Agent(
   instructions=instructions
)
response = client.run(
   agent=agent,
   messages=[{"role":"user", "content": "Hi!"}],
   context_variables={"user_name":"John"}
)
print(response.messages[-1]["content"])
```

```
Hi John, how can I assist you today?
```

##功能

-Swarm“Agent”可以直接调用python函数。
-函数通常应返回“str”（将尝试将值转换为“str”）。
-如果函数返回“Agent”，则执行将转移到该“Agent”。
-如果一个函数定义了一个`context_variables`参数，它将由传递给`client.run（）`的`context-variables`填充。

```python
def greet(context_variables, language):
   user_name = context_variables["user_name"]
   greeting = "Hola" if language.lower() == "spanish" else "Hello"
   print(f"{greeting}, {user_name}!")
   return "Done"

agent = Agent(
   functions=[print_hello]
)

client.run(
   agent=agent,
   messages=[{"role": "user", "content": "Usa greet() por favor."}],
   context_variables={"user_name": "John"}
)
```

```
你好，约翰！
```

-如果“Agent”函数调用有错误（缺少函数、参数错误、错误），则会在聊天中附加错误响应，以便“Agent”可以正常恢复。
-如果“Agent”调用了多个函数，它们将按此顺序执行。

###切换和更新上下文变量

一个“代理”可以通过在“函数”中返回它来切换到另一个“代理人”。

```python
sales_agent = Agent(name="Sales Agent")

def transfer_to_sales():
   return sales_agent

agent = Agent(functions=[transfer_to_sales])

response = client.run(agent, [{"role":"user", "content":"Transfer me to sales."}])
print(response.agent.name)
```

```
销售代理
```

它还可以通过返回更完整的“Result”对象来更新“context_variables”。这也可以包含一个“值”和一个“代理”，以防您希望单个函数返回值、更新代理和更新上下文变量（或这三个变量的任何子集）。

```python
sales_agent = Agent(name="Sales Agent")

def talk_to_sales():
   print("Hello, World!")
   return Result(
       value="Done",
       agent=sales_agent,
       context_variables={"department": "sales"}
   )

agent = Agent(functions=[talk_to_sales])

response = client.run(
   agent=agent,
   messages=[{"role": "user", "content": "Transfer me to sales"}],
   context_variables={"user_name": "John"}
)
print(response.agent.name)
print(response.context_variables)
```

```
Sales Agent
{'department': 'sales', 'user_name': 'John'}
```

> [!NOTE]
>如果“Agent”调用多个函数来切换到“Agent”，则只会使用最后一个切换函数。

###函数模式

Swarm会自动将函数转换为JSON模式，并传递给聊天完成“工具”。

-文档字符串被转化为函数“description”。
-没有默认值的参数设置为“必需”。
-类型提示被映射到参数的“Type”（默认为“string”）。
-没有明确支持每个参数的描述，但如果只是添加到docstring中，则应该类似地工作。（将来可能会添加docstring参数解析。）

```python
def greet(name, age: int, location: str = "New York"):
   """Greets the user. Make sure to get their name and age before calling.

   Args:
      name: Name of the user.
      age: Age of the user.
      location: Best place on earth.
   """
   print(f"Hello {name}, glad you are {age} in {location}!")
```

```javascript
{
   "type": "function",
   "function": {
      "name": "greet",
      "description": "Greets the user. Make sure to get their name and age before calling.\n\nArgs:\n   name: Name of the user.\n   age: Age of the user.\n   location: Best place on earth.",
      "parameters": {
         "type": "object",
         "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"},
            "location": {"type": "string"}
         },
         "required": ["name", "age"]
      }
   }
}
```

## Streaming

```python
stream = client.run(agent, messages, stream=True)
for chunk in stream:
   print(chunk)
```

使用与[聊天完成API流式处理]相同的事件(https://platform.openai.com/docs/api-reference/streaming). 请参阅“/swarm/repl/repl.py”中的“process_and_print_streaming_response”作为示例。

新增了两种事件类型：

- `{"delim":"start"}` and `{"delim":"start"}`, to signal each time an `Agent` handles a single message (response or function call). This helps identify switches between `Agent`s.
- `{"response": Response}` will return a `Response` object at the end of a stream with the aggregated (complete) response, for convenience.

#评价

评估对任何项目都至关重要，我们鼓励开发人员带来自己的eval套件来测试他们的集群的性能。作为参考，我们在“airline”、“weather_agent”和“triage_agent”快速入门示例中有一些如何评估swarm的示例。有关更多详细信息，请参阅README。

#Utils

使用`run_demo_loop`测试你的蜂群！这将在您的命令行上运行REPL。支持流媒体。

```python
from swarm.repl import run_demo_loop
...
run_demo_loop(agent, stream=True)
```

# Core Contributors

- Ilan Bigio - [ibigio](https://github.com/ibigio)
- James Hills - [jhills20](https://github.com/jhills20)
- Shyamal Anadkat - [shyamal-anadkat](https://github.com/shyamal-anadkat)
- Charu Jaiswal - [charuj](https://github.com/charuj)
- Colin Jarvis - [colin-openai](https://github.com/colin-openai)
