---
title: TLDR; - LangGraph Development Cheatsheet and Guidelines
permalink: /cheatsheet/tldr/
layout: post
---

**Motto:** _Control Flow, Master State._

LangGraph reimagines AI application development beyond simple chains, offering a graph-based framework for building sophisticated workflows with Large Language Models. It addresses the limitations of linear chains by providing developers with fine-grained control over application logic, state management, and execution flow, enabling the creation of robust and complex agentic systems.

This cheatsheet provides a concise, actionable guide to LangGraph development, covering core concepts, best practices, debugging techniques, and common use cases, empowering developers to build high-quality, production-ready AI applications with greater efficiency and mastery.

#### Core Concepts

- **Graph:** Workflow Blueprint. _Interconnected Nodes & Edges_. Defines app logic. `StateGraph` (versatile), `MessageGraph` (chatbots).
- **Node:** Computation Building Block. _Python/JS function_. Input: `State`. Output: State updates/`Command`. _Executes tasks: LLM call, tool use, routing._
- **Edge:** Defines Execution Flow. _Connections between Nodes_. Normal (sequential), Conditional (dynamic routing), `Send` (parallel tasks), `Command` (state+control).
- **State:** Shared Application Memory. _Data Structure_. Schema (`TypedDict`/Pydantic `BaseModel`). _Channels for node communication_.
- **Reducer:** Concurrent State Update Manager. _Functions to combine/merge State updates_. Crucial for lists, parallel branches.

#### Quick Setup

- **Install (Python):** `pip install langgraph langchain_openai`
- **Import:** `from langgraph.graph import StateGraph, START, END; from langgraph.types import Command, Send, interrupt; from langchain_core.runnables import RunnableConfig`
- **State (Pydantic):**
  ```python
  class MyState(BaseModel): user_input: str; llm_output: str = ""
  ```

#### Key Operations

- **Add Node:** `builder.add_node("node_name", node_function)`
- **Normal Edge:** `builder.add_edge("node_a", "node_b")`
- **Conditional Edge:** `builder.add_conditional_edges("node_a", route_fn)`
- **Compile Graph:** `graph = builder.compile(checkpointer=MemorySaver())`
- **Invoke Graph:** `graph.invoke({"input": "val"}, config={"configurable": {"thread_id": "thread_1"}})`
- **Stream Output:** `graph.stream({"input": "val"}, config=config, stream_mode="updates")`
- **`Command` in Node:** `def command_node(state: MyState) -> Command[Literal["node_b"]]: return Command(goto="node_b")`
- **`interrupt` (HITL):** `user_input = interrupt({"q": "Approve?"}); return Command(resume=user_input)`

#### Key Features

- **Persistence:** Checkpointers, Threads, Memory, Time Travel, Fault-Tolerance. _Save progress, enable advanced features_.
- **Streaming:** `values`, `updates`, `tokens` modes. _Real-time UX, responsiveness_.
- **Configuration:** `config_schema`, `configurable`. _Dynamic, reusable, adaptable graphs_.
- **Recursion Limit:** Prevents infinite loops. _Safety net for runaway executions_.
- **Subgraphs:** Modular apps, nested graphs, agent teams. _Encapsulation, reusability_.
- **Breakpoints:** `interrupt_before`, `interrupt_after`, `NodeInterrupt`. _Debugging, human-in-the-loop control_.
- **`interrupt`:** Human-in-the-loop. _Pause, resume, human feedback integration_.

#### Use Cases & Patterns

- **Router Agents:** LLM Traffic Controllers. Route to specialized nodes. _Intent-based, Adaptive RAG, Multi-Agent dispatch_.
- **Tool-Calling (ReAct) Agents:** Reason-Act-Observe. LLM + Tools for complex tasks. _Planning, memory, external interaction_.
- **Multi-Agent Systems:** AI Teams. Network, Supervisor, Hierarchical. _Modularity, handoffs, agent communication & coordination_.
- **Human-in-the-Loop (HITL):** Human-AI Collaboration. `interrupt` for approval, editing, review, multi-turn. _Human oversight, guidance, error correction_.
- **Map-Reduce:** Parallel Power. `Send` for parallel tasks, Reducers for aggregation. _Scalability, efficient data processing_.
- **Self-Correcting Agents (Reflection):** AI Learns. LLM or rule-based evaluators, feedback loops. _Iterative improvement, self-optimization_.

#### Troubleshooting, Performance, Fixes & "Gotchas"

- **`StateGraph` vs `MessageGraph`?:** `StateGraph` = default, `MessageGraph` = chatbots (simple).
- **Node Errors?:** `try-except` + logging + graceful degradation.
- **Debug LangGraph?:** Breakpoints, Time Travel, LangSmith, Logging (essential tools).
- **Human-in-the-loop?:** Use `interrupt(value)` & `Command(resume=value)`.
- **Optimize Performance?:** Streaming, Parallel, Lean State, Recursion Limit (tune).
- **Side Effects & `interrupt`:** Avoid side effects _before_ `interrupt` - re-execution trap!
- **Resume Re-executes Node:** `interrupt` resumes _entire node_, not line-by-line.
- **Subgraph State:** Isolated. Explicitly manage State flow between graphs.
- **Reducers (Parallel):** _Mandatory_ for concurrent State updates. Prevent `InvalidUpdateError`.
- **Recursion Limit:** Safety net, not design fix. Design graphs for termination.
- **Dynamic `interrupt` Nodes:** Complex, use with caution.

Happy LangGraph building! 🚀
