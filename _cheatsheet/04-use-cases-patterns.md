---
title: IV. Use Cases & Patterns
permalink: /cheatsheet/use-cases-patterns/
layout: post
---

Okay, theory time is (mostly) over! Let's get into the exciting part: what can you actually _build_ with LangGraph? This chapter is all about common agentic patterns and some more advanced use cases where LangGraph really shines.

### 4.1. Common Agentic Patterns

These are some of the fundamental patterns you'll see pop up again and again when building agentic applications. LangGraph makes it easier to implement these in a structured and controllable way.

#### 4.1.1. Router Agents

Router agents are like smart traffic controllers for your AI workflows. They use an LLM to decide which path to take next, routing execution to different specialized nodes based on the input or the current `State`. Think of them as intelligent dispatchers.

- **Structured Output for Routing:** Router agents often use _structured outputs_ (remember Section 2.3.1 on configuration and schemas?) to make sure the LLM gives its routing decision in a predictable format. You can use tools like `with_structured_output` (from LangChain) and define a schema that lists the possible routing options (e.g., using `Literal["path_a", "path_b"]` in Python to say the LLM must choose between "path_a" and "path_b").

- **Conditional Edges for Routing:** Conditional edges (Section 2.2.2) are the mechanism for implementing the actual routing logic. The `routing_function` in your conditional edge uses the LLM's structured output to figure out which node to execute next. It's like reading the traffic controller's signal and choosing the right lane.

- **Use Cases for Router Agents:**
  - **Intent-Based Routing:** Route user requests to different nodes based on the user's _intent_. For example:
    - If the intent is "question answering," route to a node that handles question answering.
    - If the intent is "task execution" (like booking a flight), route to a task execution node.
    - If it's just "chit-chat," route to a conversational node.
  - **Adaptive RAG (Retrieval-Augmented Generation):** Route questions to different _retrieval strategies_ based on the type of question. For instance:
    - For keyword-based questions, use keyword search.
    - For semantic questions, use semantic search.
    - For knowledge-graph related questions, query a knowledge graph.
  - **Multi-Agent Coordination:** In multi-agent systems (Section 4.1.3), a router agent can decide which specialized agent should handle a particular task based on the task type or complexity.

#### 4.1.2. Tool-Calling (ReAct) Agents

Tool-calling agents, often built using the ReAct (Reason + Act) pattern, are incredibly powerful. They use LLMs to dynamically figure out which _tools_ to use and _how_ to use them to solve complex problems. Think of them as agents with access to a toolbox.

- **`create_react_agent`:** LangGraph provides a handy `create_react_agent` function to make building ReAct-style agents much easier. It sets up a lot of the boilerplate for you.

- **Tool Binding:** Use `llm.bind_tools(tools)` (from LangChain) to attach a list of tools to your LLM. This makes those tools available for the agent to use when it's reasoning and acting. It's like giving the agent access to its toolbox.

- **Tool Calling Logic:** The agent uses an LLM to decide:

  - Should I call a tool? (Reasoning)
  - If yes, _which_ tool should I call? (Tool Selection)
  - What _arguments_ should I pass to the tool? (Parameter Generation)
    This decision-making is based on the user input, the current `State`, and the tools available.

