# Code Reading Knowledge Points

This file records important knowledge points and insights accumulated during code reading sessions.

---

## 2026-06-15: Project Overview

### Project Identity
- **LlamaIndex** = a "data framework" bridging LLMs with private/external data (RAG infrastructure)
- v0.14.22, monorepo: `llama-index-core` + `llama-index-integrations` (300+ plugins) + utils + instrumentation
- Core package namespace: `llama_index.core.xxx`; integrations: `llama_index.xxx.yyy`

### High-Level Architecture (3-Phase Pipeline)
1. **Ingestion**: `Document → [Transformations...] → Nodes → VectorStore/DocStore`
2. **Indexing**: Build data structures from Nodes (vector index, keyword table, knowledge graph, tree...)
3. **Querying**: `Query → Retriever → NodePostprocessor → ResponseSynthesizer → Response`

### Key Design Patterns
1. **TransformComponent** (`schema.py:191`) — universal abstraction: `(nodes) -> nodes` for all pipeline steps
2. **Settings singleton** (`settings.py:291`) — lazy-init global defaults (LLM, embed_model, node_parser, transformations)
3. **Index → Retriever → QueryEngine chain** — `.as_retriever()` → `.as_query_engine()`
4. **Workflow event engine** (`workflow/`) — async event-driven DAG with `@step` decorator + `ctx.send_event()`
5. **Plugin integration pattern** — `llama_index.core.xxx.ClassABC` (abstract) → `llama_index.xxx.yyy.ProviderConcrete` (implementation)

### Core Modules Map
| Module | Path | Role |
|---|---|---|
| Schema | `core/schema.py` | BaseNode, Document, ImageDocument, TransformComponent |
| Settings | `core/settings.py` | Global lazy singleton for LLM/embed_model/callbacks/etc |
| Ingestion | `core/ingestion/` | IngestionPipeline, run_transformations, IngestionCache |
| Indices | `core/indices/` | BaseIndex, VectorStoreIndex, SummaryIndex, KeywordTableIndex, KnowledgeGraphIndex, TreeIndex, PropertyGraphIndex |
| Query Engine | `core/query_engine/` | RetrieverQueryEngine, RouterQueryEngine, SubQuestionQueryEngine, SQL+Vector hybrid |
| Response Synth | `core/response_synthesizers/` | refine, tree_summarize, compact_and_refine, accumulate |
| Workflows | `core/workflow/` | Event-driven DAG engine for complex multi-step agents |
| Agent | `core/agent/` | AgentRunner, ReAct, Function Calling agents |
| Base | `core/base/` | Abstract ABCs: BaseLLM, BaseEmbedding, BaseQueryEngine, BaseRetriever |

### Integration Ecosystem
- 104 LLM providers, 78 vector stores, plus embeddings, readers, storage, tools, retrievers, etc.
- 29 integration categories under `llama-index-integrations/`

### Best Code to Read (Recommended Reading Order)
1. `core/schema.py` — TransformComponent, BaseNode, Document, TextNode data model
2. `core/ingestion/pipeline.py:72-156` — `run_transformations` ingest pipeline core
3. `core/indices/base.py` — BaseIndex base class
4. `core/settings.py` — Settings global config
5. `core/workflow/workflow.py` + `decorators.py` — Event-driven workflow engine
6. `core/base/llms/base.py` — LLM abstraction
7. `core/indices/vector_store/base.py` — VectorStoreIndex (most used index)
8. `core/query_engine/retriever_query_engine.py` — Typical RAG query pipeline
9. `core/response_synthesizers/base.py` — Response synthesis base
10. `core/agent/runner/base_agent_runner.py` — Agent execution loop

---

## 2026-06-15: schema.py 深度解析

### 继承全景图
```
BaseComponent                        ← 万物之源（序列化能力: class_name, to_dict, from_dict, pickle）
  ├─ TransformComponent              ← 管道工位统一接口: (nodes) -> nodes
  ├─ RelatedNodeInfo                 ← 轻量节点引用 (只存 node_id + type + metadata, 无实际内容)
  ├─ NodeWithScore                   ← 检索结果包装 (node + score)
  └─ BaseNode                        ← 可检索原子单元 (id_, embedding, metadata, relationships)
       ├─ TextNode                   ← 纯文本节点（旧, 向后兼容）
       │    ├─ ImageNode             ← 图片节点（旧, 向后兼容）
       │    └─ IndexNode             ← "指针"节点（指向其他索引/对象）
       └─ Node                       ← 多模态节点（新, 含 text/image/audio/video resource）
            └─ Document              ← 用户原始文档入口
                 └─ ImageDocument    ← 图片文档（旧, 向后兼容）

QueryBundle (@dataclass)             ← 查询请求（非 pydantic, 用 dataclass_json）
```

### 各层详解

#### 四个枚举 (标签系统)
- **NodeRelationship**: SOURCE/PREVIOUS/NEXT/PARENT/CHILD — 节点间关系图谱
- **ObjectType**: TEXT/IMAGE/INDEX/DOCUMENT/MULTIMODAL — 对象类型标签
- **Modality**: TEXT/IMAGE/AUDIO/VIDEO — 纯模态区分
- **MetadataMode**: ALL/EMBED/LLM/NONE — **控制 metadata 在不同场景的选择性展示**（embed 用一套字段, LLM 用另一套）

#### BaseComponent (line 81) — 所有类的根
- 继承 pydantic BaseModel
- 核心能力: `class_name()` 身份证号 + 序列化全家桶 (to_dict/from_dict/to_json/from_json)
- 自定义 pickle: 自动跳过不可序列化的属性, 不崩溃
- 设计理念: **框架中流转的一切都必须能持久化后完整恢复**

#### TransformComponent (line 191) — 管道统一接口
- 核心契约: `__call__(nodes: Sequence[BaseNode]) -> Sequence[BaseNode]`
- 同时提供 `acall()` 异步版本
- 设计理念: **管道模式** — 所有处理步骤遵循同一接口, 像乐高自由组合, 缓存的 hash 机制依赖此统一签名

