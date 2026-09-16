本手册包含 2 份源文件：E:\GithubProject\docs\general\计算机基础知识整合指南\07-分布式系统中间件与后台任务.md、E:\GithubProject\RabbitMQ架构详解.md

> 来源：由总文档拆分生成。原补充材料会按主题整理插入到对应位置。

# 分布式系统、中间件与后台任务

```cypher
// 节点：人（演员、导演）
CREATE (p1:Person {id:"p_tom_hanks",  name:"汤姆·汉克斯",        birth_year:1956, nationality:"美国"})
CREATE (p2:Person {id:"p_zhang_yimou",name:"张艺谋",            birth_year:1950, nationality:"中国"})
CREATE (p3:Person {id:"p_leonardo",   name:"莱昂纳多·迪卡普里奥", birth_year:1974, nationality:"美国"})
CREATE (p4:Person {id:"p_angelina",   name:"安吉丽娜·朱莉",      birth_year:1975, nationality:"美国"})

```text
                ┌─────────────────────┐
                │   二十世纪福克斯      │
                └──────────┬──────────┘
                           │ PRODUCED_BY
                           ▼
   汤姆·汉克斯 ──ACTED_IN──→ 阿甘正传 ──HAS_GENRE──→ 剧情
        │  │                      │
        │  │ ACTED_IN             │ HAS_GENRE
        │  ▼                      ▼
        │ 莱昂纳多 ──ACTED_IN──→ 泰坦尼克号 ──HAS_GENRE──→ 爱情
        │
        │ WON_AWARD
        ▼
   奥斯卡最佳男主角

```text
- 节点类型不只一种：人、电影、公司、类型、奖项、国家
- 边代表不同语义关系：演过、导演过、出品、属于类型、获奖、位于
- 边也可以有属性：汤姆·汉克斯演阿甘正传时片酬 7000 万美元
- 节点之间是多跳路径：演员 → 电影 → 公司 → 国家 → 其他公司
- 属性多种多样：数字、字符串、日期、列表都可以挂
```

```text
Google Knowledge Graph：约 500 亿条事实，覆盖数亿实体
Wikidata：超过 1 亿个实体，10 亿多条关系
Amazon Product Graph：商品节点超过 4 亿
Freebase（已合并入 Wikidata）：约 19 亿三元组
医疗 SNOMED CT：概念节点超过 35 万，关系数百万级
中文 CN-DBpedia：数千万实体，覆盖人物、地点、电影、机构
```

```text
问题：「汤姆·汉克斯演过的电影里，哪些是和二十世纪福克斯合作的？」

构建流程一般分 6 步：

```text
1. 明确领域和 schema（限定图谱边界）
2. 收集原始数据（结构化、半结构化、非结构化）
3. 抽取实体和关系（Information Extraction）
4. 实体归一和融合（Entity Resolution）
5. 写入图数据库
6. 补全文索引、向量索引、图索引
```

```cypher
// 电影 KG 的 schema
Node types:  Person, Movie, Company, Genre, Award, Country
Relations:
  Person  -[ACTED_IN]->    Movie
  Person  -[DIRECTED]->    Movie
  Movie   -[PRODUCED_BY]-> Company
  Movie   -[HAS_GENRE]->   Genre
  Person  -[WON_AWARD]->   Award
  Company -[LOCATED_IN]->  Country
```

```cypher
CREATE CONSTRAINT person_id  FOR (p:Person)  REQUIRE p.id IS UNIQUE
CREATE CONSTRAINT movie_id   FOR (m:Movie)   REQUIRE m.id IS UNIQUE
CREATE CONSTRAINT company_id FOR (c:Company) REQUIRE c.id IS UNIQUE
```

```text
LLM + schema（灵活、覆盖广）
  - 给 LLM 一个明确的 schema prompt
  - 要求输出 JSON 结构
  - 适合长尾实体、跨领域文本
```

```python
import openai

| 类型     | 例子                                                   |
| -------- | ------------------------------------------------------ |
| 结构化   | 维基百科 infobox、IMDb、豆瓣电影、企查查、关系型数据库 |
| 半结构化 | 表格、列表、HTML 表格、JSON / XML                      |
| 非结构化 | 新闻报道、影评、剧情介绍、人物专访文本                 |

#### 一个真实的知识图谱长什么样？

**问题**：知识图谱在真实生产环境里到底长什么样？节点、边、属性具体怎么展开？

**回答**：

用一个相对完整的「电影-人物-公司」知识图谱来举例，下面是 Neo4j 中的真实存储形态（用 Cypher 创建）：

// 节点：电影
CREATE (m1:Movie {id:"m_forrest", title:"阿甘正传",   year:1994, rating:9.5, language:"英语"})
CREATE (m2:Movie {id:"m_titanic", title:"泰坦尼克号", year:1997, rating:9.3, language:"英语"})
CREATE (m3:Movie {id:"m_hero",    title:"英雄",       year:2002, rating:7.9, language:"中文"})

// 节点：公司
CREATE (c1:Company {id:"c_paramount", name:"派拉蒙影业",     country:"美国", founded:1912})
CREATE (c2:Company {id:"c_fox",       name:"二十世纪福克斯", country:"美国", founded:1915})
CREATE (c3:Company {id:"c_nmg",       name:"新画面影业",     country:"中国", founded:2000})

// 节点：类型、奖项、地区
CREATE (g1:Genre  {name:"剧情"})
CREATE (g2:Genre  {name:"爱情"})
CREATE (a1:Award  {name:"奥斯卡最佳男主角", year:1995, holder_id:"p_tom_hanks"})

// 边（关系）—— 边也可以有属性
CREATE (p1)-[:ACTED_IN   {role:"Forrest Gump", salary_million:70}]->(m1)
CREATE (p3)-[:ACTED_IN   {role:"Jack"}]->(m2)
CREATE (p4)-[:ACTED_IN   {role:"Lara Croft"}]->(m1)
CREATE (p2)-[:DIRECTED]->(m3)

CREATE (m1)-[:PRODUCED_BY]->(c1)
CREATE (m2)-[:PRODUCED_BY]->(c2)
CREATE (m3)-[:PRODUCED_BY]->(c3)

CREATE (m1)-[:HAS_GENRE]->(g1)
CREATE (m2)-[:HAS_GENRE]->(g1)
CREATE (m2)-[:HAS_GENRE]->(g2)
CREATE (m3)-[:HAS_GENRE]->(g1)

CREATE (p1)-[:WON_AWARD]->(a1)
CREATE (c1)-[:LOCATED_IN]->(:Country {name:"美国"})
CREATE (c3)-[:LOCATED_IN]->(:Country {name:"中国"})
```

可视化后大致是：

   安吉丽娜·朱莉 ──ACTED_IN──→ 阿甘正传

   张艺谋 ──DIRECTED──→ 英雄 ──PRODUCED_BY──→ 新画面影业
                                                │
                                                │ LOCATED_IN
                                                ▼
                                              中国
```

这才是真实知识图谱的样子：

真实世界里的规模：

真实查询常常需要 3-5 跳甚至更多跳。例如：

最短路径：
  Person(汤姆·汉克斯)
    -[ACTED_IN]->
  Movie
    -[PRODUCED_BY]->
  Company(二十世纪福克斯)

三跳就出答案。换成 SQL 通常要 5-8 次 JOIN，越深越慢；
换成图数据库，三跳是常数时间复杂度。
```

---

#### 知识图谱怎么找数据源、怎么构建？

**问题**：从零搭建一个真实可用的知识图谱，数据从哪来？流程是什么？

**回答**：

##### 第 1 步：定义 schema

没有 schema 的图谱会迅速变脏。schema 是图谱的「类型系统」，必须先定义允许的节点类型和关系类型：

约束示例（Neo4j 4.x）：

##### 第 2 步：找数据源

数据来源分三类：

##### 第 3 步：抽取实体和关系

**非结构化文本必须用 NLP 抽取**

用 LLM 按 schema 抽取的示例：

text = "汤姆·汉克斯凭借《阿甘正传》获得奥斯卡最佳男主角，导演是罗伯特·泽米吉斯。"

prompt = f"""
从下面文本中抽取实体和关系。
节点类型：Person, Movie, Award
关系类型：ACTED_IN, DIRECTED, WON_AWARD
输出 JSON 格式：
{{"entities":[{{"label":"...", "name":"...", "properties":{{...}}}}, ...],
  "relations":[{{"source":"...", "type":"...", "target":"..."}}, ...]}}

文本：{text}
"""

resp = openai.ChatCompletion.create(
    model="gpt-4o-mini",
    messages=[{"role":"user", "content":prompt}],
    response_format={"type":"json_object"}
)

# 输出大致：
# {
#   "entities": [
#     {"label":"Person", "name":"汤姆·汉克斯"},
#     {"label":"Movie",  "name":"阿甘正传"},
#     {"label":"Award",  "name":"奥斯卡最佳男主角"},
#     {"label":"Person", "name":"罗伯特·泽米吉斯"}
#   ],
#   "relations": [
#     {"source":"汤姆·汉克斯",       "type":"ACTED_IN",   "target":"阿甘正传"},
#     {"source":"汤姆·汉克斯",       "type":"WON_AWARD",  "target":"奥斯卡最佳男主角"},
#     {"source":"罗伯特·泽米吉斯",   "type":"DIRECTED",   "target":"阿甘正传"}
#   ]
# }
```

