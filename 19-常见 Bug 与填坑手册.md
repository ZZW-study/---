本手册包含 2 份源文件：E:\GithubProject\docs\general\计算机基础知识整合指南\总是出现的bug.md、E:\GithubProject\docs\general\计算机基础知识整合指南\问题回答文档.txt

# 一、Clash 代理：内外网访问

## 1.1 问题与方案

### 问题根源

```
勾上 → 所有流量都走代理（国内站也变慢或打不开）
不勾 → 所有流量都直连（国外站访问不了）
```

在 Windows 的"局域网(LAN)设置"里手动填了代理（一般是 `127.0.0.1:7890`）。这种"硬编码"方式会强制所有 HTTP/HTTPS 流量都走代理，绕过了 Clash 自己的"规则分流"引擎。

走代理就是走本地回环地址：想发什么东西，就直接发给本机的 clash，clash 帮你去发；数据回来先给 clash，clash 再给你。

### 推荐方案：让 Clash GUI 接管系统代理

- 国内 IP/域名 → DIRECT（直连）
- 国外 IP/域名（如 gpt、google）→ 走代理

迁移完的 ZBot 协议对照(再贴一次):

```text
先找到所有 a = 1
再逐条判断 b 是否等于 2
```

```text
在联合索引中直接定位到 (a=1, b=2) 这个范围
然后只扫描这个范围内的数据
```

```text
a  b  c
1  1  1
1  1  2
1  1  3
1  2  1
1  2  2
1  2  3
1  3  1
1  3  2
2  1  1
2  2  1
```

```sql
where a = 1 and b = 2
```

```text
1  2  1
1  2  2
1  2  3
```

```text
1  1  1
1  1  2
1  1  3
1  2  1
1  2  2
1  2  3
1  3  1
1  3  2
```

```sql
where a = 1 and b = 2
```

```sql
where a = 1 and c = 3
```

```text
a -> b -> c
```

```text
a  b  c
1  1  3
1  2  1
1  2  3
1  3  2
1  4  3
```

```sql
where a = 1
```

```sql
where a = 1 and b = 2
```

```sql
where a = 1 and b = 2 and c = 3
```

```sql
where a = 1 and c = 3
```

```sql
where b = 2
```

```text
先按 a 排序
a 相同，再按 b 排序
b 相同，再按 c 排序
```

```text
a
a,b
a,b,c
```

```text
a,c
b,c
c
```

```sql
index(a, b, c)
```

```text
(a, b, c, 主键id)
```

```text
(1,1,1,id=101)
(1,1,2,id=102)
(1,2,1,id=103)
(1,2,2,id=104)
(1,3,1,id=105)
(2,1,1,id=106)
(2,2,1,id=107)
```

```text
                              根节点
                    [(1,3,1)        (2,2,1)]
                     /        |              \
                    /         |               \

```sql
where a = 1 and b = 2
```

```text
(1,2,最小值)
```

```text
根节点: [(1,3,1)  (2,2,1)]
目标:   (1,2,最小值)
```

```text
先比 a：1 = 1
再比 b：2 < 3
```

```text
走向：内部节点A
```

```text
内部节点A: [(1,1,3)  (1,2,3)]
目标:      (1,2,最小值)
```

```text
a：1 = 1
b：2 > 1
```

```text
a：1 = 1
b：2 = 2
c：最小值 < 3
```

```text
走向：叶子2
```

```text
叶子2: (1,2,1)  (1,2,2)  (1,2,3)
```

```text
a = 1 且 b = 2
```

```text
(1,2,1)  满足
(1,2,2)  满足
(1,2,3)  满足
```

```text
(1,3,1)
```

```text
b = 3，不是 2
```

```text
where a=1 and b=2
```

```text
(1,2,最小值)
```

```text
先比 a
a 相等再比 b
b 相等再比 c
```

```python
def build_list(nums: list[int]) -> ListNode | None:
    ...
