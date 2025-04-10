---
title: V. Troubleshooting & Debugging
permalink: /cheatsheet/troubleshooting-debugging/
layout: post
---

Debugging LangGraph apps is key. This guide provides concise solutions to common errors and debugging techniques.

### 5.1. Common Errors and Solutions

- **Error 5.1.1: `InvalidUpdateError` - State Update Conflicts**

  - **Explanation:** Concurrent nodes update same State key without a reducer. _State collision._
  - **Symptoms:** `InvalidUpdateError` exception. Graph halts.
  - **Solution:** Define a **reducer function** (using `Annotated`) for the conflicting State key.
  - **Code Snippet (Python - Reducer for `InvalidUpdateError`):**

    ```python
    from operator import add
    from typing import Annotated
    from typing_extensions import TypedDict

    class ListState(TypedDict):
        items: Annotated[list[str], add]  # 'add' reducer
    ```

    _Caption: Reducer example for `InvalidUpdateError`._

- **Error 5.1.2: `GraphRecursionError` - Runaway Graphs**

  - **Explanation:** Graph exceeds `recursion_limit`. Likely infinite loop or excessive complexity.
  - **Symptoms:** `GraphRecursionError` exception. Graph stops.
  - **Solution:**
    1. **Increase `recursion_limit` (Cautiously):** `config={"recursion_limit": <higher_value>}`.
    2. **Review Graph Design (Crucially):** Simplify workflow, fix loops.
  - **Caution:** Fix design first. Increasing limit is a workaround.

- **Error 5.1.3: Type Errors - Schema Mismatches**

  - **Explanation:** Node/edge State data incompatible with `State` schema. _Data type mismatch._
  - **Symptoms:** `TypeError`, `ValidationError`, unexpected behavior.
  - **Solution:**
    1. **Verify State Schema:** Check `State` definitions.
    2. **Inspect Node/Edge Functions:** Ensure type adherence.
    3. **Use Type Hints:** Static analysis for early detection.

- **Error 5.1.4: API Key Errors - Authentication Failures**

  - **Explanation:** Missing or invalid API keys for LLMs/services. _"Authentication Failed!"_
  - **Symptoms:** `AuthenticationError`, `HTTPError` (401, 403) API exceptions.
  - **Solution:**
    1. **Verify Env Vars:** Check API keys in environment variables.
    2. **Check API Key Validity:** Ensure keys are active and valid.
    3. **Permissions:** Verify key permissions.

- **Error 5.1.5: Serialization/Deserialization Errors - State Storage Issues**
  - **Explanation:** State saving/loading fails due to serialization issues. _"Cannot save/load State!"_
  - **Symptoms:** Checkpointing exceptions during graph execution.
  - **Solution:**
    1. **Serializable State:** Ensure `State` schema is JSON-serializable.
    2. **Pydantic for Robustness:** Use Pydantic `BaseModel` for State.
    3. **Custom Serializers (Advanced):** For highly specialized data types only.

### 5.2. Debugging Techniques - Your Streamlined Toolkit

- **Technique 5.2.1: Breakpoints - Pause and Inspect**

  - **Explanation:** Pause graph execution for step-by-step debugging and State inspection.
  - **Types:**
    - **Static Breakpoints:** `interrupt_before`/`interrupt_after` (compile/runtime). Node-by-node stepping.
    - **Dynamic Breakpoints:** `NodeInterrupt` (conditional, within nodes). Pause on specific State.
  - **Action:** Inspect `State`, variables. Resume step-by-step.
  - **Code Snippet (Python - Static Breakpoint - `interrupt_before` Compilation):**
    ```python
    graph_with_breakpoint = builder.compile(interrupt_before=["my_node"], checkpointer=memory_saver) # Breakpoint before 'my_node'
    ```
    _Caption: Breakpoint set before `my_node`._

- **Technique 5.2.2: Time Travel - Rewind and Fork**

  - **Explanation:** Revisit past runs to reproduce errors and explore alternative paths.
  - **Actions:**
    - **Replaying:** Re-run past executions up to a checkpoint. Analyze decisions.
      - `graph.get_state_history(thread_config)`: Get checkpoint history.
      - `config={'configurable': {'thread_id': 'thread_id', 'checkpoint_id': 'checkpoint_id'}}`: Invoke for replay.
    - **Forking:** Branch from a checkpoint, modify `State`, explore alternatives.
      - `graph.update_state(config, update_values, checkpoint_id=...)`: Fork and modify State.
  - **Benefit:** Debug non-deterministic agents, explore scenarios efficiently.

- **Technique 5.2.3: LangSmith Tracing - Visual Run Analysis**

  - **Explanation:** Visual, detailed traces of graph runs. Powerful observability for LangGraph.
  - **LangSmith Features:** Run traces, node timings, visual graph representations, State transitions, error tracking.
  - **Benefit:** Identify bottlenecks, understand complex flows, pinpoint errors visually.

- **Technique 5.2.4: Logging - Detailed Execution Audits**
  - **Explanation:** Comprehensive logging within nodes for execution tracking and issue diagnosis.
  - **Log:** Node entry/exit, input/output State, variables, errors, API calls.
  - **Use Logging Levels:** DEBUG, INFO, WARNING, ERROR for verbosity control.

### 5.3. Understanding Stack Traces in LangGraph - Decode Error Messages Quickly

- **Explanation:** Stack traces pinpoint error locations. Learn to read them for efficient debugging.
- **Traceback Context:** Follow the function call chain.
  - **Node Function Names:** Identify the failing node.
  - **File and Line Numbers:** Exact error location in code.
- **Node Function Errors:** Errors within your node's code (LLM calls, tools). Stack trace points to node code.
- **Graph Execution Errors:** Errors in LangGraph internals (`InvalidUpdateError`, `GraphRecursionError`). Focus on the **error message**.
- **LangSmith Trace Enrichment:** LangSmith enhances stack traces with visual context, linking errors to nodes and State.
