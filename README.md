# LangGraph Project

Building stateful, multi-actor AI agents using LangGraph - a framework for creating complex agent workflows as graphs.

## What is LangGraph?

LangGraph extends LangChain to enable:
- **Stateful workflows** - Persist state across agent steps
- **Cyclic graphs** - Create loops for iterative reasoning (ReAct, reflection)
- **Multi-agent systems** - Coordinate multiple agents with different roles
- **Human-in-the-loop** - Pause execution for human approval
- **Streaming** - Stream both tokens and intermediate steps

## Project Structure

```
LangGraph-Project/
├── main.py              # Entry point
├── pyproject.toml       # Dependencies
├── .env                 # API keys (TAVILY_API_KEY, etc.)
└── .python-version      # Python 3.14
```

## Setup

### 1. Install dependencies

```bash
uv sync
```

### 2. Configure environment

Create a `.env` file:

```env
TAVILY_API_KEY=your_tavily_api_key
LANGCHAIN_API_KEY=your_langsmith_api_key
LANGCHAIN_TRACING_V2=true
```

### 3. Install Ollama models

```bash
ollama pull llama3.2
```

### 4. Run

```bash
uv run python main.py
```

## Dependencies

From `pyproject.toml`:

| Package | Purpose |
|---------|---------|
| `langgraph>=1.1.3` | Graph-based agent orchestration |
| `langchain>=1.2.13` | LLM framework |
| `langchain-ollama>=1.0.1` | Ollama integration |
| `langchain-tavily>=0.2.17` | Web search tool |
| `python-dotenv>=1.2.2` | Environment management |

## LangGraph Core Concepts

### 1. StateGraph

Define a typed state that flows through your graph:

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph = StateGraph(State)
```

### 2. Nodes

Functions that transform state:

```python
def chatbot(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

graph.add_node("chatbot", chatbot)
```

### 3. Edges

Define transitions between nodes:

```python
# Simple edge
graph.add_edge(START, "chatbot")
graph.add_edge("chatbot", END)

# Conditional edge
def should_continue(state: State):
    if state["messages"][-1].tool_calls:
        return "tools"
    return END

graph.add_conditional_edges("chatbot", should_continue)
```

### 4. Compile and Run

```python
app = graph.compile()
result = app.invoke({"messages": [HumanMessage(content="Hello!")]})
```

## Common Patterns

### ReAct Agent (Reasoning + Acting)

```python
from langgraph.prebuilt import create_react_agent

tools = [TavilySearch()]
agent = create_react_agent(model=llm, tools=tools)
```

### Agent with Tools

```python
from langchain.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return tavily.search(query)

# Bind tools to LLM
llm_with_tools = llm.bind_tools([search])
```

### Human-in-the-Loop

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory, interrupt_before=["sensitive_action"])

# Resume after human approval
app.invoke(None, config={"configurable": {"thread_id": "1"}})
```

### Multi-Agent System

```python
# Supervisor routes to specialized agents
def supervisor(state):
    # Decide which agent should handle the task
    return {"next": "researcher" | "coder" | "END"}

graph.add_conditional_edges("supervisor", lambda s: s["next"])
```

## Example: Search Agent

```python
from dotenv import load_dotenv
from langchain_ollama import ChatOllama
from langchain_tavily import TavilySearch
from langgraph.prebuilt import create_react_agent

load_dotenv()

llm = ChatOllama(model="llama3.2")
tools = [TavilySearch()]

agent = create_react_agent(model=llm, tools=tools)

result = agent.invoke({
    "messages": [("user", "What's the latest news about AI?")]
})

print(result["messages"][-1].content)
```

## Visualization

Generate a graph diagram:

```python
from IPython.display import Image

Image(app.get_graph().draw_mermaid_png())
```

Or get Mermaid syntax:

```python
print(app.get_graph().draw_mermaid())
```

## LangGraph vs LangChain

| Feature | LangChain | LangGraph |
|---------|-----------|-----------|
| Flow | Linear chains | Cyclic graphs |
| State | Passed through chain | Persistent, typed state |
| Loops | Not native | First-class support |
| Agents | `create_react_agent` | Full graph control |
| Use case | Simple pipelines | Complex multi-step agents |

## Resources

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [LangGraph Tutorials](https://langchain-ai.github.io/langgraph/tutorials/)
- [LangGraph Examples](https://github.com/langchain-ai/langgraph/tree/main/examples)

## Next Steps

1. **Simple chatbot** - Basic graph with single node
2. **Tool-using agent** - Add Tavily search
3. **ReAct agent** - Reasoning + acting loop
4. **Reflection agent** - Self-critique and improve
5. **Multi-agent** - Supervisor + worker pattern
