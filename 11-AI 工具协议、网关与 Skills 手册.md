本手册包含 2 份源文件：E:\GithubProject\docs\general\计算机基础知识整合指南\08-AI-RAG-模型与智能体.md、E:\GithubProject\docs\general\ClaudeCode_实战指南.txt

核心不是直接让大模型读整个知识库，而是先检索相关片段。

```text
显存有限
内存有限
API 有请求限制
模型有最大输入长度
```

```text
所有文档切成 chunk
把 chunk 分成多个 batch
每个 batch 输入 embedding 模型
每个 chunk 生成一个向量
把向量存入向量数据库
继续处理下一个 batch
```

```text
第 1 批：1~32 号 chunk → 32 个向量
第 2 批：33~64 号 chunk → 32 个向量
第 3 批：65~96 号 chunk → 32 个向量
第 4 批：97~128 号 chunk → 32 个向量
第 5 批：129~130 号 chunk → 2 个向量
```

```text
130 个 chunk → 130 个向量
```

```text
embedding 模型会把整个 chunk 的语义压缩成一个固定维度向量。
向量化不是训练，是用预训练 embedding 模型做推理。
所有 chunk 不会一次性全塞进去，而是按 batch 分批处理。
```

```text
chunk_id
chunk_text
chunk_vector
```

```text
文档 ID
标题
页码
章节
来源链接
创建时间
权限信息
```

```json
{
  "chunk_id": "doc1_chunk_003",
  "text": "Redis 分布式锁用 SET NX EX 实现...",
  "vector": [0.12, -0.03, 0.88],
  "metadata": {
    "doc_id": "doc1",
    "page": 5,
    "title": "Redis 锁机制"
  }
}
```

流程是：

```text
用户问题
→ 用同一个 embedding 模型生成 query 向量
→ 去向量数据库里找相似 chunk 向量
→ 取回对应 chunk 原文
→ 把 chunk 原文塞进大模型上下文
→ 大模型基于这些内容回答
```

```text
向量库里存的是"向量 + 原文 chunk + 元数据"，不是只有向量。
RAG 是先用向量检索找资料，再让大模型基于资料回答。
```

#### 所有 chunk 是不是一次性放进 embedding 模型

不是一次性全塞进去，而是**分批 batch 处理**。

原因是：

正确流程是：

例如有 130 个 chunk，`batch_size=32`：

最终：

一句话：

---

#### 向量库里到底存什么

不是只存向量。

向量数据库通常至少存三类东西：

还会存一些元数据：

例如一条记录可能是：

检索时先用向量找相似 chunk，再把原文 chunk 取出来交给大模型回答。

#### RAG 检索时是怎么用这些向量的

一句话：

---

### 3. RRF 融合排序、去重和 Rerank 的完整逻辑是什么？为什么很多教程把去重位置写错了？

#### 为什么 RRF 融合前不能去重

```text
同一个文档被多个检索器都排在前面，说明它更可能是好结果。
```

```text
A 和 B 失去来自 BM25 的加分
多路召回的交叉验证信号被抹掉
最终排序失真
```

```text
收集所有召回结果
按文档 ID 分组
同一个 ID 的不同排名贡献分数累加
最后每个 ID 只输出一次
```

```text
A 在向量检索第 1 名
A 在 BM25 第 2 名
```

```text
A 的总分 = 向量第 1 名贡献 + BM25 第 2 名贡献
```

```text
分块时有 overlap 重叠
同一文档被重复上传
不同文档中有相同段落
模板化内容太多
```

```text
chunk A：Redis 分布式锁使用 SET NX EX 实现...
chunk B：Redis 分布式锁一般使用 SET NX EX 命令实现...
```

```text
RRF 融合后
计算 chunk 之间的相似度
相似度超过阈值，比如 0.85~0.9
只保留排名更高的那个
```

```text
用户问题
→ 向量检索召回 TopK
→ BM25 关键词召回 TopK
→ 其他召回通道召回 TopK
→ RRF 融合排序
→ 语义去重
→ Rerank 重排
→ 取 TopN chunk
→ 放入大模型上下文
→ 生成回答
```

```text
ID 去重不应该放在 RRF 前。
语义去重可以放在 RRF 后、Rerank 前。
```

