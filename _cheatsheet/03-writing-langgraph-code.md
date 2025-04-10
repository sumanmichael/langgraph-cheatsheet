---
title: III. Writing LangGraph Code - Guidelines & Best Practices
permalink: /cheatsheet/writing-langgraph-code/
layout: post
---

Write clean, maintainable, and robust LangGraph code with these guidelines. Focus on clarity, efficiency, and best practices for building production-ready applications.

### Quick Wins: Top 3 Best Practices for LangGraph Code

- **Modular Nodes:** Keep nodes focused on single tasks for clarity and reusability.
- **Type Hints Everywhere:** Use type hints for error prevention and readability.
- **Embrace `async`:** Use async nodes for I/O-bound operations to maximize responsiveness.

### 3.1. Structuring Nodes and Edges

- **Guideline 3.1.1: Modular Nodes**

  - **Explanation:** Each node should perform a single, well-defined task. Improves readability, debugging, and code reuse. Adhere to Single Responsibility Principle.
  - **Do:** Separate nodes for distinct operations: LLM calls, tool executions, routing, data transformations.
  - **Avoid:** "God Nodes" performing multiple unrelated tasks.
  - **Code Snippet (Python - Example of Modular Nodes):**

    ```python
    # Good: Modular Nodes
    def generate_query_node(state: SearchState):
        return {"search_query": query}  # Query generation only

    def web_search_node(state: SearchState):
        return {"search_results": results} # Web search only

    # Bad: Non-Modular Node
    def search_and_generate_query_node(state: SearchState): # Node doing too much
        return {"search_results": results, "search_query": query}
    ```

    _Caption: Modular (good) vs. non-modular "God Node" (bad) examples._

- **Guideline 3.1.2: Clear Node Signatures**

  - **Explanation:** Use type hints for node function signatures (`State` input, State updates/`Command` output). Improves code understanding and error detection.
  - **Do:** Always annotate node functions with expected State and return types.
  - **Benefit:** Enhances readability, enables static analysis, reduces runtime errors.
  - **Code Snippet (Python - Node Function with Type Hints):**

    ```python
    from typing_extensions import TypedDict, Optional
    from langchain_core.runnables import RunnableConfig

    class SearchState(TypedDict):
        search_query: str
        search_results: str

    def web_search_node(state: SearchState, config: Optional[RunnableConfig] = None) -> Dict[str, str]:
        query = state['search_query']
        results = perform_search(query)
        return {"search_results": results}
    ```

    _Caption: Node function with clear type hints._

- **Guideline 3.1.3: Descriptive Node Names**

  - **Explanation:** Use node names that clearly describe the node's function. Improves graph readability and self-documentation.
  - **Do:** Choose action-oriented names reflecting the node's purpose (e.g., `generate_summary`, `call_api`).
  - **Avoid:** Generic names (e.g., `node1`, `step_2`).
  - **Examples:** `generate_search_query`, `fetch_web_page`, `extract_information`, `summarize_text`, `route_based_on_intent`.

- **Guideline 3.1.4: Routing Function Best Practices**

  - **Explanation:** Routing functions should be concise and focused on decision-making, avoiding complex computations or side effects.
  - **Do:** Focus routing logic on determining the next node based on State.
  - **Avoid:** Complex computations, LLM calls, API requests within routing functions.
  - **Return Values:** Return clear node names or symbolic outputs that are easily mapped.
  - **Code Snippet (Python - Well-structured Routing Function):**

    ```python
    from typing import Literal
    from langgraph.types import Command
    from typing_extensions import TypedDict

    class MyState(TypedDict):
        intent: str

    def route_based_on_intent(state: MyState) -> Command[Literal["search_node", "question_node"]]:
        intent = state["intent"]
        if intent == "search":
            return Command(goto="search_node")
        else:
            return Command(goto="question_node")
    ```

    _Caption: Concise routing function for decision-making._

- **Guideline 3.1.5: Sync vs. Async Nodes**

  - **Explanation:** Use sync (`def`) for CPU-bound nodes and async (`async def`) for I/O-bound nodes to optimize performance.
  - **Use Sync Nodes (`def`):** For CPU-intensive tasks, in-memory processing.
  - **Use Async Nodes (`async def`):** For I/O-bound tasks: LLM calls, API requests, network operations.
  - **Benefit of Async:** Improves responsiveness, concurrency, resource utilization for I/O heavy tasks.
  - **Code Snippet (Python - Async Node Example):**

    ```python
    from langchain_openai import ChatOpenAI
    from typing_extensions import TypedDict, Optional
    from langchain_core.runnables import RunnableConfig

    class LLMState(TypedDict):
        user_query: str
        llm_response: str

    async def llm_call_node(state: LLMState, config: Optional[RunnableConfig] = None) -> Dict[str, str]:
        llm = ChatOpenAI(model="gpt-4o")
        response = await llm.ainvoke(state['user_query']) # Non-blocking async LLM call
        return {"llm_response": response.content}
    ```

    _Caption: Async node for non-blocking I/O operations._