```text
"汤姆·汉克斯"、"Tom Hanks"、"Thomas Jeffrey Hanks"  →  同一个 Person
"阿甘正传"、"Forrest Gump"、"阿甘"                   →  同一个 Movie
"派拉蒙"、"Paramount Pictures"、"派拉蒙影业公司"      →  同一个 Company
```

```text
1. 主键匹配：直接用 Wikidata QID、IMDb ID 对齐（最稳）
2. 别名表：维护一个手工或半自动的 alias 字典
3. 规则匹配：去除括号、标点、统一大小写、繁简转换
4. 模糊匹配：Edit Distance、Jaro-Winkler
5. 向量相似度：用句向量模型计算名字相似度（bge-large-zh）
6. 关系增强：看邻居节点是否一致（同名但无共同邻居，可能是不同实体）
7. 人工审核：低置信度对进入审核队列（< 0.8 置信度）
```

```json
{
  "merged_id": "p_tom_hanks",
  "candidates": [
    {"name":"汤姆·汉克斯",      "source":"豆瓣", "confidence":0.99},
    {"name":"Tom Hanks",        "source":"IMDb", "confidence":0.99},
    {"name":"Thomas J. Hanks",  "source":"Wikidata", "confidence":0.97}
  ],
  "merge_method": "wikidata_qid_match",
  "review_status": "auto_approved"
}
```

```python
from py2neo import Graph, Node, Relationship

| 数据库          | 特点                              | 适用场景               |
| --------------- | --------------------------------- | ---------------------- |
| **Neo4j** | 最主流，Cypher 查询语言、社区成熟 | 通用、企业内、知识图谱 |
|                 |                                   |                        |

##### 第 4 步：实体归一和融合

不同来源写同一个实体，名字可能完全不同：

归一方法按强度递增：

归一过程中要保留**证据**，方便人工核查：

##### 第 5 步：写入图数据库

常用图数据库选型：

写入流程示例（py2neo）：

graph = Graph("bolt://localhost:7687", auth=("neo4j", "password"))

# 写入节点：按 id 去重
for ent in entities:
    node = Node(ent.label, id=ent.id, **ent.properties)
    graph.merge(node, ent.label, "id")

# 写入关系：必须先确认两端节点已存在
for rel in relations:
    s_node = graph.evaluate(f"MATCH (n {{id:'{rel.source_id}'}}) RETURN n")
    t_node = graph.evaluate(f"MATCH (n {{id:'{rel.target_id}'}}) RETURN n")
    if s_node and t_node:
        r = Relationship(s_node, rel.type, t_node, **rel.properties)
        graph.merge(r, rel.type, "source_id")
```

#### 什么是微服务？

```cypher
UNWIND $entities AS ent
MERGE (n:Person {id: ent.id})
SET n += ent.properties

```text
图数据库  Neo4j / NebulaGraph     存放节点、关系、属性（事实层）
全文索引  Elasticsearch / Solr    存放节点名字、属性、文本描述（关键字检索）
向量索引  Milvus / Qdrant          存放节点描述向量（语义近似检索）
```

```python
# 节点入库同时同步三个存储
for node in nodes:
    graph.merge(node)                                  # 图数据库
    es.index("kg_nodes", id=node.id, body={            # 全文索引
        "name": node.name,
        "description": node.description
    })
    vec = embedding_model.encode(node.description)     # 向量索引
    milvus.insert(ids=[node.id], vectors=[vec])
```

```cypher
// 查询 1：汤姆·汉克斯演过的所有电影
MATCH (p:Person {name:"汤姆·汉克斯"})-[r:ACTED_IN]->(m:Movie)
RETURN m.title, m.year, r.role

```text
中心性算法：PageRank、Betweenness Centrality（找重要人物/关键节点）
社区发现：Louvain、Label Propagation（发现流派、圈子、团伙）
相似度：  Node Similarity、Jaccard（推荐相似电影/商品）
路径分析：最短路径、所有路径（关联挖掘）
嵌入学习：Node2Vec、GraphSAGE、TransE（把节点变成向量）
```

```cypher
CALL gds.pageRank.stream('movie-kg')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS actor, score
ORDER BY score DESC LIMIT 10
```

```cypher
CALL gds.louvain.stream('movie-kg')
YIELD nodeId, communityId
RETURN communityId, COLLECT(gds.util.asNode(nodeId).name) AS members
ORDER BY SIZE(members) DESC
```

```text
用户问题："汤姆·汉克斯有什么作品得过奥斯卡？"
   |
   v
LLM 识别实体和意图
   |
   v
并行三种检索：
  1. 图查询：     Cypher     Person -[WON_AWARD]-> Award
  2. 向量检索：   Milvus    "汤姆·汉克斯 奥斯卡" 句向量 → 近似节点
  3. 全文检索：   Elasticsearch 节点描述里 grep 关键词
   |
   v
合并 + 重排序（Reranker，如 BGE Reranker）
   |
   v
LLM 基于证据生成答案（带引用）
```

```text
- 答案有出处（不是 LLM 编的）
- 支持多跳推理（"哪些和汤姆·汉克斯合作过的演员也拿过奥斯卡？"）
- 可审计（每条事实可以追到原始 evidence）
- 不幻觉（事实来自图谱，不来自模型记忆）
```

```cypher
// 规则 1：得过奥斯卡最佳男主角的演员自动标记为 TopActor
MATCH (p:Person)-[:WON_AWARD]->(a:Award)
WHERE a.name STARTS WITH "奥斯卡最佳男主角"
SET p:TopActor

```
单体应用：所有功能在一个程序里
  ┌───────────────────────────┐
  │  一个大程序                 │
  │  用户管理 + 订单 + 支付 + ...│
  └───────────────────────────┘
  
  问题：改一个功能要重新部署整个程序，代码越来越臃肿

```
- 独立开发、独立部署
- 一个服务挂了不影响其他
- 可以用不同语言写不同服务
```

批量写入推荐用 `UNWIND` + `MERGE`：

UNWIND $relations AS rel
MATCH (s {id: rel.source_id}), (t {id: rel.target_id})
MERGE (s)-[r:ACTED_IN]->(t)
SET r += rel.properties
```

##### 第 6 步：补全三类索引

知识图谱系统通常需要「图数据库 + 全文索引 + 向量索引」三层协作：

写入节点时同步写索引：

---

#### 知识图谱构建好之后怎么用？

**问题**：知识图谱有哪些典型查询和用法？

**回答**：

##### 用法 1：结构化查询（Cypher）

Cypher 是图数据库的 SQL，等价于「关系路径上的 select」。

// 查询 2：和汤姆·汉克斯合作过 2 次以上的演员
MATCH (p1:Person {name:"汤姆·汉克斯"})
      -[:ACTED_IN]->(m:Movie)
      <-[:ACTED_IN]-(p2:Person)
WHERE p1 <> p2
WITH p2, COUNT(DISTINCT m) AS times
WHERE times >= 2
RETURN p2.name, times
ORDER BY times DESC

// 查询 3：最短路径（汤姆·汉克斯如何连接到张艺谋）
MATCH p = shortestPath(
  (a:Person {name:"汤姆·汉克斯"})-[*..6]-(b:Person {name:"张艺谋"})
)
RETURN p

// 查询 4：两个演员是否出演过同一家公司出品的电影
MATCH (p1:Person)-[:ACTED_IN]->(m:Movie)-[:PRODUCED_BY]->(c:Company)
      <-[:PRODUCED_BY]-(m2:Movie)<-[:ACTED_IN]-(p2:Person)
WHERE p1.name = "汤姆·汉克斯" AND p2.name = "莱昂纳多"
RETURN c.name, m.title, m2.title
```

传统 SQL 做这种多跳查询通常要 5-8 次 JOIN，图数据库一跳搞定。

##### 用法 2：图算法（Graph Algorithm）

Neo4j Graph Data Science、NetworkX、阿里 GraphScope、PyTorch Geometric 都提供：

PageRank 找出图谱里最有影响力的演员：

社区发现找出演员圈子：

##### 用法 3：GraphRAG（和 LLM 结合的检索增强）

这是 2024 年开始最常见的用法，前面 schema 部分已经讲过思路。再强调一次混合检索的实际执行：

优势：

##### 用法 4：推理（Rule-based Reasoning）

图谱上可以跑规则推理。例如：

// 规则 2：标记动作片演员
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)-[:HAS_GENRE]->(g:Genre)
WHERE g.name IN ["动作","冒险","战争"]
WITH p, COUNT(DISTINCT m) AS cnt
WHERE cnt >= 3
SET p:ActionActor