#### MediaResource (line 507) — 万能媒体容器
- 四种存放方式: data(bytes) / path(Path) / url(AnyUrl) / text(str)
- 自动 base64 编码 + mimetype 自动推断 (filetype 库)
- 基于内容 SHA256 的 hash 去重
- 设计理念: **多态数据源** — 一个资源多种形态, 使用者无需关心底层来源

#### BaseNode (line 264) — 最核心的数据模型
- 关键字段: `id_`(UUID), `embedding`(向量), `metadata`(自由键值对), `relationships`(关系图)
- `excluded_embed_metadata_keys` / `excluded_llm_metadata_keys` — 选择性展示 metadata
- 关键方法: `get_content(metadata_mode)` — 根据不同模式拼接文本
- 便捷属性: source_node / prev_node / next_node / parent_node / child_nodes
- 设计理念: **检索原子单位** — 自身可嵌入可检索, 同时通过 relationships 记住来源和邻居, 支持上下文补全

#### Node (line 638) — 多模态节点 (新)
- 四种 resource: text_resource / image_resource / audio_resource / video_resource (均为 MediaResource)
- `get_content_blocks()` 转成 LLM 可理解的多模态 block 列表
- 设计理念: **多模态 RAG 的新架构**, 用 MediaResource 统一管控, 替代旧版多字段方式

#### TextNode (line 765) — 纯文本节点 (旧)
- 字段: text + start_char_idx + end_char_idx (原文位置)
- 设计理念: 历史遗留向后兼容, 新代码应使用 Node(text_resource=MediaResource(text="..."))

#### ImageNode (line 852) — 图片节点 (旧)
- 核心方法: `resolve_image()` — base64/path/url 统一转 PIL BytesIO
- 设计理念: 向后兼容, 统一图片访问入口

#### IndexNode (line 948) — 指针节点
- `index_id` + `obj` — 指向另一个索引或对象
- `from_text_node()` — 从 TextNode 生成 IndexNode
- 设计理念: **组合模式** — 构建"索引的索引", 实现分层 RAG (路由 → 子索引检索)

#### NodeWithScore (line 1035) — 检索结果
- `node`(BaseNode) + `score`(float) — 内容 + 相关度分数
- 透传属性: `.text`, `.metadata`, `.embedding` 直接代理到内部 node
- 设计理念: **检索结果的标准包装**, 下游 reranker/synthesizer 可同时获取内容和分数

#### Document (line 1097) — 用户入口
- 继承 Node, 代表用户最初传入的原始文档
- `__init__` 做大量向后兼容转换: doc_id→id_, extra_info→metadata, text→text_resource
- 附全套互转方法: to/from_langchain, to/from_haystack, to/from_semantic_kernel 等
- 设计理念: **Document 是"用户所见", Node 是"系统所用"** — 一篇 Document 被 split 成 N 个 TextNode

#### ImageDocument (line 1330) — 图片文档 (旧)
- 构造时做 PIL 图片有效性校验
- 设计理念: 向后兼容的图片文档包装

#### QueryBundle (line 1449) — 查询请求
- `query_str`(用户输入) + `custom_embedding_strs`(改写后用于 embedding 的文本) + `embedding`(预计算向量)
- 支持 image_path 图搜图
- 设计理念: **解耦用户输入和检索 embedding text** — HyDE、多查询改写等高级技巧在此层实现

### 核心设计理念总结
| 理念 | 体现 |
|---|---|
| 持久化优先 | BaseComponent 的序列化全家桶 + 安全 pickle |
| 管道模式 | TransformComponent 统一签名, 步骤可自由组合和缓存 |
| 多态数据源 | MediaResource 统一 bytes/path/url/text |
| 检索原子单位 | BaseNode 自带 embedding + relationships 关系图谱 |
| 选择性暴露 | MetadataMode 控制 metadata 在不同场景的展示 |
| 向后兼容 | TextNode/ImageNode/ImageDocument 保留旧接口, Document.__init__ 做字段转换 |
| 组合模式 | IndexNode 构建"索引的索引"实现分层检索 |
| 关注点分离 | Document(用户所见) ≠ Node(系统所用), query_str ≠ embedding_str |

---

## 2026-06-15: settings.py 深度解析

### 一句话定位

`Settings` = **框架的"全局遥控器"** —— 用 `@dataclass` 实现的懒加载单例, 一键换掉整个框架里的 LLM / embed_model / node_parser / transformations / callback_manager / tokenizer 六个核心组件。

### 它是什么

```python
_settings = _Settings()   # 模块加载那一刻创建唯一实例
```

任何地方 `from llama_index.core.settings import Settings`, 拿到的都是同一个人。**模块级单例**。

### 管的六件事

| 属性 | 默认 | 作用 |
|---|---|---|
| `.llm` | OpenAI gpt-3.5-turbo | 大模型 |
| `.embed_model` | OpenAI text-embedding | 转向量 |
| `.callback_manager` | 空 CallbackManager | 追踪/日志 |
| `.tokenizer` | tiktoken | 算 token 数 |
| `.node_parser` | SentenceSplitter | 文档切块 |
| `.transformations` | `[node_parser]` | ingest 处理管道 |

衍生属性: prompt_helper(根据 context_window 自动生成), chunk_size, chunk_overlap, num_output, context_window

### 核心机制: 懒加载

```python
@property
def llm(self) -> LLM:
    if self._llm is None:                          # 第一次访问才初始化
        self._llm = resolve_llm("default")
    if self._callback_manager is not None:
        self._llm.callback_manager = self._callback_manager  # 自动注入 callback
    return self._llm
```

- 不是 import 时初始化, 而是**第一次访问时才造**(省资源、允许运行时改环境变量)
- 造完自动注入 callback_manager(用户设一次 callback, 所有组件自动获得)

### 使用方式

```python
# 入门: 什么都不改, 全默认 (OpenAI 全家桶), 5 行跑通 RAG
index = VectorStoreIndex.from_documents(documents)

# 进阶: 改一行, 全局生效
Settings.llm = Ollama(model="llama3.1")

# 高级: 完全不用 Settings, 每个组件构造时显式传参
index = VectorStoreIndex.from_documents(documents, llm=my_llm, embed_model=my_embed)
```

