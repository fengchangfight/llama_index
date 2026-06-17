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

