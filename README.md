# Building production agentic AI systems

Notes from shipping an agentic AI platform to a small internal user base — covering how I integrate vendor agent loops, author custom skills, stream tool-call output, and design RAG pipelines that compound knowledge over time.

I'm a software engineer working on agent systems. This page is the public companion to a private codebase — the architectural patterns and the lessons, without the parts that aren't mine to share.

---

## What I built

A two-panel chat product where users converse with an LLM agent that can:

- Generate live artifacts (PPTX, DOCX, XLSX, code) that render alongside the chat
- Search the web, fetch URLs, ingest documents into a personal knowledge wiki
- Stream every token and tool-call to the UI in real time
- Save chat outputs back into the wiki, where the agent indexes them and re-reads them on later turns — knowledge compounds across sessions

The agent loop itself is a fork of [MiniMax Mini-Agent](https://github.com/MiniMax-AI/MiniMax-Agent). The wrapper around it — backend streaming layer, frontend, model proxy, RAG and wiki pipeline, auth — is what I designed and shipped.

![Architecture](./architecture.png)

> Source: [`architecture.excalidraw`](./architecture.excalidraw) — drop into [excalidraw.com](https://excalidraw.com) to edit.

---

## 1 · Integrating an agent harness

Vendor agent loops (Mini-Agent, OpenAI Agents SDK, Anthropic's Claude Agent SDK, etc.) ship with their own runtime — but they expect to be driven from a CLI or notebook. To make one feel native inside a chat product, I had to build:

- **Streaming bridge.** The harness emits raw token + tool-call events; the frontend needs SSE frames it can render incrementally. I wrote a streaming adapter that turns the harness event bus into a typed SSE stream the React layer consumes via `assistant-ui`.
- **Session lifecycle.** Mini-Agent thinks in single sessions; users think in threads they leave and come back to. I built a session manager that persists each thread's state (messages, tool history, artifacts) and rehydrates on resume.
- **Tool-call rendering.** Every tool invocation needs a UI: a search shows the queried URL, an image-gen shows the inflight prompt, a long bash command collapses into a header you can expand. I designed a generic `ToolCallUI` component that registers per-tool renderers.
- **Artifact channel.** Long-form outputs (a 12-slide deck, a 600-line report, a generated chart) shouldn't fit in chat bubbles. The agent emits them to a separate artifact bus, which renders in a side panel with preview and download.

Takeaway: integrating a harness is 20% wiring the loop and 80% making its outputs feel native in your product surface.

---

## 2 · Authoring agent skills

A "skill" is a `SKILL.md` file plus reference docs and helper scripts, packaged as a directory the agent can load on demand. The skill tells the agent *when* to invoke it, *what tools* it exposes, and *how* to chain them.

I authored skills for deck/doc/spreadsheet generation, document parsing, image generation, and deployment automation. The non-obvious lessons:

- **Models copy patterns from templates regardless of how many anti-pattern rules you write.** If your `SKILL.md` shows a fenced code block with a `tool_call(...)` example, the model will reproduce that fenced code block as text in its output instead of invoking the tool. Fix: write tool invocations as imperative markdown, never as code samples.
- **Skill loading must be lazy.** Loading every skill at session start blows up the context window and confuses the model. The harness should load a skill only when the model asks for it (or when a router matches it).
- **Skills should fail loudly.** A skill that silently no-ops is the worst-case bug — you can't tell from the chat whether the model didn't try, the skill didn't run, or the output was dropped. Every skill I shipped raises a clear error string the agent surfaces to the user.
- **Rules files beat narrative.** A folder of one-rule-per-file behavioural notes (`.claude/rules/*.md`) outperformed long narrative `SKILL.md` files for steering model behavior. Models scan rule files; they skim narrative.

---

## 3 · LLM proxy: OpenAI-compatible → SageMaker

The agent harness expects an OpenAI-compatible API. The models we run live on SageMaker endpoints, which use a different request shape and auth model. I wrote a thin FastAPI proxy that:

- Translates OpenAI `chat/completions` and `embeddings` shapes to SageMaker `invoke_endpoint` calls
- Handles streaming (`text/event-stream` ↔ SageMaker streaming responses)
- Routes between multiple model endpoints by model ID — a single chat client can target the primary chat model, an alternate provider, an embedding model, or a reranker
- Normalizes errors so every consumer sees consistent response codes

Roughly 200 lines of code; saves us from maintaining a custom client in every consumer.

---

## 4 · Two-phase RAG with compounding knowledge

The naive RAG flow — chunk, embed, store, retrieve — has a latency problem when you ingest documents interactively. Upload a 200-page PDF and you wait three minutes before you can ask anything.

The pattern I shipped splits ingestion into two phases:

- **Phase 1 — fast, blocking.** Parse → chunk → embed → write to vector store. The user can now query the document. Target: ≤30s for typical inputs.
- **Phase 2 — slow, background.** A separate agent re-reads the document, extracts entities and concepts, builds wiki pages with cross-links and backlinks, classifies entities into a knowledge graph. Runs invisibly while the user keeps working.

The result is a wiki the agent grows over time. When the user asks a question, retrieval pulls back chunks (Phase 1 product) **and** the wiki entries that reference those chunks (Phase 2 product). The split also lets us toggle Phase 2 per-environment for cost control.

Two design touches I'm particularly happy with:

- A live ingestion timeline in the UI — Phase 1 → Phase 2 → "extracting entities" → "wiki page created" — so the user sees their doc moving through the pipeline instead of staring at a spinner.
- A "save to wiki" action that pushes the agent's chat output back through the same two-phase pipeline. The agent's prior synthesis becomes input to its next turn, which is what makes the knowledge actually compound.

---

## 5 · Frontend: streaming + tool-calls + artifacts

The chat surface uses [`assistant-ui`](https://github.com/assistant-ui/assistant-ui) as the message-list primitive, with custom layers on top:

- **Custom content parts** for tool calls, artifacts, file references, errors. Each registered with the assistant-ui content-parts API.
- **Mermaid diagram inline rendering** with a validation step — we run candidate diagrams through a pre-render check before mounting them, so a malformed diagram surfaces an inline error instead of crashing the whole message.
- **Resizable two-panel layout.** Chat thread on the left, artifact viewer on the right. The split lets users keep iterating in chat while reviewing the latest deck/doc/code without losing scroll position.
- **Real-time tool-call collapsing.** Long tool calls collapse to a one-line summary by default with an expand affordance, so a chat with 20 tool calls is still scannable.

---

## Stack

| Layer | What I used |
|---|---|
| Backend | FastAPI · Python 3.11 · supervisord |
| Frontend | React 18 · TypeScript · Tailwind · Vite · assistant-ui |
| Agent | Forked MiniMax Mini-Agent + custom skills |
| Models | Self-hosted LLMs on SageMaker (chat, embeddings, rerankers); third-party APIs for web search and image generation |
| Storage | PostgreSQL for memory; S3 for documents and artifacts |
| Deployment | Docker Compose for dev; multi-process supervisord on EC2 for prod |

---

## What I'd do differently

- **Pick the harness later.** I committed to one agent loop early; some of the integration friction would have evaporated with a smaller, less opinionated runtime. If I were starting over I'd prototype the streaming + tool-call surface against a thin in-house loop first, then swap a vendor harness in once the contract is stable.
- **Treat skills as a product surface.** Skills started as throwaway prompts and ended up being the highest-leverage thing in the system. I would version them, test them, and ship them through CI from day one.
- **Bake observability into the skill protocol.** When a skill misbehaves the easiest debugging signal is a structured event log — entries / exits / tool calls / outputs. Adding this in retrospect was painful; designing it into the skill API would have been cheap.

---

## Why no source

The codebase belongs to the team I built it for. This page is the public-facing version: the architectural patterns and the lessons, without the parts that are someone else's IP.

If you'd like to talk about agent harness integration, skill authoring, or RAG-heavy product design, I'm on LinkedIn.
