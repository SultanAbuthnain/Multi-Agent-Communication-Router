# Write-Up: Multi-Agent Communication Router

This notebook implements a supervisor-style LangGraph agent that routes user requests between an HR-email workflow and a calendar-scheduling workflow, backed by a Gemini LLM, a FAISS-based RAG pipeline, short-term (thread) memory, and long-term (cross-thread) memory.

## LLM-Driven Drafting and Routing

Intent classification is not done with keyword matching or if/else string checks. Inside `supervisor_agent`, the user's message is passed to `llm.invoke()` with a system prompt instructing Gemini to act as a strict classifier and return exactly one label — `EMAIL_HR`, `CALENDAR`, or `UNKNOWN`. The agent's branching logic depends entirely on this LLM output.

Similarly, email content is not templated: `draft_email` sends the retrieved policy text to `llm.invoke()` with a prompt asking Gemini to compose a professional HR email, and the model's generated text is used as the draft.

## Retrieval-Augmented Context

Before drafting, `retrieve_knowledge` queries a FAISS vector store (built from three mock HR/facilities documents, embedded with `GoogleGenerativeAIEmbeddings`) to pull the most relevant policy snippet, which is then fed into the drafting prompt as grounding context.

## Long-Term Memory

An `InMemoryStore` is attached to the `entrypoint` and is separate from the `MemorySaver` checkpointer that handles thread-level state. Inside the agent, a namespace `("user_profile", user_id)` is used to look up a persisted user profile (department, status); if none exists, one is written with `store.put()`. Because this is keyed by `user_id` rather than by `thread_id`, the profile would persist and be retrievable across different conversation threads for the same user, not just within a single session.

## Retry Handling

The RAG retrieval task is wrapped with a genuine `RetryPolicy` object (`RetryPolicy(max_attempts=3, initial_interval=1.0)`) imported from `langgraph.types` and passed to `@task(retry_policy=rag_retry)`, so a transient `ValueError` raised on an empty retrieval result triggers automatic re-attempts by LangGraph rather than by custom retry code.

## Human-in-the-Loop Approval

`send_email_with_approval` calls `interrupt()` to pause graph execution and wait for explicit human approval before the email is considered "sent." The driver cell demonstrates this end-to-end: it streams the graph until the interrupt fires, collects a yes/no input from the user, and resumes execution with a `Command(resume=...)` carrying the approval decision.