```

```text
1. 创建一个函数对象 build_list
2. 把函数名 build_list 放到当前模块的全局命名空间里
3. 记录参数名 nums、函数体代码、类型注解等信息
```

```python
nums
```

```python
build_list([1, 2, 3])
```

```text
nums  ->  [1, 2, 3]
```

```python
def build_list(nums: list[int]) -> ListNode | None:
```

```python
nums: list[int]
```

```python
build_list.__annotations__
```

```python
{
    "nums": list[int],
    "return": ListNode | None
}
```

```text
nums 这个参数变量：调用函数时才绑定
list[int]、ListNode | None 这些类型注解：定义函数时可能会被解析/记录
```

不要在 Windows 设置里手填代理，改成让客户端自动接管：

> MySQL 可以利用联合索引 `(a,b,c)` 直接定位到 `a=1 且 b=2` 的那一段索引范围。

> `import` 文件时会执行顶层的 `def`，创建 `build_list` 函数对象；但 `nums` 不会被创建。只有调用 `build_list(...)` 时，`nums` 才作为局部变量绑定到传进来的参数对象。

> **redo log 是为了把“随机写数据页”变成“顺序写日志”，先快速保证事务安全。**

1. 打开你的 Clash 桌面客户端（Clash for Windows / Clash Verge / mihomo-party / Clash Meta 等任选）
2. 把模式切到 **Rule（规则）**——关键，不要用 Global 或 Direct
3. 找到客户端里的 **"System Proxy" 按钮**，点亮它
4. Windows 设置里那个"为 LAN 使用代理服务器"**保持未勾选**

这样 Clash 会自动把系统代理设成 `127.0.0.1:7890`，并按规则分流：


---

1. Claude Code
传输层:纯 SSE,完全不用 WebSocket。

证据:

Claude Code CLI 调的是 Anthropic Messages API(POST /v1/messages with stream: true),响应是 text/event-stream。这是 Anthropic API 服务端原生支持的 — 它不提供 WebSocket 入口。
CLI 内部用 @anthropic-ai/sdk 的 Messages.stream() 拿到 SSE 流,AsyncIterator 消费事件,事件类型固定为 Anthropic 的标准集:
message_start / content_block_start / content_block_delta / content_block_stop / message_delta / message_stop / ping / 错误事件
取消机制:AbortController.abort() — 直接断开底层 fetch。Anthropic 服务端会读 connection close 信号,优雅停机。
会话状态:完全无状态。每条消息要带完整 messages: [...] 数组,服务端不存。
重连:没有。要重发就重新 stream() 整段,带上历史 messages。
多端:每个端独立持有 context,服务端不协同。
为什么不用 WS:

Anthropic API 整体设计就是 HTTP+SSE,WebSocket 在 Anthropic 的产品矩阵里不存在
SSE 能复用 HTTP 的所有基础设施(CORS、auth header、proxy、CDN、curl 调试、postman、看 devtools)
Agent CLI 是短到中长连接(一次对话几秒到几分钟),SSE 浏览器自动重连 + AbortController 取消就够用
WS 双向通信的优势(client → server 推指令)在 agent CLI 场景用不到 — Claude Code 是"用户发一次消息,流式收完整段",中间不需要 client 主动推指令
2. Codex (OpenAI)
传输层:也是纯 SSE + 本地 JSONL 落盘,不用 WebSocket。

证据:

Codex CLI 跟 server(本地 codex exec 子进程或者云端)之间的事件 schema 在 codex-rs/protocol/src/protocol.rs 定义。你 ZBot 的 backend/schemas/agent.py:220-230 的 envelope 就是直接抄的 Codex:

RunEventType = Literal["session_meta", "turn_context", "event_msg", "response_item"]
Codex 的实际传输是:
本地:codex exec --json 把事件序列化成 JSONL(每行一个 JSON 对象)写到 stdout,前端(TUI / 别的 shell)读 stdin/stdout。
服务端云模式(Codex Cloud / Web):走 HTTP SSE,POST /threads/{id}/turns + stream 字段,响应 SSE 流。
Codex 还有 rollout JSONL 文件落盘(~/.codex/sessions/.../*.jsonl)— 这是它独有的设计:所有事件同时写到磁盘,这样断线 / 重启 / 别的端可以回放。
取消:POST /threads/{id}/turns/{turn_id}/cancel — REST,不是 WS 帧。
会话:thread_id + turn_id 是服务端一等资源,断线可重连、其它端可 resume。
WebSocket 在 Codex 整个产品里不存在。
为什么不用 WS:

Codex 设计目标之一是可重放、可调试 — JSONL 落盘 + SSE 都支持用 curl / cat 离线读
Codex Cloud 服务端直接用 OpenAI 的 Responses API(/responses + stream: true),底层就是 SSE
Codex 想做"任何前端都能消费"(TUI、IDE plugin、Web UI),用 SSE + JSON envelope 不绑死传输,后续能改 gRPC、QUIC、管道都行
3. 直接对比(都用 SSE 的原因)
维度	WS	SSE(主流选择)
方向	双向	服务器→客户端(单向)
协议	独立 upgrade	普通 HTTP,response text/event-stream
调试	wscat,devtools 难	curl -N 就能看,devtools 网络面板友好
代理/防火墙	常被中间件拦(尤其是 corporate proxy)	走 HTTP,几乎不会被拦
取消	客户端发关闭帧 + 服务端读 disconnect	AbortController.abort() + 客户端 close,服务端读 disconnect
服务端推送多个并发	一条连接多流复杂(per-channel multiplexing)	一个 HTTP 连接就是一个流,简单
浏览器自动重连	❌ 自己写	✅ EventSource 原生
鉴权	子协议头 / URL token	标准 Authorization header
工具链	专用	复用 HTTP — 任何 fetch 库都能干
WS 真正胜过 SSE 的场景:

双向高频消息(游戏、协同编辑、实时聊天)— client 要主动推指令
超低延迟(金融撮合、IoT)
需要服务端主动给 client 推任意消息,不局限于"事件流"
Agent CLI / 长 LLM 调用场景:这三条全部不满足:

Client 只在开始时发一条消息,之后都是被动收
LLM token 生成延迟是几百毫秒级,不是毫秒级
服务端推的就是事件流,SSE 原生语义
所以 Claude Code 和 Codex 都选 SSE,不是巧合,是 agent CLI 场景的最优解。

4. 你 ZBot 的现状:跟主流一致
你 backend/handlers/agent_sse.py 里的事件翻译表,头几行就写了:


# 事件类型映射(旧 -> 新):
#   run.started           -> (忽略,session_meta 已经覆盖)
#   turn.started          -> event_msg / task_started
#   turn.completed        -> (忽略,task_complete 覆盖)
#   run.completed         -> event_msg / task_complete {status: completed}
event_msg / response_item / session_meta / task_started / task_complete — 这就是 Codex rollout JSONL 的事件名。说明你的后端设计者本来就以 Codex 为参考目标。WS 那一套是历史包袱,这次切干净完全合理。



# 二、Agent 事件类型映射（旧 -> 新 翻译表）

WS(已删)	SSE(现在)	Claude Code	Codex
传输	双向帧	HTTP+SSE	HTTP+SSE	HTTP+SSE / JSONL
启动	ws.send({type:'run.start',...})	POST /runs	POST /messages?stream=true	POST /threads/.../turns?stream=true
取消	ws.send({type:'run.cancel'})	DELETE /runs/{id}	AbortController.abort()	POST /turns/{id}/cancel
事件	扁平 AgentEvent	envelope session_meta/event_msg/response_item	message_start/content_block_delta/...	session_meta/event_msg/response_item(你的 schema 抄的就是它)
错误	run.failed 事件	event_msg/type=error	error event	event_msg/type=error
会话	闭包内 state	thread/turn 一等资源(run_id 走 URL)	stateless(每次带 messages)	thread/turn 一等资源
一句话回答你
对,Claude Code 和 Codex 都是纯 SSE,不用 WebSocket。 不是因为他们穷/懒,是因为 agent CLI 场景 SSE 在调试、代理、鉴权、取消、自动重连、工具链六个维度都比 WS 强,而 WS 唯一优势"双向高频"在 agent CLI 用不上。

一句话本质区别
WebSocket	SSE
方向	全双工(双向)	服务器 → 客户端 单向
协议	HTTP 升级(Upgrade: websocket)	普通 HTTP,response 头 Content-Type: text/event-stream
帧	文本/二进制 frame	UTF-8 文本,每条以 \n\n 结尾
浏览器自动重连	❌ 自己写	✅ EventSource 原生
多路复用	一条连接多 channel(复杂)	一条连接 = 一个流(简单)
工具链	wscat、专用客户端	curl -N、任何 HTTP 客户端
协议机制差异(为什么不是同一个东西)
WebSocket
客户端先发 HTTP 请求,带 Upgrade: websocket + Connection: Upgrade + Sec-WebSocket-Key
服务端返回 101 Switching Protocols,从此连接脱离 HTTP,变成二进制帧通道
服务端和客户端平等,都可以随时发消息
关闭要发 close 帧(0x8),双方都要 ack
帧结构:1 bit FIN + 1 bit RSV + 4 bit opcode + 1 bit mask + 7 bit payload length + masking key + payload
SSE
客户端发普通 GET,服务端响应 200 OK + Content-Type: text/event-stream + Cache-Control: no-cache
连接保持打开,服务端持续写:

id: 1
event: message
data: {"text": "hello"}

id: 2
event: delta
data: {"token": "world"}

客户端只能读,不能写
关闭 = 关 TCP 连接(浏览器自动重连)
注释行以 : 开头(用来发心跳 / keep-alive)
详细对比表
维度	WebSocket	SSE	谁赢
方向	全双工	单向(server→client)	看场景
协议类型	独立(从 HTTP 升级)	普通 HTTP	SSE 复用 HTTP 基础设施
浏览器 API	new WebSocket(url)	new EventSource(url)	平手
自动重连	❌ 没有	✅ 原生(可配 retry interval)	SSE 完胜
自动重连时携带 last-event-id	❌	✅(Last-Event-ID header)	SSE 完胜(断点续传友好)
事件类型	自己定(用 JSON 字段)	原生 event: 字段(浏览器分发到 onmessage / addEventListener)	SSE 略胜
数据格式	文本/二进制	只能 UTF-8 文本	WS 略胜(二进制帧更省带宽)
代理/防火墙	常被拦(非 HTTP 流量)	走 HTTP,几乎不被拦	SSE 完胜
公司网络环境	经常被中间件 reset	正常	SSE 完胜
CORS	单独处理	跟普通 HTTP 一样	SSE 完胜
鉴权	子协议 / URL token	标准 Authorization header	SSE 完胜
调试	wscat,devtools 二进制帧难读	curl -N 直接看	SSE 完胜
服务器实现	任意 TCP 长连接服务	任意能写 chunked HTTP 的服务	平手
CDN 友好	多数 CDN 不支持 WS	✅ 任何 CDN/反代都支持	SSE 完胜
Nginx 反代	需要 Upgrade 转发	默认就行(可能要关 proxy_buffering)	SSE 略胜
服务端推送并发多流	一连接 multiplex(per-channel)	一连接 = 一流(简单但费连接)	WS 略胜
Header 开销	升级后几乎无	每个 chunk 都带 HTTP chunked framing	WS 略胜
客户端向服务端发消息	✅ 原生	❌ 不行(要走另开 fetch)	WS 完胜
服务端向多个客户端广播	需要 pub/sub 中间件	简单循环 write 即可	SSE 略胜
取消 / 停止	发 close 帧	关 TCP(EventSource.close())	平手
在 React/Vue 里用	略复杂(自己处理 readyState)	简单(EventSource 事件循环)	SSE 略胜
库生态	socket.io、ws	原生,几乎不需要库	SSE 完胜
Playwright / e2e 测试	特殊处理	当成普通 HTTP 测试	SSE 完胜
各自适用的场景
WebSocket 该用的时候
判断标准:你需要客户端 → 服务端也持续推消息,且消息频繁

协同编辑 / 白板(Figma、Google Docs)

多人同时编辑,光标位置、选区、撤销栈都要双向同步
延迟要求 < 100ms
多人游戏 / 实时战斗

玩家操作 → 服务端 → 其它玩家,双向高频
每秒几十到几百条消息
金融行情 / 撮合

行情推送 + 客户端下单 + 实时回报,都是高频
延迟 < 50ms
实时聊天 / 直播弹幕

用户发消息 + 别人收 + 礼物特效推送
大量小消息
IoT 设备控制

设备上报状态 + 服务端下发指令
双向 + 长连接
通知中心 / 看板 的"双向"版本

客户端能主动 ack / 标记已读
SSE 该用的时候
判断标准:你只需要服务端 → 客户端单向推,客户端只在开始/偶尔发指令

LLM 流式输出(Claude Code、Codex、ChatGPT Web)

客户端发一次 prompt,服务端流式吐 token
agent CLI 场景:发送 → 收完,中间不需要再推任何东西
AI 图像生成进度

提交任务 → 推 0% / 30% / 80% / 完成
新闻 / 股价 / 体育比分推送

服务端推实时数据,客户端纯展示
进度条 / 长任务状态

文件上传处理、订单处理、模型训练进度
服务端 → 客户端的单向通知

系统公告、版本更新提醒
构建日志 / CI 流水线实时输出

Jenkins / GitHub Actions 的实时日志
断点续传友好的场景

SSE 的 Last-Event-ID 头原生支持,断线后用 last id 重连只拿增量
模糊地带:两者都能,看偏好
实时 dashboard(Grafana、监控面板)— 多数选 SSE,因为单向 + 调试友好 + 不需要客户端推
客服系统 — 多数选 WS,因为需要双向
在线课堂 — 选 WS,因为要上行(学生答题、举手)
你的 ZBot 场景:为什么 SSE 完胜
你的需求	SSE 是否够
客户端发一条消息	❌ 不行,但可以开一个普通 POST 走
服务端流式推 token / 工具调用 / 进度	✅
取消正在运行的 run	❌ 不能用 SSE 发,但可以走 DELETE
切 session / 切 run	同上,普通 HTTP 即可
浏览器 / 代理 / 公司内网	✅ SSE 友好,WS 常被拦
DevTools 调试	✅ curl -N 直接看
多端协同(resume run / 看历史)	✅ run_id 是一等资源,SSE 重连拿到 last-event-id
WS 在你这个场景唯一能多做的是:把上面"客户端发消息"和"服务端推流"塞到一条连接里。但代价是:

鉴权复杂(Cookie / Header / 子协议头)
代理 / 公司网络常被拦
devtools 难调试
写客户端 / 服务端都要管 readyState、心跳、重连
完全不值。

决策 checklist(以后遇到类似问题照抄)
客户端要不要主动发消息?
不发 → SSE
发,但只在开始 → HTTP POST + SSE(Claude Code、Codex、你的 ZBot)
发,而且过程中持续发 → WebSocket
延迟要求多少?
200ms → 两者都行,选 SSE

50-200ms → 看场景
< 50ms → WebSocket(或直接 TCP/QUIC)
要过公司代理 / CDN / 防火墙?
要 → SSE
自家机房 / 内网 → 都行
要给其它端 resume / 共享?
要 → SSE(run 一等资源 + Last-Event-ID)
纯单端 → 都行
预算紧 + 不想引库?
SSE 完胜(浏览器原生,服务端只要能写 chunked HTTP 就行)
一句话决策
单向流 + HTTP 工具链 = SSE
双向高频 + 自定义协议 = WebSocket
其它 = 看延迟 / 网络环境 / 调试要求


不是的，`where a = 1 and b = 2` **不是先查到所有 `a=1`，再一个个遍历判断 `b=2`**。

更准确地说：

它不是这样：

而是这样：

---

假设联合索引 `(a,b,c)` 里的顺序是：

当你查：

MySQL 可以直接定位到这一段：

它不会先把这些全扫一遍：

所以 **b 是参与索引定位的**。

---

你可能混淆的是这两种情况。

第一种：

这个可以用到联合索引的 `a,b`。

第二种：

这个一般只能用到 `a` 做定位，`c=3` 更多是过滤条件。

因为联合索引是：

你跳过了 `b`，直接查 `c`，MySQL 没法在 `a=1` 的范围内按 `c` 快速定位。

比如：

在 `a=1` 的范围内，数据是按 `b` 排的，不是按 `c` 排的。
所以 `c=3` 分布在不同的 `b` 下面，不能直接连续定位，只能扫 `a=1` 的范围再判断 `c=3`。

---

所以结论是：

用到 `a` 定位。

用到 `a,b` 定位，不是只用 `a`。

用到 `a,b,c` 定位。

通常只用 `a` 定位，`c` 作为过滤条件。

通常无法正常利用 `(a,b,c)` 的最左前缀。

---

你可以这样理解：

联合索引 `(a,b,c)` 不是三个独立索引，而是一个组合排序规则：

所以只要你从最左边连续使用：

都可以参与索引定位。

但如果中间断了：

后面的列就很难用于定位，只能更多作为过滤条件。



可以，下面画一个**简化但接近真实结构**的联合索引 B+ 树。

假设有联合索引：

索引项真实存的不是单独的 `a`，而是类似：

比如：

B+ 树大概长这样：

        内部节点A              内部节点B              内部节点C
   [(1,1,3) (1,2,3)]          [(2,1,2)]              [(3,1,1)]
      /      |      \            /     \                /     \

叶子1       叶子2       叶子3       叶子4       叶子5

叶子1:  (1,1,1)  (1,1,2)
          ↓
叶子2:  (1,2,1)  (1,2,2)  (1,2,3)
          ↓
叶子3:  (1,3,1)  (2,1,1)
          ↓
叶子4:  (2,1,2)  (2,2,1)
          ↓
叶子5:  (3,1,1)  (3,2,1)
```

