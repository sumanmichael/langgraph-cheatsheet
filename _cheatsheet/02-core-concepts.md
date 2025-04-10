---
title: II. Core Concepts
permalink: /cheatsheet/core-concepts/
layout: post
---

**Motto:** _Control the Flow, Master the State._

LangGraph workflows are built on interconnected **Graphs**, **Nodes**, and **Edges**, all working together to manipulate and evolve **State**. Understanding these core concepts is crucial for building robust, manageable, and powerful LangGraph applications. These are the fundamental building blocks for creating powerful AI applications.

### 2.1. Graph: The Workflow Blueprint

- **Definition:** The interconnected structure or blueprint of your LangGraph application. A directed graph defining the flow of execution. Use `StateGraph` for most applications, or `MessageGraph` for simple chatbots.
- **Purpose:** Defines the application's logic and how data flows between processing steps.
- **Key Classes:** `StateGraph`, `MessageGraph`
- **Code Snippet (Python - Graph Initialization):**

  ```python
  from langgraph.graph import StateGraph

  builder = StateGraph(MyState) # MyState assumed to be defined (e.g., TypedDict or Pydantic model)
  ```

  _Caption: Initializing a StateGraph with a defined State schema._

### 2.2. Node: The Building Blocks of Computation

- **Definition:** A Python (or JavaScript) function representing a single processing step within the graph.
- **Function Signature (Python Example):**

  ```python
  from typing_extensions import TypedDict, Optional
  from langchain_core.runnables import RunnableConfig

  class MyState(TypedDict): # Example State Schema - REPLACE with your actual State
      user_input: str
      llm_output: str

  def my_node(state: MyState, config: Optional[RunnableConfig] = None): # Node function
      # Node logic here (e.g., LLM call, tool use, data processing)
      updated_state = {"llm_output": "Updated output based on input"} # Dictionary representing State updates
      return updated_state
  ```

  _Caption: Example of a Node function signature in Python, showing State input and dictionary output._

  - **Input:** Takes the current `State` (replace `MyState` with your schema).
  - **Output:** Returns a dictionary representing updates to the `State`.
  - **Optional `config`:** Accepts `RunnableConfig` for runtime parameters (like `thread_id`). `# Optional config argument for runtime parameters`

- **Purpose:** Executes a specific task within the workflow.

### 2.3. Edge: Defining the Flow of Execution

