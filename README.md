# Multi-Agent Communication Router

**Name:** Sultan Saad Abu Thnain

## Project Write-Up

**Agent Fundamentals:** The supervisor agent triggers deterministic Python functions wrapped in `@task` decorators to execute real tool calls (email drafting and calendar scheduling), ensuring outputs are structured and predictable rather than hardcoded text.

**Multi-agent / Routing Architecture:** Track A was implemented utilizing an Orchestrator-Worker pattern. The `@entrypoint` supervisor analyzes the natural language query and routes execution to either the Calendar sub-workflow or the Email/RAG sub-workflow, matching the course's recommended domain routing.

**RAG Pipeline:** I implemented an Agentic RAG architecture. This choice empowers the agent to conditionally query the FAISS vector store only when HR policy context is required, rather than forcing a standard 2-step retrieval on every prompt. This saves compute and reduces hallucinations when handling non-knowledge-based queries (like scheduling).

**Context & State Management:** A persistent thread is maintained via the LangGraph `MemorySaver` checkpointer. The explicit `thread_id` acts as long-term memory to bridge the gap during the human-in-the-loop pause, while the graph's internal variables manage the short-term execution state.

**Human-in-the-Loop & Error Handling:** The system relies on the LangGraph Functional API. Error handling includes a transient retry policy on the RAG retrieval task in case of timeouts, and a user-fixable interrupt using `langgraph.types.interrupt` that securely pauses execution to demand explicit human approval via terminal input before dispatching sensitive emails.

**Workflow Pattern:** The system explicitly implements the Orchestrator-Worker pattern, where the main entrypoint acts as the orchestrator delegating specific domains to the specialized tool nodes.

**LangSmith Observability:** Tracing was enabled globally via environment variables. The trace revealed that the orchestrator accurately identified the "vacation" keyword, routed to the RAG tool, and successfully suspended the thread execution precisely at the email node, waiting idly without consuming compute resources until the human approval payload was injected.