叶子节点之间是有链表连接的，所以可以顺序扫描。

---

现在执行：

MySQL 要找的起点可以理解成：

然后开始走 B+ 树。

---

第一步，看根节点：

比较目标 `(1,2,最小值)` 和 `(1,3,1)`：

所以目标在 `(1,3,1)` 左边，走左侧子节点。

---

第二步，看内部节点A：

先和 `(1,1,3)` 比：

所以目标在它右边。

再和 `(1,2,3)` 比：

所以目标在 `(1,2,3)` 左边。

于是走到中间那个叶子节点。

---

第三步，到叶子节点：

这里就是符合：

的一整段。

然后 MySQL 顺着叶子节点往后扫描：

继续往后到下一个：

发现：

停止扫描。

---

所以重点是：

走树时不是只拿 `a=1` 去走，而是拿：

这个组合值去走。

比较规则是：

所以 `b=2` 在走 B+ 树的时候已经参与定位了。它不是先查完所有 `a=1`，再一个个过滤 `b=2`。


假设创建 Read View 的那一刻，系统里事务状态是：

事务 10：已经提交
事务 11：还没提交
事务 12：已经提交
事务 13：还没提交
下一个事务 id：14

那么 Read View 是：

m_ids = [11, 13]
min_trx_id = 11
max_trx_id = 14