```text
RRF 前要保留多路重复出现的信息。
RRF 后再处理语义重复，更合理。
```

| 向量检索 | BM25 检索 |
| -------- | --------- |
| 1. A     | 1. B      |
| 2. B     | 2. A      |
| 3. C     | 3. D      |

**合并后的问题：**
多路召回时，比如向量检索和 BM25 都返回了一些结果，为什么不能先去重再做 RRF 融合？如果多个检索器返回同一个文档，RRF 最后会不会重复出现？既然 RRF 自带 ID 聚合，那 RAG 里还需要去重吗？如果需要，去什么重？如果有向量检索、BM25、RRF、去重、rerank，正确顺序是什么？为什么很多 RAG 教程会写"召回后先去重再融合"？这个错在哪里？

**答案：**

RRF 的核心思想是：

RRF 会把同一个文档在多个召回列表里的排名分数累加。

比如：

正确情况下，A 和 B 都被两个检索器召回，而且排名都靠前，所以它们应该得高分。

如果先去重，把 BM25 里的 A、B 删除了，就会导致：

也就是说，先去重会把"多个检索器共同认可某个结果"这个重要信号删掉。

#### RRF 本身是不是已经做了去重

标准 RRF 本身就是按文档 ID 聚合分数的。

它的逻辑是：

所以 RRF 输出结果天然就是去重后的。

例如文档 A 同时出现在向量检索和 BM25 中：

RRF 会计算：

最终 A 只出现一次，但得分更高。

#### RAG 里真正需要去重的是什么

需要，但主要不是 ID 去重，而是**语义去重**。

因为很多 chunk 虽然 ID 不同，但内容几乎一样。

常见原因：

例如：

它们 ID 不同，但语义高度重复。

正确做法：

#### 正确的 RAG 多路召回流程

正确顺序应该是：

关键点：

原因是：

## AI 工具协议、网关、OpenAI API 与 Skills

### 10.1 MCP协议

   启动流程：
   客户端进程                    MCP服务器进程
       │                              │
       │  fork + exec                 │
       │  ─────────────────────────→  │
       │                              │
       │  stdin ──→ JSON-RPC请求       │
       │  ─────────────────────────→  │
       │                              │
       │  ←── stdout ── JSON-RPC响应   │
       │  ←─────────────────────────  │

```
问题：不同AI模型、不同客户端的工具调用方式不同
  - Claude有自己的工具调用格式
  - OpenAI有自己的Function Calling格式
  - 每个客户端都要适配不同的格式

```
Function Calling：
  - 模型内部的函数调用能力
  - 每个模型有自己的格式
  - 需要在API请求中定义函数

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ AI客户端     │────→│ MCP服务器    │────→│ 工具/资源    │
│ (Claude等)  │     │ (stdio/SSE) │     │ (文件/API)  │
└─────────────┘     └─────────────┘     └─────────────┘

```
MCP支持两种传输方式：

```
请求消息：
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/tmp/test.txt"
    }
  }
}

```
1. 客户端请求工具列表：
   {
     "jsonrpc": "2.0",
     "id": 2,
     "method": "tools/list"
   }

错误响应：
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params"
  }
}
```

#### MCP是什么？

**问题**：MCP 服务是什么？MCP 和 Function Calling 有什么区别？

**回答**：

**是什么**：
MCP（Model Context Protocol）是AI模型与工具交互的协议标准，由Anthropic推出。

**为什么需要**：

MCP解决：
  - 统一的工具协议标准
  - 一次开发，多端使用
  - 工具可以在不同AI模型之间共享
```

**与Function Calling的区别**：

MCP：
  - 跨模型、跨客户端的工具协议
  - 工具独立于模型存在
  - 客户端通过MCP协议连接工具
  - 模型只需要知道工具的功能描述
```

**MCP架构**：

MCP服务器提供：
  - Tools：可调用的函数
  - Resources：可读取的资源
  - Prompts：预定义的提示词模板