// 查询：既得过奥斯卡又是动作片演员的人
MATCH (p:TopActor:ActionActor)
RETURN p.name
```

更复杂的形式化推理可以用 OWL / RDF 推理机（RDF4J、Jena），但工程上慎用，推理链长容易爆炸。

**问题**：什么是微服务？和单体应用有什么区别？

**回答**：

**是什么**：
微服务是将应用拆分为多个独立小服务的架构。

**对比单体**：

微服务：每个功能是独立的小服务
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ 用户服务   │  │ 订单服务   │  │ 支付服务   │
  │ 端口8001  │  │ 端口8002  │  │ 端口8003  │
  └──────────┘  └──────────┘  └──────────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
              通过HTTP/RPC通信
```

**好处**：

---

### 6.5 高并发处理

```
方式1：多进程
  ┌─────────┐
  │ 进程1    │ ← 处理请求1-250
  │ 进程2    │ ← 处理请求251-500
  │ 进程3    │ ← 处理请求501-750
  │ 进程4    │ ← 处理请求751-1000
  └─────────┘
  每个进程运行相同的代码，占独立内存

#### 高并发是怎么处理的？

**问题**：1000个用户同时访问，服务器怎么处理？

**回答**：

方式2：多线程
  ┌─────────────────────────┐
  │ 一个进程                  │
  │  线程1 ← 请求1           │
  │  线程2 ← 请求2           │
  │  ...                    │
  │  线程N ← 请求N           │
  │  共享内存                │
  └─────────────────────────┘

方式3：异步/协程（Python的asyncio）
  ┌─────────────────────────┐
  │ 一个线程                  │
  │  协程1: 等数据库响应...    │
  │  → 切到协程2处理         │
  │  协程2: 等Redis响应...    │
  │  → 切到协程3处理         │
  │  协程1: 数据库返回了！继续 │
  └─────────────────────────┘

实际生产环境通常组合使用：
  Nginx（负载均衡）→ 多个Gunicorn进程 → 每个进程多个线程/协程
```

---

## 性能优化与高并发

### 8.1 限流与防刷

#### 什么是接口限流？

- **限流**：限制请求频率，防止系统过载
- **防刷**：防止恶意请求（如刷票、爬虫）
- **流量控制**：更广泛的流量管理

```
Redis实现令牌桶：

```
Redis实现固定窗口：

```
Redis实现滑动窗口：

```
漏桶算法：请求以恒定速率流出

```
请求被限流后：

**问题**：什么是接口限流、接口防刷和流量控制？

**回答**：

**是什么**：

**令牌桶算法的实现**

1. 数据结构：
   - tokens：当前令牌数量
   - last_time：上次更新时间
   - rate：令牌生成速率（如10个/秒）
   - max_tokens：桶容量上限

2. 请求到达时：
   // Lua脚本（保证原子性）
   local tokens = redis.call('GET', KEYS[1] .. ':tokens')
   local last_time = redis.call('GET', KEYS[1] .. ':last_time')
   local now = redis.call('TIME')[0]
   
   // 计算新增令牌
   local elapsed = now - last_time
   local new_tokens = elapsed * rate
   
   // 更新令牌数量（不超过上限）
   tokens = math.min(tokens + new_tokens, max_tokens)
   
   // 尝试取令牌
   if tokens >= 1 then
       tokens = tokens - 1
       redis.call('SET', KEYS[1] .. ':tokens', tokens)
       redis.call('SET', KEYS[1] .. ':last_time', now)
       return 1  // 允许请求
   else
       redis.call('SET', KEYS[1] .. ':tokens', tokens)
       redis.call('SET', KEYS[1] .. ':last_time', now)
       return 0  // 拒绝请求
   end

3. CPU执行流程：
   - 客户端发送请求
   - Redis收到Lua脚本
   - Redis执行脚本（原子操作）：
     a. 读取当前令牌数和时间
     b. 计算新增令牌
     c. 更新令牌数
     d. 判断是否允许
   - 返回结果给客户端
```

**固定窗口算法的实现**

1. 数据结构：
   - key：用户ID + 时间窗口（如 user:123:2024-01-01-10:00:00）
   - value：请求计数

2. 请求到达时：
   // Lua脚本
   local window = math.floor(redis.call('TIME')[0] / 60)  // 每分钟一个窗口
   local key = KEYS[1] .. ':' .. window
   
   local count = redis.call('INCR', key)
   
   // 设置过期时间（窗口结束后自动删除）
   if count == 1 then
       redis.call('EXPIRE', key, 120)  // 2分钟后过期
   end
   
   if count > LIMIT then
       return 0  // 拒绝请求
   else
       return 1  // 允许请求
   end

3. 问题：临界时刻突发
   窗口1（10:00-10:01）：用户发了100个请求（达到限制）
   窗口2（10:01-10:02）：计数器重置，用户又能发100个请求
   
   10:00:59 和 10:01:01 之间只隔2秒，但用户发了200个请求
```

**滑动窗口算法的实现**

1. 数据结构：
   - 使用ZSET（有序集合）
   - member：请求时间戳
   - score：请求时间戳

2. 请求到达时：
   // Lua脚本
   local now = redis.call('TIME')[0]
   local window_start = now - 60  // 查看最近60秒
   
   // 删除窗口外的请求记录
   redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, window_start)
   
   // 统计窗口内的请求数量
   local count = redis.call('ZCARD', KEYS[1])
   
   if count >= LIMIT then
       return 0  // 拒绝请求
   else
       // 记录这次请求
       redis.call('ZADD', KEYS[1], now, now .. ':' .. math.random())
       redis.call('EXPIRE', KEYS[1], 120)
       return 1  // 允许请求
   end

3. 优点：精确控制
   - 始终看最近60秒的请求数
   - 不存在临界突发问题
```

**漏桶算法的实现**

1. 数据结构：
   - queue：请求队列
   - rate：流出速率（如10个/秒）

2. 实现方式：
   // 请求到达时
   if queue.size() < max_size then
       queue.push(request)
       return 200  // 接收请求
   else
       return 429  // 拒绝请求
   end
   
   // 后台线程以恒定速率处理
   while true:
       sleep(1 / rate)  // 如每0.1秒处理一个
       if queue.not_empty():
           request = queue.pop()
           process(request)

3. 特点：
   - 输出速率恒定
   - 请求排队等待
   - 不适合实时性要求高的场景
```

**限流返回的处理**

1. 返回HTTP 429状态码：
   HTTP/1.1 429 Too Many Requests
   Content-Type: application/json
   
   {
       "error": "Rate limit exceeded",
       "retry_after": 60
   }

2. 客户端处理：
   if response.status_code == 429:
       retry_after = response.headers.get('Retry-After')
       sleep(retry_after)
       retry_request()

3. 服务器日志：
   - 记录被限流的请求
   - 用于分析异常流量
   - 发现恶意用户
```

### Celery + RabbitMQ 核心总结

```text
定义任务
发送任务
执行任务
重试任务
记录任务状态
支持定时任务
```

```text
接收任务消息
保存任务消息
把任务消息投递给 Worker
```

> Celery 管“任务怎么调度和执行”，RabbitMQ 管“任务消息怎么排队和投递”。

#### 1. 它们分别是什么

**Celery** 是 Python 的异步任务框架，负责：

**RabbitMQ** 是消息中间件，负责：

一句话：

---

### 2. 基本架构

```text
Web 服务 / Celery Beat
        |
        | 发送任务消息
        v
RabbitMQ
        |
        | 投递任务消息
        v
Celery Worker
        |
        | 执行业务代码
        v
数据库 / Redis / 第三方接口 / 文件系统
```

---

### 3. Worker 是什么

启动命令：

```bash
celery -A celery_app worker -Q order,report -l info
```

```text
启动一个 Celery Worker 进程
导入 celery_app 模块
连接 RabbitMQ
监听 order、report 队列
等待任务消息
收到消息后执行本地代码
```

含义：

`-Q order,report` 表示这个 Worker 只消费 `order` 和 `report` 两个队列。

Worker 是一个**长期运行的进程**。开发时它在终端前台运行；生产环境一般用 systemd、Supervisor、Docker、Kubernetes 托管。

---

### 4. Beat 是什么

启动命令：

```bash
celery -A celery_app beat -l info
```

```text
启动 Celery Beat 调度器进程
定期检查哪些任务到时间了
到时间后把任务消息发到 RabbitMQ
```

```text
Beat = 定时发任务
Worker = 消费任务并执行
RabbitMQ = 保存和投递任务消息
```

含义：

Beat **不执行业务逻辑**，它只负责“按时间发任务”。

真正执行任务的是 Worker。

---

### 5. 定时任务怎么设计

启动：

```python
from celery.schedules import crontab

```bash
celery -A celery_app beat -l info
celery -A celery_app worker -Q order -l info
```

```text
django-celery-beat
```

```text
1. Beat 只负责投递任务，不写业务逻辑
2. 生产环境 Beat 和 Worker 分开启动
3. 同一套定时任务只能跑一个 Beat，避免重复投递
4. 防止任务重叠，需要 Redis 锁或数据库锁
5. 定时任务必须幂等
6. 不同类型任务拆不同队列
```

静态定时任务可以写在配置里：

app.conf.beat_schedule = {
    "close-expired-orders": {
        "task": "tasks.close_expired_orders",
        "schedule": crontab(minute="*"),
        "options": {
            "queue": "order",
            "expires": 50,
        },
    },
}
```

