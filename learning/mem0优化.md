可以。我刚对照了 mem0 当前 GitHub `main` 分支和官方 V2→V3 migration 文档。你贴的文章**大方向是对的**，但它一下子塞了太多实现细节，所以很难形成直觉。

我建议你先不要管 sigmoid、entity boost、9 阶段这些细节。**先抓住 V3 真正改了哪两个地方。**

## 先用一句话理解 V2 → V3

V2：

```text
写入时很聪明
检索时比较简单
```

V3：

```text
写入时尽量简单
检索时变聪明
```

官方迁移文档也把核心变化概括为：**ADD-only 单次提取 + semantic/BM25/entity hybrid search**。([GitHub](https://github.com/mem0ai/mem0/blob/main/docs/migration/oss-v2-to-v3.mdx?utm_source=chatgpt.com))

------

# 一、先看 V2 到底有什么问题

假设用户先说：

```text
我喜欢喝牛奶。
```

后来又说：

```text
我最近不喝牛奶了，改喝燕麦奶。
```

## V2 的思路

V2 会先抽取：

```text
用户现在喝燕麦奶。
```

然后搜索已有记忆：

```text
用户喜欢喝牛奶。
```

再把：

```text
旧记忆：用户喜欢喝牛奶
新事实：用户现在喝燕麦奶
```

一起交给 LLM：

```text
你判断一下：

ADD？
UPDATE？
DELETE？
NONE？
```

LLM 可能决定：

```text
UPDATE

用户喜欢喝牛奶
↓
用户喜欢喝燕麦奶
```

所以 V2 的核心实际上是：

```text
Conversation
    ↓
LLM 提取事实
    ↓
搜索旧 Memory
    ↓
LLM 判断 ADD / UPDATE / DELETE / NONE
    ↓
修改 Memory DB
```

你可以把它理解成：

> **LLM 不仅负责理解信息，还负责维护数据库状态。**

这就带来一个很明显的问题：

```text
今天模型认为 UPDATE
明天模型可能认为 ADD
换个模型可能认为 NONE
```

**Memory DB 的最终状态依赖 LLM 的判断。**

而且 V2 更新 prompt 在当前代码里确实还保留着，可以看到 `ADD / UPDATE / DELETE / NONE` 这套逻辑；但 V3 主路径已经改成 additive extraction。([GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com))

------

# 二、V3 第一刀：干脆不让 LLM 管数据库了

这是我认为 V3 **最重要的优化**。

还是：

```text
以前：
用户喜欢牛奶

现在：
用户改喝燕麦奶
```

V3 不再问：

```text
我要不要修改「用户喜欢牛奶」？
```

而是直接：

```text
Memory 1:
用户以前喜欢喝牛奶

Memory 2:
用户后来从牛奶改成燕麦奶
```

也就是：

```text
旧状态不动
新事实继续 ADD
```

代码里的 `ADDITIVE_EXTRACTION_PROMPT` 明确把角色定义成 ADD-only；`main.py` 当前写入路径也是先检索一批已有 memory 给提取 prompt 作上下文，然后**一次 LLM 调用完成 extraction**。([GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com))

所以你可以先记：

> **V2 = Mutable Memory**
>
> **V3 = Append-oriented Memory**

这和日志系统其实有一点像。

你做过日志/ClickHouse，可以这样理解：

### V2

像维护一张：

```text
user_current_state
```

不断：

```sql
UPDATE user_state ...
DELETE ...
INSERT ...
```

### V3

更像：

```text
event log
```

不断追加：

```text
2025-01 喜欢牛奶
2025-06 开始喝燕麦奶
2026-01 更喜欢豆奶
```

先**保存事实演化过程**，查询的时候再判断哪些信息最相关。

------

# 三、为什么这样反而可能更准？

这点非常关键。

很多人第一反应都是：

> 「不 UPDATE，那旧记忆不是越来越多吗？」

是的。

但 V3 实际上是在做一个取舍：

```text
V2：

写入时
    ↓
必须正确判断新旧关系
    ↓
一旦判断错
    ↓
可能直接破坏历史 Memory
```

例如：

```text
Memory:
用户住南京

用户：
我下周去上海出差。
```

LLM 如果误判成：

```text
UPDATE:
用户住上海
```

原来的正确信息就被覆盖掉了。

而 V3：

```text
Memory A:
用户住南京

Memory B:
用户下周去上海出差
```

哪怕 Memory B 的处理不够完美：

> **至少不会破坏 Memory A。**

所以 V3 的思想其实非常符合你之前研究 Memory Adjudication 时碰到的问题：

> **如果 LLM 对“冲突/更新关系”的判断不够可靠，就不要让这个概率判断直接产生 destructive mutation。**

这才是 ADD-only 真正重要的地方。

------

# 四、但是 V3 马上产生了一个新问题

这就是整套设计最精彩的地方。

如果永远 ADD：

```text
用户喜欢牛奶
用户改喝燕麦奶
用户买了燕麦奶
用户推荐燕麦奶
用户后来尝试豆奶
用户觉得燕麦奶不错
……
```

Memory 会越来越多。

于是问题从：

```text
写入时怎么维护正确 Memory
```

变成：

```text
搜索时怎么从大量 Memory
找到真正需要的那几条？
```

所以：

> **V3 并没有消灭复杂度，而是把复杂度从 Write Path 搬到了 Read Path。**

这就是你那篇文章最值得记住的一句话。

------

# 五、所以 V3 第二刀：大幅加强 Search

以前可以粗略理解成：

```text
query
 ↓
Embedding
 ↓
Vector Search
 ↓
Top K
```

例如：

```text
用户问：
我之前去 Paris 玩是什么时候？
```

如果 Memory 写的是：

```text
用户 2025 年 5 月去了巴黎旅游
```

纯 embedding 大概率能找到。

但真实世界没这么简单。

------

# 六、为什么只靠向量搜索不够？

假设 Memory：

```text
M1：用户喜欢 iPhone 17 Pro
M2：用户喜欢苹果派
M3：用户计划购买苹果公司的 MacBook
```

查询：

```text
iPhone 17 Pro
```

这里存在三类信息。

### 语义

```text
Embedding Similarity
```

判断：

```text
iPhone ↔ 手机
跑步 ↔ jogging
```

比较强。

------

### 关键词

如果用户明确搜索：

```text
iPhone 17 Pro
```

这个字符串本身非常有信息量。

所以 BM25：

```text
iPhone
17
Pro
```

可能比纯 embedding 更靠谱。

------

### 实体

还可以识别：

```text
Apple
iPhone 17 Pro
Paris
OpenAI
张三
南京
```

如果 query 和 Memory 都包含：

```text
iPhone 17 Pro
```

这是一个很强的相关性信号。

因此 V3：

```text
                 Query
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
   Semantic       BM25       Entity
       │           │           │
       └───────────┼───────────┘
                   ↓
             Score Fusion
                   ↓
                 Top K
```

当前官方文档明确把它叫作 **multi-signal hybrid search**。([GitHub](https://github.com/mem0ai/mem0/blob/main/docs/migration/oss-v2-to-v3.mdx?utm_source=chatgpt.com))

------

# 七、你可以把三路信号理解成三个“评委”

比如搜索：

```text
我之前用的 ClickHouse 索引是什么？
```

Memory A：

```text
用户在日志项目中使用 ClickHouse 的 minmax、Bloom Filter 和跳数索引。
```

三个评委分别评分：

```text
语义评委：

ClickHouse / 索引 / 日志
→ 0.86


BM25 评委：

ClickHouse 精确命中
索引命中
→ 0.91


Entity 评委：

ClickHouse entity 命中
→ +0.5
```

最后：

```text
Semantic
   +
BM25
   +
Entity Boost
   ↓
Combined Score
```

而当前 `score_and_rank` 的设计确实是把 semantic、归一化后的 BM25、entity boost 做加性融合。([GitHub](https://github.com/BANG644/project-collection/blob/main/full-analysis/mem0ai-mem0-深度调研.md?utm_source=chatgpt.com))

这其实就是：

> **不要相信任何一个 Retriever，让几个不同性质的 Retriever 互相补盲区。**

------

# 八、Over-fetch 又是在解决什么？

这个也非常容易理解。

假设最终只需要：

```text
Top 5
```

如果 Semantic Search 一开始就只返回：

```text
5 条
```

那么 BM25 和 Entity 只能在这 5 条里重新排序。

问题是：

```text
真正相关的 Memory
可能 Semantic 排第 12

但是：
BM25 非常高
Entity 又完全命中
```

你一开始 Top5 就把它扔了：

```text
后面再牛逼的融合算法
也救不回来。
```

所以 V3：

```text
最终想要 Top 5

        ↓

Semantic 先拿 60 条左右

        ↓

BM25 / Entity 一起重新打分

        ↓

再选 Top 5
```

当前代码的候选池策略确实是：

```text
internal_limit = max(limit * 4, 60)
```

即至少给 hybrid scoring 留一个较宽的候选池。([GitHub](https://github.com/mem0ai/mem0/discussions/6726?utm_source=chatgpt.com))

这个设计你应该很熟悉：

> **Recall → Rank**

本质和 RAG 两阶段检索一样。

------

# 九、Threshold 前置，我反而建议你重点理解

V3 有个很有意思的决定：

```text
Semantic < threshold
        ↓
直接淘汰
```

而不是：

```text
Semantic 很低

但 BM25 很高
Entity 很高

        ↓
救回来
```

为什么？

例如：

```text
Query：
我用的苹果手机是什么型号？

Memory：
用户很喜欢苹果派。
```

可能出现：

```text
BM25：
“苹果”完全命中

Entity：
Apple/苹果可能也产生某些关联

Semantic：
很低
```

如果：

```text
semantic + BM25 + entity
```

什么都允许互相救：

```text
苹果派
```

可能因为关键词太强混进来。

所以 mem0 当前 scoring 是先做：

```text
if semantic_score < threshold:
    continue
```

之后才进行其他信号融合。([GitHub](https://github.com/mem0ai/mem0/issues/7068?utm_source=chatgpt.com))

你可以把 Semantic Threshold 理解成：

> **入场券。**

BM25 和 Entity 是：

> **进场后的加分项。**

于是：

```text
Semantic
   ↓
相关性底线
   ↓
 ┌───────┐
 │ 通过？ │
 └───────┘
   ↓ Yes
BM25 + Entity
   ↓
重新排序
```

这样就很好理解了。

------

# 十、Prompt 的优化其实也非常重要

这一块你贴的文章写得有点复杂。

本质只有一句：

> **Memory 不仅要“存得对”，还要“以后搜得到”。**

例如原始对话：

```text
我上周去了 Paris，
住的是 Ritz，
在那里待了 3 天。
```

差的 Memory：

```text
用户最近出去旅游了。
```

这句话没有事实错误。

但它作为 Memory **几乎是垃圾**。

以后搜索：

```text
Paris
Ritz
3 天
什么时候去的
```

全部很难命中。

好的 Memory：

```text
用户在 2026 年 9 月去了 Paris，
入住 Ritz，并停留 3 天。
```

里面保留：

```text
Paris
Ritz
2026-09
3 天
```

这几个东西就是：

> **Retrieval Anchor**

当前 `ADDITIVE_EXTRACTION_PROMPT` 确实要求提取 self-contained、context-rich 的 memory，并使用已有 memory 做去重/关联上下文。([GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/configs/prompts.py?utm_source=chatgpt.com))

所以你以后设计自己的 Memory Extraction Prompt，也应该有这个意识：

```text
Conversation
      ↓
不是简单 Summary
      ↓
而是生成
Retrieval-Friendly Memory
```

这个观点我认为非常值得你借鉴。

------

# 十一、所以把 V3 整个系统压缩成这张图

```text
                    Conversation
                         │
                         ▼
                 Memory Extraction
                    只负责提取
                         │
                    ADD-only
                         │
                         ▼
              ┌──────────────────┐
              │   Memory Store   │
              │                  │
              │ M1: ...          │
              │ M2: ...          │
              │ M3: ...          │
              │ M4: ...          │
              │ ...不断积累       │
              └────────┬─────────┘
                       │
                       │ Query
                       ▼
                Over-fetch 候选
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Semantic        BM25        Entity
          │            │            │
          └────────────┼────────────┘
                       │
                 Score Fusion
                       │
                Semantic Gate
                       │
                       ▼
                     Top K
                       │
                 optional rerank
                       │
                       ▼
               给 Agent / LLM
```

这就是 V3 的主体。

------

# 十二、从 Agent Memory 设计角度，我认为最值得你学的是 4 件事

不是具体的 sigmoid 参数，那些东西以后都可能改。真正值得你吸收到自己 Agent 项目里的，是：

1. **尽量减少 LLM 对 Memory DB 做破坏性决策。** LLM 很适合 extraction，但让它直接决定 DELETE/UPDATE，需要非常谨慎。
2. **Memory Extraction 要为未来 Retrieval 服务。** Memory 不是普通 summary，要保留实体、时间、数字、状态变化这些 retrieval anchors。
3. **Memory Retrieval 不要只依赖 embedding。** `Dense + Sparse + Entity + 可选 Reranker` 是比单纯 vector search 更稳健的思路。
4. **复杂 Retrieval 一定要可观测。** `explain=True` 能返回 semantic、BM25、entity boost、combined score 等细节，这对于你做 Memory Evaluation / Badcase Analysis 特别有价值。官方搜索文档已经暴露了这套 `score_details`。([GitHub](https://github.com/mem0ai/mem0/blob/main/docs/core-concepts/memory-operations/search.mdx?utm_source=chatgpt.com))

------

## 但我发现你这篇文章有几个地方值得警惕

尤其如果你是准备用它来学习源码。

第一，**V3 不是简单的“完全不去重、疯狂 ADD”**。当前 `add()` 会先检索相关已有 memory，把它们传给 additive extraction prompt，让模型避免重新抽取语义等价内容；另外还有确定性去重逻辑。因此更准确的说法应该是：

> **不再通过 ADD/UPDATE/DELETE/NOOP 维护唯一当前事实，而不是完全取消 dedup。**

([GitHub](https://github.com/mem0ai/mem0/blob/main/mem0/memory/main.py?utm_source=chatgpt.com))

第二，`linked_memory_ids` 这一块当前还有一个值得关注的 OSS issue：有人指出 prompt 会让 LLM 生成 memory link，但 Python add pipeline 中该字段可能没有真正进入存储路径。这意味着**“设计意图”和“当前代码实际行为”需要分开看**。([GitHub](https://github.com/mem0ai/mem0/issues/6982?utm_source=chatgpt.com))

第三，也是工程上很值得你注意的一点：V3 的 hybrid scoring 强依赖各 Vector Store 返回统一的“higher-is-better similarity score”。最近 GitHub 上就有 pgvector、LangChain/FAISS、Neptune 等 backend score 方向不一致导致 threshold/ranking 出错的 issue。换句话说：

```text
Hybrid Search 很漂亮

但前提：

所有 Retriever 的 score contract
必须严格统一
```

([GitHub](https://github.com/mem0ai/mem0/issues/7068?utm_source=chatgpt.com))

这个反而是你这种做后端/平台的人非常值得关注的工程点。

------

### 最后给你一个最简单的记忆方式

不要记一大堆细节，就记：

```text
                V2
        ┌─────────────────┐
        │ 写入聪明         │
        │ 检索简单         │
        └─────────────────┘
                 ↓

     LLM 判断 Memory 应该
   ADD / UPDATE / DELETE / NONE


                V3
        ┌─────────────────┐
        │ 写入简单         │
        │ 检索聪明         │
        └─────────────────┘
                 ↓

          ADD-only 积累
                 +
     Semantic / BM25 / Entity
                 +
            Hybrid Ranking
```

**本质就是：V3 不再执着于写入时维护一个“完美的当前 Memory 状态”，而是更倾向于可靠地积累事实，再通过更强的 Retrieval 把需要的信息找出来。**

这和你之前研究的“长期记忆 / 情景记忆 / Memory Adjudication / LoCoMo 评测”其实已经连起来了：下一步最值得拆的是 **`Memory.add()` 和 `Memory.search()` 两条源码链路**。尤其 `search()`，我可以按实际源码给你画成一张**函数调用级流程图**，把 `vector search → BM25 → entity → score_and_rank → reranker` 每一步输入输出讲清楚。