### 设计理念

| 理念 | 体现 |
|---|---|
| **开箱即用** | 所有属性都有默认值, 不配置也能跑 |
| **全局一致性** | 改一次 Settings, 后续所有操作自动生效, 避免"建索引用 A 模型, 查询忘了换" |
| **渐进式定制** | 新手不碰 Settings → 进阶改 `.llm` → 高级绕过 Settings 显式传参 |
| **依赖注入(穷人版)** | Settings 作为"模块级注册表", 框架内部统一从这里取值 |
| **关注点分离** | `transformations` 把"框架默认管道"和"用户自定义管道"解耦 |

---

## 2026-06-15: ingestion/pipeline.py 深度解析

### 文件定位

整个 LlamaIndex 数据处理的"总装流水线": **原始文档 → 去重 → 加工(切块+嵌入+...) → 入库**。882行, 两个类(`IngestionPipeline` 调度器 + `DocstoreStrategy` 去重策略枚举) + 一组工具函数。

### 整体流程 (以 `run()` 为例)

```
Phase 1: _prepare_inputs()  → 从 documents/nodes/readers 四个来源收集输入
Phase 2: _handle_duplicates / _handle_upserts → 查 docstore 去重/增量判断
Phase 3: run_transformations() → 顺序执行管线步骤 (带缓存)
Phase 4: vector_store.add() + _update_docstore() → 写入向量库 + 更新文档hash记录
```

### 加工引擎: `run_transformations()` (72行) — 管线心脏

```python
for transform in transforms:
    if cache:
        hash = get_transformation_hash(nodes, transform)  # (内容, 配置) 的 hash
        cached = cache.get(hash)
        if cached: nodes = cached     # 命中缓存, 跳过
        else:
            nodes = transform(nodes)  # 真执行
            cache.put(hash, nodes)    # 存缓存
    else:
        nodes = transform(nodes)
```

**缓存 key 的精妙设计**: `hash = sha256(节点内容 + 步骤配置(去掉内存地址等不稳定值))`。改了 `chunk_size` 参数 → hash 自动变 → 缓存自动失效, 不需要手动清。

**`remove_unstable_values()`** (46行): 用正则删掉 `<xxx at 0x7fb9...>` 这种内存地址, 保证同一份逻辑在不同进程/时间点产生相同 hash。

### 异步 + 多进程的巧妙处理

- `arun_transformations()`: 异步版引擎, 调用 `await transform.acall(nodes)` 
- `arun_transformations_wrapper()` (159行): 在子进程创建新 event loop 跑异步 — 解决 ProcessPoolExecutor 子进程无 event loop 的痛点
- `_run_transformations_worker()` / `_arun_transformations_worker()` (186/214行): 多进程 worker, 返回 `(结果节点, 内存缓存条目)` 让主进程合并

### 去重策略 `DocstoreStrategy` (242行)

| 策略 | 逻辑 | 场景 |
|---|---|---|
| UPSERTS(默认) | ID不存在→新增; ID存在+hash不同→更新; ID存在+hash相同→跳过 | 增量更新 |
| DUPLICATES_ONLY | 只看hash, 不关心更新 | 纯去重 |
| UPSERTS_AND_DELETE | UPSERTS + 删除本次未出现的旧文档 | 全量替换 |

**去重粒度**: 以 `ref_doc_id`(源文档ID) 为准, 而非 `node_id`。一篇 Document split 成 10 个 Node, 它们有各自的 node_id 但共享一个 ref_doc_id, 去重按"源文档"粒度。

### 智能降级

当用户配了 `docstore` 但没配 `vector_store`, 却选了 `UPSERTS`(需要删旧向量) 时, 自动降级为 `DUPLICATES_ONLY` + 发出 Warning — 不崩溃, 给用户补救提示。

### 写入阶段的细节

- 只有 **带 embedding 的节点** 才写 vector_store (纯文本节点跳过)
- 多进程模式下, 内存缓存在 worker 中独立产生, 通过返回值带回主进程合并
- docstore 在去重阶段就实时更新 hash(避免并发重复), 最终 `_update_docstore` 做批量补充

### 设计理念

| 理念 | 体现 |
|---|---|
| **管道模式** | TransformComponent 统一签名, 循环顺序执行 |
| **增量缓存** | hash=(内容,配置), 改参数自动失效, 不改永久命中 |
| **策略模式** | DocstoreStrategy 三选一, 同一代码适配不同场景 |
| **多进程安全** | 独立 worker 函数 + 返回值合并缓存 |
| **同步/异步双轨** | run()/arun() 完全对称, 每个内部方法都有 sync/async 版 |
| **渐进式配置** | 0配置可跑(Settings默认) → 配transformations → 配cache/docstore |
| **智能降级** | 配置不兼容时自动降级 + Warning, 不崩溃 |

---

## 2026-06-15: 五种索引类型详解

### 1. VectorStoreIndex — 最常用, 向量语义检索
- 数据结构: `IndexDict` (向量库ID → node_id 映射)
- 建索引: 对节点做 embedding, 批量写入外部 vector_store (不调 LLM)
- 检索: 查询转向量 → 向量库 ANN 搜索 → 返回 top-k
- 适用: 绝大多数 RAG 场景, 语义搜索
- 建索引成本: 低 (只需 embedding)

### 2. SummaryIndex (ListIndex) — 最简单, 全量返回
- 数据结构: `IndexList` (有序 node_id 列表)
- 建索引: 不做任何处理, 只按顺序记录 node_id (不调 LLM, 不做 embedding)
- 检索: 默认返回全部节点; 也支持 embedding 检索模式和 LLM 挑选模式
- 适用: 小文档, 需看全文, baseline 测试
- 建索引成本: 零