现在判断某条版本记录的 trx_id。

情况 1：trx_id = 11
11 在 [11, 14) 范围内
并且 11 在 m_ids 里

说明事务 11 在创建 Read View 时还没提交。

所以：

不可见
情况 2：trx_id = 12
12 在 [11, 14) 范围内
但是 12 不在 m_ids 里

这说明什么？

说明事务 12 的 id 虽然在 Read View 创建前已经分配过，但是它在创建 Read View 时已经提交了。

所以：

可见

关键就在这里：

[11, 14) 里面有 11、12、13
但真正活跃的只有 11 和 13

也就是：

m_ids = [11, 13]

中间的 12 已经提交了，所以不在 m_ids 里。








`nums` 这个变量名是 **函数被调用时才会绑定/创建为局部变量**，不是别人 `import` 这个文件时就创建。

你的代码：

当 Python 执行到这段 `def` 时，只会做几件事：

但此时：

这个局部变量并没有真正产生。

---

真正调用时才有：

这时 Python 才会创建一个函数调用栈帧，然后把传入的列表绑定给局部变量 `nums`：

所以：

里面的 `nums` 只是一个**参数名**，提前被记录在函数对象里，不是提前创建了变量。

---

但是有一个细节：**类型注解可能会在定义函数时被处理。**