### 3.2. State Management Strategies

- **Guideline 3.2.1: Well-Defined State Schema**

  - **Explanation:** Design a clear, comprehensive State schema upfront. Foundation for organized data flow and maintainability.
  - **Do:** Define State schema (TypedDict/Pydantic) representing all necessary application data.
  - **Benefit:** Improves code clarity, reduces errors, enhances maintainability.
  - **Recommendation:** Plan State schema before coding nodes/edges.

- **Guideline 3.2.2: Choose the Right State Type (`TypedDict` vs. Pydantic)**

  - **Explanation:** Select State type based on complexity and validation needs.
  - **Use `TypedDict`:** For simpler structures, type hinting, lightweight.
  - **Use Pydantic `BaseModel`:** For robust validation, defaults, serialization, complex structures.
  - **Trade-off:** Pydantic adds overhead, but often worth it for validation and features.

- **Guideline 3.2.3: Reducer Strategy**

  - **Explanation:** Use reducers for State keys (especially lists/complex types) to manage concurrent updates and prevent conflicts.
  - **Do:** Define reducers using `Annotated` for keys needing custom update logic.
  - **When to Use:** Always for concurrent updates in parallel branches.
  - **Reducer Types:** `operator.add` (lists), `add_messages` (chat history), custom functions.
  - **Code Snippet (Python - Reducer Example with `operator.add`):**

    ```python
    from operator import add
    from typing import Annotated
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START

    class ListState(TypedDict):
        items: Annotated[list[str], add]  # 'add' reducer for 'items'

    # ... (rest of the code showing node_a, node_b, graph definition, and invoke)
    ```

    _Caption: `operator.add` reducer concatenates list updates._

- **Guideline 3.2.4: Minimize State Updates**

  - **Explanation:** Optimize graph performance by reducing unnecessary State updates.
  - **Do:** Update State keys only when values change or information needs to be passed downstream.
  - **Avoid:** Redundant or trivial State updates in every node.

- **Guideline 3.2.5: Memory Management**
  - **Explanation:** Manage memory in long-running apps to avoid excessive State growth and LLM context limits.
  - **Strategies:** Message trimming, summarization, external memory stores.
  - **Goal:** Balance context retention with performance and cost.

### 3.3. Error Handling & Robustness

- **Guideline 3.3.1: Node-Level Error Handling**

  - **Explanation:** Implement `try-except` within nodes to handle potential errors gracefully and prevent crashes.
  - **Do:** Wrap error-prone operations (LLM calls, API requests) in `try-except`.
  - **Error Handling:** Retry, fallback, logging, signal error to graph (advanced).

- **Guideline 3.3.2: Logging**

  - **Explanation:** Use logging extensively for debugging, monitoring, and understanding execution flow.
  - **Log:** Node entry/exit, input/output State, variable values, errors.
  - **Use Logging Levels:** DEBUG, INFO, WARNING, ERROR for verbosity control.

- **Guideline 3.3.3: Graceful Degradation**

  - **Explanation:** Design graphs to handle errors without complete failure. Implement fallbacks and alternative paths for resilience.
  - **Techniques:** Conditional edges, fallback nodes, caching.

- **Guideline 3.3.4: Fault-Tolerance with Persistence**
  - **Explanation:** Use checkpointers for fault-tolerance and error recovery, enabling graph resumption after interruptions.
  - **Benefit:** Minimize data loss, enable error recovery, improve reliability.
  - **Implementation:** Compile graph with checkpointer, use `thread_id` in `config`.

### 3.4. Asynchronous Programming in LangGraph

- **Guideline 3.4.1: Use `async def` for I/O-Bound Nodes**

  - **Explanation:** Define I/O-bound nodes (LLM calls, APIs) as async functions (`async def`) for non-blocking operations.
  - **I/O-Bound Operations:** Network requests, file I/O.
  - **Benefit:** Improved responsiveness, concurrency, resource utilization.

- **Guideline 3.4.2: `await` Asynchronous Operations**

  - **Explanation:** Use `await` when calling async functions within `async def` nodes to prevent blocking.
  - **`await` Keyword:** Pauses execution only for the awaited operation, allowing concurrent tasks.
  - **Avoid:** Blocking synchronous calls in async nodes.

- **Guideline 3.4.3: Asynchronous Checkpointers and Stores**

  - **Explanation:** Use async checkpointers/stores (`AsyncSqliteSaver`, `AsyncPostgresSaver`) for async graphs to ensure end-to-end non-blocking flow.

- **Guideline 3.4.4: Benefits of Async Execution**
  - **Improved Responsiveness:** Smoother user experience, even during long LLM calls.
  - **Increased Concurrency:** Handle multiple tasks concurrently, maximizing throughput.
  - **Better Resource Utilization:** Efficient resource usage, improved scalability.

### 3.5. Configuration Best Practices

