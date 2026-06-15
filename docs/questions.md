Q: Ingestion 和Indexing定义的便捷在哪里？有何区别？

Q: pipeline ，吐出来可以直接入库检索的节点, 这些节点怎么生成index?

---

## Agent 系统设计面试

### Q1: 架构重构动机 — 旧 AgentRunner while-loop → 新 Workflow 事件驱动 DAG

旧版 Agent 系统是经典的 `AgentRunner` + while-loop 模式。现在整个 agent 系统被重写为继承 `Workflow` 的事件驱动 DAG——`BaseWorkflowAgent(Workflow, BaseModel, PromptMixin)`。agent 的执行逻辑被拆成多个 `@step` 方法：`init_run → setup_agent → run_agent_step → parse_agent_output → (call_tool → aggregate_tool_results) → 循环`。

请阐述这次架构重构的核心动机。为什么一个简单的 while-loop 不够用？事件驱动模型解决了哪些老架构无法解决的问题？

**A:** 在以下几个方面事件驱动的DAG都有绝对的优势：
1. 代码可扩展性
2. 代码可读性
3. 并发能力（asyncio 的支持）
4. 可观测性
5. 代码耦合度

### Q2: 并发能力 — 并行工具调用的精确机制

`parse_agent_output` 发现多个 tool_calls 时，它不直接调用工具，而是给每个 tool_call `ctx.send_event(ToolCall(...))`，然后返回 `None`。下游 `call_tool` step 对每一个 ToolCall 事件各自触发生成一个协程。最终 `aggregate_tool_results` 用这行收拢：

```python
tool_call_results = ctx.collect_events(ev, expected=[ToolCallResult] * num_tool_calls)
```

请解释 `ctx.send_event` + `ctx.collect_events(expected=[...])` 这套同步屏障机制在 Workflow 引擎内部是怎样工作的。核心问题：
- 为什么 `call_tool` 能对同一事件产生多个并行实例？引擎如何知道要等 N 个结果才放行？
- `aggregate_tool_results` 返回 `None`（还没收齐）时，引擎做了什么？它如何被重新唤醒？
- 如果某个 tool_call 失败抛异常，`call_tool` 捕获后返回 `ToolCallResult(is_error=True)`，屏障还能正常解除吗？

**A:** call_tool 事件只要发送了多次，`@step`就能生成多个协程。`collect_events` 里面有 `num_tool_calls` 参数就知道等几个结果。返回 `None` 时，当前协程结束，step 保持订阅，等下一个事件到达时重新执行本 step。只要返回 `ToolCallResult`，就计入计数；屏障正常解除。

### Q3: Scratchpad vs Reasoning State — 两种内部状态管理模式的对比

`FunctionAgent` 和 `ReActAgent` 在处理 tool call 结果时采用了截然不同的策略：

- **FunctionAgent**: 维护一个 `scratchpad`（`List[ChatMessage]`），把 LLM 的 assistant 消息和 tool 结果都 append 进去。下一次 `take_step` 时拼接 `[llm_input, *scratchpad]`。最终 `finalize` 时一次性 `memory.aput_messages(scratchpad)` 写入 memory，然后清空 scratchpad。

- **ReActAgent**: 维护 `current_reasoning`（`List[BaseReasoningStep]`），每条是 `ActionReasoningStep` / `ObservationReasoningStep` / `ResponseReasoningStep`。每次 `take_step` 调 `formatter.format(tools, chat_history, current_reasoning)` 把整个 reasoning 链注入 prompt。最终 `finalize` 时把整串 reasoning 拼成一条 assistant 消息写入 memory。

1. 为什么 FunctionAgent 需要 scratchpad 而 ReActAgent 不需要直接把中间消息写 memory？
2. ReActAgent 的 formatter 每轮都把完整的 `current_reasoning` 拼进 prompt，这会导致什么隐患？

**A:** 
1. FunctionAgent 是格式化的 tool 调用（native function calling via `achat_with_tools`），而 ReActAgent 是基于 prompt 的调用。前者更格式化后者更灵活。为了下一轮 LLM 调用能够知道我调用了什么 tool，就需要维护 scratchpad。ReActAgent 不写 memory 没关系因为它是基于 prompt，而 prompt 已经存了完整的 reasoning 链路。**补充**: scratchpad 的另一层意图是隔离"进行中的工作"与"已提交的历史"——如果 agent 中途 handoff 给另一个 agent，对方不应该继承未完成的 tool 调用碎片。