比如：

这个 `list[int]` 是类型注解，可能会在函数定义时记录到：

大概类似：

所以注意区分：

一句话总结：







不是多此一举，核心是：

---

# 三、redo log 与数据库



## 1. 写数据为什么慢？

```text
第 10 页
第 800 页
第 23000 页
第 500000 页
```

```text
磁盘跳到 A 位置写一下
再跳到 B 位置写一下
再跳到 C 位置写一下
```

MySQL 的真实数据存在很多数据页里，比如：

你更新一条记录时，MySQL 要找到这条记录所在的数据页。这个页可能在磁盘很远的位置。

所以写数据页像这样：

这叫 **随机写**，比较慢。

---

## 2. redo log 为什么快？

```text
日志1
日志2
日志3
日志4
```

redo log 是日志文件，写的时候基本就是往文件末尾追加：

就像记账本一样，一条一条往后写。

这叫 **顺序写**，比较快。

---

## 3. redo log 到底解决什么？

```sql
update user set age = 20 where id = 1;
```

```text
1. 在内存里修改数据页
2. 把“我改了什么”写入 redo log
```

> **先用很快的顺序写 redo log 保证安全，真实数据页以后再慢慢刷盘。**

比如你执行：

MySQL 不一定马上把真实数据页写回磁盘。

