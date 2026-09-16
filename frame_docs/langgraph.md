# LangGraph 框架架构详解

> 源码级解读 | 面向有 LangChain 基础的开发者 | 覆盖核心原理与高级特性

源码依据以当前工作区为准：

- 主库：`E:/GithubProject/langgraph/libs/langgraph/langgraph`
- 图构建：`E:/GithubProject/langgraph/libs/langgraph/langgraph/graph`
- Pregel 执行器：`E:/GithubProject/langgraph/libs/langgraph/langgraph/pregel`
- Channel：`E:/GithubProject/langgraph/libs/langgraph/langgraph/channels`
- Checkpoint：`E:/GithubProject/langgraph/libs/checkpoint/langgraph/checkpoint`
- Prebuilt：`E:/GithubProject/langgraph/libs/prebuilt/langgraph/prebuilt`

---

## 目录

- [0. 核心本质与定位](#0-核心本质与定位)
- [1. 与 LangChain 的关系](#1-与-langchain-的关系)
- [2. 核心概念](#2-核心概念)
- [3. Graph 定义与编译](#3-graph-定义与编译)
- [4. Pregel 执行引擎](#4-pregel-执行引擎)
- [5. Channel 状态机制](#5-channel-状态机制)
- [6. 状态管理](#6-状态管理)
- [7. Checkpoint 持久化](#7-checkpoint-持久化)
- [8. Human-in-the-Loop](#8-human-in-the-loop)
- [9. 流式输出](#9-流式输出streaming)
- [10. 预构建组件](#10-预构建组件)
- [11. Send API 与 Map-Reduce](#11-send-api-与-map-reduce)
- [12. 函数式 API](#12-函数式-api)
- [13. 错误处理](#13-错误处理)
- [14. LangGraph Platform](#14-langgraph-platform)
- [15. 设计模式总结](#15-设计模式总结)

---

## 0. 核心本质与定位

### 0.1 LangGraph 是什么？

LangGraph 是 LangChain 团队开发的**有状态图编排框架**，用于构建复杂的 AI Agent 应用。

一句话理解：**LangChain 解决"怎么调用模型和工具"，LangGraph 解决"怎么把多个步骤编排成可控的工作流"。**

如果说 LangChain 是积木块（模型、工具、Prompt、Parser），那 LangGraph 就是把这些积木块拼成复杂机器人的图纸和发动机。

### 0.2 为什么需要 LangGraph？

LangChain 的 `AgentExecutor` 足以处理简单的"思考 → 工具调用 → 回答"循环，但在复杂场景下力不从心。详细对比见[第 1 章](#1-与-langchain-的关系)。

一句话总结：**LangGraph 让你用图的方式编排 Agent，而不是用 while 循环硬写。**

### 0.3 核心思想：有状态的图计算

LangGraph 的设计灵感来自 Google 2010 年发表的 **Pregel** 论文（大规模图计算模型）和 **Apache Beam**（统一批流处理模型）。

核心思想：

1. **图结构**：用节点（Node）表示计算单元，用边（Edge）表示数据流
2. **全局状态**：所有节点共享一个状态对象，通过 Channel 机制读写
3. **超步执行**：按"超步"（Superstep）迭代执行，每步并行执行所有就绪节点
4. **检查点**：每个超步结束后保存状态快照，支持恢复和时间旅行

### 0.4 拆包生态

```
libs/
├── langgraph/          ← 核心库（图定义、编译、执行引擎、Channel）
├── checkpoint/         ← Checkpoint 抽象接口（BaseCheckpointSaver）
├── checkpoint-sqlite/  ← SQLite 检查点实现
├── checkpoint-postgres/← PostgreSQL 检查点实现
├── prebuilt/           ← 预构建组件（React Agent、ToolNode）
├── sdk-py/             ← Python SDK（与 LangGraph Server 交互）
├── sdk-js/             ← JavaScript SDK
└── cli/                ← 命令行工具
```

---

## 1. 与 LangChain 的关系

### 1.1 各自的定位

```
┌──────────────┐  ┌──────────────┐
│  LangGraph   │  │   langchain  │
│  图编排引擎   │  │  Agent/Chain │
│  状态机+循环  │  │  Memory/RAG  │
└──────┬───────┘  └──────┬───────┘
       │                 │
       └────────┬────────┘
                ▼
     ┌─────────────────────┐
     │   langchain-core    │
     │  Runnable / Message  │
     │  Tool / ChatModel   │
     └─────────────────────┘
                ▼
     ┌─────────────────────┐
     │  langchain-community │
     │  partner 包          │
     │  OpenAI / FAISS ... │
     └─────────────────────┘
```

LangGraph 和 langchain 是**并列关系**，都依赖 langchain-core。LangGraph 不是 langchain 的上层，而是替代 langchain 中 Agent/Chain 的另一种编排方式。

### 1.2 LangGraph 使用 LangChain 的组件

LangGraph 重度依赖 `langchain-core` 的基础抽象：

- **消息类型**：`AIMessage`、`ToolMessage`、`SystemMessage`（来自 `langchain_core.messages`）
- **工具抽象**：`BaseTool`、`@tool` 装饰器（来自 `langchain_core.tools`）
- **模型接口**：`BaseChatModel`（来自 `langchain_core.language_models`）
- **运行时**：`RunnableConfig`、`Runnable`（来自 `langchain_core.runnables`）

### 1.3 LangGraph 的独立性

尽管与 LangChain 深度集成，核心图引擎是**完全独立**的：

- 节点可以是**任意 Python 函数**，不强制要求 LangChain Runnable
- 状态管理、Channel 系统、Checkpoint 机制都是独立实现
- Pregel 执行引擎是纯图计算框架，不依赖任何 LLM 提供商

正如官方所说：*"LangGraph is built by LangChain Inc, but can be used without LangChain."*

### 1.4 依赖关系

```
langgraph（核心）
    ├── 依赖 checkpoint（抽象接口）
    │       ├── checkpoint-sqlite（SQLite 实现）
    │       └── checkpoint-postgres（Postgres 实现）
    └── 被 prebuilt 依赖（预构建组件依赖核心库）

sdk-py ──► langgraph + cli
sdk-js（独立实现，无 Python 依赖）
```

简单说：**langgraph 是核心，checkpoint 是它的持久化层，prebuilt 是它的上层封装**。

---

## 2. 核心概念

### 2.1 StateGraph（状态图）

StateGraph 是 LangGraph 的**构建器**，用于定义图的节点、边和状态结构。

```python
from langgraph.graph import StateGraph

graph = StateGraph(MyState)
```

核心数据结构（`graph/state.py`）：

```python
class StateGraph:
    nodes: dict[str, StateNodeSpec]       # 节点名 -> 节点规格
    edges: set[tuple[str, str]]           # 普通边集合
    branches: dict[str, dict[str, BranchSpec]]  # 条件边
    channels: dict[str, BaseChannel]      # 状态通道
    schemas: dict[type, dict[str, BaseChannel]]  # schema 映射
```

### 2.2 Node（节点）

节点是图中的**计算单元**，可以是任意 Python 函数或 LangChain Runnable。

```python
def my_node(state: MyState) -> dict:
    # 读取状态，执行逻辑，返回状态更新
    return {"messages": [new_message]}
```

每个节点封装为 `StateNodeSpec`（`_node.py`），包含：runnable、metadata、input_schema、retry_policy、ends 等。

### 2.3 Edge（边）

边定义节点之间的**数据流方向**。

**普通边**：A 执行完直接到 B

```python
graph.add_edge("node_a", "node_b")
```

**条件边**：根据状态决定下一步去哪

```python
graph.add_conditional_edges("node_a", routing_function, {
    "path1": "node_b",
    "path2": "node_c",
})
```

**多源等待边**：等待多个节点全部完成后再继续

```python
graph.add_edge(["node_a", "node_b"], "node_c")  # A 和 B 都完成后才到 C
```

### 2.4 State（状态）

状态是所有节点共享的**全局数据**，通过 TypedDict 定义：

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class MyState(TypedDict):
    messages: Annotated[list, add_messages]  # 带 Reducer 的字段
    context: str                              # 普通字段（LastValue）
```

状态字段的类型注解决定了它使用哪种 Channel：

- 普通类型 → `LastValue`（只保留最后一个值）
- `Annotated[type, binary_fn]` → `BinaryOperatorAggregate`（Reducer 累积）

### 2.5 Channel（通道）

Channel 是状态的**底层存储机制**，每种 Channel 有不同的并发语义。

**为什么不能用普通 dict？**

假设两个节点并行执行，都更新 `messages` 字段：

```python
# 节点 A 返回
{"messages": [AIMessage("搜索完成")]}

# 节点 B 返回
{"messages": [AIMessage("分析完成")]}
```

如果用普通 dict 存储状态，两个节点同时写 `messages`，结果会是**后写覆盖前写**，丢失一条消息。

Channel 的作用就是定义"多个写入如何合并"：

- `LastValue` Channel：只允许一个写入，多个写入报错（适合普通字段）
- `BinaryOperatorAggregate` Channel：用 Reducer 函数合并多个写入（适合 `Annotated[list, add_messages]`）
- `Topic` Channel：收集所有写入到列表（适合 Send 的 TASKS）

Channel 还负责：版本号管理、Checkpoint 序列化、触发条件判断。

### 2.6 Reducer（归并器）

Reducer 是状态合并的策略函数。例如 `add_messages` 会把新消息追加到已有列表：

```python
def add_messages(left: list, right: list) -> list:
    # 合并逻辑：按 ID 去重，新消息追加
    ...
```

用户通过 `Annotated[list, add_messages]` 注解将 Reducer 绑定到状态字段。编译时自动映射为 `BinaryOperatorAggregate` Channel。

---

## 3. Graph 定义与编译

### 3.1 定义图

```python
from langgraph.graph import StateGraph, START, END

# 1. 创建构建器
graph = StateGraph(MyState)

# 2. 添加节点
graph.add_node("agent", call_model)
graph.add_node("tools", tool_node)

# 3. 添加边
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {
    "continue": "tools",
    "end": END,
})
graph.add_edge("tools", "agent")

# 4. 编译
app = graph.compile()
```

### 3.2 add_node 做了什么？

`add_node`（`state.py` 第 578-803 行）接受函数/Runnable 作为节点，自动从类型注解推断输入 schema。每个节点封装为 `StateNodeSpec` 数据类。

```python
# 等价写法
graph.add_node("agent", call_model)  # 函数
graph.add_node("agent", ChatOpenAI(...).bind_tools(tools))  # Runnable
graph.add_node("agent", compiled_subgraph)  # 子图
```

### 3.3 add_conditional_edges 做了什么？

`add_conditional_edges`（`state.py` 第 859-907 行）将路径函数包装为 `BranchSpec`（`_branch.py`），存储在 `self.branches[source][name]` 中。

BranchSpec 的 `_route` 方法执行路径函数得到目标节点，再通过 `writer` 将写入分发到对应的 Channel。

### 3.4 compile() 做了什么？（四阶段详解）

`compile()`（`state.py` 第 1055-1210 行）将 StateGraph 构建器转换为可执行的 `CompiledStateGraph`（继承自 Pregel）。转换分四个阶段：

**阶段一：验证与初始化（第 1101-1179 行）**

- 调用 `self.validate()` 验证所有边的起止节点都存在、图有入口点
- 创建 `CompiledStateGraph` 实例（继承自 Pregel）
- `input_channels` 设为 `START`，同时创建一个 `EphemeralValue` 类型的 START channel（写入后下一步消费即消失）
- `output_channels` 根据 output_schema 确定
- `stream_channels` 包含所有非 managed 的 channel

**阶段二：attach_node — 构建 PregelNode（第 1253-1355 行）**

对每个用户定义的节点：

1. **创建 `branch:to:{key}` 触发 channel**（第 1334-1339 行）。这是一个 `EphemeralValue`。所有指向该节点的边最终都写入这个 channel。这是**统一确定性边、条件分支和 Command.goto 的关键设计**。

2. **构建输出写入器 `write_entries`**（第 1304-1314 行）。包含两个 `ChannelWriteTupleEntry`：
   - 第一个通过 `_get_updates` 将节点返回值映射为 channel 写入
   - 第二个通过 `_control_branch` 处理 Command.goto 控制流

3. **组装 PregelNode**（第 1340-1353 行）。`triggers=[branch_channel]`，`channels=state_keys`，`writers=ChannelWrite(write_entries)`，`bound=user_func`

**阶段三：attach_edge — 连接节点（第 1357-1381 行）**

- **单起点边**：在起点节点的 writers 中追加 `ChannelWrite`，写入 `branch:to:{end}` channel。起点执行完毕后，writer 向目标节点的触发 channel 写入值，下一步触发目标节点。
- **多起点边**：创建 `NamedBarrierValue` channel，所有起点都向这个 channel 写入。只有当所有起点都写入后，barrier channel 才变为 available，从而触发目标节点。

**阶段四：attach_branch — 条件分支（第 1383-1430 行）**

为条件边创建 reader + writer 组合：
- reader 是 `partial(ChannelRead.do_read, ...)` 用于读取当前状态
- writer 是 `branch.run(get_writes, reader)` 返回的 `RunnableCallable`
- 节点执行后，writer 运行条件函数，根据返回值决定写入哪个 `branch:to:{target}` channel

编译后的核心读写模式：

```
触发 Channel (branch:to:node)
  → 读取状态 Channels
  → 执行 Runnable
  → 写回状态 Channels
```

**编译前后对比 — 用一个简单例子**：

```python
# 用户写的代码
graph = StateGraph(MyState)
graph.add_node("agent", call_model)
graph.add_node("tools", tool_node)
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {"continue": "tools", "end": END})
graph.add_edge("tools", "agent")
app = graph.compile()
```

编译后，内部变成这样：

```
用户视角：                          编译后内部结构：
                                    ┌─────────────────────────────────┐
START → agent → tools              │ Channels:                        │
                                    │   START (EphemeralValue)         │
                                    │   messages (BinaryOperatorAgg)   │
                                    │   branch:to:agent (LastValue)    │
                                    │   branch:to:tools (LastValue)    │
                                    ├─────────────────────────────────┤
                                    │ Nodes:                           │
                                    │   agent → reads: messages        │
                                    │           writes: branch:to:tools│
                                    │   tools → reads: messages        │
                                    │           writes: branch:to:agent│
                                    └─────────────────────────────────┘
```

每个节点的触发 Channel（`branch:to:node`）就像一个"门铃"——当有人按下门铃（写入这个 Channel），节点就被唤醒执行。

### 3.5 Channel 的自动生成

编译时，`_get_channels` 函数（`state.py` 第 1621-1641 行）解析 State 的类型注解，自动生成对应的 Channel：

```python
class MyState(TypedDict):
    messages: Annotated[list, add_messages]  # → BinaryOperatorAggregate（Reducer）
    context: str                              # → LastValue（默认）
    cache: Annotated[dict, CustomChannel()]   # → 直接使用自定义 Channel
```

判断逻辑（`_get_channel`，第 1656-1679 行）：

1. 若 `Annotated` 中包含 BaseChannel 实例 → 直接使用
2. 若最后一个元数据是二元函数（2 个参数）→ 创建 `BinaryOperatorAggregate`
3. 否则 → 创建 `LastValue`

---

## 4. Pregel 执行引擎（核心！）

### 4.1 Pregel 是什么？

Pregel 是 Google 2010 年提出的**大规模图计算模型**，核心思想是 **BSP（Bulk Synchronous Parallel，批量同步并行）**。

LangGraph 将 Pregel 模型应用于 Agent 编排。Pregel 类文档（`pregel/main.py` 第 354-416 行）明确定义了三阶段执行模型：

> - **Plan**：决定本步要执行哪些 Actor（节点）
> - **Execution**：并行执行所有选中的 Actor，直到全部完成或超时。此阶段 Channel 更新对 Actor 不可见
> - **Update**：用 Actor 的写入更新 Channel

重复，直到没有 Actor 被选中，或达到最大步数。

**超步执行的具体例子 — React Agent 走一遍**：

```
React Agent 图结构：
  agent 节点 → 条件边 should_continue → tools 节点（或 END）
  （should_continue 是条件边函数，不是独立节点）

超步 0：
  Plan:   检测到 START Channel 有新数据 → 触发 agent 节点
  Execute: agent 节点调用 LLM，返回 AIMessage(tool_calls=[search("天气")])
  Update:  将 AIMessage 写入 messages Channel
  → agent 完成，条件边 should_continue 判断有 tool_calls → 写入 branch:to:tools

超步 1：
  Plan:   检测到 branch:to:tools Channel 有更新 → 触发 tools 节点
  Execute: tools 节点执行 search("天气")，返回 ToolMessage("晴天 25°C")
  Update:  将 ToolMessage 写入 messages Channel
  → tools 完成，条件边判断无 tool_calls → 写入 branch:to:agent

超步 2：
  Plan:   检测到 branch:to:agent Channel 有更新 → 触发 agent 节点
  Execute: agent 看到工具结果，生成最终回答 AIMessage("今天天气晴，25°C")
  Update:  将 AIMessage 写入 messages Channel
  → agent 完成，条件边判断无 tool_calls → 写入 branch:to:END

超步 3：
  Plan:   没有节点被选中 → 执行结束
```

**关键点**：每个超步内，Channel 更新对其他节点**不可见**（直到 Update 阶段）。这保证了同一超步内并行执行的节点看到的状态是一致的。

### 4.2 PregelLoop — 超步执行循环

`PregelLoop`（`pregel/_loop.py`）是执行循环的核心，有两个子类：`SyncPregelLoop` 和 `AsyncPregelLoop`。

**初始化**（`__enter__`/`__aenter__`，第 1223-1290 行）：

- 从 Checkpointer 加载或创建 Checkpoint
- 从 Checkpoint 恢复 Channels 和 Managed Values
- 计算 step 和 stop（step + recursion_limit + 1）

**每个超步的执行流程**：

```python
# tick() — 超步开始（第 506-583 行）
1. 检查是否超过迭代限制 → out_of_steps
2. 调用 prepare_next_tasks() 准备任务
3. 匹配之前的 pending writes 到新任务
4. 检查是否需要 interrupt_before
5. 返回 True 表示继续执行

# 执行所有任务（并行）

# after_tick() — 超步结束（第 585-618 行）
1. 调用 apply_writes() 将写入合并到 Channel
2. 发射 values 输出
3. 保存 Checkpoint
4. 检查是否需要 interrupt_after
```

### 4.3 调度算法 — PUSH vs PULL

`prepare_next_tasks()`（`_algo.py` 第 382-503 行）是任务调度的核心，支持两种调度模式：

**PULL 任务（静态边）**：

- 遍历所有候选节点
- 通过版本号比较判断触发条件
- 若 Channel 有更新且可用，则触发该节点

```python
def _triggers(channels, versions, seen, null_version, proc):
    for chan in proc.triggers:
        if channels[chan].is_available() and \
           versions.get(chan, null_version) > seen.get(chan, null_version):
            return True
    return False
```

**PUSH 任务（动态 Send）**：

- 从 TASKS Channel（Topic 类型）读取所有 Send 包
- 为每个 Send 创建独立的执行任务
- 支持 Map-Reduce 并行

**什么时候用 PULL，什么时候用 PUSH？**

| 场景                                 | 使用 | 原因                               |
| ------------------------------------ | ---- | ---------------------------------- |
| `add_edge("A", "B")`               | PULL | 静态边，A 完成后触发 B             |
| `add_conditional_edges("A", func)` | PULL | 条件边，A 完成后根据 func 结果触发 |
| `Send("worker", data)`             | PUSH | 动态任务，运行时决定发多少个       |
| `create_react_agent` v2 工具调用   | PUSH | 每个 tool_call 一个 Send           |

简单说：**静态定义用 PULL，动态生成用 PUSH**。

### 4.4 版本号机制

LangGraph 用版本号实现精确的变更检测：

- `channel_versions`：Channel 的当前版本号（全局）
- `versions_seen`：每个节点上次看到的 Channel 版本号

当 `channel_versions[chan] > versions_seen[node][chan]` 时，说明 Channel 有新数据，节点应该被触发。

### 4.5 apply_writes — 写入合并

`apply_writes()`（`_algo.py` 第 230-335 行）将任务写入应用到 Channel：

1. 按 task path 排序（保证确定性）
2. 更新 `versions_seen`
3. 按 Channel 分组写入值
4. 调用每个 Channel 的 `update()` 方法
5. 通知未被更新但可用的 Channel
6. 如果是最后一步，调用 `finish()`

### 4.6 invoke() 的完整调用链（源码级）

`invoke()` 并不直接驱动执行循环，它本质上是对 `stream()` 的**薄封装**（`main.py` 第 3305-3422 行）。`invoke()` 调用 `self.stream()` 并消费所有 yield 的 chunk，最后返回最后一个 values chunk。

`stream()` 方法（`main.py` 第 2505-2822 行）是真正的入口：

```python
# stream() 的核心结构（简化）
with SyncPregelLoop(...) as loop:
    runner = PregelRunner(...)
    while loop.tick():                    # 超步前半段
        for _ in runner.tick(tasks):      # 并行执行任务
            yield from _output(stream.get())  # 流式输出
        loop.after_tick()                 # 超步后半段
```

**关键设计**：`stream()` 创建一个 `SyncQueue` 作为数据管道，把 `stream.put` 传给 `StreamProtocol`，然后传给 `PregelLoop`。所有内部节点的写入都通过 `StreamProtocol` 最终流入这个队列，外部通过 `stream.get()` 消费。

### 4.7 PregelNode 的内部结构（源码级）

PregelNode（`_read.py` 第 97-287 行）**不是 Runnable**，而是一个"容器"，用于在运行时构造 `PregelExecutableTask`。

```python
class PregelNode:
    channels: str | list[str]      # 输入来源：读哪些 Channel
    triggers: list[str]            # 触发条件：哪些 Channel 被写入时触发
    mapper: Callable | None        # 输入转换：dict → schema 类实例
    writers: ChannelWrite          # 输出写入：执行后写回哪些 Channel
    bound: Runnable                # 用户函数的 Runnable 包装
```

**`node` 属性**（第 209-222 行）：是一个 `cached_property`，将 `bound` 和 `writers` 串联为一个 `RunnableSeq`。执行顺序是**先 bound 后 writers**。

```
node = RunnableSeq(bound, *writers)
      ┌─────────┐    ┌──────────┐
      │  bound  │ →  │ writers  │
      │ 用户函数 │    │ ChannelWrite │
      └─────────┘    └──────────┘
```

### 4.8 ChannelRead / ChannelWrite 的内部实现（源码级）

**ChannelRead**（`_read.py` 第 25-91 行）：

ChannelRead **不直接访问 Channel**。它从 `config[CONF][CONFIG_KEY_READ]` 获取一个读函数，这个函数是在 `prepare_single_task`（`_algo.py` 第 715-725 行）中通过 `partial(local_read, ...)` 注入的。

`local_read`（`_algo.py` 第 186-222 行）实现了**"带写入预览"的读取**：如果 `fresh=True`，会将当前 task 已有的 writes 应用到 channel 的副本上再读取，从而让条件分支能看到当前节点刚刚写入但尚未 commit 到全局状态的值。

**ChannelWrite**（`_write.py` 第 46-170 行）：

ChannelWrite **不直接写入 Channel**。它从 `config[CONF][CONFIG_KEY_SEND]` 获取写函数（绑定到 `task.writes.extend`），将 `(channel_name, value)` 元组追加到 task 的 writes 列表中。真正应用到 Channel 发生在 `after_tick()` 的 `apply_writes()` 调用中。

**CONFIG_KEY_SEND / CONFIG_KEY_READ 的注入模式**是连接 compile 时静态结构和运行时动态行为的桥梁：
- compile 时确定 channel 结构
- prepare 时注入具体读写函数
- 节点执行时通过 config 隐式调用

### 4.9 任务执行器（源码级）

**BackgroundExecutor**（`_executor.py` 第 40-119 行）：
- 使用 `ThreadPoolExecutor`（同步）或 `asyncio` 事件循环（异步）
- `submit()` 方法接受 callable 和标志参数
- `__next_tick__=True` 时会在执行前调用 `time.sleep(0)` 让出 CPU，确保更新先于新任务被提交
- 使用 `copy_context()` 传递 Python 的 contextvars，使得 node 内部能访问 config 等上下文

**PregelRunner**（`_runner.py`）是连接 PregelLoop 和 BackgroundExecutor 的中间层：

```
PregelLoop.tick()
  → prepare_next_tasks() 得到任务列表
  → PregelRunner.tick(tasks)
      → 快速路径：单任务直接在当前线程执行
      → 批量路径：提交到线程池，wait(FIRST_COMPLETED) 等待
      → 每完成一个 task → commit() → put_writes()
  → PregelLoop.after_tick()
      → apply_writes() → 保存 checkpoint
```

### 4.10 pending_writes 机制（源码级）

`checkpoint_pending_writes` 是 `list[tuple[str, str, Any]]`（task_id, channel, value）。

**写入路径**（`_loop.py` 第 351-426 行 `put_writes`）：
1. task 完成时，runner 调用 `put_writes`
2. 对特殊 channel（ERROR, INTERRUPT）last-write-wins 去重
3. 追加到 `checkpoint_pending_writes`
4. 异步提交给 checkpointer 的 `put_writes`

**读取路径**（`_loop.py` 第 628-633 行 `_match_writes`）：
- 在 `tick()` 中被调用
- 将 pending writes 中匹配 task_id 的写入追加到 task.writes
- 跳过 ERROR/INTERRUPT/RESUME 类型

**恢复时的作用**：当从 checkpoint 恢复时，pending writes 中已完成的写入会被匹配到新任务的 writes 中，确保恢复时不会丢失已完成的工作。

### 4.11 节点执行的完整数据流（源码级）

以 `graph.invoke({"input": "hello"})` 为例，数据从输入到输出的完整路径：

```
步骤 1：输入映射
  invoke({"input": "hello"})
    → stream() → SyncPregelLoop.__enter__()
    → _first() → map_input() 将输入映射为 (START, value) writes
    → apply_writes() 写入 START Channel
    → 保存 input checkpoint

步骤 2：触发判定
  tick() → prepare_next_tasks()
    → _triggers() 检查每个节点的 trigger channel 版本
    → 版本有更新且 channel available → 该节点被触发

步骤 3：输入准备
  prepare_single_task()
    → _proc_input() 读取节点 channels 对应的所有 channel 值
    → 应用 mapper 转换
    → 注入 CONFIG_KEY_READ = partial(local_read, ...)
    → 注入 CONFIG_KEY_SEND = writes.extend

步骤 4：节点执行
  PregelRunner.tick(tasks)
    → 提交到线程池（或单任务直接执行）
    → 执行 RunnableSeq(bound, *writers)
        → bound: ChannelRead 读取状态 → 执行用户函数 → 返回结果
        → writers: ChannelWrite 将结果追加到 task.writes
    → commit() → put_writes() 写入 checkpoint_pending_writes

步骤 5：写入应用
  after_tick()
    → apply_writes() 将所有 task 的 writes 批量应用到 channels
    → 更新 channel_versions
    → 保存 checkpoint
    → 检查 interrupt_after
    → 下一轮 tick() 中，新版本号触发下游节点
```

---

## 5. Channel 状态机制

> Channel 的基本概念已在 [2.5 节](#25-channel通道)介绍。本节深入 Channel 的内部接口和七种类型。

### 5.1 BaseChannel 接口

抽象基类 `BaseChannel`（`channels/base.py`）定义了核心接口：

```python
class BaseChannel(Generic[Value, Update, Checkpoint]):
    def get(self) -> Value: ...           # 读取当前值
    def update(self, values: Sequence[Update]) -> bool: ...  # 应用更新
    def is_available(self) -> bool: ...   # 是否有值可读
    def consume(self) -> bool: ...        # 通知已消费（触发后清除）
    def finish(self) -> bool: ...         # 通知超步结束
    def checkpoint(self) -> Checkpoint: ... # 序列化
    def from_checkpoint(self, checkpoint) -> Self: ...  # 反序列化
```

### 5.2 七种 Channel 类型

| Channel                           | 文件                       | 用途                     | 使用场景                                              |
| --------------------------------- | -------------------------- | ------------------------ | ----------------------------------------------------- |
| **LastValue**               | `last_value.py`          | 默认 Channel，存储单个值 | 普通状态字段：`context: str`                        |
| **BinaryOperatorAggregate** | `binop.py`               | Reducer Channel          | 消息累积：`messages: Annotated[list, add_messages]` |
| **Topic**                   | `topic.py`               | 发布-订阅，存储多个值    | Send 的 TASKS Channel、消息收集                       |
| **AnyValue**                | `any_value.py`           | 接受任意多个相同值       | 多个节点返回相同值时取最后一个                        |
| **EphemeralValue**          | `ephemeral_value.py`     | 临时值，跨步后清除       | START 输入 Channel、一次性输入                        |
| **NamedBarrierValue**       | `named_barrier_value.py` | 名称屏障同步             | 多源等待边：`add_edge(["A", "B"], "C")`             |
| **UntrackedValue**          | `untracked_value.py`     | 不持久化到 Checkpoint    | 临时中间数据，不需要恢复                              |

**每种 Channel 的实际例子**：

```python
# LastValue（默认）— 每步只能有一个节点写入
class State(TypedDict):
    result: str  # → LastValue

# BinaryOperatorAggregate（Reducer）— 多个节点写入时自动合并
class State(TypedDict):
    messages: Annotated[list, add_messages]  # → BinaryOperatorAggregate

# Topic — 收集所有值到列表（内部用于 TASKS Channel）
# 用法：Send("node", data) 时，Send 被写入 TASKS Topic Channel

# EphemeralValue — 输入 Channel，超步结束后清除
# 用法：START Channel 就是 EphemeralValue

# NamedBarrierValue — 等待多个节点全部完成
# 用法：graph.add_edge(["node_a", "node_b"], "node_c")
# → 编译后创建 NamedBarrierValue Channel，收集 node_a 和 node_b 的完成信号
```

### 5.3 Reducer 到 Channel 的映射

当用户定义 `Annotated[list, operator.add]` 时，`_is_field_binop` 函数（`state.py` 第 1696-1714 行）检测到 `Annotated` 的最后一个元数据是二元函数，自动创建 `BinaryOperatorAggregate`。这样多个节点并发更新同一 key 时，值会通过 Reducer 累积而非冲突报错。

---

## 6. 状态管理

### 6.1 定义 State

**方式一：TypedDict（推荐）**

```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    context: str
```

**方式二：Pydantic BaseModel**

```python
from pydantic import BaseModel

class AgentState(BaseModel):
    messages: list
    context: str = ""
```

### 6.2 状态的读写

节点函数通过**参数读取状态**，通过**返回字典写入状态**：

```python
def my_node(state: AgentState) -> dict:
    # 读取：通过参数
    messages = state["messages"]

    # 写入：返回字典（只返回需要更新的字段）
    return {"messages": [new_message], "context": "updated"}
```

**重要**：节点返回的 dict 不是直接覆盖状态，而是通过对应的 Channel 合并：

- `context: str`（LastValue Channel）→ 直接覆盖
- `messages: Annotated[list, add_messages]`（BinaryOperatorAggregate Channel）→ 通过 add_messages 函数合并

这个合并发生在 `apply_writes()` 中，是超步 Update 阶段的核心操作。

### 6.3 add_messages — 最常用的 Reducer

`add_messages` 是 LangGraph 内置的消息合并 Reducer，实现了智能去重和追加：

```python
# 已有消息
existing = [HumanMessage(id="1", content="你好"), AIMessage(id="2", content="你好！")]

# 新消息（节点 A 返回）
new_a = [AIMessage(id="3", content="搜索完成")]

# 新消息（节点 B 返回）
new_b = [AIMessage(id="2", content="你好！有什么可以帮你？")]  # ID 与已有消息相同

# 合并结果
# add_messages(existing, new_a) → [Human("1"), AI("2"), AI("3")]  ← 追加
# add_messages(结果, new_b)     → [Human("1"), AI("2" 更新), AI("3")]  ← 替换（ID 匹配）
```

**合并规则**：

1. 新消息按 ID 与已有消息匹配
2. ID 匹配 → **替换**（用新消息覆盖旧消息）
3. ID 不匹配 → **追加**到末尾
4. 这保证了幂等性：同一个节点多次返回相同 ID 的消息不会重复

---

## 7. Checkpoint 持久化

### 7.1 什么是 Checkpoint？

Checkpoint 是图在某个超步结束后的**完整状态快照**。

```python
class Checkpoint(TypedDict):
    v: int                        # 格式版本
    id: str                       # UUID6 格式的唯一 ID，单调递增
    ts: str                       # ISO 8601 时间戳
    channel_values: dict[str, Any] # 通道名称到值的映射
    channel_versions: dict[str, int]  # 通道版本号
    versions_seen: dict[str, dict[str, int]]  # 每个节点看到的版本
    updated_channels: list[str] | None  # 本次更新的通道列表
```

`CheckpointMetadata` 记录来源信息：

```python
class CheckpointMetadata(TypedDict):
    source: Literal["input", "loop", "update", "fork"]  # 创建来源
    step: int           # 步数
    parents: dict[str, str]  # 父 Checkpoint ID 映射
    run_id: str         # 运行 ID
```

### 7.2 为什么需要 Checkpoint？

1. **故障恢复**：Agent 执行中断后，可以从最近的 Checkpoint 恢复
2. **人机交互**：interrupt 暂停后，状态需要持久化才能等待人类响应
3. **时间旅行**：回到任意历史状态，修改后重新执行
4. **多轮对话**：跨会话保持上下文

### 7.3 BaseCheckpointSaver 接口

`BaseCheckpointSaver`（`checkpoint/base/__init__.py`）是所有 Checkpoint 实现的抽象基类：

| 方法                                    | 作用                       |
| --------------------------------------- | -------------------------- |
| `get_tuple(config)`                   | 获取单个 Checkpoint 元组   |
| `list(config, filter, before, limit)` | 按条件列举 Checkpoints     |
| `put(config, checkpoint, metadata)`   | 存储 Checkpoint            |
| `put_writes(config, writes, task_id)` | 存储中间写入               |
| `delete_thread(thread_id)`            | 删除某个 thread 的全部数据 |
| `get_next_version(current, channel)`  | 生成通道的下一个版本号     |

### 7.4 三种 Checkpoint 存储

| 存储                    | 适用场景   | 特点                                  |
| ----------------------- | ---------- | ------------------------------------- |
| **InMemorySaver** | 开发测试   | 内存存储，进程退出即丢失              |
| **SqliteSaver**   | 轻量级生产 | SQLite 文件，单机部署，线程安全       |
| **PostgresSaver** | 生产环境   | PostgreSQL，JSONB + BYTEA，支持高并发 |

**InMemorySaver** 使用三层嵌套的 `defaultdict` 存储：`thread_id → checkpoint_ns → checkpoint_id → (bytes, metadata, parent_id)`。通道值以 blob 形式单独存储，key 为 `(thread_id, checkpoint_ns, channel, version)` 四元组。

**PostgresSaver** 建表结构：

- `checkpoints` — 主表（主键：thread_id + checkpoint_ns + checkpoint_id）
- `checkpoint_blobs` — 通道值 blob 表
- `checkpoint_writes` — 中间写入表

### 7.5 使用方式

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 执行时指定 thread_id
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke({"messages": [...]}, config=config)

# 获取状态快照
state = app.get_state(config)
print(state.values)  # 当前状态值
print(state.next)    # 待执行的节点

# 获取状态历史（时间旅行）
history = list(app.get_state_history(config))

# 修改状态
app.update_state(config, {"context": "modified"}, as_node="my_node")
```

### 7.6 Checkpoint 保存时机（源码级）

Checkpoint 在**三个时机**保存（`_loop.py`）：

| 时机 | 方法 | 说明 |
|------|------|------|
| 输入后 | `_first()` 第 824 行 | 保存 input checkpoint，记录初始状态 |
| 每步结束后 | `after_tick()` 第 610 行 | 保存 loop checkpoint，记录执行后的状态 |
| 退出时 | `_suppress_interrupt()` 第 967 行 | 仅在 durability="exit" 模式下保存 |

`_put_checkpoint` 内部调用 `create_checkpoint()` 创建新 checkpoint，然后通过 `self.submit()` 异步提交给 `_checkpointer_put_after_previous`。后者确保 checkpoint **按顺序保存**（前一个 save 完成后才执行下一个）。

**三种 Durability 模式的具体行为**：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `"sync"` | 每步后**等待** checkpoint save 完成再进入下一步 | 数据安全优先 |
| `"async"` | checkpoint save **异步**进行，不阻塞下一步（默认） | 性能优先 |
| `"exit"` | 仅在 loop **退出时**保存一次 | 临时计算、不需要中间状态 |

---

## 8. Human-in-the-Loop

### 8.1 interrupt() 的实现原理

`interrupt()`（`types.py` 第 706-829 行）是实现人机交互的核心函数。

**内部流程**：

1. 通过 `get_config()` 获取当前配置的 `CONFIG_KEY_SCRATCHPAD`
2. 使用 `scratchpad.interrupt_counter()` 跟踪中断索引
3. 检查是否有匹配的 resume 值：
   - 如果有 → 通过 `CONFIG_KEY_SEND` 发送 RESUME 写入并返回值
   - 如果没有 → 抛出 `GraphInterrupt` 异常

**关键设计**：同一个节点内的多次 `interrupt()` 调用按顺序匹配 resume 值列表，而非单个值。

**为什么需要 SCRATCHPAD？**

假设一个节点内有两次 `interrupt()`：

```python
def multi_interrupt_node(state):
    name = interrupt("请输入姓名")      # interrupt #0
    age = interrupt("请输入年龄")        # interrupt #1
    return {"name": name, "age": age}
```

SCRATCHPAD 用 `interrupt_counter` 跟踪当前是第几次 interrupt。第一次执行时 counter=0，匹配 resume 列表的第 0 个值；第二次执行时 counter=1，匹配第 1 个值。这样恢复时可以传入多个值：

```python
# 恢复时传入多个值
app.invoke(Command(resume=["张三", 25]), config)
# → interrupt #0 返回 "张三"
# → interrupt #1 返回 25
```

### 8.2 两种中断方式

LangGraph 提供两种中断方式，不要混淆：

**方式一：`interrupt()` 函数（推荐）** — 在节点代码中显式调用

```python
from langgraph.types import interrupt, Command

def approval_node(state: State):
    # 在节点内调用 interrupt()，暂停并等待人类响应
    human_response = interrupt({
        "question": "请审批以下方案：",
        "proposal": state["proposal"]
    })
    return {"approved": human_response == "approved"}

# 不需要 interrupt_before 参数
app = graph.compile()

# 执行到 interrupt() 就暂停
config = {"configurable": {"thread_id": "1"}}
app.invoke({"messages": [...]}, config=config)

# 人类审批后恢复
app.invoke(Command(resume="approved"), config=config)
```

**方式二：`interrupt_before` / `interrupt_after` 参数** — 在编译时指定哪些节点前后中断

```python
# 在 approval 节点执行前中断（不需要 interrupt() 函数）
app = graph.compile(interrupt_before=["approval"])

# 执行到 approval 节点前暂停
app.invoke({"messages": [...]}, config=config)

# 恢复（不需要 Command，直接再次 invoke）
app.invoke(None, config=config)  # 继续执行 approval 节点
```

**区别**：

- `interrupt()` 函数：节点代码主动控制中断点，可以传值给用户，恢复时用 `Command(resume=...)`
- `interrupt_before/after`：编译时声明式中断，不传值，恢复时直接 `invoke(None)`

### 8.3 interrupt_before / interrupt_after

| 模式                 | 时机                   | 实现位置                       | 用途                 |
| -------------------- | ---------------------- | ------------------------------ | -------------------- |
| `interrupt_before` | 节点执行**之前** | `tick()` 第 569-573 行       | 审批、确认、修改输入 |
| `interrupt_after`  | 节点执行**之后** | `after_tick()` 第 612-616 行 | 检查结果、人工验证   |

`PregelLoop` 的 `status` 属性：`"input" | "pending" | "done" | "interrupt_before" | "interrupt_after" | "out_of_steps"`

### 8.4 Command 恢复机制

`Command`（`types.py` 第 653-703 行）是恢复被中断图的命令对象：

```python
@dataclass
class Command(Generic[N]):
    graph: str | None = None      # 目标图（None=当前，PARENT=父图）
    update: Any | None = None     # 状态更新
    resume: Any | None = None     # resume 值（传给 interrupt() 的返回值）
    goto: str | Send | list = ()  # 跳转到指定节点
```

在 `_loop.py` 的 `_first()` 方法中，Command 被映射为写入（通过 `map_command`）。当 resume 是字典且所有 key 都是 xxh3_128 哈希值时，会被解析为 `resume_map`（用于多中断场景）。

### 8.5 Interrupt 在 Loop 中的处理（源码级）

**GraphInterrupt 的传播路径**：

```
节点函数调用 interrupt()
  → 抛出 GraphInterrupt 异常
  → PregelRunner.commit() 捕获（_runner.py 第 437-443 行）
  → 将 (INTERRUPT, value) 写入 task.writes
  → 通过 put_writes() 保存到 checkpoint_pending_writes
  → _suppress_interrupt() 抑制异常向上传播（_loop.py 第 952-1013 行）
  → 保存 durability="exit" 的 checkpoint
  → 发射最后一个 "values" 事件
  → 返回 True，异常不再向上传播
```

**恢复时的 versions_seen 更新**：

当 `is_resuming=True` 时，`_first()` 方法会设置 `checkpoint["versions_seen"][INTERRUPT]` 为当前所有 channel 的版本号。这确保了中断的节点不会因为 channel 版本未变而被跳过——它会认为 INTERRUPT channel 有新数据，从而重新触发执行。

---

## 9. 流式输出（Streaming）

### 9.1 七种流式模式

| 模式              | 说明                        | 数据内容             |
| ----------------- | --------------------------- | -------------------- |
| `"values"`      | 每步之后发射完整状态        | 完整的 State 字典    |
| `"updates"`     | 仅发射增量更新              | 节点名 → 节点返回值 |
| `"custom"`      | 节点通过 StreamWriter 发射  | 自定义数据           |
| `"messages"`    | 逐 token 发射 LLM 消息      | (message, metadata)  |
| `"checkpoints"` | Checkpoint 创建时发射       | Checkpoint 数据      |
| `"tasks"`       | 任务开始/结束时发射         | 任务事件             |
| `"debug"`       | 同时发射 checkpoint + tasks | 调试信息             |

### 9.2 使用方式

```python
# 单一模式
for chunk in app.stream(input, stream_mode="updates"):
    print(chunk)

# 多模式组合
for chunk in app.stream(input, stream_mode=["updates", "messages"]):
    print(chunk)

# 节点级自定义流式
def my_node(state, writer):
    writer("中间结果")  # 通过 StreamWriter 发射
    return {"result": "done"}
```

### 9.3 stream() 实现

`Pregel.stream()`（`pregel/main.py` 第 2505 行起）的核心流程：

1. 创建 `SyncQueue`（或 `AsyncQueue`）作为流式输出通道
2. 创建 `PregelLoop` 作为执行引擎
3. 循环调用 `loop.tick()` 和 `after_tick()`
4. 在 tick 内部通过 `loop._emit()` 向流写入事件

### 9.4 Durability 模式

控制 Checkpoint 持久化时机：

| 模式        | 说明                             |
| ----------- | -------------------------------- |
| `"sync"`  | 同步持久化，下一步开始前完成     |
| `"async"` | 异步持久化，与下一步并行（默认） |
| `"exit"`  | 仅在图退出时持久化               |

---

## 10. 预构建组件

### 10.1 create_react_agent — React Agent

`create_react_agent`（`prebuilt/chat_agent_executor.py`）是 LangGraph 最核心的预构建工厂函数。

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4"),
    tools=[search_tool, calculator_tool],
    prompt="You are a helpful assistant.",
)

result = agent.invoke({"messages": [("user", "今天天气怎么样？")]})
```

**内部图结构**：

```
START → agent（调用 LLM）
           ↓
       should_continue（路由）
           ├── 有 tool_calls → tools（执行工具）→ agent（循环）
           └── 无 tool_calls → END
```

**AgentState 定义**：

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    remaining_steps: NotRequired[RemainingSteps]  # 限制循环步数（默认 25）
```

**v1 vs v2 的区别**：

| 版本 | 工具执行方式                                  | 特点                   |
| ---- | --------------------------------------------- | ---------------------- |
| v1   | 所有 tool_calls 在同一个 ToolNode 内并行      | 简单直接               |
| v2   | 使用 Send API 将每个 tool_call 分发到独立实例 | 支持精细的人机交互控制 |

v2 还支持 `pre_model_hook`（消息裁剪/摘要）和 `post_model_hook`（guardrail/验证）两个可选钩子节点。

### 10.2 ToolNode — 工具节点

ToolNode（`prebuilt/tool_node.py`）是执行工具的核心组件。

**工作流程**：

1. 解析输入（`_parse_input`）：支持 dict、消息列表、tool_call 列表、ToolCallWithContext
2. 并行执行：使用线程池通过 `executor.map` 并行执行多个工具调用
3. 参数注入（`_inject_tool_args`）：自动注入 InjectedState、InjectedStore、ToolRuntime
4. 结果归一化（`_normalize_tool_response`）：统一处理 ToolMessage、Command、列表返回值

**错误处理**：

```python
# 捕获所有异常
tool_node = ToolNode(tools, handle_tool_errors=True)

# 自定义错误消息
tool_node = ToolNode(tools, handle_tool_errors="工具执行失败")

# 仅捕获特定异常
tool_node = ToolNode(tools, handle_tool_errors=ValueError)

# 自定义处理函数
tool_node = ToolNode(tools, handle_tool_errors=lambda e: f"错误：{e}")
```

**关键设计**：`GraphBubbleUp`（包含 `GraphInterrupt`）始终被重新抛出，不被错误处理器拦截，确保人机交互中断机制不会被吞掉。

### 10.3 多 Agent 协作

LangGraph 通过**子图（Subgraph）**机制支持多 Agent 架构：

```python
# 子图作为节点
main_graph = StateGraph(MainState)
main_graph.add_node("researcher", research_app)  # 子图
main_graph.add_node("writer", writing_app)        # 子图
```

子图拥有独立的状态通道，通过输入/输出映射与父图通信。`compile()` 支持 `checkpointer` 参数控制子图的检查点行为：`None` 继承父图、`True` 启用独立检查点、`False` 禁用。

---

## 11. Send API 与 Map-Reduce

### 11.1 Send 是什么？

`Send`（`types.py` 第 575-648 行）是 LangGraph 实现 Map-Reduce 模式的核心原语：

```python
from langgraph.types import Send

Send(node="process_item", arg={"item": item_data})
```

在执行引擎中，Send 被写入 TASKS Topic Channel，在下一个超步被 `prepare_next_tasks` 的 PUSH 分支消费。

### 11.2 Map-Reduce 模式

```python
def route_to_workers(state: State):
    """Map 阶段：将任务分发到多个工作节点"""
    return [
        Send("worker", {"task": task})
        for task in state["tasks"]
    ]

graph.add_conditional_edges("router", route_to_workers)
```

`create_react_agent` 的 v2 版本正是利用此机制将每个 tool_call 分发到独立的 ToolNode 实例。

---

## 12. 函数式 API

### 12.1 @entrypoint 装饰器

函数式 API（`func/__init__.py`）提供了更简洁的编程模型：

```python
from langgraph.func import entrypoint, task

@task
def compute(x: int) -> int:
    return x * 2

@entrypoint(checkpointer=InMemorySaver())
def workflow(inputs: list[int]):
    futures = [compute(x) for x in inputs]
    return [f.result() for f in futures]
```

`entrypoint` 将普通函数转换为 Pregel 图。可注入参数：`config`、`previous`（上一次返回值）、`runtime`。

### 12.2 entrypoint.final — 解耦返回值和保存值

```python
@entrypoint(checkpointer=checkpointer)
def workflow(input_data):
    return entrypoint.final(
        value=result,                    # 返回给调用者
        save={"history": result}         # 存入 Checkpoint
    )
```

### 12.3 函数式 vs 图式 API 对比

| 维度               | 函数式 API                  | 图式 API                                   |
| ------------------ | --------------------------- | ------------------------------------------ |
| **定义方式** | `@entrypoint` + `@task` | `StateGraph` + `add_node`/`add_edge` |
| **状态管理** | 通过 `previous` 参数      | 通过 TypedDict 状态                        |
| **并行**     | future-based 并行           | `Send` + fan-out                         |
| **流式模式** | 默认 `updates`            | 默认 `values`                            |
| **底层**     | 同样编译为 Pregel           | 编译为 Pregel                              |

---

## 13. 错误处理

### 13.1 核心错误类型

| 错误类                  | 继承               | 用途                         |
| ----------------------- | ------------------ | ---------------------------- |
| `GraphRecursionError` | `RecursionError` | 步数超过 `recursion_limit` |
| `InvalidUpdateError`  | `Exception`      | 通道更新不合法               |
| `GraphBubbleUp`       | `Exception`      | 所有图级异常的基类           |
| `GraphInterrupt`      | `GraphBubbleUp`  | 子图中断，由根图抑制         |
| `ParentCommand`       | `GraphBubbleUp`  | 向父图传递 Command           |
| `NodeTimeoutError`    | `TimeoutError`   | 节点执行超时                 |
| `EmptyChannelError`   | `Exception`      | 读取未初始化的通道           |

### 13.2 重试策略

```python
from langgraph.types import RetryPolicy

graph.add_node(
    "api_call",
    call_external_api,
    retry_policy=RetryPolicy(
        max_attempts=3,
        backoff_factor=2,
        jitter=True,
    )
)
```

---

## 14. LangGraph Platform

### 14.1 langgraph-cli

```bash
langgraph dev      # 开发模式启动
langgraph build    # 构建 Docker 镜像
langgraph up       # 本地部署（Docker Compose）
langgraph deploy   # 云端部署
```

配置文件 `langgraph.json`：

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {"my_agent": "./app/graph.py:agent"},
  "env": ".env",
  "python_version": "3.11"
}
```

### 14.2 SDK

**Python SDK**：

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:2024")
thread = await client.threads.create()
result = await client.runs.create(
    thread_id=thread["thread_id"],
    assistant_id="my_agent",
    input={"messages": [...]},
)
```

核心资源客户端：`AssistantsClient`、`ThreadsClient`、`RunsClient`、`CronClient`、`StoreClient`。

---

## 15. 设计模式总结

### 15.1 LangGraph 用到的设计模式

| 模式               | 在 LangGraph 中的体现                         | 核心价值                                         |
| ------------------ | --------------------------------------------- | ------------------------------------------------ |
| **Builder**  | StateGraph 定义图 → compile() 生成可执行对象 | 用户只管"定义什么"，不管"怎么执行"               |
| **Strategy** | 7 种 Channel 类型可替换                       | 不同字段用不同并发策略，用户通过类型注解自动选择 |
| **Memento**  | Checkpoint 保存/恢复完整状态                  | 中断、恢复、时间旅行的基石                       |
| **Command**  | Command 对象封装 resume/goto/update           | 节点不直接操作图结构，通过命令对象间接控制       |
| **Mediator** | Channel 作为节点间中介                        | 节点之间不直接通信，通过 Channel 解耦            |

### 15.2 从 Chain 到 Graph 的范式转变

```
LangChain 范式（Chain）：
  输入 → Prompt → LLM → OutputParser → 输出
  线性、单向、无状态

LangGraph 范式（Graph）：
  任意节点 → 条件路由 → 并行执行 → 状态累积 → 循环
  非线性、有状态、可恢复
```

核心转变：

1. **从链到图**：不再局限于线性流程，支持循环、分支、并行
2. **从无状态到有状态**：全局状态通过 Channel 管理，支持持久化
3. **从一次性到可恢复**：Checkpoint 机制支持中断、恢复、时间旅行
4. **从隐式到显式**：状态、路由、执行流程都是显式定义的

---

## 16. 完整端到端示例

下面是一个完整的"带人机审批的搜索助手"示例，覆盖定义 → 编译 → 执行 → 中断 → 恢复的全链路。

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph import add_messages
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import HumanMessage

# ========== 1. 定义状态 ==========
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # 消息列表（带 Reducer）
    approved: bool                            # 是否已审批

# ========== 2. 定义节点 ==========
def search_node(state: AgentState) -> dict:
    """搜索节点：模拟搜索"""
    return {"messages": [("ai", "搜索完成，找到 3 条相关结果。")]}

def approval_node(state: AgentState) -> dict:
    """审批节点：中断等待人类确认"""
    result = interrupt({                       # ← 暂停执行
        "question": "是否继续分析搜索结果？",
        "results": state["messages"][-1].content
    })
    return {"approved": result == "approve"}

def analyze_node(state: AgentState) -> dict:
    """分析节点：根据搜索结果生成分析"""
    return {"messages": [("ai", "分析完成：结论是...")]}

def should_continue(state: AgentState) -> str:
    """路由：审批通过则分析，否则结束"""
    if state.get("approved"):  # approved 由 approval_node 设置
        return "analyze"
    return END  # approved=False 或不存在时结束

# ========== 3. 构建图 ==========
graph = StateGraph(AgentState)
graph.add_node("search", search_node)
graph.add_node("approval", approval_node)
graph.add_node("analyze", analyze_node)
graph.add_edge(START, "search")
graph.add_edge("search", "approval")
graph.add_conditional_edges("approval", should_continue, {
    "analyze": "analyze",
    END: END,
})
graph.add_edge("analyze", END)

# ========== 4. 编译 ==========
checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# ========== 5. 执行（会中断在 approval 节点） ==========
config = {"configurable": {"thread_id": "user-1"}}

# 第一次执行：走到 interrupt 就暂停
result = app.invoke({"messages": [("user", "搜索 LangGraph 教程")]}, config)
print(result)
# → {"messages": [...], "approved": False}
# → 执行在 approval 节点暂停

# 查看当前状态
state = app.get_state(config)
print(state.next)      # → ("approval",)  ← 待执行的节点
print(state.tasks[0].interrupts)  # → [Interrupt(value={"question": "是否继续..."}, ...)]

# ========== 6. 恢复执行 ==========
# 人类审批后恢复
result = app.invoke(Command(resume="approve"), config)
print(result)
# → {"messages": [..., ("ai", "分析完成：...")], "approved": True}
```

**数据流全景**：

```
输入: {"messages": [("user", "搜索 LangGraph 教程")]}
  ↓
search_node → {"messages": [("ai", "搜索完成...")]}
  ↓
approval_node → interrupt() → 暂停！保存 Checkpoint
  ↓ (等待人类响应)
Command(resume="approve") → 恢复
  ↓
should_continue → "analyze"
  ↓
analyze_node → {"messages": [("ai", "分析完成...")]}
  ↓
END → 返回最终结果
```

**关键点**：

1. `interrupt()` 在 approval_node 中暂停，状态通过 Checkpoint 持久化
2. `app.get_state()` 可以查看当前状态和待执行节点
3. `Command(resume="value")` 恢复执行，`interrupt()` 返回 "value"
4. 整个过程只用了一个 `config = {"configurable": {"thread_id": "user-1"}}`

---

## 参考资料

- [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [Pregel 论文](https://kowshik.github.io/JPregel/pregel_paper.pdf)
- [LangChain 官方文档](https://python.langchain.com/docs/)
