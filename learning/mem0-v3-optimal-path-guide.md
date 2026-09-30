# mem0 V3 最优路径导读:只读这条线

> 本文是**导读**,不是讲解。目标:在这个清理分支上,把 V3 当前**唯一活着的主路径**从头到尾走一遍,告诉你每段读哪个文件、看什么、以及——**哪些老代码别再去找了**。前置阅读:`mem0-memory-write-storage.md`(设计思想)+ `mem0-memory-optimization-decisions.md`(为什么这么选)。读本文时打开代码,按指引跳转。

---

## 0. 先看清这个分支做了什么

本分支(`learn/cleanup-v2-legacy-prompts`)把 V3 之前、后续可能废弃的老特性从代码里**物理移除**了,目的是让你学一个"没有历史包袱"的 mem0。被移除的是 V2 时代那套**写入时让 LLM 做四选一决策**的残留:

- 4 个老 prompt:`FACT_RETRIEVAL_PROMPT`、`USER_MEMORY_EXTRACTION_PROMPT`、`AGENT_MEMORY_EXTRACTION_PROMPT`、`DEFAULT_UPDATE_MEMORY_PROMPT`
- 3 个老 prompt 装配函数:`get_update_memory_messages`、`get_fact_retrieval_messages`、`get_fact_retrieval_messages_legacy`
- 2 个老分支选择器:`_should_use_agent_memory_extraction`(同步 + 异式各一)
- 1 个只测上述函数的测试:`tests/configs/test_prompts.py`

**关键认知**:网上很多讲 mem0 的旧文仍在讲"ADD/UPDATE/DELETE/NOOP 四操作决策"——那是 V2,V3 主路径早不走了,本分支连代码带测试一起删了。你读本文时如果碰到旧文讲那套,对不上是正常的。

---

## 1. 全景:V3 只有两条主路径 + 一组显式 API

把整个 mem0 记成三块,就不会迷路:

```
                  ┌─────────────────────────────────────────┐
   add(messages)  │  写入路径(批处理事务,9 阶段,只 ADD)        │
        ─────────►│  _add_to_vector_store                     │
                  │  蒸馏 → 去重 → 批嵌入 → 批落库 → 实体链接   │
                  └─────────────────────────────────────────┘
                  ┌─────────────────────────────────────────┐
  search(query)   │  检索路径(三信号融合,只读)               │
        ─────────►│  _search_vector_store → score_and_rank   │
                  │  语义 + BM25 + 实体boost → 自适应融合     │
                  └─────────────────────────────────────────┘
                  ┌─────────────────────────────────────────┐
 update/delete/   │  显式管理 API(确定性,不归 LLM)           │
 get/get_all/     │  按 ID 操作,走代码不走判断题              │
 history          └─────────────────────────────────────────┘
```

**思想**:V3 的架构纪律是——**写入廉价积累,检索精确定位,管理显式确定**。三条线各司其职,没有一条让 LLM 做"该不该改/删哪条"的判断题。这个分工是 V3 的脊梁,读代码时每走到一个分叉,先问"这一步是 LLM 理解,还是代码确定?",基本不会跑偏。

---

## 2. 读写入路径:`m.add()` → `_add_to_vector_store`

入口 `mem0/memory/main.py` 的 `Memory.add`(同步)与 `Memory.add_async`(异步)。真正干活的是 `_add_to_vector_store`(同步 858 行 / 异步 2491 行)。一次 `add` 是一条 9 阶段流水线,**按这个顺序读每一段**:

| 阶段 | 做什么 | 读哪里 | 看什么 |
|---|---|---|---|
| 1 解析消息 | 多轮消息拼成单串 | `mem0/memory/utils.py` 的 `parse_messages` | 它跳过 `content=None` 的 tool-call 消息——agent 场景必备 |
| 2 取上下文 | 向量召回已有相关记忆(~10 条)喂给抽取器 | `_add_to_vector_store` 内的召回段 | **增量写入**的本钱:让 LLM 知道"已记过什么",别重复抽 |
| 3 单次 LLM 抽取 | 只 ADD,产事实列表 | `ADDITIVE_EXTRACTION_PROMPT`(prompts.py:100);`generate_additive_extraction_prompt`(648)处理 custom_instructions | V3 的核心:LLM 只理解、只产出,不做增删改决策 |
| 4 json 兜底 | 保证 prompt 里有 "json" 字 | `utils.py` 的 `ensure_json_instruction` | 对内置 prompt 是 no-op,只护自定义 prompt 不被 OpenAI json_object 模式拒 |
| 5 解析归一 | LLM 输出 → 事实字符串列表 | `utils.py` 的 `extract_json` + `normalize_facts` | `normalize_facts` 兜小模型返回 `{"fact":...}` 这种畸形 |
| 6 hash 去重 | md5 完全一致就跳过 | `_add_to_vector_store` 内 hash 段 | 确定性去重第一道闸:挡"完全一样",挡不了"语义重复" |
| 7 批嵌入+批落库 | `embed_batch` + `vector_store.insert`,失败回退逐条 | `_add_to_vector_store` 批处理段 | **事务思想**:N 条事实 = 1 次 LLM + 1 次 embed + 1 次 insert,不是 N 次循环 |
| 8 实体链接 | spaCy 抽实体 → embed → upsert到 `{collection}_entities` | `mem0/utils/entity_extraction.py` + main.py 实体段 | 实体存另一个 collection 复用向量库基建,没上 Neo4j(够用就好) |
| 9 历史留痕 | SQLite 记 ADD 事件(前后文本+时间) | `_add_to_vector_store` 末尾 + history 表 | event-sourcing 轻量版:本体可变,日志不可变 |