```

**MCP协议的通信机制**

1. stdio（标准输入/输出）
   - MCP服务器作为子进程启动
   - 客户端通过stdin发送请求
   - 服务器通过stdout返回响应

2. SSE（Server-Sent Events）
   - MCP服务器作为HTTP服务运行
   - 客户端通过HTTP POST发送请求
   - 服务器通过SSE推送事件

   通信流程：
   客户端                        MCP服务器
       │                              │
       │  HTTP POST /message          │
       │  ─────────────────────────→  │
       │                              │
       │  HTTP 200 + SSE stream       │
       │  ←─────────────────────────  │
       │                              │
       │  事件: tool_call             │
       │  ←─────────────────────────  │
```

**MCP消息格式（JSON-RPC 2.0）**

响应消息：
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "文件内容..."
      }
    ]
  }
}

**工具调用流程**

2. 服务器返回工具定义：
   {
     "jsonrpc": "2.0",
     "id": 2,
     "result": {
       "tools": [
         {
           "name": "read_file",
           "description": "读取文件内容",
           "inputSchema": {
             "type": "object",
             "properties": {
               "path": {
                 "type": "string",
                 "description": "文件路径"
               }
             },
             "required": ["path"]
           }
         }
       ]
     }
   }

3. AI模型决定调用工具，客户端发送调用请求：
   {
     "jsonrpc": "2.0",
     "id": 3,
     "method": "tools/call",
     "params": {
       "name": "read_file",
       "arguments": {
         "path": "/tmp/test.txt"
       }
     }
   }

4. 服务器执行工具并返回结果：
   {
     "jsonrpc": "2.0",
     "id": 3,
     "result": {
       "content": [
         {
           "type": "text",
           "text": "Hello, World!"
         }
       ]
     }
   }
```

---

---

---

Claude Code 实战指南

目录：

1. Slash Commands 基础与安装
2. CLAUDE.md 与 Memory 系统
3. Checkpoints 与 CLI 使用
4. Skills + Hooks 自动化
5. MCP + Subagents 集成
6. Agent Orchestration
7. REPL 与管道基础

---

# 1. Slash Commands 基础与安装

- 创建目录 `.claude/commands/`
- 将 Markdown 文件复制进去，例如：
  ```bash
  cp 01-slash-commands/optimize.md .claude/commands/
  ```

- PowerShell 中没有 `cp`，可使用 `Copy-Item`。
- 命令必须在 Claude Code 打开的项目根目录下，否则不会加载。
- 文件名就是 slash command 命令名，文件内容是 prompt 模板。
- 文件内可以使用 `$ARGUMENTS` 或 `$0/$1` 接收参数。
- 可通过 `!` 执行 shell 命令获取动态上下文，Claude 自动执行，无需手动运行。

```md
---
name: commit
description: 使用上下文创建 git commit
allowed-tools: Bash(git *)
---

注意事项：

Slash command 是 Claude Code 中通过 Markdown 文件定义的可复用任务模板。文件名即命令名，例如 `.claude/commands/optimize.md` 对应 `/optimize` 命令。安装方法：

举例：

## 上下文
- 当前 git 状态：!`git status`
- 当前 diff：!`git diff HEAD`
- 当前分支：!`git branch --show-current`
- 最近提交：!`git log --oneline -5`

## 任务
根据以上生成 commit message
```

Slash command 本质是 prompt function + 任务模板，Claude 解析 Markdown 文件后自动生成结构化任务，并可基于当前项目环境自动执行 shell 命令或生成提示词。

---

# 2. CLAUDE.md 与 Memory 系统

核心理解：

- `CLAUDE.md`：手动编写的项目规则文件，包括编码规范、架构说明、团队工作流等。是持久化规则，加载到 Claude 会话上下文。
- `/memory`：查看和编辑当前 session 能感知的记忆，包括项目级和用户级 memory。
- `#` 前缀：快速告诉 Claude 记住一条规则，例如

```
# = 快速记忆
CLAUDE.md = 正式项目规则
/memory = 查看和编辑当前记忆体系
```

  ```text
  # 每次提交前都运行 npm test
  ```

  这条规则会进入当前会话记忆或 auto memory，但不会自动写入 `CLAUDE.md`。
- Auto Memory：Claude 根据你的偏好和有用信息自动记录在项目的 MEMORY.md（例如 `~/.claude/projects/<project>/memory/MEMORY.md`），不会覆盖 CLAUDE.md。
- 用户偏好可写入 `~/.claude/CLAUDE.md`。

---

# 3. Checkpoints 与 CLI 使用

使用方法：