- **Introduction:** LangGraph provides different types of Edges to handle various workflow needs:

  - **2.3.1. Normal Edges - Sequential Flow**

    - **Definition:** Unconditional, direct flow from one node to the next. Ensures sequential execution.
    - **How to Use:** Use `builder.add_edge("node_a", "node_b")` to connect "node_a" to "node_b".
    - **Code Snippet (Python - Normal Edge):**
      ```python
      builder = StateGraph(MyState) # Assuming you have a State defined as MyState
      builder.add_node("node_a", node_a_function) # Assuming node_a_function is defined
      builder.add_node("node_b", node_b_function) # Assuming node_b_function is defined
      builder.add_edge("node_a", "node_b") # Normal edge from node_a to node_b - always goes to node_b after node_a
      ```
      _Caption: Example of adding a normal edge for sequential node execution._

  - **2.3.2. Conditional Edges - Dynamic Routing**

    - **Definition:** Enables dynamic routing based on a routing function that evaluates the current `State`. Creates branches in the workflow.
    - **How to Use:** Use `builder.add_conditional_edges("node_a", routing_function, mapping=optional_mapping)`.
      - `"node_a"`: The node from which the conditional branching originates.
      - `routing_function`: A Python function that determines the next node(s).
        - **Input:** Takes the current `State`.
        - **Decision:** Evaluates the `State` to decide the next execution path.
        - **Return:** Returns the name of the next node (string) or a list of node names (for parallel execution). Can also return symbolic outputs to be mapped via `mapping`.
      - `mapping` (Optional): A dictionary to map symbolic outputs from `routing_function` to actual node names (e.g., `{"condition_true": "node_b", "condition_false": "node_c"}`).
    - **Code Snippet (Python - Conditional Edge with Routing Function - No Mapping):**

      ```python
      from typing import Literal
      from langgraph.types import Command

      class MyState(TypedDict): # Assuming MyState is defined
          condition: bool

      def route_fn(state: MyState) -> Command[Literal["node_b", "node_c"]]:
          if state["condition"]: # Check for some condition in the state
              return Command(goto="node_b") # Route to node_b if condition is true
          else:
              return Command(goto="node_c") # Route to node_c otherwise

      builder = StateGraph(MyState)
      builder.add_node("node_a", lambda state: {"condition": True}) # Example Node A setting condition to True
      builder.add_node("node_b", lambda state: {})
      builder.add_node("node_c", lambda state: {})
      builder.add_conditional_edges("node_a", route_fn) # Routing function decides next node
      ```

      _Caption: Example of conditional edges using a routing function that returns node names directly._

    - **Code Snippet (Python - Conditional Edge with Mapping):**

      ```python
      from typing_extensions import TypedDict
      from langgraph.graph import StateGraph, START

      class MyState(TypedDict): # Assuming MyState is defined
          condition_type: str

      def route_fn(state: MyState):
          condition_type = state["condition_type"] # Determine route based on condition_type
          if condition_type == "type_a":
              return "condition_true" # Symbolic output for condition type A
          else:
              return "condition_false" # Symbolic output for other condition types

      builder = StateGraph(MyState)
      builder.add_node("node_a", lambda state: {"condition_type": "type_a"}) # Example Node A setting condition_type
      builder.add_node("node_b", lambda state: {})
      builder.add_node("node_c", lambda state: {})
      builder.add_conditional_edges(
          "node_a",
          route_fn,
          {"condition_true": "node_b", "condition_false": "node_c"} # Mapping: symbolic outputs to node names
      )
      ```

      _Caption: Example of conditional edges with a mapping dictionary for symbolic routing function outputs._

  - **2.3.3. `Send` - Dynamic Parallelism**

    - **Definition:** A special object used in routing functions to dynamically create edges to a node for parallel execution. Enables map-reduce patterns.
    - **How to Use:** Return a list of `Send` objects from a routing function within `add_conditional_edges`.
    - **`Send` Object Components:**
      - `goto`: The name of the node to execute in parallel.
      - `config`: The `State` to be passed to each instance of the destination node.
    - **Code Snippet (Python - `Send` for Map-Reduce):**

      ```python
      from langgraph.constants import Send
      from typing import Annotated, List, TypedDict
      import operator

      class MapState(TypedDict): # State for our map-reduce example
          items_to_process: list[str]
          processed_results: Annotated[list, operator.add] # Reducer to combine results

      def map_node(state: MapState):
          # For each item in items_to_process, create a Send object to "process_item_node"
          sends = [Send("process_item_node", {"item": item}) for item in state["items_to_process"]]
          return sends # Return a list of Send objects - dynamic edge creation!

      def process_item_node(state: dict): # Note: State here can be just a dict, not necessarily MapState
          item = state["item"] # Get the 'item' from the state passed by Send
          # ... do some processing on the item ...
          processed_item = f"Processed: {item}" # Example processing
          return {"processed_results": [processed_item]} # Return the processed result


      builder = StateGraph(MapState)
      builder.add_node("map_node", map_node)
      builder.add_node("process_item_node", process_item_node)
      builder.add_conditional_edges("map_node", map_node, ["process_item_node"]) # Dynamic edges based on map_node's output
      ```

      _Caption: Example of using `Send` in a routing function to create dynamic edges for parallel processing._

  - **2.3.4. `Command` - State Updates + Control in One**

    - **Definition:** A special object returned by nodes to combine state updates and control flow decisions within a single node function.
    - **How to Use:** Return a `Command` object instead of a State update dictionary from a node function.
    - **`Command` Object Arguments:**
      - `update` (optional): A dictionary of State updates.
      - `goto` (optional): The name of the next node to execute (string).
      - `graph` (optional): For subgraph navigation, use `Command.PARENT` to navigate to a node in the parent graph.
    - **Type Annotations:** Crucially, annotate node functions returning `Command` with the possible destination node names: `-> Command[Literal["node_a", "node_b"]]`.
    - **Code Snippet (Python - Basic `Command` Usage):**

      ```python
      from langgraph.types import Command
      from typing import Literal, TypedDict

      class MyState(TypedDict): # Assuming MyState is defined
          value: str

      def command_node(state: MyState) -> Command[Literal["node_b"]]: # Return type annotation is key here
          return Command(update={"value": "updated_value"}, goto="node_b") # Update state AND route to node_b
      ```

      _Caption: Example of a node function returning a `Command` object for combined state update and routing._

    - **Code Snippet (Python - `Command` for Dynamic Routing within a Node):**

      ```python
      from langgraph.types import Command
      from typing import Literal, TypedDict

      class MyState(TypedDict): # Assuming MyState is defined
          value: str

      def command_routing_node(state: MyState) -> Command[Literal["node_b", "node_c"]]: # Possible destinations: node_b or node_c
          if state["value"] == "condition_met":
              return Command(goto="node_b") # Route to node_b if condition met
          else:
              return Command(goto="node_c") # Route to node_c otherwise
      ```

      _Caption: Example of using `Command` for dynamic routing logic within a node._

    - **Code Snippet (Python - `Command` for Navigating to a Parent Graph from a Subgraph):**

      ```python
      from langgraph.types import Command
      from typing import Literal, TypedDict

      class SubgraphState(TypedDict): # Assuming SubgraphState is defined
          value: str

      def subgraph_node(state: SubgraphState) -> Command[Literal["parent_graph_node"]]: # Going to "parent_graph_node" in parent graph
          return Command(goto="parent_graph_node", graph=Command.PARENT) # Navigate UP to the parent graph!
      ```

      _Caption: Example of using `Command` to navigate from a subgraph node to a node in the parent graph._

    - **When to Use `Command` vs. Conditional Edges?**
      - Use `Command` when you need to **both** update the graph state **and** route to a different node from _within_ a node function.
      - Use Conditional Edges for routing between nodes based on state _without_ combining it directly with state updates in the routing logic itself.

