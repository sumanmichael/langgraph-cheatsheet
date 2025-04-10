---
title: I. Getting Started
permalink: /cheatsheet/getting-started/
layout: post
---

Alright, ready to dive in? Let's get LangGraph set up on your system. You'll need to install the core library and any extras you need, like integrations for specific LLM providers. Good news: LangGraph speaks both Python and JavaScript (Node.js)!

### 1.1. Installation

Pick your potion – Python or JavaScript! And, I picked python poison.

**Python:**

```bash
pip install langgraph langchain_openai  # For OpenAI models (super common)
pip install langgraph langchain_anthropic # If you're using Anthropic
pip install langgraph-checkpoint-sqlite # For saving progress locally (SQLite option)
pip install langgraph-checkpoint-postgres # For serious, production-level saving (Postgres)
# ... and install any other integrations you need (check out langchain-community) ...
```

**Dependencies - The Fine Print:**

Heads up, depending on what you're building, you might need to install more LangChain integrations (like `langchain-openai`, `langchain-anthropic`, `langchain-community`), some checkpointers (`langgraph-checkpoint-sqlite`, `langgraph-checkpoint-postgres` are good options), and any other libraries your nodes and edges rely on. Basically, install what you need as you go!

**Compatibility Check:**

LangGraph plays nicely with Python 3.8+ (but really, aim for 3.11 or 3.12 – they're faster and have more goodies)

### 1.2. Setup & Environment Configuration

Okay, software installed! Now, let's get your workspace prepped. This means setting up API keys, environment variables, and getting your project folders organized.

**API Keys - Keep 'em Secret, Keep 'em Safe:**

LangGraph apps often talk to LLMs and other services that need API keys. **Never, ever hardcode these directly into your code!** Instead, use environment variables. It's way more secure and makes things easier to manage. Common ones you'll probably need:

- `OPENAI_API_KEY` (for, you guessed it, OpenAI models)
- `ANTHROPIC_API_KEY` (for Anthropic models)
- `LANGCHAIN_API_KEY` (for LangSmith tracing and cool LangGraph Platform features)
- `TAVILY_API_KEY` (if you're using the Tavily search tool)

You can set these directly in your terminal, but for easier project management, create a `.env` file in your project's root directory. Libraries like `python-dotenv` (Python) can help load these into your environment.

**Project Structure - Let's Get Organized:**

A typical LangGraph project folder might look something like this:

```
my-langgraph-app/
├── my_graph/                # Where your graph code lives
│   ├── nodes.py           # Your node functions
│   ├── edges.py           # Edge/routing logic
│   ├── state.py           # State definitions (TypedDict, Pydantic)
│   ├── tools.py           # Custom tools (optional)
│   └── __init__.py
├── langgraph.json         # For LangGraph Platform deployments (config file)
├── requirements.txt       # Python dependencies (or package.json for JS)
├── .env                   # Where you store your API keys (environment variables)
└── main.py               # Entry point to run your app (optional)
```

**`langchain.json` - Deployment Time (Later):**

The `langchain.json` file in your project root is key if you plan to deploy to the LangGraph Platform. It tells the platform about your project's dependencies, where your graph starts, and environment variables. Don't worry too much about this right now, but here's a peek:

```
{
  "dependencies": [
    "langchain_openai",
    "./my_graph"         # Path to your local graph package
  ],
  "graphs": {
    "my_agent": "./my_graph/agent.py:my_graph_instance" # Path to your compiled graph instance
  },
  "env": "./.env"         # Path to your .env file
}
```

**Deployment Environments - Local vs. Cloud:**

- **Local Dev:** `.env` files are your friend for local development. Just make sure those API keys are set before you run anything!
- **Cloud Deployments:** When you go live (LangGraph Platform or your own cloud setup), you'll usually configure environment variables directly in the cloud environment (like in LangSmith Deployment settings, Docker Compose, or your cloud provider's control panel). Again, avoid hardcoding secrets in your code!