### 3. KeywordTableIndex — 关键词哈希表
- 数据结构: `KeywordTable` (keyword → {node_id集合})
- 建索引: 对每个节点调 LLM 提取关键词, 建关键词→节点映射表 (需调 LLM)
- 检索: 查询中提取关键词, 按匹配关键词数量排序返回节点
- 适用: 精确术语匹配, 结构化术语明确的知识库
- 建索引成本: 中

### 4. KnowledgeGraphIndex — 知识图谱 (已废弃, 用 PropertyGraphIndex)
- 数据结构: `KG` (关键词映射 + 三元组图 + embedding)
- 建索引: LLM 提取 (主语, 关系, 宾语) 三元组, 写入 graph_store
- 检索: 支持关键词匹配实体 + 图遍历 + embedding 搜索三元组
- 状态: 自 v0.10.53 废弃

### 5. TreeIndex — 树状分层摘要
- 数据结构: `IndexGraph` (叶子=原文, 往上=LLM摘要, 根=最高层摘要)
- 建索引: 自底向上, 每 N 个节点用 LLM 生成一个摘要父节点, 逐层递归直到根
- 检索: 从根向下遍历, 每步让 LLM 判断"该去哪个子节点", 逐层下到叶子
- 适用: 大文档, 多层次问答 (宏观走高处, 微观走深处)
- 建索引成本: 高 (大量 LLM 摘要调用)

### 索引选型速查
| 索引 | 建索引成本 | 检索方式 | 适合场景 |
|---|---|---|---|
| VectorStoreIndex | 低 | 向量 ANN | 语义搜索, 通用 RAG |
| SummaryIndex | 零 | 全量返回/嵌入/LLM择 | 小文档, baseline |
| KeywordTableIndex | 中 | 关键词匹配 | 精确术语查找 |
| TreeIndex | 高 | 树遍历 | 大文档, 分层问答 |

### 核心要点
- **只有 VectorStoreIndex 依赖外部向量数据库**, 其余四种全在内存自管理
- **除 SummaryIndex 和 VectorStoreIndex 外, 其余建索引都需调 LLM**
- **`from_documents()` 统一入口**: 所有索引都走 BaseIndex._from_documents → run_transformations → _build_index_from_nodes
- **`as_retriever()` 是多态转换**: 每个索引返回自己的检索器, `as_query_engine()` 和 `as_chat_engine()` 是通用包装

---

## 2026-06-17: VectorStore / Index / Retriever 三者边界

### 一句话定位

| 组件 | 一句话 | 核心接口 |
|------|--------|----------|
| **VectorStore** | 引擎 — 底层向量存储与近邻检索 | `add(nodes)`, `query(VectorStoreQuery) → VectorStoreQueryResult` |
| **Index** | 工厂+编排者 — 管理摄入、维护状态、生产 Retriever | `from_documents(docs)`, `as_retriever(kwargs) → BaseRetriever`, 自己不检索 |
| **Retriever** | 策略 — 封装查询方式（怎么嵌入、怎么调 VectorStore、怎么组装结果） | `retrieve(str) → List[NodeWithScore]` |

### 依赖链

```
Index ──owns──→ VectorStore (via StorageContext)
  │
  └── as_retriever() ──creates──→ VectorIndexRetriever
                                      │
                                      ├── embed query
                                      ├── vector_store.query()  ← 调 VectorStore
                                      ├── docstore.get_nodes()  ← 补全 Node 对象
                                      └── return List[NodeWithScore]
```

### VectorStore 核心方法 (Protocol)

定义在 `core/vector_stores/types.py:268`，`typing.Protocol` 运行时检查接口：

| 方法 | 签名 | 职责 |
|------|------|------|
| `add` | `(nodes: List[BaseNode]) → List[str]` | 写入节点 + embedding，返回 IDs |
| `delete` | `(ref_doc_id: str) → None` | 按源文档 ID 删除 |
| `query` | `(query: VectorStoreQuery) → VectorStoreQueryResult` | ANN 近邻搜索 |
| `stores_text` | `bool` 属性 | 自身是否能完整还原 Node（影响 Index 是否使用 docstore） |

`BasePydanticVectorStore` (ABC) 额外提供 `get_nodes`, `delete_nodes`, `clear` 等。

### `stores_text` 的关键作用

当 `stores_text=True` 时，`VectorStoreIndex._add_nodes_to_index()` (indices/vector_store/base.py:234) 会**跳过写入 docstore/index_struct**，只把 TextNode 存入 VectorStore。只有 ImageNode/IndexNode 这类非文本节点才额外存入 docstore。

这意味着 VectorStore 必须能独立重建完整 Node 对象。

### Index 不自己检索

`BaseIndex` 提供 `as_retriever()`, `as_query_engine()`, `as_chat_engine()` 等工厂方法，
但**自身不执行检索**。检索逻辑完全委托给 Retriever：
- `as_retriever()` → `VectorIndexRetriever`
- `as_query_engine()` → `as_retriever()` + 外套 `RetrieverQueryEngine`

### Milvus 写入手动封装需实现的 3 个核心方法

如果不用官方 `MilvusVectorStore`，自己封装 pymilvus 对接 LlamaIndex，只需实现：

1. **`add(nodes, **kwargs) → List[str]`**
   - `node_to_metadata_dict(node, remove_text=True, text_field=self.text_key)` 序列化 node
   - 写入字段：`id`(PK = node.node_id), `text`(node.text), `embedding`(node.embedding)
   - `_node_content`(JSON blob 含全部元数据), `_node_type`, `ref_doc_id`, `doc_id`
   - 批量 `self.client.insert(collection_name, data)`

2. **`query(query: VectorStoreQuery, **kwargs) → VectorStoreQueryResult`**
   - 用 `query.query_embedding` 做 ANN 搜索
   - 结果中解析 `_node_content` JSON 重建 Node，返回 `VectorStoreQueryResult(nodes=[...], similarities=[...], ids=[...])`

3. **`delete(ref_doc_id: str, **kwargs) → None`**
   - 先 query 查出 `ref_doc_id` 对应的所有 PK
   - `self.client.delete(pks=ids)` 批量删除

只需这 3 个方法 + `stores_text = True` 即可接入 `VectorStoreIndex`。