**每段的设计意图**不在这里重复——`mem0-memory-write-storage.md` 已展开。读这段代码时只抓一件事:**每一步都"能批就批、失败能局部回退"**。批量是性能,降级是健壮,缺一不可。

> 一个容易看漏的点:阶段 2 的"取上下文喂给抽取器"和阶段 6 的"hash 去重"是**两层闸**——hash 挡完全重复,prompt 挡语义重复。少了上下文那层,hash 兜不住同义不同字,记忆库会被同义重复撑爆。

---

## 3. 读检索路径:`m.search()` → `_search_vector_store` → `score_and_rank`

入口 `Memory.search`(同步)/ `Memory.search_async`(异步)。核心在 `_search_vector_store`(同步 1607 / 异步 3251)调 `mem0/utils/scoring.py` 的 `score_and_rank`(60 行)。**按这个顺序读**:

1. **over-fetch 召回**:语义检索不直接取 top_k,取 `max(top_k×4, 60)` 作候选池。读 `_search_vector_store` 开头。
2. **阈值门控前置**:`semantic_score < threshold` 的候选**直接丢,不让 BM25/实体救**。读 `score_and_rank` 内的 `continue`。**思想**:门控在融合前 = 保精度不保召回,因为写入侧只增不改,候选池本就大,缺的是别放噪声。
3. **BM25 召回 + 自适应归一**:读 `scoring.py` 的 `get_bm25_params`(16,按查询长度调中点陡度)+ `normalize_bm25`(43,sigmoid 压到 [0,1])。**思想**:三路分数量纲不同,直接加会被 BM25 碾压;先各自归一再融合。
4. **实体 boost + 枢纽惩罚**:读 `_compute_entity_boosts`(同步 1712 / 异式 3354)。命中实体的关联记忆被加权,但被 `memory_count_weight = 1/(1+0.001×(n-1)²)` 稀释。**思想**:链接越多的实体(如"用户")区分度越低,boost 越要衰减——IDF 思想变体。
5. **加性融合**:`(语义 + BM25 + 实体boost) / max_possible`,max_possible 随活跃信号数变(1.0/2.0/2.5)。读 `score_and_rank` 融合段。**思想**:不用调权重,用"有几路除以几"做自适应归一,融合分始终 [0,1] 可比。
6. **可选 reranker + explain**:读 `mem0/configs/rerankers/`(Cohere/HF/sentence_transformers/LLM/zero_entropy 五种)+ `score_and_rank` 的 `score_details` 输出。**思想**:多信号系统必须自带可观测性,否则调参靠玄学。

**这条路径的全部意图**在 `mem0-memory-optimization-decisions.md` 的"决策 3~6"里有展开——本文只给你读代码的顺序和锚点,不重复 why。

---

## 4. 读显式管理 API:确定性,不归 LLM

这一组在 `main.py` 里,**都是调用方主动发起、按 ID 操作、走代码不走判断题**:

- `update(memory_id, text)`:知道改哪条(有 ID)→ 给新文本 → 重嵌入原子替换。**不依赖 LLM 决策**。读 `Memory.update`。
- `delete(memory_id)` / `delete_all()`:删向量库 point + SQLite 记 DELETE。
- `get(memory_id)` / `get_all()`:读当前状态(走向量库)。
- `history(memory_id)`:读变更轨迹(走 SQLite 历史表)。

**思想**:V3 删的是 add() 里**隐式**的 LLM UPDATE;`update()` 这个**显式** API 一直在,且 V3 没动它。区分清楚:隐式(被砍)= LLM 在写入时自作主张改记忆;显式(保留)= 调用方拿着 ID 主动改。后者在 agent 运行时是否"有用"是另一回事(调用方不好判断改哪条),但它**存在且确定**,不要误以为 V3 把更新能力也删了。

---

## 5. 三类代码:别追错路

学这套系统最大的坑是**追根本不存在或已被删的代码**。本分支上,代码分三类:

### A. LIVE —— 读这些
| 文件/符号 | 作用 |
|---|---|
| `mem0/configs/prompts.py` 的 `ADDITIVE_EXTRACTION_PROMPT`(100) | V3 唯一的写入抽取 prompt(ADD-only) |
| `prompts.py` 的 `AGENT_CONTEXT_SUFFIX`(579) | agent 视角抽取时的后缀(V2 用独立 prompt,V3 用后缀) |
| `prompts.py` 的 `generate_additive_extraction_prompt`(648) | custom_instructions 注入 |
| `prompts.py` 的 `MEMORY_ANSWER_PROMPT`(4) | 检索后生成自然语言答复(代理路径在用,**别误删**) |
| `prompts.py` 的 `PROCEDURAL_MEMORY_SYSTEM_PROMPT`(15) | 过程性记忆 |
| `mem0/memory/utils.py` 的 `ensure_json_instruction` | json_object 模式兜底 |
| `mem0/utils/scoring.py` | 三信号融合的全部实现 |
| `mem0/utils/entity_extraction.py` | spaCy 实体抽取 |
| `mem0/configs/rerankers/` | 五种可选 reranker |
| `mem0/memory/notices.py` 的 `get_temporal_feature_error_message` | OSS/Platform 边界的抛错点 |

### B. LEGACY-REMOVED —— 本分支已删,别再找
| 已删符号 | 它是干嘛的(V2) | 为什么删 |
|---|---|---|
| `FACT_RETRIEVAL_PROMPT` 等 4 个老 prompt | V2 的抽取/四操作决策 prompt | V3 主路径不调,留着误导 |
| `get_update_memory_messages` | 装配 V2 四操作 prompt | 无人调用 |
| `get_fact_retrieval_messages` / `_legacy` | 装配老抽取 prompt | 无人调用 |
| `_should_use_agent_memory_extraction` | V2 在 user/agent 抽取间二选一 | V3 统一走 ADDITIVE + 后缀,不再分支 |

### C. OSS-GAP —— README 宣传,但 OSS 没有,别找
| 能力 | OSS 现状 | 哪去了 |
|---|---|---|
| Temporal Reasoning(检索时给最新记忆加权) | `search(reference_date=...)` 抛错 | Platform 专有 |
| Decay(记忆衰减治理) | `add(timestamp=...)` 抛错 | Platform 专有 |
| memory-to-memory `linked_memory_ids` 持久化 | 不落库 | 实体库自己用这个字段做反向索引,但记忆间的显式关联在 OSS 没存 |

**思想**:mem0 把**机制**(蒸馏、去重、融合、实体关联)留给 OSS 开源建生态,把**需要重度基础设施的能力**(时序检索、衰减治理)留给托管平台商业化。读 OSS 代码时把这条边界记牢,否则会到处找根本不存在的 `reference_date` 逻辑。OSS 侧的时间能力仅限**提取阶段的时间锚定**(相对词→绝对日期,写进 prompt 的),那是 LLM 自带的,不是检索侧时间感知。

---

## 6. 推荐阅读顺序

如果从头读,按这个序最不易迷路:

1. `mem0-memory-write-storage.md`(设计思想,先建心智模型)
2. `mem0-memory-optimization-decisions.md`(V3 为什么这么选)
3. 本文(代码导读)
4. 打开代码,按本文 §2 → §3 → §4 的顺序读三条路径
5. 读 `ADDITIVE_EXTRACTION_PROMPT` 全文——它是 V3 最大的"软优化",把"防丢失"规则编码进了 prompt(反首主题主导、保专名、时间锚定、记全变迁、数量精确)

读完这五步,你脑里的 mem0 就是"去历史包袱的 V3 最优形态",没有 V2 残留,也没有 Platform 幻觉。

---

## 7. 最小代码索引(便于跳转,不是讲解)

| 路径 | 入口 | 核心 |
|---|---|---|
| 写入 | `main.py: Memory.add` / `add_async` | `_add_to_vector_store`(858 / 2491) |
| 检索 | `main.py: Memory.search` / `search_async` | `_search_vector_store`(1607 / 3251)+ `scoring.py: score_and_rank`(60) |
| BM25 | — | `scoring.py: get_bm25_params`(16)/ `normalize_bm25`(43) |
| 实体 boost | — | `main.py: _compute_entity_boosts`(1712 / 3354_async) |
| 抽取 prompt | — | `prompts.py: ADDITIVE_EXTRACTION_PROMPT`(100)+ `AGENT_CONTEXT_SUFFIX`(579) |
| json 兜底 | — | `utils.py: ensure_json_instruction` |
| 显式管理 | `main.py: update/delete/get/get_all/history` | 按 ID,确定性 |
| OSS 边界 | — | `notices.py: get_temporal_feature_error_message`(774) |