### 2.4. State: Shared Memory and Data Structure

- **Definition:** A shared data structure that holds the application's data throughout its execution, acting as the graph's memory.
- **Schema:** Defines the structure, data types, and validation rules for the State. Crucial for data integrity and clarity.

  - **`TypedDict`:** Lightweight and straightforward for defining State schema using Python's type hinting system. Best for simpler state structures where runtime validation isn't critical.
  - **`Pydantic BaseModel`:** Provides robust data validation at runtime, default values, and serialization capabilities. Ideal for complex applications requiring data integrity and structured data management.
  - **Channels:** Keys within the State schema act as communication channels, enabling data flow and information sharing between different Nodes in the graph. Nodes read from and write to these channels to process and update the application's data.
  - **Code Snippet (Python - State Schema with Pydantic):**

    ```python
    from pydantic import BaseModel, field_validator, ValidationError

    class MyState(BaseModel):
        user_input: str = Field(..., description="User's free-form text input")
        llm_output: str = Field("", description="Raw output from the Language Model")
        processed_output: str = Field("", description="Processed and refined output")
        is_valid_input: bool = Field(False, description="Flag indicating if user input is valid")

        @field_validator('user_input') # Example validation using Pydantic validators
        @classmethod
        def check_user_input(cls, value):
            if not isinstance(value, str) or len(value.strip()) == 0:
                raise ValueError("User input must be a non-empty string")
            return value
    ```

    _Caption: Example of defining a State schema using Pydantic BaseModel with data validation and field descriptions._

### 2.5. Reducer: Managing Concurrent State Updates (Conflict Resolution)

- **Definition:** Functions that specify how updates to the `State` are merged or combined, especially important when multiple nodes attempt to update the same State key concurrently (e.g., in parallel branches).
- **Purpose:** Resolves potential conflicts arising from concurrent state modifications, ensuring data consistency and predictable State evolution.
- **Default Behavior (Overwrite Reducer):** If no reducer is explicitly defined for a State key, the default behavior is to simply overwrite the existing value with the latest update. This can lead to data loss or unpredictable behavior in concurrent scenarios.
- **Custom Reducers (via `Annotated`):** To implement custom merging or combining logic, use `Annotated` from the `typing` module to associate a reducer function with a specific State key in your schema. This allows fine-grained control over state updates.

  - **Example: `operator.add` for lists:** Uses Python's `operator.add` to concatenate lists. Useful for accumulating data from parallel branches into a single list within the State.
  - **Example: `add_messages` for message history:** A pre-built reducer specifically designed for managing chat message histories. Appends new messages to the existing list of messages in the State, preserving conversation flow.
  - **Example: Custom Reducer Function:** Define your own Python function to implement highly specific merging or conflict resolution logic (e.g., merging dictionaries based on keys, applying custom aggregation rules).

  - **Code Snippet (Python - Reducer Example with `operator.add`):**

    ```python
    from operator import add
    from typing import Annotated
    from typing_extensions import TypedDict

    class ListState(TypedDict):
        items: Annotated[list[str], add]  # 'add' reducer for the 'items' key - appends lists

    def node_a(state: ListState):
        return {"items": ["item_a"]} # Node A returns a list of items

    def node_b(state: ListState):
        return {"items": ["item_b"]} # Node B also returns a list of items

    builder = StateGraph(ListState)
    builder.add_node("node_a", node_a)
    builder.add_node("node_b", node_b)
    builder.add_edge(START, "node_a")
    builder.add_edge(START, "node_b") # Parallel edges from START to node_a and node_b
    graph = builder.compile()

    result = graph.invoke({"items": []}) # Initial state with an empty list
    print(result['items']) # Output will be ['item_a', 'item_b'] - lists are concatenated by the 'add' reducer
    ```

    _Caption: Example of using `operator.add` reducer to combine list-based State updates from parallel nodes._

### 2.6. START and END Nodes: Defining Graph Boundaries

- **START Node:** A virtual node representing the graph's entry point. Graph execution begins at the node(s) directly connected to `START` via edges. Use `builder.add_edge(START, "node_name")` to designate starting nodes.
- **END Node:** A virtual node signifying the graph's termination point. When execution reaches a node with an outgoing edge to `END`, the graph run is considered complete. Use `builder.add_edge("node_name", END)` to define terminal nodes in the workflow.