它可以先做两件事：

只要 redo log 写入磁盘，即使数据库突然宕机，也能根据 redo log 恢复数据。

所以它的意义是：

---

## 4. 一句话理解

核心就这几句话：

```text
我要去磁盘各个位置找页再写，慢，随机写。
```

```text
我只是在日志末尾追加一条记录，快，顺序写。
```

```text
先用 redo log 恢复数据页和 undo 页
再用 undo log 回滚未提交事务
```

```text
1. 事务回滚
2. MVCC 快照读
```

```sql
BEGIN;

```text
redo log：崩溃后把已经做过的修改恢复出来；
undo log：事务失败或未提交时，把数据改回原来的样子；
MVCC：通过 undo log 读取旧版本数据。
```

```sql
SELECT * FROM goods WHERE id = 1;
```

```sql
SELECT * FROM goods WHERE id = 1 FOR UPDATE;
```

```sql
UPDATE goods SET stock = stock - 1 WHERE id = 1;
```

```text
普通 SELECT：只看，不锁，读 MVCC 快照
SELECT ... FOR UPDATE：先看，再锁，不改
UPDATE：先锁，再改
```

```text
我怎么安全、高效地读数据
```

```text
谁能修改这行数据
```

```sql
UPDATE goods
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
```

```sql
BEGIN;