- **Observation and Reasoning Loop:** ReAct agents work in a loop:

  1.  **Reasoning:** The LLM analyzes the input and decides if it needs to use a tool or if it can respond directly.
  2.  **Acting (Tool Call):** If the LLM decides to use a tool, it executes the tool with the determined arguments and gets back an _observation_ (the tool's output).
  3.  **Observation:** The tool's output (observation) is fed back to the LLM as part of the context for the next reasoning step.
  4.  **Repeat:** This loop continues – reason, act (maybe), observe – until the LLM decides it has enough information to respond directly to the user _without_ calling any more tools.

- **Memory for ReAct Agents:** ReAct agents often incorporate memory (Section 2.3.1) to keep track of the conversation history and context across multiple turns. This helps them reason more effectively over time.

- **Planning with ReAct:** ReAct agents can perform complex, multi-step planning. They can iteratively call tools, reason about the outputs, and refine their plan as they go to achieve complex goals. It's like breaking down a big task into smaller steps and using tools at each step.

**Code Snippet (Creating a ReAct Agent with `create_react_agent`):**

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI # Or your LLM of choice
from langchain.tools import DuckDuckGoSearchRun # Example tool

llm = ChatOpenAI(model_name="gpt-4o-mini", temperature=0) # Your LLM
tools = [DuckDuckGoSearchRun()] # Example tool - DuckDuckGo search or Tavily Search

llm_with_tools = llm.bind_tools(tools) # Bind the tools to the LLM

react_agent = create_react_agent(llm_with_tools, tools=tools) # BAM! ReAct agent created using the pre-built function!

# Now you can use 'react_agent' as a node in your LangGraph graph.
```

#### 4.1.3. Multi-Agent Systems

Multi-agent systems are all about composing multiple agents to work together to solve complex problems. LangGraph's graph structure is _perfect_ for building and managing these kinds of applications because it lets you clearly define how agents interact and coordinate.

- **Agent Architectures:** There are various ways to structure multi-agent systems. Common architectures include:

  - **Network Architecture:** Agents can communicate freely with each other in a many-to-many fashion. Think of a decentralized team where everyone can talk to everyone.
  - **Supervisor Architecture:** One "supervisor" agent orchestrates and directs other agents (one-to-many). Think of a team lead assigning tasks to team members.
  - **Hierarchical Architecture:** Multiple levels of supervisors, managing teams of agents in a hierarchy. Like a corporate org chart, but with AI agents.
  - **Custom Workflow Architecture:** Agents communicate in a predefined sequence or a custom, more complex flow. This flow can be deterministic (always the same sequence) or dynamic (flow changes based on conditions).

- **Agents as Nodes:** In LangGraph, you typically represent each agent as a _node_ within a larger graph. Each agent node encapsulates the specific logic, capabilities, and (potentially) private state for that agent.

- **Handoffs:** Multi-agent systems often involve _handoffs_, where one agent completes its task and passes control and information to another agent. This can be implemented in LangGraph using `Command` objects (Section 2.2.4) to control routing between agent nodes and pass along State updates.

- **Communication Between Agents:** Agents can communicate in different ways:

  - **Shared Graph State:** Agents can read from and write to a _shared_ graph `State`. This is like a shared whiteboard where agents can exchange information by updating and reading values in the State.
  - **Tool Calls:** Agents can call _other agents_ as tools. This is a powerful way for agents to request services or information from each other. Agent A can call Agent B as a tool, passing information as tool arguments and getting results back as tool outputs.

- **State Schemas for Agents:** Agents can have different approaches to managing their "memory" or state:

  - **Shared State Schema:** All agents operate on a single, unified `State` schema. They all work with the same shared memory space.
  - **Different State Schemas:** Agents can have their _own private_ `State` schemas, tailored to their specific roles and tasks. They can still communicate with the parent graph or other agents through _overlapping State keys_ (if they share keys in their schemas) or by using State transformations when passing information.

- **Message History Sharing:** Agents can communicate through:
  - **Shared Message List:** Maintain a single, shared list of messages (conversation history) in the graph's State. All agents contribute to and read from this shared history.
  - **Private Message Histories:** Each agent can maintain its own private message history. They might only exchange final results or summaries with other agents, rather than sharing the entire conversation history.

**Code Snippet (Multi-Agent System - Supervisor Architecture Example):**

```python
from langgraph.graph import StateGraph, START, END

# Assuming you have defined node functions:
# supervisor_agent_node, agent_1_node, agent_2_node
# and a routing function: supervisor_routing_function
# and a MessagesState TypedDict

builder = StateGraph(MessagesState) # Or your chosen State

builder.add_node("supervisor_agent", supervisor_agent_node) # Node for the supervisor agent
builder.add_node("agent_1", agent_1_node) # Node for agent 1
builder.add_node("agent_2", agent_2_node) # Node for agent 2

# Define edges to connect the nodes and define the flow:
builder.add_edge(START, "supervisor_agent") # Start at the supervisor
builder.add_conditional_edges( # Supervisor agent routes to agent_1 or agent_2
    "supervisor_agent",
    supervisor_routing_function, # Routing function decides which agent to use
    {"agent_1": "agent_1", "agent_2": "agent_2"} # Mapping: route "agent_1" to node "agent_1", etc.
)
builder.add_edge("agent_1", "supervisor_agent") # Agent 1 hands back to the supervisor after its task
builder.add_edge("agent_2", "supervisor_agent") # Agent 2 hands back to the supervisor

# ... (add edges to END node if needed to terminate the graph)

# Now you have a basic supervisor-agent system in LangGraph!
```

### 4.2. Advanced Use Cases

LangGraph really shines when you start tackling more complex and nuanced use cases. Here are a few examples of where it can get _really_ interesting.

#### 4.2.1. Human-in-the-Loop Workflows

LangGraph is _excellent_ for building human-in-the-loop workflows. This means you can seamlessly integrate human intelligence and feedback into your automated AI processes, leading to more robust, trustworthy, and effective applications. Remember the `interrupt` feature (Section 2.3.8)? This is where it becomes super powerful.

- **Review Tool Calls:** Implement human review and approval of tool calls _before_ they are executed. This is crucial for safety and control, especially in sensitive operations (like making financial transactions, sending emails, or modifying data). Use `interrupt` _before_ the node that makes the tool call to pause and get human validation.

- **Edit Graph State:** Allow humans to directly modify the graph's `State` at breakpoints or interruptions. This lets humans correct errors, refine information, or provide guidance to the AI during execution. Use `graph.update_state()` (Section 2.3.9 - Time Travel) to inject human feedback into the State.

- **Multi-Turn Conversations:** Design conversational agents that can engage in multi-turn interactions with humans. The agent can ask questions, seek clarification, and adapt to user feedback over multiple exchanges. Use `interrupt` to pause for user input and `Command(resume=...)` to continue the conversation flow.

- **Validation of Human Input:** Implement input validation within human-in-the-loop nodes. Make sure that user-provided data meets specific criteria (e.g., format, content, validity) _before_ resuming graph execution. You can use loops with `interrupt` and validation logic within a node to ensure you get good quality human input.

- **Approve or Reject Actions:** Create workflows where humans can explicitly approve or reject actions proposed by the agent before they are carried out. Use `interrupt` to present action details to the user and then route the graph execution based on their approval or rejection decision (using `Command(goto=...)`).

#### 4.2.2. Map-Reduce in LangGraph

LangGraph's `Send` API (Section 2.2.3) makes it surprisingly straightforward to implement map-reduce patterns. This is incredibly useful for parallel processing and aggregation of results, especially when dealing with large datasets or tasks that can be broken down into independent chunks.

- **Parallel Processing (Map Step):** Use `Send` to dynamically create and execute multiple parallel tasks. Each `Send` object represents a separate branch of execution for a sub-task. It's like launching multiple workers to handle different parts of a problem at the same time.

- **Aggregation (Reduce Step):** After all the parallel tasks complete, use a dedicated "reduce" node to gather and combine the results from all the parallel branches. Reducers (Section 2.1.3) are essential here for correctly combining `State` updates from parallel nodes, especially if you're working with lists, collections, or other complex data structures in your State.

- **Use Cases for Map-Reduce:**
  - **Document Processing:** Parallelize the processing of multiple documents (e.g., summarization, information extraction, analysis). Process each document in parallel and then combine the summaries or extracted info.
  - **Data Analysis:** Distribute data analysis tasks across multiple nodes for faster processing of large datasets.
  - **Multi-Agent Collaboration (Parallel Tasks):** Run multiple agents in parallel to work on different aspects of a larger problem and then combine their outputs to achieve a common goal.

**Code Snippet (Conceptual Structure of a Map-Reduce Workflow in LangGraph):**

```python
from langgraph.graph import StateGraph, START, END

# Assuming you have defined node functions:
# map_node_function, process_item_function, reduce_node_function
# and a MapReduceState TypedDict

builder = StateGraph(MapReduceState) # Or your chosen State

builder.add_node("map_node", map_node_function) # Node to initiate parallel tasks (using Send)
builder.add_node("process_item_node", process_item_function) # Node for each parallel processing task
builder.add_node("reduce_node", reduce_node_function) # Node to aggregate results from parallel tasks

# Define the edges to create the map-reduce flow:
builder.add_edge(START, "map_node") # Start at the map node
builder.add_conditional_edges( # Dynamic edges from map_node to multiple 'process_item_node' instances
    "map_node",
    map_routing_function, # Routing function in map_node will return a list of Send objects
    ["process_item_node"] # Destination node for each Send object (could be a list of destinations if needed)
)
builder.add_edge("process_item_node", "reduce_node") # Edges from each parallel task to the reduce node (fan-in)
builder.add_edge("reduce_node", END) # Reduce node leads to the end

# This structure sets up a basic map-reduce pattern in LangGraph.
```

#### 4.2.3. Self-Correcting Agents (Reflection)

Reflection mechanisms allow agents to evaluate their own performance, identify errors, and iteratively improve their behavior. LangGraph provides the flexibility to implement reflection in various ways, creating agents that can learn and adapt over time.

- **LLM-Based Reflection:** Use LLMs themselves to evaluate agent outputs, provide feedback, and suggest improvements to prompts or strategies. Create dedicated "evaluator" nodes that use LLMs to assess the quality of agent-generated content or actions. It's like having an AI critic for your AI agent.

- **Deterministic Reflection:** Implement deterministic reflection mechanisms based on predefined rules or metrics. For example:

  - Code compilation errors: If an agent generates code that doesn't compile, that's a clear error signal.
  - Validation failures: If agent output fails to meet certain validation criteria (format, content, etc.), that's also a deterministic error.
    Use conditional edges or `Command` objects to route execution back to "correction" nodes based on these reflection results.

- **Iterative Improvement Loop:** Create feedback loops within your graph where reflection results are used to update agent prompts, strategies, or even the `State` itself. This leads to iterative performance improvements over multiple executions or interactions. The agent learns from its "mistakes" and gets better over time.

- **Prompt Rewriting:** Agents can even modify their _own prompts_ based on reflection feedback. Use reflection nodes to analyze agent outputs and generate improved prompts that are then used in subsequent LLM calls. This is a form of meta-learning, where the agent learns how to improve its prompting strategy.

**Code Snippet (Evaluator-Optimizer Pattern for Self-Correction):**

```python
from langgraph.graph import StateGraph, START, END

# Assuming you have defined node functions:
# generator_function, evaluator_function, optimizer_function
# and an OptimizationState TypedDict

builder = StateGraph(OptimizationState) # Or your chosen State

builder.add_node("generator_node", generator_function) # Node to generate some output (e.g., text, code)
builder.add_node("evaluator_node", evaluator_function) # Node to evaluate the output (using LLM or rules)
builder.add_node("optimizer_node", optimizer_function) # Node to optimize the generation process based on feedback

# Define the feedback loop:
builder.add_edge(START, "generator_node") # Start with generation
builder.add_edge("generator_node", "evaluator_node") # Output goes to the evaluator
builder.add_conditional_edges( # Evaluator decides: optimize or end?
    "evaluator_node",
    evaluator_routing_function, # Routing function based on evaluation result
    {"Needs Optimization": "optimizer_node", "Accepted": END} # Route to optimizer if needed, or end if accepted
)
builder.add_edge("optimizer_node", "generator_node") # Optimizer feeds back into generator for the next iteration

# This sets up a basic evaluate-optimize loop for self-correction.
```

### 4.3. Rare Use Cases (Just a Quick Mention)

These are use cases that are less common but worth being aware of, especially if you're pushing the limits of LangGraph in very specific scenarios.

- **Complex Graph Migrations:** LangGraph supports graph migrations when you change your graph structure or State schema. However, if you have _extremely_ complex schema changes or topology changes in _production systems_ with persistent `State`, it might require very careful planning and potentially custom migration strategies. In _most_ cases, LangGraph handles migrations smoothly, but for edge cases, be aware of potential complexity.

- **Highly Specialized Persistence Setups:** For applications with _extreme_ persistence requirements (e.g., very high transaction throughput, specific database needs beyond PostgreSQL/SQLite, custom data serialization needs), you might need to implement custom checkpointer or store implementations. LangGraph's base interfaces are designed to be extensible if you need to go beyond the built-in options for persistence.
