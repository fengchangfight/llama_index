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