### MilvusVectorStore 写入全流程

```python
# base.py:436-501
def add(self, nodes: List[BaseNode], **add_kwargs) -> List[str]:
    for node in nodes:
        # 1. 序列化 node 为 metadata dict，清掉 text 和 embedding 避免重复
        entry = node_to_metadata_dict(node, remove_text=True, text_field=self.text_key)
        # 2. 单独写 text 列
        entry[self.text_key] = node.dict()[self.text_key]
        # 3. PK
        entry[MILVUS_ID_FIELD] = node.node_id  # "id"
        # 4. dense embedding
        entry[self.embedding_field] = node.embedding  # "embedding"
        # 5. sparse embedding (可选, BGEM3/BM25)

    # 批量 insert/upsert
    for insert_batch in iter_batch(insert_list, self.batch_size):
        self.client.insert(collection_name, insert_batch, partition_name=...)
    return [n.node_id for n in nodes]
```

Schema 字段：`id`(VARCHAR PK), `text`(VARCHAR), `embedding`(FLOAT_VECTOR), `sparse_embedding`(SPARSE_FLOAT_VECTOR 可选)。
因为 `enable_dynamic_field=True`，其余元数据（`_node_content`, `_node_type`, `ref_doc_id`, 用户自定义 metadata）自动作为 dynamic fields 存储。

### Binlog 结构 (Milvus 存储层)

Binlog 是 Milvus 的**物理持久化格式**，Segment 是逻辑单元，Binlog 是落盘文件。

**6 种类型**：InsertBinlog(0), DeleteBinlog(1), DDLBinlog(2), IndexFileBinlog(3), StatsBinlog(4), BM25Binlog(5)

**二进制格式**：
```
[MagicNumber: 4 bytes = 0xfffabc]
[DescriptorEvent: collectionID, partitionID, segmentID, fieldID, timestamps, datatype]
[Event 1: eventHeader + 列数据]
[Event 2: eventHeader + 列数据]
...
```

**三层 Protobuf 结构**：
```
SegmentBinlogs          ← 一个 Segment 的完整 binlog 清单
  ├─ FieldBinlogs[]     ← insert 数据 (每字段一个)
  ├─ Statslogs[]        ← PK 统计
  └─ Deltalogs[]        ← 删除记录

FieldBinlog
  └─ Binlogs[]          ← 该字段可能拆成多个物理文件

Binlog                  ← 单文件描述：LogPath, LogSize, EntriesNum, Timestamps, MemorySize
```

**写入流程**：`BulkPackWriter.Write()` (flushcommon/syncmgr/pack_writer.go:70) → writeInserts/writeStats/writeDelta/writeBM25Stats → 写入对象存储路径 `{rootPath}/insert_log/{collectionID}/{partitionID}/{segmentID}/{fieldID}/{logID}`

---

## 2026-06-17: VectorStore 与 docstore 的角色定位

### VectorStore — 向量引擎

定义在 `core/vector_stores/types.py:269`，`typing.Protocol` 运行时检查接口。只负责存储和检索 **embedding 向量**：
- `add(nodes) → List[str]`：存入节点 + 向量
- `query(VectorStoreQuery) → VectorStoreQueryResult`：ANN 近邻搜索
- `delete(ref_doc_id)`：按源文档 ID 删除

**只存向量索引，不一定是完整数据**（取决于 `stores_text` 属性）。

### docstore — 文档/节点实体存储

定义在 `core/storage/docstore/types.py:24`，ABC 抽象基类。本质是 **key-value 存储**（`KVDocumentStore`），以 `node_id` 为 key 存储完整 `BaseNode` 对象（文本、metadata、relationships 等）：
- `add_documents(nodes)`：存入节点
- `get_document(doc_id) → BaseNode`：按 ID 取节点
- 维护 `ref_doc_id → [node_ids]` 映射（用于按源文档批量管理）
- 维护文档哈希（用于去重和增量更新判断）

### 检索时的协作流程

```
Retriever._retrieve()
  ├─ embed query
  ├─ vector_store.query()      ← 向量 ANN 搜索 → 得到相似 node_id 列表
  ├─ docstore.get_nodes()      ← 用 node_id 去 docstore 取完整 Node 对象
  └─ return List[NodeWithScore]
```

**一句话**：VectorStore 负责"找到哪些节点"（相似度），docstore 负责"这些节点是什么"（完整内容）。

### Node 对应的是 chunk，不是原始文档

**继承关系**：`Document` → `Node` → `BaseNode` (`schema.py:1097,638,264`)

实际流程：
1. `Document` 是用户输入的原始文档
2. 通过 `NodeParser`/`SentenceSplitter` 将 Document 切分成多个 `Node`（chunk）
3. 每个 chunk 生成 embedding，存入 VectorStore 和 docstore

检索命中的 node 基本都是 chunk 而非原始文档。设计理念：**Document 是"用户所见"，Node 是"系统所用"**。

---

## 2026-06-17: 多 Retriever 共享同一个 Index

### 多个 Retriever 不会互相干扰（读操作）

`VectorIndexRetriever`（`retriever.py:24`）不持有数据，只持有 Index 的引用：

```python
self._index = index
self._vector_store = self._index.vector_store   # line 61 — 共享引用
self._docstore = self._index.docstore           # line 63 — 共享引用
self._embed_model = embed_model or self._index._embed_model
```

- Retriever 只做读操作：`_vector_store.query()` + `_docstore.get_nodes()`，无任何写/删
- 每个 Retriever 有独立的查询配置（`similarity_top_k`, `filters`, `query_mode`），存在自身实例变量里
- 底层存储是指向同一个 Python 对象的引用，关键不存在"副本"问题

### 并发写入时的可见性

**正确路径**：通过 `index.insert_nodes()` → 同步更新三样：
- `vector_store.add()` — 向量入库
- `index_struct.add_node()` — 更新 `nodes_dict` 映射（向量 ID → node_id）
- `docstore.add_documents()` — 节点内容入库

三者指向同一个内存对象，其他 Retriever 立即可见。