动态定时任务，比如后台页面配置、用户自定义提醒，常用：

它把定时任务存到数据库里，可以动态增删改查。

定时任务生产设计原则：

---

### 6. Celery 发给 RabbitMQ 的是什么

```json
{
  "id": "task-uuid",
  "task": "orders.close_order",
  "args": [1001],
  "kwargs": {},
  "retries": 0,
  "eta": null,
  "expires": null
}
```

```text
请执行 orders.close_order 这个任务，参数是 1001
```

不是函数。

不是文件。

不是类。

不是整个项目代码。

Celery 发给 RabbitMQ 的是**任务调用消息**，核心内容类似：

本质是：

---

### 7. Worker 怎么执行任务

```python
@app.task(name="orders.close_order")
def close_order_task(order_id):
    return close_order(order_id)
```

```text
orders.close_order -> close_order_task 函数
```

```text
1. 取出 task 名称
2. 在本地注册表里找到对应函数
3. 反序列化 args / kwargs
4. 执行函数
5. 成功后 ACK 消息
6. 如果配置了 backend，写入任务状态和结果
```

```python
close_order_task(1001)
```

Worker 启动时会导入你的代码，注册任务。

例如：

Worker 本地会形成一个任务注册表：

收到 RabbitMQ 的消息后：

所以 Worker 执行的是：

不是执行整个文件，也不是从 RabbitMQ 拿代码。

---

### 8. 如果任务函数调用其他文件的函数或类怎么办

```python
@app.task(name="orders.close_order")
def close_order_task(order_id):
    return close_order_service(order_id)
```

```text
tasks/
services/
models/
utils/
配置文件
依赖包
环境变量
```

例如：

这里 `close_order_service` 可能在其他文件里。

Celery 不会把它传给 RabbitMQ。

Worker 机器或容器里必须提前部署完整代码：

任务消息只负责触发入口函数。

入口函数内部调用的其他函数、类、ORM、数据库连接，全部来自 Worker 本地环境。

---

### 9. Web 服务关闭了，任务还会执行吗

```text
Web 服务关了
新请求进不来
新任务发不出去
但旧任务如果已经在 RabbitMQ 里，Worker 还能消费
```

分情况。

#### Web 服务关了，Worker 没关

已经进入 RabbitMQ 的任务，Worker 仍然可以继续执行。

#### Worker 也关了

任务不会执行。

RabbitMQ 里如果还有消息，会继续排队。等 Worker 重新启动后，再继续消费。

#### RabbitMQ 关了

Web 发不出任务，Worker 也拿不到任务，异步任务链路中断。

---

### 10. Worker 会不会自动启动 Web 服务代码

```bash
uvicorn main:app
python manage.py runserver
gunicorn app:app
```

不会。

Worker 不会启动：

Worker 只启动 Celery Worker 进程。

但它会 import 你的业务模块，并在收到任务后直接调用本地函数。

如果任务要用 Django ORM，只要 Worker 环境里有 Django 项目代码、settings、数据库连接配置，就可以直接操作数据库，不需要 Web 服务正在运行。

---

### 11. `backend="redis://localhost:6379/0"` 是什么

```text
任务状态
任务返回值
异常信息
traceback
任务完成时间
group/chord 结果
```

```python
backend="redis://localhost:6379/0"
```

```text
用 Redis 作为任务结果存储
Redis 地址是 localhost
端口是 6379
使用 Redis 的 0 号数据库
```

```text
redis://username:password@host:port/db
```

```python
backend = "redis://:password@localhost:6379/0"
```

这里的 `backend` 指的是 **Celery Result Backend**，也就是任务结果后端。

它不是 Web 后端。

它负责保存：

这行配置：

意思是：

完整格式：

例如：

---

### 12. Broker 和 Backend 区别

对比：

```python
broker="amqp://guest:guest@localhost:5672//"
backend="redis://localhost:6379/0"
```

```text
RabbitMQ 作为 broker，负责传任务消息
Redis 作为 backend，负责存任务结果
```

| 组件    | 职责               |
| ------- | ------------------ |
| Broker  | 保存和投递任务消息 |
| Worker  | 执行业务代码       |
| Backend | 保存任务状态和结果 |
| Beat    | 定时产生任务消息   |

非常重要：

含义是：

### 14. 推荐的生产理解模型

```text
Web 服务
  接收 HTTP 请求
  调用 delay/apply_async 发任务

```text
用户请求
   |
   v
Web 服务
   |
   | task.delay(order_id)
   v
RabbitMQ
   |
   | task="orders.close_order", args=[1001]
   v
Celery Worker
   |
   | 本地执行 close_order_task(1001)
   v
业务代码 / 数据库 / 第三方服务
   |
   v
Redis Backend 记录结果
```

RabbitMQ
  保存任务消息
  把消息投递给 Worker

Worker
  长期运行
  监听队列
  收到任务后执行本地代码

Beat
  长期运行
  按时间规则发任务

Redis Backend
  保存任务状态和结果
```

完整链路：

---

### 15. 最核心结论

```text
1. Celery 不传代码，只传任务名和参数
2. RabbitMQ 不执行代码，只保存和投递消息
3. Worker 必须提前部署完整业务代码
4. Worker 是独立长期运行进程，不依赖 Web 服务活着
5. Web 服务关了，已进入 RabbitMQ 的任务仍可被 Worker 执行
6. Worker 关了，任务会在 RabbitMQ 里排队
7. Beat 只负责定时发任务，不负责执行任务
8. Backend 只负责存任务状态和结果，不负责传任务
9. 生产环境要拆分 web、worker、beat、rabbitmq、redis
10. 任务参数尽量传 ID，不要传复杂对象
```

> **Celery + RabbitMQ 的本质是：发任务的一方只发送“任务名 + 参数”；RabbitMQ 负责排队；Worker 在自己本地代码环境里找到对应任务函数并执行。**

你只需要牢牢记住这几句话：

一句话总结：


---

# RabbitMQ 架构详解（八股文）

> 面试向 | 小白友好 | 每个概念都讲透 | 覆盖核心原理与 15 道高频面试题

---

## 目录

