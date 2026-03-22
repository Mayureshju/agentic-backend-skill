# Agentic RAG Reference

## Table of Contents
1. [Agentic RAG Architecture](#agentic-rag-architecture)
2. [LangGraph Implementation](#langgraph-implementation)
3. [CrewAI Implementation](#crewai-implementation)
4. [LlamaIndex Agents Implementation](#llamaindex-agents-implementation)
5. [OpenAI Agents SDK Implementation](#openai-agents-sdk-implementation)
6. [Tool Registry](#tool-registry)
7. [State Management](#state-management)
8. [Agent API Endpoints](#agent-api-endpoints)

---

## Agentic RAG Architecture

```
                    ┌──────────────────────────────────┐
                    │         AGENT ORCHESTRATOR        │
                    │  (LangGraph / CrewAI / LlamaIndex)│
                    └──────────┬───────────────────────┘
                               │
                    ┌──────────▼───────────────────────┐
                    │         DECISION ENGINE           │
                    │  • Should I retrieve?             │
                    │  • What should I search for?      │
                    │  • Do I need more context?        │
                    │  • Should I use a tool?           │
                    └──────────┬───────────────────────┘
                               │
              ┌────────────────┼────────────────────┐
              ▼                ▼                     ▼
    ┌─────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │  RAG Tool   │  │  Web Search     │  │  Custom Tools   │
    │  (retrieve  │  │  Tool           │  │  (code exec,    │
    │   chunks)   │  │                 │  │   API calls)    │
    └──────┬──────┘  └────────┬────────┘  └────────┬────────┘
           │                  │                     │
           └──────────────────┼─────────────────────┘
                              ▼
                    ┌─────────────────────────────────┐
                    │       RESPONSE SYNTHESIZER      │
                    │  Combine results → Generate     │
                    │  → Stream → Log traces          │
                    └─────────────────────────────────┘
```

### Module Structure

```
app/
├── agents/
│   ├── __init__.py
│   ├── orchestrator.py       # Main agent execution loop
│   ├── state.py              # Agent state definition
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── registry.py       # Tool registry + dispatcher
│   │   ├── rag_tool.py       # Document retrieval tool
│   │   ├── web_search.py     # Web search tool
│   │   ├── calculator.py     # Math/calculation tool
│   │   └── code_exec.py      # Code execution (sandboxed)
│   │
│   ├── prompts/
│   │   ├── __init__.py
│   │   └── agent_prompts.py  # System prompts for agents
│   │
│   └── memory/
│       ├── __init__.py
│       └── conversation.py   # Conversation history + memory
│
├── pipeline/                  # Standard RAG pipeline (shared)
│   ├── ingestion/
│   ├── retrieval/
│   └── generation/
```

---

## LangGraph Implementation

LangGraph is the recommended choice for production agentic RAG. It models agents as state machines with explicit control flow.

### Dependencies
```
# Python
langgraph>=0.3.0
langchain>=0.3.15
langchain-openai>=0.3.0       # or langchain-anthropic
langchain-community>=0.3.15
langgraph-checkpoint-postgres>=2.0.0  # For persistent state (PostgreSQL)

# JavaScript
@langchain/langgraph: ^0.2.0
@langchain/core: ^0.3.0
@langchain/openai: ^0.4.0     # or @langchain/anthropic
```

### Python — LangGraph Agent

```python
from typing import TypedDict, Annotated, Sequence
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI
import operator

# 1. Define State
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    retrieved_docs: list[dict]
    current_query: str
    iteration: int
    max_iterations: int

# 2. Define Nodes
async def should_retrieve(state: AgentState) -> dict:
    """LLM decides whether to retrieve documents or respond directly."""
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    messages = state["messages"]
    response = await llm.ainvoke(messages)
    return {"messages": [response]}

async def retrieve_documents(state: AgentState) -> dict:
    """Execute RAG retrieval using the search query."""
    # Uses the RAG pipeline from pipeline/retrieval/
    query = state["current_query"]
    # ... retrieval logic ...
    return {"retrieved_docs": results, "messages": [AIMessage(content=f"Found {len(results)} relevant documents.")]}

async def generate_response(state: AgentState) -> dict:
    """Generate final response using retrieved context."""
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    context = "\n".join([doc["content"] for doc in state["retrieved_docs"]])
    messages = state["messages"] + [HumanMessage(content=f"Context:\n{context}\n\nAnswer based on this context.")]
    response = await llm.ainvoke(messages)
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    """Router: decide next step based on current state."""
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    if state["iteration"] >= state["max_iterations"]:
        return "generate"
    return "generate"

# 3. Build Graph
def build_agent_graph(tools):
    workflow = StateGraph(AgentState)

    workflow.add_node("agent", should_retrieve)
    workflow.add_node("tools", ToolNode(tools))
    workflow.add_node("retrieve", retrieve_documents)
    workflow.add_node("generate", generate_response)

    workflow.set_entry_point("agent")

    workflow.add_conditional_edges("agent", should_continue, {
        "tools": "tools",
        "generate": "generate",
    })
    workflow.add_edge("tools", "agent")
    workflow.add_edge("retrieve", "generate")
    workflow.add_edge("generate", END)

    return workflow.compile()

# 4. Usage
async def run_agent(query: str, tools: list):
    graph = build_agent_graph(tools)
    result = await graph.ainvoke({
        "messages": [HumanMessage(content=query)],
        "retrieved_docs": [],
        "current_query": query,
        "iteration": 0,
        "max_iterations": 5,
    })
    return result["messages"][-1].content
```

### TypeScript — LangGraph Agent

```typescript
import { StateGraph, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { HumanMessage, AIMessage, BaseMessage } from "@langchain/core/messages";
import { Annotation } from "@langchain/langgraph";

const AgentState = Annotation.Root({
  messages: Annotation<BaseMessage[]>({ reducer: (a, b) => [...a, ...b] }),
  retrievedDocs: Annotation<Record<string, unknown>[]>(),
  currentQuery: Annotation<string>(),
  iteration: Annotation<number>(),
});

type AgentStateType = typeof AgentState.State;

async function agentNode(state: AgentStateType) {
  const llm = new ChatOpenAI({ model: "gpt-4o", temperature: 0 });
  const response = await llm.invoke(state.messages);
  return { messages: [response] };
}

async function generateNode(state: AgentStateType) {
  const llm = new ChatOpenAI({ model: "gpt-4o", temperature: 0 });
  const context = state.retrievedDocs.map((d) => d.content).join("\n");
  const messages = [
    ...state.messages,
    new HumanMessage(`Context:\n${context}\n\nAnswer based on this context.`),
  ];
  const response = await llm.invoke(messages);
  return { messages: [response] };
}

function buildAgentGraph() {
  const workflow = new StateGraph(AgentState)
    .addNode("agent", agentNode)
    .addNode("generate", generateNode)
    .addEdge("__start__", "agent")
    .addEdge("agent", "generate")
    .addEdge("generate", END);

  return workflow.compile();
}
```

### LangGraph Persistent State (PostgreSQL)

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async def create_persistent_agent(db_url: str):
    checkpointer = AsyncPostgresSaver.from_conn_string(db_url)
    await checkpointer.setup()

    graph = build_agent_graph(tools)
    return graph.compile(checkpointer=checkpointer)

# Each conversation gets a thread_id for state persistence
result = await agent.ainvoke(
    {"messages": [HumanMessage(content="...")]},
    config={"configurable": {"thread_id": "conversation-123"}},
)
```

---

## CrewAI Implementation

CrewAI uses role-based agents that collaborate on tasks. **Python only**.

### Dependencies
```
crewai>=0.100.0
crewai-tools>=0.17.0
```

### Python — CrewAI Agentic RAG

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import tool

# 1. Define Tools
@tool("Search Knowledge Base")
def search_knowledge_base(query: str) -> str:
    """Search the internal knowledge base for relevant documents."""
    # Uses the RAG pipeline retrieval
    results = retriever.retrieve_sync(query)
    return "\n\n".join([f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results)])

@tool("Web Search")
def web_search(query: str) -> str:
    """Search the web for current information."""
    # Uses a web search API
    results = web_search_client.search(query)
    return "\n\n".join([f"{r['title']}: {r['snippet']}" for r in results])

# 2. Define Agents
researcher = Agent(
    role="Research Analyst",
    goal="Find accurate and relevant information to answer the user's question",
    backstory="You are a thorough researcher who always verifies information from multiple sources.",
    tools=[search_knowledge_base, web_search],
    verbose=True,
    allow_delegation=True,
)

writer = Agent(
    role="Response Writer",
    goal="Synthesize research findings into a clear, well-structured answer with citations",
    backstory="You are an expert communicator who presents complex information clearly.",
    tools=[],
    verbose=True,
)

# 3. Define Tasks
def create_research_task(query: str) -> Task:
    return Task(
        description=f"Research the following question thoroughly: {query}",
        expected_output="A comprehensive summary of findings with source references",
        agent=researcher,
    )

def create_writing_task(query: str) -> Task:
    return Task(
        description=f"Write a clear answer to: {query}\nUse the research findings and cite sources.",
        expected_output="A well-structured answer with [Source N] citations",
        agent=writer,
    )

# 4. Create Crew
def run_agentic_rag(query: str) -> str:
    crew = Crew(
        agents=[researcher, writer],
        tasks=[create_research_task(query), create_writing_task(query)],
        process=Process.sequential,
        verbose=True,
    )
    result = crew.kickoff()
    return result.raw
```

---

## LlamaIndex Agents Implementation

### Dependencies
```
# Python
llama-index>=0.12.0
llama-index-agent-openai>=0.4.0
llama-index-tools-tavily-research>=0.3.0  # For web search

# JavaScript
llamaindex: ^0.8.0
```

### Python — LlamaIndex ReAct Agent

```python
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import QueryEngineTool, FunctionTool
from llama_index.core import VectorStoreIndex, Settings
from llama_index.llms.openai import OpenAI

# 1. Create RAG query engine tool
index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine(similarity_top_k=5)

rag_tool = QueryEngineTool.from_defaults(
    query_engine=query_engine,
    name="knowledge_base",
    description="Search the knowledge base for answers to questions about our documents.",
)

# 2. Create custom tools
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        result = eval(expression)  # In production, use a safe math parser
        return str(result)
    except Exception as e:
        return f"Error: {e}"

calc_tool = FunctionTool.from_defaults(fn=calculate, name="calculator", description="Calculate math expressions")

# 3. Build Agent
llm = OpenAI(model="gpt-4o", temperature=0)
agent = ReActAgent.from_tools(
    tools=[rag_tool, calc_tool],
    llm=llm,
    verbose=True,
    max_iterations=10,
)

# 4. Run
response = await agent.achat("How much did revenue grow last quarter?")
```

---

## OpenAI Agents SDK Implementation

### Dependencies
```
# Python
openai-agents>=0.1.0  # or openai>=1.60.0 with agents API
```

### Python — OpenAI Agents

```python
from agents import Agent, Runner, function_tool

@function_tool
def search_knowledge_base(query: str) -> str:
    """Search the internal knowledge base for relevant documents."""
    results = retriever.retrieve_sync(query)
    return "\n\n".join([f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results)])

@function_tool
def web_search(query: str) -> str:
    """Search the web for current information."""
    results = web_search_client.search(query)
    return "\n\n".join([f"{r['title']}: {r['snippet']}" for r in results])

# Define agent
research_agent = Agent(
    name="Research Assistant",
    instructions=(
        "You are a helpful research assistant. Use the knowledge base tool to find relevant "
        "information. If the knowledge base doesn't have enough information, use web search. "
        "Always cite your sources."
    ),
    tools=[search_knowledge_base, web_search],
    model="gpt-4o",
)

# Run
async def run_agent(query: str) -> str:
    result = await Runner.run(research_agent, query)
    return result.final_output
```

---

## Tool Registry

A centralized tool registry for agents to discover and use tools:

```python
from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class ToolDefinition:
    name: str
    description: str
    function: Callable
    parameters: dict  # JSON Schema for parameters
    requires_auth: bool = False
    rate_limit: int | None = None  # calls per minute

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}

    def register(self, tool: ToolDefinition):
        self._tools[tool.name] = tool

    def get(self, name: str) -> ToolDefinition | None:
        return self._tools.get(name)

    def list_tools(self, role: str | None = None) -> list[ToolDefinition]:
        tools = list(self._tools.values())
        if role:
            # Filter tools based on user role/permissions
            tools = [t for t in tools if not t.requires_auth or role == "ADMIN"]
        return tools

    async def execute(self, name: str, **kwargs) -> Any:
        tool = self._tools.get(name)
        if not tool:
            raise ValueError(f"Tool not found: {name}")
        result = tool.function(**kwargs)
        if hasattr(result, "__await__"):
            return await result
        return result
```

---

## State Management

### Conversation Memory

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Message:
    role: str  # "user" | "assistant" | "system" | "tool"
    content: str
    timestamp: datetime = field(default_factory=datetime.utcnow)
    metadata: dict = field(default_factory=dict)  # tool results, sources, etc.

class ConversationMemory:
    def __init__(self, max_messages: int = 50, max_tokens: int = 8000):
        self.max_messages = max_messages
        self.max_tokens = max_tokens
        self.messages: list[Message] = []

    def add(self, message: Message):
        self.messages.append(message)
        self._trim()

    def get_context(self) -> list[dict]:
        """Get messages formatted for LLM context."""
        return [{"role": m.role, "content": m.content} for m in self.messages]

    def _trim(self):
        """Keep within token budget by removing oldest messages (keep system prompt)."""
        while len(self.messages) > self.max_messages:
            # Remove oldest non-system message
            for i, m in enumerate(self.messages):
                if m.role != "system":
                    self.messages.pop(i)
                    break
```

---

## Agent API Endpoints

```
# Agent Conversations
POST   /api/v1/agent/chat                    # Send message to agent (triggers agent loop)
POST   /api/v1/agent/chat/stream             # Streaming agent response (SSE)

# Conversation Management
POST   /api/v1/conversations                 # Create new conversation
GET    /api/v1/conversations                 # List conversations
GET    /api/v1/conversations/:id             # Get conversation with history
DELETE /api/v1/conversations/:id             # Delete conversation

# Agent Tools
GET    /api/v1/agent/tools                   # List available tools
POST   /api/v1/agent/tools/:name/execute     # Execute a tool directly (admin only)

# Agent State (for debugging)
GET    /api/v1/agent/state/:thread_id        # Get agent state for a conversation (admin only)
```