- Esc+Esc 或 `/rewind` 进入 checkpoint 选择界面
- 可恢复代码、对话或两者
- 交互模式：`claude "explain this project"`，进入 REPL 会话，可持续提问
- 输出模式（Print mode）：`claude -p "query"`，一次性输出结果后退出
- 管道：`cat file | claude -p "query"`，将文件内容通过 pipe 输入给 Claude，一次性分析

- Checkpoint 不替代 Git，删除文件后 `/rewind` 可能无法恢复
- 输出模式适合一次性分析或日志处理
- 管道在 PowerShell 中可用 `Get-Content file | claude -p "query"`

注意事项：

Checkpoint 是 Claude Code 提供的本地撤销点功能，只跟踪通过 Claude 编辑器修改的文件。系统命令（rm、mv、cp）修改的文件不在 checkpoint 范围。

---

# 4. Skills + Hooks 自动化

区别：

- Skill：Claude 可调用的重复任务模板，可配置 YAML frontmatter（name、description、effort、shell、allowed-tools）
- Hook：事件驱动自动化，可在工具执行前/后或会话结束时自动触发命令/HTTP/prompt/agent
- 常用 hook 事件：`PreToolUse`、`PostToolUse`、`Stop` 等
- 示例安装 Skill：
  ```powershell
  New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills"
  Copy-Item -Recurse ".-skills\code-review" "$env:USERPROFILE\.claude\skills\code-review"
  ```
- 配置 Hook（PowerShell 示例）
  ```json
  {
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hooks": [
            {
              "type": "command",
              "shell": "powershell",
              "command": "& "$HOME\.claude\hooks\pre-tool-check.ps1""
            }
          ]
        }
      ]
    }
  }
  ```

| 机制  | 功能           | 举例                                              |
| ----- | -------------- | ------------------------------------------------- |
| Skill | 可复用任务能力 | /code-review                                      |
| Hook  | 事件自动化     | PreToolUse 拦截 Bash 命令，PostToolUse 自动跑测试 |

---

# 5. MCP + Subagents 集成

- MCP（Model Context Protocol）：连接外部系统，如 GitHub、数据库、Slack
- Subagents：专门处理任务的 AI 子代理，有独立 context 和权限
- Hooks：事件触发自动化控制

```powershell
$env:GITHUB_TOKEN="your_token"
claude mcp add github --transport http https://api.githubcopilot.com/mcp/ --header "Authorization: Bearer $env:GITHUB_TOKEN"
```

```text
/mcp__github__list_prs
```

```powershell
New-Item -ItemType Directory -Force .\.claudegents
Copy-Item .-subagents\*.md .\.claudegents```
4. 完整工作流
```text
获取 GitHub PR → 交给 code-reviewer subagent → hook 自动运行测试
```

实操步骤：

1. 配置 GitHub MCP

2. 测试 MCP

3. 安装 Subagents

成功标准：能获取 PR 数据、Subagent 执行任务、Hook 自动触发

---

# 6. Agent Orchestration

区别：

- AGENTS.md：顶层调度规则文件，告诉 Claude 在什么任务下调用哪个 subagent
- 作用：Routing Policy，模拟多角色协作
- Subagent：独立运行、独立上下文、独立工具权限
- 例子：
  - 复杂功能 → planner
  - 新功能/bug → tdd-guide
  - 代码修改 → code-reviewer
  - 架构决策 → architect
- 多视角分析：事实校验、架构专家、安全专家、冗余检查
- 并行任务：同类任务可同时执行，提高效率

| 层            | 功能              |
| ------------- | ----------------- |
| CLAUDE.md     | 项目规则、规范    |
| Slash command | 可调用任务模板    |
| AGENTS.md     | 子 agent 调度策略 |

---

# 7. REPL 与管道基础

- REPL：Read-Eval-Print Loop，交互式会话环境
  - Read：读取输入
  - Eval：执行
  - Print：输出结果
  - Loop：循环等待下一条输入
- 管道（Pipe `|`）：把前一个命令输出作为后一个命令输入

```bash
cat file | claude -p "query"
# 等于将 file 内容一次性喂给 Claude 处理并输出结果
```

在 PowerShell 中可用 `Get-Content file | claude -p "query"`

---

文档整理完毕。