- [一、概述与定位](#一概述与定位)
- [二、核心概念详解](#二核心概念详解)
- [三、整体架构](#三整体架构)
- [四、Exchange 路由机制深入](#四exchange-路由机制深入)
- [五、消息存储机制](#五消息存储机制)
- [六、集群架构](#六集群架构)
- [七、高可用机制](#七高可用机制)
- [八、可靠性保证](#八可靠性保证)
- [九、八股面试题精选（15 题）](#九八股面试题精选15-题)
- [十、与 Kafka/RocketMQ 对比](#十与-kafkarocketmq-对比)

---

### 1.2 为什么需要消息队列？—— 三大核心场景

核心对象如下：

总结：RabbitMQ 通过 Exchange 和 Binding 把“消息生产”和“消息消费”解耦。Producer 不直接找 Consumer，Consumer 也不直接依赖 Producer，双方通过队列和路由规则协作。

总结：RabbitMQ 的可靠性不是一个单独开关，而是生产确认、消息持久化、消费确认、重试、死信和业务幂等共同组成的链路。

问题：

```
用户下单 --> 订单服务直接调用 --> 库存服务（HTTP）
                            --> 物流服务（HTTP）
                            --> 积分服务（HTTP）
```

```
用户下单 --> 订单服务 --> 发送"订单已创建"消息到 RabbitMQ
                              |
                              ├── 库存服务订阅消息 --> 扣减库存
                              ├── 物流服务订阅消息 --> 创建运单
                              ├── 积分服务订阅消息 --> 增加积分
                              └── 通知服务订阅消息 --> 发送短信（新增，订单服务无需改动）
```

```
用户注册请求
  --> 写入数据库（50ms）
  --> 发送验证邮件（2000ms）
  --> 初始化推荐数据（1000ms）
  --> 返回响应
总耗时：50 + 2000 + 1000 = 3050ms
```

```
用户注册请求
  --> 写入数据库（50ms）
  --> 发送"用户注册"消息到 RabbitMQ（5ms）
  --> 返回响应
总耗时：50 + 5 = 55ms

```
秒杀开始瞬间：10 万个请求涌入
  --> 直接打到数据库
  --> 数据库每秒只能处理 2000 个请求
  --> 数据库崩溃 --> 系统雪崩
```

```
秒杀开始瞬间：10 万个请求涌入
  --> 全部进入 RabbitMQ 队列（队列能承受百万级写入）
  --> 数据库按自己的节奏从队列消费（每秒 2000 条）
  --> 数据库正常运行
```

```text
Producer 发送消息
  ↓
消息到达 Exchange
  ↓
Exchange 根据类型、Routing Key 和 Binding 规则匹配 Queue
  ↓
消息写入一个或多个 Queue
  ↓
Consumer 从 Queue 获取消息
  ↓
Consumer 执行业务逻辑
  ↓
Consumer 返回 ack，RabbitMQ 删除已确认消息
```

```text
Exchange: order.events
Routing Key: order.created
Queue: order.created.email
Binding: order.created
Consumer: email-service
```

```text
Producer → RabbitMQ → Queue → Consumer
```

```text
Producer 发送消息
  ↓
RabbitMQ 成功接收并处理
  ↓
返回 confirm
  ↓
Producer 才认为发送成功
```

```text
Consumer 收到消息
  ↓
执行业务逻辑
  ↓
业务成功后发送 ack
  ↓
RabbitMQ 删除消息
```

```text
发送端：publisher confirm 确认 broker 收到
  ↓
broker：durable queue + persistent message 尽量防重启丢失
  ↓
消费端：manual ack 确认业务成功后再删除消息
  ↓
失败路径：retry + DLQ 保存异常消息
```

| 对象        | 作用                                    |
| ----------- | --------------------------------------- |
| Producer    | 生产消息的应用程序                      |
| Exchange    | 接收消息并决定投递到哪些队列            |
| Queue       | 保存消息，等待消费者消费                |
| Binding     | 连接 Exchange 和 Queue 的路由规则       |
| Routing Key | 生产者发送消息时带上的路由键            |
| Consumer    | 从 Queue 拉取或接收消息并处理的应用程序 |

| 类型        | 路由方式                             |
| ----------- | ------------------------------------ |
| `direct`  | Routing Key 完全匹配                 |
| `topic`   | Routing Key 按通配符匹配             |
| `fanout`  | 广播到所有绑定队列，忽略 Routing Key |
| `headers` | 根据消息 headers 匹配                |

| 配置               | 解决什么                                |
| ------------------ | --------------------------------------- |
| durable queue      | RabbitMQ 重启后队列定义还在             |
| persistent message | 消息可以写入磁盘，broker 重启后尽量恢复 |

| 机制         | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| 重试         | 临时失败时重新消费，例如外部接口超时                         |
| 最大重试次数 | 防止坏消息无限循环                                           |
| 死信队列 DLQ | 多次失败、过期、被拒绝的消息进入专门队列，便于人工排查或补偿 |

> 分布式锁的完整问答已收入第四部分“锁与并发控制”。

#### 场景一：解耦 —— 让系统之间不再强依赖

**没有消息队列时**：

1. 订单服务需要知道所有下游服务的地址和接口
2. 任何一个下游服务接口变更，订单服务都要改代码
3. 如果积分服务挂了，可能影响下单主流程
4. 新增一个"通知服务"，订单服务又要改代码

**引入消息队列后**：

好处：

1. 订单服务只需要发一条消息，不需要知道下游有谁
2. 下游服务接口变更，不影响订单服务
3. 某个下游服务挂了，不影响其他服务和订单主流程
4. 新增服务只需订阅消息，订单服务零改动

**这就是解耦**：发送方和接收方不直接依赖，通过消息中间件通信。

#### 场景二：异步 —— 让主流程不被耗时操作阻塞

**没有消息队列时（同步调用）**：

用户要等 3 秒多才能看到注册成功，体验很差。

**引入消息队列后（异步处理）**：

后台消费者异步处理：
  --> 发送验证邮件（2000ms）
  --> 初始化推荐数据（1000ms）
```

用户只需要等 55ms 就能看到注册成功，体验大幅提升。耗时的邮件发送和数据初始化在后台异步完成。

**这就是异步**：主流程只做必要的事情，耗时操作放到后台异步处理。

#### 场景三：削峰 —— 让系统不被瞬间流量冲垮

**没有消息队列时**：

**引入消息队列后**：

RabbitMQ 就像一个巨大的缓冲池，把瞬间的高流量变成平稳的低流量，保护后端系统不被冲垮。

**这就是削峰**：高流量时先排队，系统按自己的节奏处理。

#### RabbitMQ 的核心模型是什么？一条消息如何从发送到消费？

**问题**：RabbitMQ 里的 Producer、Exchange、Queue、Binding、Routing Key、Consumer 分别是什么？它们如何组成一条完整消息链路？

**回答**：

RabbitMQ 的核心不是“生产者直接把消息塞给消费者”，而是：**生产者把消息发给交换机，交换机根据规则把消息路由到队列，消费者从队列取消息处理**。

完整流程是：

Exchange 常见类型包括：

例如订单事件可以这样设计：

这样生产者只关心“发生了订单创建事件”，不需要知道邮件服务、积分服务、风控服务各自怎么消费。新增消费者时，只需要新增队列和绑定关系，生产者代码可以不变。

#### RabbitMQ 如何尽量保证消息不丢失？

**问题**：RabbitMQ 的 publisher confirm、持久化队列、持久化消息、consumer ack、死信队列、消息重试分别解决什么问题？

**回答**：

消息可靠性要按链路拆开看。消息可能丢在三个位置：

因此可靠性设计也分三段。

第一段，生产者到 RabbitMQ：使用 publisher confirm。

如果没有 confirm，生产者把消息写到 socket 后就以为成功，但 RabbitMQ 可能还没真正接收，网络或 broker 故障会导致消息丢失。

第二段，RabbitMQ 自身存储：队列和消息都要持久化。

只设置 durable queue 不够，因为队列定义持久化不等于消息本身持久化。消息也需要设置持久化标记。

第三段，Consumer 消费：使用手动 ack。

如果 Consumer 拿到消息就自动 ack，随后业务执行失败或进程崩溃，RabbitMQ 会认为消息已经处理完，消息就可能丢失。手动 ack 的原则是：**业务真正成功后再确认**。

失败处理通常还需要重试和死信队列：

可靠性链路可以总结为：

还要注意：RabbitMQ 的可靠机制降低丢失概率，但不等于业务天然“只执行一次”。网络重试、消费者崩溃、ack 丢失都可能导致重复投递。因此消费者逻辑必须做幂等，例如用业务唯一键、数据库唯一索引、状态机校验来避免重复扣款、重复发货。

---

### 2.7 消息完整流转流程 —— 从发出到被消费的每一步

```
┌─────────┐                                                    ┌─────────┐
│Producer │                                                    │Consumer │
└────┬────┘                                                    └────┬────┘
     │ ① 建立 TCP Connection                                      │
     │ ② 创建 Channel                                             │
     │ ③ 发送消息到 Exchange（携带 Routing Key）                    │
     ▼                                                              │
┌──────────┐                                                        │
│ Exchange │ ④ 查询 Binding 规则表                                   │
│          │ ⑤ 根据类型和 Routing Key 匹配                           │
└────┬─────┘                                                        │
     │ ⑥ 将消息路由到匹配的 Queue                                    │
     ▼                                                              │
┌──────────┐                                                        │
│  Queue   │ ⑦ 存储消息（持久化则写入磁盘）                           │
│          │ ⑧ 等待消费者                                            │
└────┬─────┘                                                        │
     │ ⑨ Broker 将消息推送给 Consumer（Push 模式）                   │
     │    或 Consumer 主动拉取（Pull 模式）                           │
     ├──────────────────────────────────────────────────────────────►│
     │                                                              │
     │ ⑩ Consumer 处理完毕后发送 ACK                                │
     │◄─────────────────────────────────────────────────────────────┤
     │ ⑪ Broker 收到 ACK 后将消息从 Queue 中删除                    │
```

**关键点**：

1. Producer 从不直接往 Queue 里塞消息，**必须经过 Exchange 中转**
2. 一个消息可以被路由到多个 Queue（每个 Queue 收到一个副本）
3. 消息在 Queue 中等待，直到被消费者取走并确认
4. ACK 机制保证消息不丢失——消费者没确认的消息会被重新投递

---

### 3.3 Broker 内部核心进程

```
队列进程崩溃
  → 监督者检测到进程退出
  → 根据重启策略重启队列进程
  → 从持久化存储中恢复消息
  → 其他队列和服务不受影响
```

| 核心进程                    | 职责                      | 说明                                       |
| --------------------------- | ------------------------- | ------------------------------------------ |
| `rabbit_connection`       | 每个 TCP 连接对应一个进程 | 处理连接握手、心跳检测、连接关闭           |
| `rabbit_channel`          | 每个 Channel 对应一个进程 | 处理 AMQP 命令的编解码、消息路由、事务管理 |
| `rabbit_amqqueue_process` | 每个队列一个独立进程      | 管理消息的入队、出队、持久化、消费者管理   |
| `rabbit_exchange`         | 管理交换机的路由逻辑      | 处理消息的路由匹配                         |
| `rabbit_binding`          | 维护绑定关系              | 管理 Exchange 与 Queue 之间的绑定规则      |
| `Mnesia`                  | Erlang 内置分布式数据库   | 存储 Exchange、Queue、Binding 等元数据     |

RabbitMQ Broker 内部有多个核心进程协同工作：

**崩溃恢复流程**：

### 3.4 架构总图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RabbitMQ Broker                              │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│  │Connection│    │Connection│    │Connection│    │Connection│     │
│  │   (TCP)  │    │   (TCP)  │    │   (TCP)  │    │   (TCP)  │     │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘     │
│       │               │               │               │            │
│  ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     │
│  │ Channel │     │ Channel │     │ Channel │     │ Channel │     │
│  │         │     │         │     │         │     │         │     │
│  └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     │
│       │               │               │               │            │
│       ▼               ▼               ▼               ▼            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Exchange Layer                            │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │ Direct  │ │  Topic  │ │ Fanout  │ │ Headers │          │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │  │
│  └───────┼───────────┼───────────┼───────────┼────────────────┘  │
│          │           │           │           │                     │
│          ▼           ▼           ▼           ▼                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Binding Layer                             │  │
│  │              (Exchange ↔ Queue 绑定规则)                     │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                             │                                      │
│                             ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                     Queue Layer                              │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │ Queue A │ │ Queue B │ │ Queue C │ │  DLX Q  │          │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │  │
│  └───────┼───────────┼───────────┼───────────┼────────────────┘  │
│          │           │           │           │                     │
│  ┌───────┴───────────┴───────────┴───────────┴────────────────┐  │
│  │                   Storage Layer                              │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                  │  │
│  │  │   msg_store     │  │  queue_index    │                  │  │
│  │  │  (消息体存储)    │  │  (队列索引)     │                  │  │
│  │  └─────────────────┘  └─────────────────┘                  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   Mnesia / Khepri                            │  │
│  │              (元数据分布式存储)                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

Producer ──► Connection ──► Channel ──► Exchange ──► Binding ──► Queue ──► Consumer
```

---

## 四、Exchange 路由机制深入

### 4.1 Direct Exchange 路由详解

```
Exchange: "payment"
├── Queue: "success_queue"    binding_key = "pay.success"
├── Queue: "fail_queue"       binding_key = "pay.failed"
└── Queue: "all_queue"        binding_key = "pay.#"  ← 这是 Topic 的规则，Direct 不支持
```

- `success_queue` ✅ 匹配（"pay.success" == "pay.success"）
- `fail_queue` ❌ 不匹配（"pay.success" != "pay.failed"）
- `all_queue` ❌ 不匹配（Direct 不支持通配符）

**精确匹配规则**：消息的 `routing_key` 必须与队列绑定的 `binding_key` **完全一致**（区分大小写）。

**实际例子**：

Producer 发送消息 routing_key = "pay.success"：

**默认交换机**：RabbitMQ 有一个默认交换机（名称为空字符串 `""`），它是 Direct 类型。每个队列创建时会自动以队列名作为 binding key 绑定到默认交换机。所以你可以直接用队列名发送消息，不需要手动创建 Exchange。

### 4.2 Topic Exchange 通配符详解

- `*`（星号）：匹配**恰好一个**单词
- `#`（井号）：匹配**零个或多个**单词
- 单词之间用 `.` 分隔

- `*` 严格匹配**一个**单词，不能多也不能少
- `#` 可以匹配**零到任意多个**单词
- `#` 放在末尾可以匹配所有以指定前缀开头的 routing_key

| routing_key             | binding_key      | 是否匹配 | 原因                                     |
| ----------------------- | ---------------- | -------- | ---------------------------------------- |
| `quick.orange.rabbit` | `*.orange.*`   | ✅       | * 各匹配 quick、rabbit                   |
| `quick.orange.rabbit` | `*.*.rabbit`   | ✅       | * 各匹配 quick、orange                   |
| `quick.orange.rabbit` | `#.rabbit`     | ✅       | # 匹配 quick.orange                      |
| `quick.orange.rabbit` | `#`            | ✅       | # 匹配所有                               |
| `quick.orange.rabbit` | `quick.#`      | ✅       | # 匹配 orange.rabbit                     |
| `quick.orange.rabbit` | `quick.orange` | ❌       | routing_key 有三段，binding_key 只有两段 |
| `lazy.orange.rabbit`  | `*.orange.*`   | ✅       | * 各匹配 lazy、rabbit                    |
| `lazy`                | `lazy.#`       | ✅       | # 匹配零个单词                           |

**通配符规则**：

**匹配示例表**：

**`*` 和 `#` 的核心区别**：

### 4.3 死信队列（DLX）深入讲解

```java
// 消费者拒绝消息，且不重新入队
channel.basicReject(deliveryTag, false);  // requeue = false
// 或
channel.basicNack(deliveryTag, false, false);  // multiple = false, requeue = false
```

```java
// 队列级别设置 TTL
Map<String, Object> args = new HashMap<>();
args.put("x-message-ttl", 60000);  // 60 秒
channel.queueDeclare("my_queue", true, false, false, args);

```java
Map<String, Object> args = new HashMap<>();
args.put("x-max-length", 1000);  // 队列最多 1000 条消息
channel.queueDeclare("my_queue", true, false, false, args);
// 当队列满时，最早的消息被丢弃（变成死信）
```

```java
// 1. 创建死信交换机
channel.exchangeDeclare("dlx_exchange", "direct");

```
Producer --> [business_queue: TTL=30s, DLX=dlx_exchange]
                  |
                  | (30 秒后 TTL 过期)
                  ▼
            [dlx_exchange] --routing_key--> [dlx_queue] --> Consumer（延迟消费）
```

**死信（Dead Letter）** 是指在以下三种情况下变为"死信"的消息：

**情况一：消息被拒绝**

**情况二：消息 TTL 过期**

// 或消息级别设置 TTL
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .expiration("60000")  // 60 秒
    .build();
channel.basicPublish("", "my_queue", props, message.getBytes());
```

**情况三：队列达到最大长度**

**死信队列的配置**：

// 2. 创建业务队列，绑定死信交换机
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx_exchange");      // 死信交换机
args.put("x-dead-letter-routing-key", "dlx_routing_key"); // 死信路由键
args.put("x-message-ttl", 30000);                         // TTL 30 秒
channel.queueDeclare("business_queue", true, false, false, args);

// 3. 创建死信队列，绑定到死信交换机
channel.queueDeclare("dlx_queue", true, false, false, null);
channel.queueBind("dlx_queue", "dlx_exchange", "dlx_routing_key");
```

**完整流程**：

### 4.4 延迟队列实现方案详解

- 不同延迟时间需要创建多个队列（因为一条消息只有一个 TTL）
- 大量消息同时设置相同 TTL 会导致集中过期，造成"消息雪崩"
- 无法精确控制延迟顺序

```java
// 声明延迟交换机
Map<String, Object> args = new HashMap<>();
args.put("x-delayed-type", "direct");
channel.exchangeDeclare("delayed_exchange", "x-delayed-message", true, false, args);

| 方式                         | 说明                           | 优先级               |
| ---------------------------- | ------------------------------ | -------------------- |
| 队列级别 `x-message-ttl`   | 整个队列中所有消息统一过期时间 | 基础                 |
| 消息级别 `expiration` 属性 | 每条消息单独设置过期时间       | 两者同时设置取较小值 |

| 场景                     | 延迟时间           | 说明                                       |
| ------------------------ | ------------------ | ------------------------------------------ |
| **订单超时未支付** | 30 分钟            | 下单后 30 分钟未支付自动取消订单并释放库存 |
| **会议/预约提醒**  | 自定义             | 提前 N 分钟发送提醒通知                    |
| **重试退避机制**   | 递增（5s/10s/30s） | 消费失败后延迟递增重试                     |
| **数据最终一致性** | N 秒               | 延迟检查分布式事务中各节点数据是否一致     |

**方案一：TTL + DLX（经典方案）**

**原理**：消息在业务队列中等待 TTL 过期后，自动进入死信队列，消费者从死信队列消费实现延迟。

**实现步骤**：

1. 创建死信交换机和死信队列（消费者监听此队列）
2. 创建业务队列，绑定 `x-dead-letter-exchange` 和 `x-message-ttl`
3. 生产者将消息发送到业务队列
4. 消息在业务队列中等待 TTL 到期后，自动被转发到 DLX
5. 消费者监听死信队列，实现延迟消费

**TTL 的两种设置方式**：

**方案一的局限**：

**方案二：`rabbitmq_delayed_message_exchange` 插件（推荐）**

RabbitMQ 3.8+ 提供的插件，通过 `x-delay` 头部直接在交换机层面实现延迟，支持任意延迟时间，无需多个队列。

// 发送延迟消息
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .headers(Map.of("x-delay", 5000))  // 延迟 5 秒
    .build();
channel.basicPublish("delayed_exchange", "routing_key", props, message.getBytes());
```

**延迟队列实际应用场景**：

---

## 五、消息存储机制

### 5.1 消息到了 Broker 后存在哪里？

- 负责消息体（payload）的持久化存储
- 所有队列**共享**同一个 msg_store
- 按消息 ID 索引
- 消息体存储在段文件（segment file，后缀 `.rdq`）中，每个约 16MB
- 采用**追加写入（append-only）**策略，写入性能高

- 负责维护每条消息在队列中的元数据和位置信息
- 包括消息是否已确认（ack）、在 msg_store 中的偏移位置等
- 每个队列有自己独立的 queue_index
- 同样使用段文件，每个段文件包含约 2048 条消息记录
- 支持批量写入以提高 I/O 性能

```
消息到达 Broker
  → 写入队列进程的内存缓冲区
  → 如果消息标记为持久化（delivery_mode=2）：
      → 同步写入 msg_store（消息体）
      → 同步写入 queue_index（元数据）
  → 对于非常小的消息（低于 queue_index_embed_msgs_below 阈值）：
      → 消息体直接嵌入 queue_index 中，避免额外的 msg_store 读取
```

消息到达 Broker 后，存储依赖两个核心模块协同工作：

**rabbit_msg_store（消息存储）**

**rabbit_queue_index（队列索引）**

**消息存储流程**：

### 5.3 持久化是怎么实现的？

```
Producer 发送消息（delivery_mode=2）
  → Broker 收到消息
  → 写入内存缓冲区
  → 同步写入 msg_store 的段文件（.rdq）
  → 同步写入 queue_index 的段文件
  → 返回 Publisher Confirm（如果开启了的话）
```

```
段文件 1: [消息A ✅已确认] [消息B ✅已确认] [消息C ✅已确认]  → 全部确认，整个文件被回收
段文件 2: [消息D ✅已确认] [消息E ❌未确认] [消息F ✅已确认]  → 还有未确认，不能回收
```

| 组件   | 持久化方式                                   | 不设置的后果            |
| ------ | -------------------------------------------- | ----------------------- |
| 交换机 | 声明时设置 `durable=true`                  | Broker 重启后交换机丢失 |
| 队列   | 声明时设置 `durable=true`                  | Broker 重启后队列丢失   |
| 消息   | 发送时设置 `delivery_mode=2`（PERSISTENT） | Broker 重启后消息丢失   |

**持久化需要三个条件同时满足**：

**持久化写入流程**：

**段文件的垃圾回收**：

单条消息的 ack 不会立即释放磁盘空间，而是等整个段文件可回收时批量释放。

### 5.4 惰性队列（Lazy Queue）

- 消息可能大量积压（如日志收集、非实时消费）
- 消费者处理速度远低于生产者
- 内存资源有限

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-mode", "lazy");
channel.queueDeclare("my_lazy_queue", true, false, false, args);
```

**核心思想**：消息优先写入磁盘，尽量不驻留内存。

**适用场景**：

**配置方式**：

**从 RabbitMQ 3.12 开始**，经典队列的默认行为已优化为类似惰性队列的智能内存管理策略，不再需要显式设置 `x-queue-mode: lazy`。

**Quorum Queue** 基于 Raft 协议的日志存储天然具有类似惰性队列的特征——消息始终写入磁盘。

### 5.5 消息什么时候从磁盘删除？

```
消费者发送 basicAck
  → RabbitMQ 在 queue_index 中将该消息标记为已删除
  → 当一个段文件中的所有消息都被确认删除后
  → 整个段文件被垃圾回收
  → 磁盘空间才真正释放
```

**注意**：单条消息的 ack 不会立即释放磁盘空间，而是等整个段文件可回收时批量释放。这意味着如果有少量消息长期未确认，会导致整个段文件无法回收。

---

## 六、集群架构

### 6.1 什么是集群？为什么需要集群？

**集群**是将多个 RabbitMQ 节点组成一个逻辑整体，共享 Exchange、Queue、Binding 等元数据。

**为什么需要集群**：

1. **水平扩展**：单节点性能有限，集群可以分散负载
2. **高可用**：单节点宕机会导致服务中断，集群可以在部分节点故障时继续服务
3. **容量扩展**：单节点内存和磁盘有限，集群可以存储更多消息

### 6.2 集群中节点的类型

| 节点类型           | 说明               | 特点                                             |
| ------------------ | ------------------ | ------------------------------------------------ |
| **内存节点** | 元数据仅存内存     | 启动快，不持久化元数据，重启后需要从其他节点同步 |
| **磁盘节点** | 元数据持久化到磁盘 | 启动慢，但重启后可以独立恢复                     |

**注意**：集群中至少需要一个磁盘节点，否则集群无法正常工作。

**所有节点都是对等的**：每个节点都能独立处理客户端连接和消息路由。没有主节点/从节点的概念（在元数据层面）。

### 6.3 元数据同步机制

- **RabbitMQ 3.x**：使用 Erlang 内置的 **Mnesia** 分布式数据库同步元数据
- **RabbitMQ 4.0+**：使用 **Khepri**（基于 Raft 协议的分布式存储），替代 Mnesia

**元数据**包括：Exchange 的定义、Queue 的定义、Binding 的关系、vhost 的配置等。

**同步方式**：

元数据在集群中**全节点复制**，任何节点都可以处理任何队列的元数据查询。

### 6.4 队列数据的分布策略

```
集群节点 A                集群节点 B                集群节点 C
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│ Queue A     │         │ Queue A     │         │ Queue A     │
│ (元数据+数据) │         │ (仅元数据)   │         │ (仅元数据)   │
│             │         │             │         │             │
│ Queue B     │         │ Queue B     │         │ Queue B     │
│ (仅元数据)   │         │ (元数据+数据) │         │ (仅元数据)   │
└─────────────┘         └─────────────┘         └─────────────┘
```

**关键点**：队列中的**消息数据默认只存储在声明该队列的节点上**，其他节点只存储队列的元数据。

**这意味着**：

1. 如果队列所在的节点宕机，该队列的消息将不可访问（除非使用 Quorum Queue）
2. 消费者连接到非队列所在节点时，消息会通过集群内部转发（有额外开销）
3. 为了减少内部转发，建议消费者连接到队列所在的节点

### 6.5 网络分区（Split Brain）处理策略

```
正常集群：  [Node A] -- [Node B] -- [Node C]

#### 策略一：ignore（忽略）

- **原理**：不处理网络分区，等网络恢复后手动解决
- **优点**：简单
- **缺点**：可能导致数据不一致，两个子群可能同时修改同一份数据
- **推荐**：**不推荐**在生产环境使用

#### 策略二：pause-minority（暂停少数派）

```
分区前：[Node A] -- [Node B] -- [Node C]

#### 策略三：autoheal（自动恢复）

- **原理**：网络恢复后自动选择一个分区保留，丢弃另一个分区的数据
- **优点**：全自动，不需要人工干预
- **缺点**：可能丢数据
- **推荐**：适合对数据一致性要求不高的场景

- **原理**：少数派节点自动暂停，多数派继续服务
- **条件**：需要 3 个或更多节点
- **优点**：避免脑裂，数据安全
- **缺点**：少数派节点暂停期间不可用
- **推荐**：**推荐**在生产环境使用

**什么是网络分区**：

集群中的节点因为网络故障被分割成多个子群，每个子群都认为自己是正确的集群。

网络分区后：[Node A] -- [Node B]    [Node C]
           子群 1                子群 2
```

**三种处理策略**：

分区后：[Node A] -- [Node B]    [Node C]
        多数派（继续服务）       少数派（自动暂停）
```

**推荐**：生产环境使用 `pause-minority` 策略，配合 3 个或 5 个节点。

---

## 七、高可用机制

### 7.1 什么是高可用？为什么单节点不够？

- 节点宕机 → 所有消息处理中断
- 磁盘损坏 → 所有持久化消息丢失
- 内存不足 → 无法处理新消息

**高可用（High Availability, HA）** 是指系统在部分组件故障时仍能继续提供服务。

**单节点的问题**：

**高可用的目标**：即使某个节点宕机，消息处理仍能继续，数据不丢失。

### 7.3 Quorum Queue（仲裁队列）—— 推荐方案

- **Leader**：负责处理所有读写请求
- **Follower**：接收 Leader 的日志复制
- **Candidate**：竞选 Leader 的节点

```
Producer --> [Leader: Queue A] --复制--> [Follower 1: Queue A]
                                   └--> [Follower 2: Queue A]
         |
         | 等待多数派确认（3 个节点中至少 2 个确认）
         ▼
    返回 Publisher Confirm

- Leader 宕机后，剩余节点通过 Raft 协议选举新 Leader
- 只要存活节点过半，就能选出新 Leader
- 选举过程通常在毫秒级完成

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
channel.queueDeclare("my_quorum_queue", true, false, false, args);
```

| 特性                 | 镜像队列                     | Quorum Queue           |
| -------------------- | ---------------------------- | ---------------------- |
| **一致性协议** | 无（主从复制）               | Raft 共识              |
| **数据安全性** | 可能丢消息                   | 强一致，不丢消息       |
| **脑裂风险**   | 有                           | 无（Raft 保证）        |
| **消息回溯**   | 不支持                       | 不支持                 |
| **性能**       | 较好                         | 略低（需要多数派确认） |
| **状态**       | **已废弃（4.0 移除）** | **推荐使用**     |

**原理**：基于 **Raft 共识协议**，每个队列有多个副本（通常 3 个或 5 个），写入需要**多数派确认**。

**什么是 Raft 协议？**

Raft 是一种分布式共识算法，保证在部分节点故障的情况下，集群仍能达成一致。核心概念：

**Quorum Queue 的工作流程**：

Consumer <-- [Leader: Queue A]
```

**写入流程**：

1. Producer 将消息发送到 Leader
2. Leader 将消息写入本地日志
3. Leader 将日志复制到所有 Follower
4. **多数派**（过半数）Follower 确认后，消息才算写入成功
5. Leader 返回 Publisher Confirm 给 Producer

**Leader 选举**：

**与镜像队列的对比**：

**Quorum Queue 的配置**：

### 7.4 Stream（流队列）

- 消息被消费后就删除
- 每条消息只能被一个消费者处理
- 不支持消息回溯

- 消息被消费后**不删除**（非破坏性读取）
- 多个消费者可以独立读取同一份数据
- 支持按 offset 或时间戳回溯
- 消息持久化在磁盘上，可以保留很长时间

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "stream");
channel.queueDeclare("my_stream", true, false, false, args);
```

| 特性               | 传统队列（Quorum Queue）   | Stream                           |
| ------------------ | -------------------------- | -------------------------------- |
| **消费模式** | 消费即删除                 | 非破坏性读取，消息保留           |
| **消息回溯** | 不支持                     | 支持（按 offset 或时间戳）       |
| **多消费者** | 每条消息只被一个消费者处理 | 多个消费者可以独立读取同一份数据 |
| **吞吐量**   | 中等                       | 高（类似 Kafka）                 |
| **适用场景** | 任务队列、工作流           | 事件溯源、日志收集、广播         |

**是什么**：RabbitMQ 3.9+ 引入的新型队列类型，类似 Kafka 的**追加日志（append-only log）**。

**与传统队列的本质区别**：

传统队列（Quorum Queue）：

Stream：

**与传统队列的对比**：

**Stream 的配置**：

## 八、可靠性保证

### 8.1 消息可能丢失的三个环节

```
生产者 --[环节1]--> Broker --[环节2]--> 消费者
         网络/代码异常    存储/宕机    ACK/处理异常
```

| 丢失原因              | 说明               |
| --------------------- | ------------------ |
| 网络抖动              | 消息未到达 Broker  |
| 发送代码异常          | 发送失败但未感知   |
| 连接/Channel 异常关闭 | 消息在传输途中丢失 |

| 丢失原因          | 说明              |
| ----------------- | ----------------- |
| 消息仅存内存      | Broker 宕机后丢失 |
| 单节点磁盘损坏    | 持久化消息丢失    |
| 队列满 / TTL 过期 | 消息被主动丢弃    |
| 内存告警触发流控  | 消息被拒绝        |

| 丢失原因              | 说明                             |
| --------------------- | -------------------------------- |
| 自动 ACK              | 消息刚收到还未处理完消费者就宕机 |
| nack 后未正确 requeue | 消息被丢弃                       |
| ACK 前网络断开        | Broker 重复投递                  |

**环节 1：生产端（Producer → Broker）**

**环节 2：Broker 端（存储阶段）**

**环节 3：消费端（Broker → Consumer）**

### 8.2 消息确认机制详解：ACK、NACK、Reject

```java
// nack 支持批量拒绝
channel.basicNack(deliveryTag, true, false);  // multiple=true，拒绝 deliveryTag 及之前的所有未确认消息

```java
// 手动 ACK 模式
channel.basicConsume("my_queue", false, (tag, delivery) -> {
    try {
        // 处理业务逻辑
        processMessage(delivery.getBody());
        // 处理成功，确认消息
        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
    } catch (Exception e) {
        // 处理失败，拒绝消息，不重新入队（进入死信队列）
        channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
    }
});
```

| 操作             | 含义                   | 效果                                                         | 使用场景                     |
| ---------------- | ---------------------- | ------------------------------------------------------------ | ---------------------------- |
| `basic.ack`    | 确认消息已被成功处理   | Broker 从队列中删除该消息                                    | 业务处理成功                 |
| `basic.nack`   | 否定确认，表示处理失败 | 可选择 requeue=true 重新入队，或 requeue=false 丢弃/进入 DLX | 业务处理失败，需要重试或丢弃 |
| `basic.reject` | 拒绝消息               | 与 nack 类似，但不支持批量操作                               | 业务处理失败，拒绝单条消息   |

**nack 和 reject 的核心区别**：

// reject 只能拒绝单条
channel.basicReject(deliveryTag, false);  // 只拒绝 deliveryTag 这一条
```

**最佳实践**：始终使用**手动 ACK** 模式（`autoAck=false`），在业务逻辑处理成功后再调用 `basicAck`。

### 8.3 Publisher Confirm 与 Consumer ACK 的区别

- Publisher Confirm 保证消息安全到达 Broker
- Consumer ACK 保证消息被消费者正确处理
- 两者结合使用可实现**至少一次（at-least-once）**投递语义

```java
// 方式一：同步确认（性能最差）
channel.confirmSelect();
channel.basicPublish(...);
if (!channel.waitForConfirms()) {
    // 消息发送失败，重试
}

| 特性                | Publisher Confirm            | Consumer ACK                   |
| ------------------- | ---------------------------- | ------------------------------ |
| **方向**      | Broker → 生产者（回调通知） | 消费者 → Broker               |
| **保证什么**  | Broker 已接收并存储了消息    | 消费者已成功处理了消息         |
| **异步/同步** | 完全异步（回调机制）         | 消费者显式调用，通常是同步的   |
| **默认行为**  | 默认关闭，需显式启用         | 取决于客户端，部分默认 autoAck |
| **覆盖阶段**  | 生产者 → Broker 这一段      | Broker → 消费者这一段         |
| **粒度**      | 按 sequence number 跟踪      | 按 delivery_tag，支持批量      |

**两者是互补关系**：

**Publisher Confirm 的三种方式**：

// 方式二：异步确认（推荐）
channel.confirmSelect();
channel.addConfirmListener(
    (sequenceNumber, multiple) -> {
        // 消息发送成功
    },
    (sequenceNumber, multiple) -> {
        // 消息发送失败，重试
    }
);

// 方式三：批量确认（性能最好，但可能丢失少量消息）
channel.confirmSelect();
channel.basicPublish(...);
channel.waitForConfirmsOrDie(5000);  // 批量等待，超时 5 秒
```

### 8.4 完整的"消息不丢失"方案

```java
// 开启 Publisher Confirm
channel.confirmSelect();

```java
// 设置 QoS，每次最多预取 10 条消息
channel.basicQos(10);
```

**生产端**：

1. 开启 **Publisher Confirm**（`confirm.select`），通过异步回调确认消息到达 Broker
2. 设置 `mandatory=true`，消息无法路由时通过 **Return 回调**通知生产者
3. 结合**本地消息表 + 定时重发**作为兜底，confirm 失败的消息存入数据库后续补偿

// 设置 mandatory
channel.basicPublish("exchange", "routing_key", true, props, body);

// Return 回调（消息无法路由时触发）
channel.addReturnListener(returnMessage -> {
    // 消息无法路由，记录到数据库，后续补偿
});
```

**Broker 端**：

1. 队列声明 `durable=true`，消息设置 `delivery_mode=2`（持久化）
2. 使用 **Quorum Queue**（仲裁队列），基于 Raft 协议多节点复制
3. 配置**死信队列（DLX）**兜底消费失败的消息

**消费端**：

1. 使用**手动 ACK**模式，在业务处理成功后才调用 `basicAck`
2. 处理失败时调用 `basicNack`，配合 requeue 或死信队列
3. 设置合理的 `prefetchCount`（QoS），控制未确认消息数量，避免消费者过载

### 8.5 消息幂等性

- 消费者处理完消息后发送 ACK 时网络中断，Broker 未收到 ACK 会重新投递
- 生产者因网络超时重试发送了重复消息
- 消费者超时未确认，消息被 requeue 后再次被消费

```java
// 生产者：为每条消息生成全局唯一 ID
String messageId = UUID.randomUUID().toString();
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .messageId(messageId)
    .build();
channel.basicPublish("exchange", "routing_key", props, body);

| 方案                             | 适用场景         | 优点             | 缺点         |
| -------------------------------- | ---------------- | ---------------- | ------------ |
| **唯一 ID + Redis SETNX**  | 高并发场景       | 性能好、实现简单 | 依赖 Redis   |
| **唯一 ID + 数据库去重表** | 数据量小         | 强一致           | 性能较低     |
| **数据库乐观锁 / 版本号**  | 更新操作场景     | 无额外组件       | 需要版本字段 |
| **状态机 + 业务流水号**    | 有状态的业务流程 | 业务语义清晰     | 实现较复杂   |

**什么是幂等性**：同一条消息被消费一次和消费多次，对系统状态的影响完全相同。

**为什么需要**：RabbitMQ 保证的是"至少一次"投递（at-least-once），以下情况会导致重复消费：

**实现方案**：

**业界最常用方案**：唯一 ID + Redis SETNX

// 消费者：处理前先检查是否已消费
String messageId = properties.getMessageId();
Boolean isNew = redis.setnx("msg:" + messageId, "1", 24, TimeUnit.HOURS);
if (isNew) {
    // 首次消费，处理业务逻辑
    processMessage(body);
    channel.basicAck(tag, false);
} else {
    // 重复消费，直接确认
    channel.basicAck(tag, false);
}
```

**一句话总结**：生产者为每条消息生成全局唯一 `messageId`，消费者处理前先通过 Redis `SETNX` 判断是否已消费（设置过期时间），实现 **"at-least-once + 幂等消费"** 的组合方案。