**绕过 Index 直接调 `vector_store.add()`**：向量入库了，但 `index_struct.nodes_dict` 不会更新。`VectorIndexRetriever._determine_nodes_to_fetch()`（`retriever.py:166`）查询时依赖 `nodes_dict` 把向量库返回的 ID 映射为 node_id，缺失映射会导致结果缺失或报错。

---

## 2026-06-17: MilvusVectorStore 的 flush 行为

### 默认不自动 flush

`add()`（`base.py:495-496`）：

```python
if add_kwargs.get("force_flush", False):
    self.client.flush(self.collection_name)
```

只有显式传入 `force_flush=True` 才触发。`async_add` 直接不支持：

```python
if add_kwargs.get("force_flush", False):
    raise NotImplementedError("force_flush is not supported in async mode.")
```

上层 `VectorStoreIndex._add_nodes_to_index()` 调用 `vector_store.add()` 时未传入 `force_flush`，所以默认路径不会 flush。

### 为什么默认不 flush

Milvus `consistency_level` 默认为 `"Session"`，growing segment 中 insert 的数据同 session 内立即可查，**不需要 flush 来保证可见性**。flush 的作用是将内存数据持久化到磁盘/对象存储，用于跨 session 持久化和崩溃恢复。

### 批量导入时频繁 force_flush 的性能损耗

**1. 强制落盘 I/O 阻塞**：每批 insert 后同步 flush 涉及磁盘/网络 I/O（几十~几百 ms 每次），百万级按 batch_size=100 就是 10000 次 flush，累积延迟巨大。

**2. 海量碎片段**：每次 flush 封一个 sealed segment。频繁 flush 导致大量极小 segment，查询时需要跨大量 segment 合并结果，查询性能严重下降；同时触发频繁 segment compaction 消耗 CPU/IO。

**3. 碎片化索引构建**：每次封段后 Milvus 为其构建向量索引，百条级别的小段建索引浪费 CPU，不如合并后在大段上集中构建。

**建议**：大批量导入不传 `force_flush`，让 Milvus 自动管理 segment。导入完成后再手动调一次 `flush` 或依赖 auto-flush（内存阈值触发）。这也是 `_add_nodes_to_index` 默认不传 `force_flush` 的设计意图。

### consistency_level 的四种级别

| 级别 | 写入 session 可见 | 跨 session 可见 | 说明 |
|------|-----------|-----------|------|
| `Strong` | ✅ | ✅ | 阻塞到 flush 完成才返回，任何连接立即可读 |
| `Session` (默认) | ✅ | ❌ | 写 session 可读 growing segment，其他 session 要等 flush |
| `Bounded` | ✅ | 限时最终可见 | 可读旧版，限时内收敛 |
| `Eventually` | ✅ | 最终可见 | 不保证时间 |

关键影响：如果两个不同服务实例共享 Milvus，实例 A 写入后实例 B 检索，在 flush 完成前 B 查不到（`Session` 模式下）。跨服务实例需要立即可见必须用 `Strong`，代价是每次 insert 阻塞等落盘。

---

## 2026-06-17: Milvus Segment 与 HNSW 索引架构

### 核心模型：每个 sealed segment 独立持有一个 HNSW

```
Growing Segment (内存, 可写, 无索引)
  └─ insert 写入
       ↓ (达到阈值: 大小/时间)
  Sealed Segment (不可变) → 建立独立 HNSW 图
       ↓ (小段过多)
  Compaction → 合并多个小 sealed segment → 建一个统一的大 HNSW
       ↓ (段过大)
  Index Compaction → 优化重建 HNSW 图
```

关键特性：
- sealed segment 不可变 → 其上的 HNSW 图也终生只读、不修改
- 多 segment 不共享 HNSW，查询时**并行搜索所有 sealed segment 各自的索引**，然后 reduce TOP-K 归并结果
- 这就是频繁 flush 产生大量小段导致查询性能差的根因：搜 N 个小图 + 合并结果的开销远大于搜一个大图

### Compaction 是自动的，无需手动介入

Milvus 的 DataCoord 内置自动 compaction 策略：

| 类型 | 触发条件 | 动作 |
|------|---------|------|
| Small Segment Compaction | sealed segment 数量/小段占比超阈值 | 多小段合并为大段 → 建新 HNSW |
| Index Compaction | 单段过大、索引老化 | 重建更优 HNSW 图 |

手动 `compact()` 只是特殊场景的补充（如导入完成后手动触发加速收敛），**长期运维依赖的是 auto-compaction**。

### 正确的写入策略总结

不要在应用层做以下操作：
- ❌ 每批 insert 后手传 `force_flush=True` — 产生海量碎片段
- ❌ 手动频繁 `compact()` — 干扰 DataCoord 的策略调度

应该做的：
- ✅ insert 不传 `force_flush`，让数据在 growing segment 积累
- ✅ 依赖 auto-seal + auto-compaction 自动管理 segment 生命周期
- ✅ 导入全量完成后可调一次 `flush`（确保落盘），后续交给自动机制

---

## 2026-06-17: 多 Index 共享单个 Milvus Collection 的并发与隔离

### 核心风险：两条查询路径都有隔离漏洞

**背景**：`node_to_metadata_dict`（`core/vector_stores/utils.py:71-73`）会自动给节点 metadata 加 `doc_id`、`document_id`、`ref_doc_id`，但值都是 `node.ref_doc_id`（源文档哈希），**没有任何 index 级别的命名空间标识**。

**查询的两种路径**（`VectorIndexRetriever._determine_nodes_to_fetch`，`retriever.py:146-170`）：

```python
if query_result.nodes:      # 路径A: Milvus 直接返回完整 Node (stores_text=True)
    # → 只过滤出非 TEXT 类型 (ImageNode/IndexNode 才需查 docstore)
    # → TextNode 直接使用, 不经过 nodes_dict 校验
    return [node.node_id for node in query_result.nodes
            if node.as_related_node_info().node_type != ObjectType.TEXT]

elif query_result.ids:      # 路径B: Milvus 只返回 ID 列表
    # → 用本 Index 的 nodes_dict 做 ID→UUID 映射
    return [self._index.index_struct.nodes_dict[idx] for idx in query_result.ids]
```