- **Guideline 3.5.1: Define `config_schema`**

  - **Explanation:** Use `config_schema` to formally declare configurable parameters in `StateGraph`. Improves code clarity.
  - **Benefit:** Self-documenting code, discoverable options, enhanced maintainability.
  - **Code Snippet (Python - StateGraph with `config_schema`):**

    ```python
    from typing_extensions import TypedDict
    from langchain_core.runnables import RunnableConfig

    class GraphConfigSchema(TypedDict):
        llm_model_name: str
        temperature: float

    builder = StateGraph(MyState, config_schema=GraphConfigSchema)
    ```

    _Caption: StateGraph initialized with `config_schema`._

- **Guideline 3.5.2: Modular Configuration**

  - **Explanation:** Group config parameters logically (e.g., `llm_config`, `data_source_config`) for better organization.
  - **Benefit:** Improved organization, reusability, simplified management.

- **Guideline 3.5.3: Dynamic Configuration**

  - **Explanation:** Use `configurable` for runtime adaptability: switch models, prompts, customize behavior dynamically.
  - **Use Cases:** Model selection, dynamic prompts, user-specific settings, A/B testing.

- **Guideline 3.5.4: Centralized Configuration**

  - **Explanation:** Manage configuration in a central location for consistency and easier updates.
  - **Options:** `langchain.json` (Platform), `.env` files (local), dedicated config systems (large deployments).

- **Guideline 3.5.5: Runtime Arguments**
  - **Explanation:** Use runtime `config` arguments (in `invoke`, `.stream`) for per-request or frequently changing parameters.
  - **Use Cases:** User-specific settings, per-request customization, A/B testing overrides.
  - **Use Sparingly:** Avoid overusing for core configuration settings.

---

### 3.6. Checklist: Key Questions to Ask Yourself When Writing LangGraph Code

- **Node Structure:**

  - Are my nodes modular and focused on a single, well-defined task? (Guideline 3.1.1)
  - Are node functions clearly named and descriptive of their purpose? (Guideline 3.1.3)
  - Are node function signatures clearly type-hinted for `State` and return values? (Guideline 3.1.2)
  - Is routing function logic concise and focused solely on routing decisions? (Guideline 3.1.4)
  - Have I used `async` nodes for all I/O-bound operations (LLM calls, APIs, network)? (Guideline 3.1.5)

- **State Management:**

  - Is my State schema well-defined and comprehensive for all application data? (Guideline 3.2.1)
  - Have I chosen `TypedDict` or `Pydantic` appropriately for my State complexity and validation needs? (Guideline 3.2.2)
  - Are reducers implemented for all State keys updated in parallel branches to prevent conflicts? (Guideline 3.2.3)
  - Have I minimized State updates, updating keys only when truly necessary? (Guideline 3.2.4)
  - For long-running apps, have I considered memory management (trimming, summarization, external stores)? (Guideline 3.2.5)

- **Error Handling & Robustness:**

  - Are error-prone operations in nodes wrapped in `try-except` blocks for graceful handling? (Guideline 3.3.1)
  - Is comprehensive logging implemented within nodes for observability and debugging? (Guideline 3.3.2)
  - Does the graph incorporate graceful degradation strategies for error conditions? (Guideline 3.3.3)
  - Is fault-tolerance enabled by compiling the graph with a checkpointer for error recovery? (Guideline 3.3.4)

- **Asynchronous Programming:**

  - Are nodes performing I/O-bound operations defined as `async def` functions? (Guideline 3.4.1)
  - Within async nodes, are all asynchronous operations called using `await` (non-blocking)? (Guideline 3.4.2)
  - Are asynchronous checkpointers and stores used for async graphs to maintain async flow? (Guideline 3.4.3)

- **Configuration:**
  - Is a `config_schema` defined for the `StateGraph` to declare configurable parameters? (Guideline 3.5.1)
  - Are configuration parameters organized logically into modular groups? (Guideline 3.5.2)
  - Is the `configurable` feature leveraged to dynamically adapt graph behavior at runtime? (Guideline 3.5.3)
  - Is configuration managed centrally (e.g., `langchain.json`, `.env`) for consistency? (Guideline 3.5.4)
  - Are runtime arguments used sparingly, only for per-request or frequent tweaks? (Guideline 3.5.5)

### 3.7. Common Mistakes to Avoid (“Anti-Patterns”)

- **"God Nodes":** Overly complex, multi-task nodes.
- **Missing Type Hints:** Lack of type hints in node/edge functions.
- **Complex Routing Functions:** Overly complex logic in routing functions.
- **Blocking Calls in Async Nodes:** Using synchronous calls in `async def` nodes.
- **Forgetting Reducers in Parallel Branches:** No reducers for concurrent State updates.
- **Ignoring Recursion Limit:** Unbounded graph execution risks.
- **Hardcoding API Keys:** Embedding secrets directly in code.
- **Lack of Error Handling:** No `try-except` for error-prone operations.

#### 📚 Further Reading:

- **LangGraph Documentation - Concepts:** [Link to LangGraph Concepts Section]
- **LangGraph Documentation - How-to Guides:** [Link to LangGraph How-to Guides]
