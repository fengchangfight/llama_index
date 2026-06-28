# Code Analysis Methodology

This file documents methodologies, techniques, and analytical frameworks for code analysis used and discovered during code reading sessions.

---

## 2026-06-17: 组件边界对比法

### 场景
分析 LlamaIndex 中 VectorStore / Index / Retriever 三个紧密耦合的组件。

### 方法：三角对比法

当代码库中有多个名称相近、职责交叉的组件时，使用"三角对比法"快速厘清边界：

1. **分别提取基类/接口定义** — 找到每个组件的 ABC 或 Protocol，列出其必须实现的抽象方法。这些方法就是该组件"不可削减的职责"。
2. **追踪创建链路** — 找到组件之间的创建关系（谁创建谁），确定依赖方向。例如 `Index.as_retriever()` → `Retriever`，Index 持有 `VectorStore`。
3. **跑通一条完整调用链** — 选一个典型场景（如 `retriever.retrieve("query")`），逐层追踪调用栈，看每一步发生在哪个组件。
4. **用一句话概括每个组件** — 如果一句话说不清楚，说明边界还没理清。

### 关键发现

- Index 是"工厂"不是"引擎"：它创建 Retriever 和 QueryEngine，但自己不检索。
- Retriever 是"策略"不是"存储"：它封装了怎么查、查多少、怎么组装结果，底层调 VectorStore。
- VectorStore 是纯粹的数据层：只关心 embedding 的 CRUD + ANN 搜索。
- `stores_text` 属性决定了 Index 是否额外使用 docstore，这是理解存储去重的关键开关。

### 推广
此方法适用于任何"多个组件协同工作"的架构分析（如 Agent / Tool / FunctionCalling, Pipeline / Cache / Docstore 等）。

---

## 2026-06-17: 接口契约分析法

### 场景
用户问"如果不用官方实现，自己封装需要重写哪些方法"。

### 方法

1. **找到最简接口定义** — 在 LlamaIndex 中，`VectorStore` Protocol（`types.py:268`）定义了最小接口契约，`BasePydanticVectorStore` ABC 是增强版。
2. **区分"必须有"和"可以有"** — Protocol 标注的方法 (= `...` 实际体) 是必须实现的；ABC 中带默认实现的方法（如 `aget_nodes` 调 `get_nodes`）是可以不覆盖的。
3. **追踪调用方检查实际使用** — 在 `VectorStoreIndex._add_nodes_to_index()` 中只调了 `add()` 和读 `stores_text`，确认最小子集。
4. **结果**：最小实现 = `add` + `query` + `delete` + `stores_text = True`

### 关键发现

- Python Protocol 是比 ABC 更"诚实"的接口定义：只声明签名，不提供实现，运行时通过 `isinstance` 检查。
- `BasePydanticVectorStore` 的很多方法（`clear`, `delete_nodes`, `get_nodes`）带默认实现或 `raise NotImplementedError`，不是必须覆盖的。
- 真正决定"能不能跑"的不是实现了多少方法，而是调用链路实际用了哪些方法。

### 推广
分析"最少需要实现什么"时，先找接口契约（Protocol/ABC），再追踪调用方的实际使用路径，两者取交集即可得到最小实现集。

---

## 2026-06-17: 存储写链路追踪法

### 场景
分析 `insert_nodes()` 数据分别写入 VectorStore / docstore / index_struct 三个存储层的过程，以及各层在一定条件下的"短路"行为。

### 方法

1. **从入口函数逐层下钻** — 从 `VectorStoreIndex.insert_nodes()` 出发，追踪 `_insert()` → `_add_nodes_to_index()` → `vector_store.add()` / `docstore.add_documents()` / `index_struct.add_node()` 的完整调用链。
2. **关注条件分支** — 不在函数签名里找逻辑，而在函数体里找 `if` 分支。例如 `if not self._vector_store.stores_text or self._store_nodes_override` 决定了写不写 docstore；`if isinstance(node, (ImageNode, IndexNode))` 决定了哪些 Node 类型额外写入。
3. **画多轨并行图** — 三个存储目标的写入不是原子事务，画出"先写谁后写谁、谁可能失败、失败后谁已经写入了"的完整状态空间。

### 关键发现

- TextNode + `stores_text=True`（Milvus 默认）场景下，docstore 完全不参与写入——数据一致性风险仅存在于 ImageNode/IndexNode。
- `index_struct.nodes_dict` 对 Milvus（PK = node UUID）退化为恒等集 `UUID→UUID`，实际作用是"已知节点白名单"而非映射。
- `node_to_metadata_dict` 自动写入的 `doc_id` 字段值 = `ref_doc_id`，不包含 index 级命名空间——这是多 Index 共享 Collection 时数据泄漏的根因。

### 推广
分析"数据写到了哪里"时，不要在接口层面猜测，必须追踪到实际的 `add()`/`add_documents()` 调用点，并检查条件分支——不同配置可能导致完全不同存储路径。

---

## 2026-06-17: 跨组件引用追踪法

### 场景
分析多个 Retriever 共享一个 Index 时是否会产生数据竞争，以及一个 Retriever 写入后另一个是否立即可见。

### 方法

1. **定位对象创建点** — 找到 Retriever 从 Index 获取 `vector_store` / `docstore` / `embed_model` 的赋值语句，确认是按值复制还是引用传递。
2. **区分读写角色** — 检查 Retriever 调用的所有方法，区分：
   - 纯读操作（`query`, `get_nodes`）
   - 写操作（`add`, `delete`）
3. **追踪 `**kwargs` 的流向** — LlamaIndex 大量使用 `**kwargs` 级联透传（`insert_nodes → _add_nodes_to_index → vector_store.add`），需要逐层确认参数是否完整到达目标。
4. **验证共享路径** — 两个组件是否指向同一个 Python 对象引用（`id(a.vector_store) == id(b.vector_store)`），还是各自创建了副本。

### 关键发现

- Retriever 不拥有存储，只持有 Index 的引用——所有共享同一 Index 的 Retriever 指向同一个底层 `vector_store` 和 `docstore` 对象。
- Retriever 自身只做读操作，不会互相干扰。但通过 Index 的写操作（`insert_nodes`）写入的数据立即可见（共享引用）。
- `**kwargs` 的透传链已经在关键路径上打通：`insert_nodes` 的 `milvus_partition_name` → `_add_nodes_to_index` 的 `**insert_kwargs` → `vector_store.add()` 的 `**add_kwargs`。

### 推广
分析"两个组件是否共享状态"时，从持有者（Index）追踪属性赋值，确认是引用还是复制；然后追踪使用者的读写操作矩阵，区分只读共享（安全）和写写竞争（需同步）。