> **用顺序写日志代替每次立即随机写数据页，提高性能，同时保证崩溃后能恢复。**

> **MVCC 让普通读不阻塞；`FOR UPDATE` 是为了在“先读后改”时提前锁住数据；`UPDATE` 是直接锁住并修改数据。**


写真实数据页：

写 redo log：

所以 redo log 不是多余，而是：




**undo log 不是不持久化。**

它是先写到内存里的 **Undo Page**，以后会刷到磁盘。因为 Undo Page 也是内存页，崩溃后可能丢失，所以 InnoDB 会把 Undo Page 的修改也记录到 **redo log** 里。

所以崩溃恢复时大概是：

undo log 的作用主要有两个：

即使你不手动执行 `rollback`，undo log 也有用。因为事务可能执行失败、发生死锁、锁等待超时，或者 MySQL 崩溃。只要事务没有成功提交，InnoDB 就需要用 undo log 把已经修改过的数据撤销。

简单例子：

UPDATE account SET money = money - 100 WHERE id = 1;

UPDATE account SET money = money + 100 WHERE id = 2;

COMMIT;
```

如果第一条成功了，账户 1 已经扣了 100 元；但第二条失败了，比如账户 2 不存在。这个事务不能只完成一半，否则钱就少了 100 元。

所以 InnoDB 会用 undo log，把账户 1 扣掉的 100 元恢复回去。

一句话记：

所以 undo log 的本质是：

**记录修改前的数据，用来支持事务回滚和 MVCC 快照读。**



**MVCC 的作用：**
让普通 `SELECT` 可以不加锁地读取一个“已提交的快照版本”，所以**读不会阻塞写，写也不会阻塞普通读**。它主要解决的是：查询、报表、列表页等普通读取场景的并发性能和一致性问题。

**普通 SELECT：**

这是**快照读**，通常不加锁。它只是“看数据”，不会阻止别人修改这行。

**SELECT ... FOR UPDATE：**

这是**当前读 + 加排他锁**。它不修改数据，但会把查到的行锁住，防止其他事务修改。适合“先查，再做业务判断，再更新”。

**UPDATE：**

这是**当前读 + 加排他锁 + 修改数据**。它会直接改数据。

所以：

MVCC 不是没用，它管的是：

锁管的是：

如果只是简单扣库存，推荐：

这是原子操作，不一定需要 `FOR UPDATE`。

如果是复杂业务判断，比如先查库存、用户状态、优惠券、风控，再决定是否修改，就用：

SELECT * FROM goods WHERE id = 1 FOR UPDATE;

-- 复杂判断

UPDATE goods SET stock = stock - 1 WHERE id = 1;

COMMIT;
```

一句话总结：