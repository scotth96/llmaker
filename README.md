<div align="center">

# 🦙 llmaker

### Self-host the modern LLM stack.

**llmaker is an open-source platform for running the complete modern LLM stack on
your own infrastructure** — large language models, vector databases, embeddings,
caching, observability, and a built-in retrieval & agent layer — provisioned,
networked, and production-shaped from a single command.

Build private retrieval-augmented chatbots, FAQ assistants, and recommendation
engines locally. No third-party API keys. No data leaving your machine.

<br/>

<!--
  A real recording: `llmaker apply` provisions an LLM + Qdrant + Redis stack,
  `ls` shows the fleet, `chat` runs live inference, `top` shows the dashboard.
  Re-record with vhs — see docs/demo.tape.
-->
<a href="#quickstart"><img src="docs/demo.gif" alt="llmaker provisioning a full LLM stack — model, vector DB, and cache — from a single command, then chatting with the model" width="820"/></a>

<br/>

[![CI](https://github.com/raiyanyahya/llmaker/actions/workflows/ci.yml/badge.svg)](https://github.com/raiyanyahya/llmaker/actions/workflows/ci.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/raiyanyahya/llmaker)](https://goreportcard.com/report/github.com/raiyanyahya/llmaker)
[![Self-hosted](https://img.shields.io/badge/100%25-self--hosted-success)](#why-self-host-your-llm-stack)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-alpha-orange.svg)](#roadmap)

[![Go](https://img.shields.io/badge/Go-1.25%2B-00ADD8?logo=go&logoColor=white)](go.mod)
[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](agent/)
[![Docker](https://img.shields.io/badge/Docker-required-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/get-docker/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)](facade/)
[![LangGraph](https://img.shields.io/badge/agent-LangGraph-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![Ollama](https://img.shields.io/badge/backend-Ollama-000000?logo=ollama&logoColor=white)](https://ollama.com)
[![GPU](https://img.shields.io/badge/GPU-partitioning%20%26%20gang%20admission-76B900?logo=nvidia&logoColor=white)](#hardware--images)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS-lightgrey?logo=linux&logoColor=white)](#hardware--images)

[![Code size](https://img.shields.io/github/languages/code-size/raiyanyahya/llmaker?logo=github)](https://github.com/raiyanyahya/llmaker)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)

[Quickstart](#quickstart) · [Why llmaker](#why-self-host-your-llm-stack) · [Stacks](#stacks) · [The agent](#the-agent) · [Architecture](#architecture) · [CLI](#cli-reference) · [Roadmap](#roadmap)

</div>

---

## Overview

Running a model locally is easy. Shipping an *application* is not. A production
retrieval system needs a vector database, an embeddings service, a caching layer,
an orchestration layer, and observability — each containerized, networked, and
configured to discover the others. Assembling that is a recurring tax: a sprawl of
`docker run` flags, a brittle Compose file, and hundreds of lines of framework glue.

llmaker removes that tax. One CLI provisions the entire stack on a private network
and **operates it as a single fleet** — live status, logs, and a resource
dashboard across every model and service. Stacks are **declarative and
reconcilable** (`apply --prune`), models are **OpenAI-compatible**, and retrieval
is **traced out of the box**. From a single model to a complete application:

```bash
# ── Build a complete application stack ──────────────────────────
llmaker stack up assistant      # one command → a private ChatGPT-style UI over a local model
llmaker stack init rag          # …or scaffold any stack to edit, then apply it:
llmaker apply                   #   assistant · voice · rag · research · code · chatbot · faq · recommend · sql

# ── …or run a single model (OpenAI-compatible) ──────────────────
llmaker up --model llama3:8b    # a local endpoint — explicit, or a preset:
llmaker up chat                 #   chat · code · small · embed · vision
llmaker chat <name>             # test it in the terminal
llmaker open <name>             # open its built-in web UI

# ── …or compose the stack à la carte, service by service ────────
llmaker service catalog         # browse what's available
llmaker service add qdrant      # vector database  → qdrant:6333
llmaker service add redis       # cache / memory   → redis:6379
llmaker service add langfuse    # observability    → langfuse:3000

# ── Operate the fleet ───────────────────────────────────────────
llmaker ls                      # every model + service, one view   (--json)
llmaker top                     # live resource dashboard (TUI)
llmaker status <name>           # gauges, loaded models, endpoints
llmaker logs <name> -f          # stream logs from any container
llmaker pull mistral --on chat  # download a model with progress
llmaker stop / start / rm       # lifecycle management

# ── Consume it — the agent's API, or any OpenAI client ──────────
AGENT=$(llmaker service ls --json | jq -r '.[]|select(.service=="agent").url')
curl "$AGENT/api/ingest"    -F file=@handbook.pdf            # add knowledge
curl "$AGENT/api/chat"      -d '{"question":"refund policy?"}'   # grounded answer + sources
curl "$AGENT/api/recommend" -d '{"like":["sku1","sku2"]}'   # semantic recommendations
```

Everything lands on a private network where each container discovers the others
by name — no Compose file and no glue code.

---

## Highlights

| | |
|---|---|
| **The complete stack, curated** | Models **and** the infrastructure around them — vector databases (Qdrant, Chroma, pgvector, Weaviate), Redis, embeddings, Open WebUI, n8n, Flowise, Whisper, Langfuse — from one versioned catalog. |
| **Automatic service discovery** | Every model and service joins a private Docker network and resolves by name. Your application reaches `chat:8080` and `qdrant:6333` with zero IP wiring. |
| **App-stack boundaries** | Group containers into isolated stacks with one `network:` line (or `--network`): members resolve each other by name, nothing outside the group can reach them, and empty group networks are garbage-collected automatically. |
| **GPU partitioning** | Split a multi-GPU box between instances: `--gpus 2` takes any two free devices, `--gpus 0,1` exactly those — exclusive, label-tracked partitions (plain `--gpu` keeps its shareable all-GPUs behavior). `apply` admits a stack's whole GPU demand as a unit (gang admission), so it comes up fully or not at all. |
| **A retrieval & tool agent, built in** | A FastAPI + LangGraph service: `rewrite → retrieve → rerank → generate` (multi-turn, MMR), a tool-calling loop (calculator, knowledge base, self-hosted web search, SQL), and a semantic recommendation API. |
| **Observability by default** | Every instance exposes Prometheus `/metrics` (requests, errors, in-flight, completion tokens, tokens/sec, CPU/RAM/GPU) for scraping, and the RAG stack ships Langfuse — every query traced (retrieval hits and scores, model and token usage) with no setup. |
| **Measurable quality** | An evaluation harness (`/api/eval`) grades answers for groundedness, relevance, and correctness with an LLM judge — retrieval quality you can track across changes, not guess at. |
| **More than RAG** | First-class endpoints for summarization (map-reduce over long docs), structured JSON extraction, and speech-to-text (Whisper), plus optional Redis-backed conversation memory. |
| **Declarative, reconcilable** | Define your stack in one file. `llmaker apply` brings it to the desired state in dependency order; `--prune` removes what's no longer declared. |
| **OpenAI-compatible** | Each model exposes a stable `/v1/*` API (chat, completions, embeddings, streaming) behind one contract — and fills in the instance's default model when a request omits one, so any OpenAI client just works. Ollama runs it today, with a llama.cpp backend [in progress](#roadmap). |
| **Private by design** | Containers bind to `127.0.0.1` by default, cross-origin browser access is off until you opt in, and the CLI warns loudly — at creation and on every boot — if an instance is publicly bound without an API key. Your documents, embeddings, and traces never leave your infrastructure. |
| **Operable** | A single static Go binary, a labeled-container model with no state file to drift, `--json` output everywhere, and a live `top` dashboard. |

---

## Why self-host your LLM stack?

- **Data ownership.** Proprietary documents, customer data, and prompts stay on
  hardware you control. Nothing is sent to a third-party API.
- **No assembly tax.** The vector DB, embeddings, cache, agent, and tracing come
  pre-integrated and networked — not as a Compose file you maintain by hand.
- **Predictable cost.** Inference and retrieval run on infrastructure you already
  pay for. No per-token billing, no rate limits.
- **Portability.** The same `stack.yaml` runs on a laptop, a CI runner, or a
  server. Swap the model or the vector database without touching your application.

| | Model runners<br/>(Ollama, LM&nbsp;Studio) | DIY<br/>Docker&nbsp;Compose | Frameworks<br/>(LangChain) | **llmaker** |
|---|:---:|:---:|:---:|:---:|
| Run local models, OpenAI-compatible | ✓ | — | — | ✓ |
| Vector DB, embeddings, cache — curated | — | manual | — | ✓ |
| Service discovery between containers | — | manual | n/a | ✓ |
| Network isolation between app stacks | — | manual | n/a | ✓ |
| Exclusive GPU partitioning, gang-admitted stacks | — | manual | n/a | ✓ |
| One-command application (RAG, recsys) | — | — | — | ✓ |
| Built-in retrieval & recommendation agent | — | — | you code it | ✓ |
| Observability / tracing integrated | — | manual | manual | ✓ |
| Declarative provisioning & reconciliation | — | partial | — | ✓ |

---

## Installation

> Requires [Docker](https://docs.docker.com/get-docker/). Run `llmaker doctor` afterward to validate your environment.

```bash
# Prebuilt binary (Linux / macOS)
curl -fsSL https://raw.githubusercontent.com/raiyanyahya/llmaker/master/scripts/install.sh | sh

# Go toolchain
go install github.com/raiyanyahya/llmaker/cmd/llmaker@latest

# From source
git clone https://github.com/raiyanyahya/llmaker && cd llmaker && make build
```

<sub>Homebrew and `winget` packages are on the [roadmap](#roadmap). The agent image is built locally with `make image-agent` until it is published to a registry.</sub>

---

## Quickstart

Provision and run a complete retrieval-augmented generation stack:

```bash
llmaker stack up assistant    # scaffold + apply in one step (assistant needs no agent image)
llmaker stack init rag        # generate stack.yaml (assistant | voice | rag | research | code | chatbot | faq | recommend | sql)
make image-agent              # build the agent image once (stacks that include the agent)
llmaker apply -f stack.yaml   # provision the stack — model + services, networked
llmaker ls                    # inspect models and services in one view
```

Resolve the agent endpoint and use it:

```bash
AGENT=$(llmaker service ls --json | jq -r '.[] | select(.service=="agent").url')

curl "$AGENT/api/ingest" -F file=@handbook.pdf                     # ingest documents
curl "$AGENT/api/chat"   -d '{"question":"…","history":[],"top_k":4}'   # query, with sources
```

llmaker also runs individual models — the easiest way to expose a local,
OpenAI-compatible endpoint:

```bash
llmaker up --model llama3:8b          # provision a model instance
```
```python
from openai import OpenAI
client = OpenAI(base_url="http://127.0.0.1:11500/v1", api_key="not-needed")
client.chat.completions.create(model="llama3:8b",
    messages=[{"role": "user", "content": "Hello"}])
```

---

## Stacks

A stack is a model plus the services around it, provisioned together. Scaffold
and run one in a single step with `llmaker stack up <name>`, or generate a
`stack.yaml` to edit with `llmaker stack init <name>` and apply it with
`llmaker apply`.

| Template | Application | Components |
|---|---|---|
| `assistant` | A private, ChatGPT-style assistant over a local model — chats, prompts, RAG in the UI. No agent image to build | LLM · Open WebUI |
| `voice` | Talk to a model — speech-to-text in the browser via self-hosted Whisper. No agent image to build | LLM · Open WebUI · Whisper |
| `rag` | Document Q&A — ingest files, query with grounded answers and sources, fully traced | LLM · Qdrant · embeddings · agent · Langfuse · Postgres |
| `research` | A tool-using assistant that searches the live web *and* your documents, then synthesizes | LLM · SearXNG · Qdrant · embeddings · agent |
| `code` | A code assistant — ingest a repo, ask grounded questions and review | code LLM · Qdrant · embeddings · agent |
| `chatbot` | A multi-turn assistant with a web UI and per-session memory | LLM · Redis · agent |
| `faq` | A knowledge-base assistant tuned for short, grounded answers | LLM · Qdrant · embeddings · agent |
| `recommend` | A semantic recommendation engine — "more like this", no LLM required | Qdrant · embeddings · agent |
| `sql` | Ask your database in plain English — the agent runs read-only SQL (enforced) and grounds in docs | LLM · Postgres · Qdrant · embeddings · agent |

---

## The agent

The catalog's `agent` is a FastAPI + LangGraph service (`agent/`) that turns a
bare model and vector store into an application. It is a standard service on the
network, configured by environment to discover the others by name.

**Retrieval as an explicit graph** — `rewrite → retrieve → rerank → generate`:

- **rewrite** — collapses multi-turn history into a standalone query, so
  follow-ups that depend on context ("and when was *it* released?") resolve
  correctly. The model is only invoked when there is history to resolve.
- **retrieve** — embeds the query and retrieves a candidate set from the vector
  store.
- **rerank** — applies [Maximal Marginal Relevance](https://en.wikipedia.org/wiki/Maximal_marginal_relevance)
  for relevant, non-redundant context.
- **generate** — produces the answer from that context and the conversation.

```
POST /api/ingest      multipart file or text  →  chunk, embed, store
POST /api/chat        { question, history?, top_k?, session_id? }  →  answer + sources
POST /api/agent       { question, history?, session_id? }  →  tool-using answer + tool calls
POST /api/summarize   { text, instructions?, max_words? }  →  summary (map-reduce for long text)
POST /api/extract     { text, fields: { name: description } }  →  JSON with exactly those keys
POST /api/transcribe  multipart audio file  →  { text }   (needs a whisper service)
POST /api/eval        { cases: [{ question, reference? }] }  →  graded answers + summary
POST /api/items       { items: [{ id, text, metadata? }] }  →  index for recommendation
POST /api/recommend   { query }  or  { like: [id, …] }  →  ranked items
```

**Tool calling.** Beyond retrieval, `/api/agent` runs a tool-calling loop where
the model decides which tools to invoke — a **calculator**, the **knowledge base**
(retrieval as a tool), the **current time**, a self-hosted **web search**
(SearXNG, no paid API), and an optional read-only **SQL** tool over your
database — and the loop executes them until it has an answer. The response
includes every tool call it made. Adding a tool is one entry in
`agent/app/tools.py`.

**Tracing.** The `rag` stack provisions Langfuse and the agent traces every query
to it, with zero configuration — each request (RAG or tool-using) appears as a
trace with its retrieval, tool, and generation steps. Tracing is enabled by the
template and is otherwise opt-in via two environment variables.

**Evaluation.** `/api/eval` runs a question set through the same pipeline and
grades each answer — *groundedness* and *relevance* by LLM-as-judge, plus
*correctness* against a reference and *context recall* against expected sources
when you supply them. You get per-case scores and an aggregate summary, and every
case is traced to Langfuse alongside your live traffic — so retrieval quality is
measurable, not a vibe.

**Beyond chat.** Two everyday tasks are first-class endpoints: `/api/summarize`
condenses text (map-reducing long inputs chunk by chunk so a whole report fits),
and `/api/extract` turns text into a typed JSON object from the fields you name —
parsed defensively so a chatty model never breaks the contract. With a `whisper`
service on the network, `/api/transcribe` adds speech-to-text.

**Memory.** The agent is stateless by default (the client passes `history`). Set
`REDIS_URL` and it persists history server-side: send a `session_id` with
`/api/chat` or `/api/agent` and prior turns are loaded, prepended, and saved
automatically — capped and expiring, and best-effort so Redis being down never
fails a chat. `llmaker stack init chatbot` wires it up.

**Recommendations** reuse the same embeddings and vector store, with no model
involved: index items once, then retrieve by free-text intent (`query`) or by
example (`like`, which averages the seed items into a profile and excludes them
from the results).

Full agent contract and configuration: [`agent/README.md`](agent/README.md).

---

## Services & networking

Compose a stack from the catalog directly, or let a template do it:

```bash
llmaker service catalog          # list available services
llmaker service add qdrant       # vector database     → qdrant:6333
llmaker service add redis        # cache / memory      → redis:6379
llmaker service add embeddings   # embeddings (HF TEI) → embeddings:80
llmaker service add searxng      # web search          → searxng:8080
llmaker service add whisper      # speech-to-text      → whisper:8000
llmaker service add open-webui   # ChatGPT-style UI    → open-webui:8080
llmaker service add langfuse     # observability       → langfuse:3000
```

| Category | Services |
|---|---|
| Vector databases | Qdrant · Chroma · pgvector (Postgres) · Weaviate |
| Cache / memory | Redis (powers per-session agent memory) |
| Embeddings | HuggingFace Text-Embeddings-Inference |
| Search | SearXNG (self-hosted metasearch) |
| Speech-to-text | Whisper (faster-whisper, OpenAI-compatible) |
| Observability | Langfuse |
| Web UI & apps | Open WebUI (ChatGPT-style UI) · n8n (workflow automation) · Flowise (visual LLM app builder) |
| Agent | LangGraph retrieval & recommendation agent |

Every model and service joins a private Docker network — the shared `llmaker-net`
by default — and is addressable there by name: service discovery without IPs,
links, or a Compose file. Applications running on the host or in their own
container reach the stack the same way:

```bash
docker run --rm --network llmaker-net redis:7-alpine redis-cli -h redis ping   # → PONG
```

**Network boundaries.** By default everything shares `llmaker-net`. When you run
several app stacks side by side, give each its own boundary: a top-level
`network:` in the stack file puts every container in the file on a private
network (`llmaker-net-<name>`) — members still resolve each other by name, but
nothing outside the group can reach them, and vice versa. Two files declaring
the same `network:` share one boundary; a per-instance/per-service `network:`
overrides the file default; `llmaker up --network <name>` and
`llmaker service add <kind> --network <name>` group ad-hoc containers the same
way. Empty group networks are cleaned up automatically when their last member
is removed (never while a member exists, even stopped). `ls` and `service ls`
show each container's group in a NETWORK column (and `ls --json` under
`network`). A container belongs to
exactly one group — infrastructure consumed by several isolated stacks belongs
on the shared network — and `apply` won't move an existing container between
networks: it warns, and `rm` + re-apply applies the change.

```yaml
# r