- **路径 A**（最常见的 TextNode + stores_text=True）：Milvus 返回的所有节点直接使用，**不过滤**，Index A 检索直接泄露 Index B 的数据
- **路径 B**（非 TextNode 或 stores_text 关闭）：`nodes_dict` 是每个 Index 私有的，查不到其他 Index 的 ID → 直接抛 `KeyError`

### 五种并发/数据问题

| 问题 | 严重程度 | 根因 |
|------|---------|------|
| **数据泄露** | 高 | 路径 A 无过滤，TextNode 跨 Index 直接可见 |
| **KeyError 崩溃** | 高 | 路径 B 中 nodes_dict 不包含其他 Index 的 ID |
| **删除扩散** | 高 | `delete(ref_doc_id)` 无分区范围限制，误删其他 Index 数据 |
| **Schema 冲突** | 中 | 共用集合强制相同向量维度、字段类型、索引配置 |
| **nodes_dict 语义退化** | 低 | 对 Milvus（PK=UUID），`nodes_dict` 退化为 UUID→UUID 恒等集 |

### 隔离方案对比

| 方案 | 原理 | 改动量 | 性能 | 适用场景 |
|------|------|--------|------|---------|
| **① Milvus Partition（推荐）** | 每 Index 一个分区，物理层隔离 | 中 | 最优，原生无额外开销 | 需要共享集合的多数场景 |
| **② Metadata Filter** | 节点加 `index_namespace` 字段，查询 filter | 中 | 需扫描过滤 | 需要跨 Index 查询能力的场景 |
| **③ 独立 Collection** | 每 Index 一个集合 | 小 | 无干扰 | 不介意多集合管理 |
| **④ doc_ids Filter** | 将 `ref_doc_id` 重写为 index 级标识 | 小 | 过滤扫描 | ❌ 不推荐（语义污染） |

### 方案① Partition 详细设计

```
Milvus Collection "shared"
  ├─ Partition "index_A"    ← Index A 数据
  ├─ Partition "index_B"    ← Index B 数据
  └─ Partition "index_C"    ← Index C 数据
```

**写入链路已打通**：`insert_nodes(**insert_kwargs)` → `_add_nodes_to_index` → `vector_store.add(nodes_batch, **insert_kwargs)`：

```python
index.insert_nodes(nodes, milvus_partition_name="index_A")
```

**检索链路已打通**：`VectorIndexRetriever.__init__` 中 `self._kwargs = kwargs.get("vector_store_kwargs", {})`，查询时 `self._vector_store.query(query, **self._kwargs)`：

```python
retriever = index.as_retriever(
    vector_store_kwargs={"milvus_partition_names": ["index_A"]}
)
```

**删除链路**：`delete(ref_doc_id, milvus_partition_name="index_A")`

### 额外注意事项

- **docstore 必须独立**：每个 Index 有独立 `docstore`（ImageNode/IndexNode 仍需它）
- **index_struct 天然隔离**：每个 Index 的 `IndexDict` 在内存中独立
- **Partition 上限**：单集合 4096 个分区
- **跨 Index 查询**：传多个 partition name 即可天然支持

---

## 2026-06-17: consistency_level 跨 Session 可见性

### 四种级别

| 级别 | 写入 session 可见 | 跨 session 可见 | 说明 |
|------|-----------|-----------|------|
| `Strong` | ✅ | ✅ | 每次 insert 阻塞等 flush 完成，任何连接立即可读 |
| `Session` (默认) | ✅ | ❌ | growing segment 只对写入 session 可见，其他 session 需等 seal+flush |
| `Bounded` | ✅ | 限时最终可见 | 可读到旧版本，限时内收敛 |
| `Eventually` | ✅ | 最终可见 | 不保证收敛时间 |

### 根因

Milvus 中 growing segment（内存 buffer）只对写入它的 session 开放。新客户端连接 = 新 session，只能看到 sealed segment（flush 后的持久化段）。默认 `consistency_level="Session"` 下，跨服务实例的数据立即可见必须在 insert 后 flush。

### 场景对照

| 场景 | 写后立即可查？ | 建议 |
|------|--------------|------|
| 同一进程内 Index→Retriever | ✅ | 默认 Session 即可 |
| 微服务 A 写, 微服务 B 查 | ❌ | flush 或改用 Strong |
| 单文档上传即搜 | flush 一次（几十 ms） | 不要用 Strong（每批 insert 都阻塞） |

---

## 2026-06-28: 去重机制全链路详解

### 核心结论

两次 ingest 同一个未变化的文档，默认配置下第二次**不做任何操作**（不 chunk、不 embed、不写向量库）。

### 机制总览

```
文档/Document
  │
  ├─ 1. 哈希计算: BaseNode.hash → SHA-256(content + metadata)
  │     schema.py:741-803 (各子类实现)
  │
  ├─ 2. 哈希存储: Docstore._metadata_collection → {doc_hash → doc_id}
  │     keyval_docstore.py:599-669
  │
  ├─ 3. 预处理判断: IngestionPipeline._handle_upserts / _handle_duplicates
  │     pipeline.py:451-507
  │     ├─ DUPLICATES_ONLY: 遍历所有已存 hash，存在则跳过
  │     ├─ UPSERTS (默认): 按 ref_doc_id 查 hash，相同跳过，变化则更新
  │     └─ UPSERTS_AND_DELETE: 同 UPSERTS + 删除本次未出现的旧文档
  │
  └─ 4. 批次内去重: current_hashes 集合，同一次 run 内的重复文档也合并
```

### 1. 哈希计算（文档身份的唯一标识）

定义在 `core/schema.py`，`BaseNode` 的 `hash` 属性为抽象方法，各子类独立实现：