2. 可能导致 prompt 爆炸（context window 膨胀）。ReAct 模式每轮把所有历史推理步骤全部塞进 prompt，token 消耗随迭代次数线性增长。

### Q4: 多重继承的元类困境

`BaseWorkflowAgent` 同时继承 `Workflow`、`BaseModel`（pydantic）和 `PromptMixin`（ABC）。三个父类各有各的元类，必须合并：

```python
class BaseWorkflowAgentMeta(WorkflowMeta, ModelMetaclass): ...
```

`__init__` 里手动分拆 kwargs，把 `timeout/verbose` 传给 `Workflow.__init__`，其余给 `BaseModel.__init__`。

为什么不设计成组合模式——即 `Agent` 持有一个 `Workflow` 实例而不是继承它？

**A:** Workflow 提供的基于事件的循环体系。如果 Agent 只是组合 Workflow 但不是一个 Workflow，那么 `@step` 就不能用了。`@step` 装饰器在元类构造期就把方法注册进 Workflow 内部的 step registry，构成 DAG 节点。组合模式无法实现声明式编程。

*[面试官注: 此题钻牛角尖, 以下转为高价值架构设计问题]*

### Q5: FunctionAgent vs ReActAgent — 实际场景下的选型决策

假设你有一个非 function-calling 的 LLM（比如通过 Ollama 跑的 llama3.1），`AgentWorkflow.from_tools_or_functions()` 会自动检测 LLM 是否为 function-calling model，然后选 `FunctionAgent` 或 `ReActAgent`：

```python
agent_cls = FunctionAgent if llm.metadata.is_function_calling_model else ReActAgent
```

但如果你的 LLM 确实支持 function calling（比如 gpt-4），你依然可以强制用 ReActAgent。在什么场景下你会明知 LLM 支持 function calling 却选择 ReActAgent？反过来，FunctionAgent 除了依赖 LLM 能力，还有哪些架构上的劣势？

待回答...

**A:** FunctionAgent 背后需要用小模型来算 tool use，如果对格式要求很严格可能容易出错，所以容错性差。为什么用某工具的可解释性也差——ReAct 的 Thought 链是显式的。**补充**: 这也是为什么 `CodeActAgent` 折中——代码生成用纯文本 `<execute>` 标签（类 ReAct），只在 handoff 才用 function calling。

### Q6: Handoff 设计 —— 为什么是"工具"而非"事件"？

*[跳过: 面试官认为此题太刁钻]*

### Q8: Agent 系统如何嵌入 LlamaIndex 的 3-Phase 管道？

Agent 出现在 Querying 层。`FunctionAgent.tools` 里可以传入 `QueryEngineTool`（包装了 `VectorStoreIndex.as_query_engine()`）。

**A:** Agent 出现在 Querying 层。是 Agent 驱动 Retrieval。但更准确的是 Agent 把 QueryEngine **升维**了：QueryEngine 是固定线性管道 → Agent 是动态循环，把 QueryEngine 降级为一个可多次调用的工具。Agent 的路由功能本质是 `RouterQueryEngine` 的 LLM 推理版本。

### Q7: 错误自愈机制 — `retry_messages` 的设计

ReActAgent 的 `take_step` 里有两类错误处理：空响应和解析失败，都构造成 `retry_messages`。`parse_agent_output` 检测到 `retry_messages` 非空时，不走 tool_calls 分支，而是重新发回 `AgentInput`。为什么不在 `take_step` 里直接 while 循环重试？

**A:** 每次重试必须经过 `parse_agent_output` 的 `num_iterations += 1` 计数器。如果在 `take_step` 里 while 循环重试，迭代计数不会递增——LLM 可能无限重试而永远不触发 `max_iterations` 上限。事件流兜一圈，本质是把"重试"当作一次正式 step，纳入迭代预算管理。同时每次重试是可观测的。

### Q9: `tool_retriever` — 工具的"延迟检索"模式

`BaseWorkflowAgent` 提供两种指定工具的方式：`tools`（静态列表）和 `tool_retriever`（`ObjectRetriever` 动态检索）。和直接把所有工具放 `tools` 有什么区别？

**A:** 直接把所有工具放 tools 太多，不够灵活，太多还会给 LLM 造成误导让它不知道该用什么工具。LlamaIndex 还需要工具经常可插拔，所以不宜只用静态。冲突时应该是 union 操作。**补充**: `tool_retriever` 是 `ObjectRetriever` 类型，可以把工具描述 embed 进向量库用语义搜索匹配——100 个工具不用全塞 prompt，LLM 只看到最相关的 3-5 个。