| 节点类型 | 行号 | 哈希构成 |
|---------|------|---------|
| `TextNode` | 800-803 | `sha256(text + str(metadata))` |
| `Node` (多模态) | 741-762 | `sha256(metadata_str + audio_hash + image_hash + text_resource_hash + video_hash)` |
| `ImageNode` | 913-922 | `sha256(image_str + image_path_str + image_url_str + text)` |
| `MediaResource` | 605-635 | `sha256(text + sha256(data) + sha256(path) + sha256(url))` |

关键设计点：
- hash 是**即时计算**的（`@property`），不作为字段持久化存储（`schema.py:272-273` 注释：`hash is computed on local field, during the validation process`）
- metadata 包含在 hash 输入中 → metadata 变了也会触发重新 ingest
- `MediaResource` 用了三层哈希（对 data/path/url 分别 sha256 再合并），保证无论数据来自哪个来源都能正确去重

### 2. 哈希存储（Docstore 中的元数据层）

`Docstore` 除了存储节点本身，还维护一个独立的 `_metadata_collection` 用于哈希映射：

```
KVDocumentStore
  ├─ _kvstore        ← 节点内容 (key: node_id, value: BaseNode)
  └─ _metadata_collection ← 哈希映射 (key: doc_id, value: {"doc_hash": "..."})
```

核心方法（`keyval_docstore.py:599-669`）：

| 方法 | 行号 | 作用 |
|------|------|------|
| `set_document_hash(doc_id, doc_hash)` | 599-602 | 存单个 hash |
| `get_document_hash(doc_id)` | 633-639 | 按 doc_id 取 hash |
| `get_all_document_hashes()` | 651-659 | 返回 `{hash: doc_id}` 全量映射（注意 key 是 hash 值） |
| `set_document_hashes(doc_hashes)` | 604-614 | 批量存入 |
| `delete_document(doc_id)` | — | 同时删除节点和其 hash 记录 |

### 3. IngestionPipeline 去重判断（pipeline.py:451-507）

#### DUPLICATES_ONLY 模式 (`_handle_duplicates`)

```
对每个 node:
  1. 从 docstore 获取所有已存在的 document hash 集合 (existing_hashes)
  2. 与当前批次的 hash 集合 (current_hashes) 合并
  3. 如果 node.hash NOT IN (existing_hashes ∪ current_hashes) → 新节点，加入处理队列
  4. 如果 node.hash 已存在 → 跳过
```

#### UPSERTS 模式 (`_handle_upserts`)

```
对每个 node:
  1. 提取 ref_doc_id（源文档 ID，多个 chunk 共享同一 ref_doc_id）
  2. 查 docstore 获取该 ref_doc_id 对应的 existing_hash
  3. existing_hash 为 None → 新文档，加入处理队列
  4. existing_hash != node.hash → 文档已变化，删除旧的（vector_store + docstore），重新 ingest
  5. existing_hash == node.hash → 跳过（核心去重逻辑！）
```

UPSERTS_AND_DELETE 在 upserts 基础上额外收集"本次 run 中未出现的旧 ref_doc_id"，统一删除。

### 4. 批次内去重

`current_hashes` 是一个 set，在同一次 `run()` 调用内追踪已处理节点的 hash。同一批次中有两个内容相同的文档，第二个直接跳过。这意味着即使不用 docstore（内存模式），也能避免同一批次内的重复处理。

### 5. 智能降级（pipeline.py:590-604）

| 条件 | 行为 |
|------|------|
| 有 docstore + 有 vector_store | 按配置策略执行（UPSERTS/DUPLICATES_ONLY/UPSERTS_AND_DELETE） |
| 有 docstore + 无 vector_store + 策略 != DUPLICATES_ONLY | 自动降级为 DUPLICATES_ONLY + 发出 Warning |
| 无 docstore | 不去重，直接处理所有输入 |

降级逻辑的设计理念：**"不崩溃，给提示"** — 配置不兼容时自动退化为最安全策略。

### 6. Index 级别的增量刷新（`refresh_ref_docs`）

`core/indices/base.py:440-480` 提供了另一种去重路径 — 不经过 IngestionPipeline，直接在已有 Index 上刷新文档：

```
refresh_ref_docs(documents):
  对每个 document:
    existing_hash = docstore.get_document_hash(doc.doc_id)
    if existing_hash is None       → insert()      # 新文档
    elif existing_hash != doc.hash → update_ref_doc()  # 文档已变化
    else                           → skip          # 未变化，跳过
```

与 IngestionPipeline 的区别：`refresh_ref_docs` 在 Index 层面直接操作，不经过完整的 transformation 管道。

### 7. PropertyGraphIndex 去重

`core/indices/property_graph/base.py:244-249`：在插入图节点前，先从 graph_store 拉取已存在的相同 ID 节点，计算 `existing_node_hashes`，过滤掉 hash 已在图中的节点。

### 8. 检索层面的去重

以上是**摄入去重**。检索结果也有去重，但粒度不同：

| 位置 | 去重键 | 文件:行号 |
|------|--------|-----------|
| Recursive Retriever | `node.id_` | `retrievers/recursive_retriever.py:68-82` |
| Property Graph Retriever | `node.text` | `indices/property_graph/retriever.py:41-49` |
| Fusion Retriever (RRF) | `node.hash` | `retrievers/fusion_retriever.py:120-148` |
| Vector Store (hybrid search) | `node_id` | 多个 vector store 的 `_dedup_results()` |

### 9. URL 级别预去重

`readers/web/async_web/base.py:64`：网页读取器在抓取前对 URL 列表做 `list(dict.fromkeys(urls))` 去重，属于 IoC 之前的最轻量去重。

### 设计理念总结

| 理念 | 体现 |
|------|------|
| **内容寻址 (Content-Addressable)** | SHA-256 hash 作为文档唯一标识，内容不变 hash 不变 |
| **元数据敏感** | metadata 参与 hash 计算，改 metadata 等价于文档变化 |
| **分层去重** | URL 层 → Pipeline 层 → Index 层 → 检索层，各层粒度不同 |
| **失败安全** | 配置不兼容时自动降级，不崩溃 |
| **读写分离** | 摄入去重（写路径）与检索去重（读路径）机制独立 |
