```
░█████╗░███╗░░░███╗███╗░░██╗██╗███╗░░██╗░█████╗░██████╗░███████╗
██╔══██╗████╗░████║████╗░██║██║████╗░██║██╔══██╗██╔══██╗██╔════╝
██║░░██║██╔████╔██║██╔██╗██║██║██╔██╗██║██║░░██║██║░░██║█████╗░░
██║░░██║██║╚██╔╝██║██║╚████║██║██║╚████║██║░░██║██║░░██║██╔══╝░░
╚█████╔╝██║░╚═╝░██║██║░╚███║██║██║░╚███║╚█████╔╝██████╔╝███████╗
░╚════╝░╚═╝░░░░░╚═╝╚═╝░░╚══╝╚═╝╚═╝░░╚══╝░╚════╝░╚═════╝░╚══════╝
```

<div align="center">

**Decentralized Multi-Agent Ecosystem with Emergent Collaborative Intelligence**

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.11+-brightgreen.svg)](https://python.org)
[![Rust](https://img.shields.io/badge/rust-1.75+-orange.svg)](https://rust-lang.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688.svg)](https://fastapi.tiangolo.com)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

[**Quickstart**](#quickstart) · [**How It Works**](#how-it-works) · [**Marketplace**](#agent-marketplace) · [**BYOK**](#bring-your-own-key) · [**Architecture**](#architecture) · [**Contributing**](#contributing)

</div>

---

## What is OmniNode?

Most multi-agent frameworks are for-loops with personas attached. An agent receives a message, emits a response, and forgets everything. There is no internal state that persists between calls, no mechanism by which being wrong changes future behavior, no genuine tension between agents.

OmniNode is different by design.

```
User: "Review this auth system. @SecurityAuditor try to break it, @BackendDev tell me if it'll scale."

SecurityAuditor  →  "JWT secret is 12 characters. This is a CRITICAL severity finding.
                     Anyone with basic compute can brute-force this offline."

BackendDev       →  "Schema looks fine for 10k users but the session table has no
                     index on user_id. You'll feel that at 100k."

DataScientist    →  [silent — not relevant to this message, correctly stays out]

SecurityAuditor  →  [interrupts] "The session table issue BackendDev mentioned would
                     also be exploitable for user enumeration. Fix both together."
```

The silence is real. The interruption is earned. The disagreement emerges from what was actually said. None of this is scripted.

---

## Quickstart

**Prerequisites**: Python 3.11+, Rust 1.75+, [Ollama](https://ollama.ai)

```bash
# 1. Clone and install
git clone https://github.com/shvgroups/omninode.git
cd omninode
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# 2. Build the Rust attention router
./scripts/build_router.sh

# 3. Configure
cp .env.example .env
# Set APP_SECRET_KEY to any 32+ character string

# 4. Initialize database and seed default agents
alembic upgrade head
python scripts/seed_marketplace.py

# 5. Pull a model and start
ollama pull llama3.1:8b
uvicorn server.main:app --reload
```

Server is live at `http://localhost:8000` · API docs at `http://localhost:8000/docs`

---

## How It Works

### The Attention Router — Rust Core

Every message passes through a Rust router before any LLM call is made. The router computes cosine similarity between the message embedding and each agent's capability embedding, modulated by live agent state.

```
effective_threshold = base_threshold + (fatigue × 0.15) − (confidence × 0.10)
```

Agents below their threshold are **not called**. The silence is computational, not filtered output. At 4μs per routing decision, it is never the bottleneck.

```
10,000 routing calls → 40ms total  (benchmarked on commodity hardware)
```

### Agent State Machine

Agents are processes, not functions. They carry behavioral state across every turn of a session.

| State        | Range       | Effect                                                                                |
|--------------|-------------|---------------------------------------------------------------------------------------|
| `confidence` | [0.0 → 1.0] | Lowers routing threshold — confident agents speak up more readily                     |
| `fatigue`    | [0.0 → 1.0] | Raises routing threshold, reduces `max_tokens` — tired agents are brief and selective |
| `engagement` | [0.0 → 1.0] | Rises on-topic, drifts toward neutral off-topic                                       |

**Hard limit**: agents with `fatigue ≥ 0.8` cannot be scheduled regardless of relevance. Structural enforcement, not instructed restraint.

**State-driven generation** — not prompt injection:
- `fatigue ≥ 0.6` → `max_tokens = 150` (brevity from constraint)
- `recently_corrected` → `temperature × 0.7` + epistemic hedge in system prompt addendum
- `confidence > 0.75` → slight temperature boost for fluent domain experts

### Three-Tier Memory

```
┌─────────────────────────────────────────────────────┐
│  WORKING MEMORY   (shared, ephemeral, all agents)   │
│  Rolling window · Current session · Released on close│
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  EPISODIC MEMORY  (per-agent, session-scoped)        │
│  What this agent said · Stance hashes · SQLite       │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  SEMANTIC MEMORY  (per-agent, cross-session)         │
│  Distilled facts · ChromaDB · Exponential decay      │
└─────────────────────────────────────────────────────┘
```

At session close, a distillation pass scores every agent response for information density. High-signal content (decisions, recommendations, key facts) is embedded and written to semantic memory. Filler responses score zero and are discarded. Memories decay across sessions unless retrieved and reinforced.

### Conversation Orchestrator

Turn scheduling follows four ordered rules:

1. **@mentioned agents** respond first, in mention order — hard bypass of all thresholds
2. **Domain-primary agents** (highest router score) before domain-secondary
3. **Higher-confidence** before lower-confidence within the same score tier
4. **Agents with a recorded stance** respond last — they react to what others said

**Debate detection**: When two agents have contradictory stance vectors on the same topic (computed from episodic content), the orchestrator flags a debate, expands the turn budget by 1, and adds a neutral tiebreaker. The conflict surfaces because it was already there in the session history — not because it was scripted.

### WebSocket Event Stream

```json
{"type": "agent_thinking", "agent_id": "...", "agent_name": "SecurityAuditor"}
{"type": "token",          "agent_id": "...", "token": "JWT"}
{"type": "token",          "agent_id": "...", "token": " secret"}
{"type": "agent_done",     "agent_id": "...", "content": "JWT secret is 12 characters..."}
{"type": "debate_signal",  "agent_a": "SecurityAuditor", "agent_b": "BackendDev", "score": -0.74}
{"type": "silence",        "reason": "no_agents_qualified"}
{"type": "turn_complete",  "turn_index": 7}
```

---

## Agent Marketplace

OmniNode ships with 10 default agents. Add them once, use them in any session.


                                                                                           | Interrupt |
| Agent                      | Domain                                                      | Threshold |
|----------------------------|-------------------------------------------------------------|-----------|
| **Backend Engineer**       | System design, databases, API architecture                  | 0.84      |
| **Security Auditor**       | OWASP, penetration testing, cryptography                    | 0.80      | 
| **Socratic Challenger**    | Logic, critical thinking, assumption testing                | 0.78      |
| **Data Scientist**         | Statistics, ML, experiment design                           | 0.83      | 
| **Product Manager**        | Strategy, prioritization, user problems                     | 0.82      |
| **Frontend Engineer**      | React, TypeScript, accessibility, performance               | 0.84      |
| **Startup Advisor**        | Go-to-market, fundraising, hard decisions                   | 0.80      |
| **Research Peer Reviewer** | Methodology, statistical validity, claim-evidence           | 0.85      |
| **Creative Director**      | Brand, copywriting, making work memorable                   | 0.78      |
| **Rubber Duck**            | Listening, thinking partnership, clarifying questions       | 0.90      |

### Adding an agent

```bash
curl -X POST http://localhost:8000/api/v1/marketplace/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Rust Expert",
    "bio": "Systems programmer obsessed with zero-cost abstractions, memory safety, and making the compiler your ally.",
    "system_prompt": "You are a senior Rust engineer. You think in terms of ownership, lifetimes, and trait bounds before thinking in terms of algorithms. When reviewing Rust code: check for unnecessary clones, misused Arc/Mutex, and blocking in async contexts. When someone reaches for unsafe: ask why the safe alternative is insufficient. Be precise about what the borrow checker is trying to tell you.",
    "capability_tags": ["rust", "systems", "memory-safety", "performance"],
    "temperature": 0.4,
    "response_threshold": 0.65,
    "interrupt_threshold": 0.86
  }'
```

### Shareable configurations

Export a team configuration as a portable bundle:

```bash
# Export by slug
python scripts/export_bundle.py export \
  --slugs backend-engineer security-auditor data-scientist \
  --output my_team.json

# Share the file — anyone can import it
python scripts/export_bundle.py import my_team.json

# Validate before importing
python scripts/export_bundle.py validate my_team.json
```

---

## Bring Your Own Key

OmniNode runs fully offline with Ollama. Cloud providers are optional.

```env
# .env

# Ollama (default, no key needed)
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_DEFAULT_MODEL=llama3.1:8b

# OpenAI (optional)
OPENAI_API_KEY=sk-...
OPENAI_DEFAULT_MODEL=gpt-4o

# Anthropic (optional)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_DEFAULT_MODEL=claude-3-5-sonnet-20241022

# Gemini (optional)
GEMINI_API_KEY=AIza...
GEMINI_DEFAULT_MODEL=gemini-1.5-pro
```

Each session specifies its backend at creation time. OpenAI-compatible custom
endpoints (Azure, Together, Groq, LM Studio) are supported via `OPENAI_BASE_URL`.

See [BYOK Setup](docs/BYOK_SETUP.md) for complete configuration instructions.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    NEXT.JS FRONTEND                         │
│         Chat UI · Agent Marketplace · Config Export         │
└──────────────────────────┬──────────────────────────────────┘
                           │ WebSocket (bidirectional stream)
┌──────────────────────────▼──────────────────────────────────┐
│                    FASTAPI SERVER                           │
│        REST routes · WebSocket manager · Auth               │
└──────┬──────────┬──────────────────┬────────────────────────┘
       │          │                  │
       ▼          ▼                  ▼
┌──────────┐ ┌──────────┐    ┌──────────────┐
│ATTENTION │ │   CONV.  │    │   MEMORY     │
│ROUTER    │ │ORCHESTR. │    │   SYSTEM     │
│(Rust)    │ │(Python)  │    │(Chroma+SQL)  │
└──────┬───┘ └────┬─────┘    └──────┬───────┘
       │          │                 │
       └────┬─────┘                 │
            ▼                       │
┌───────────────────────────────────▼───────────────────────┐
│                   AGENT RUNTIME                           │
│   AgentProcess × N  ·  StateMachine × N  ·  Context×N    │
└───────────────────────────────┬───────────────────────────┘
                                │
                ┌───────────────▼───────────────┐
                │       LLM ADAPTER LAYER        │
                │  Ollama · OpenAI · Anthropic   │
                │          Gemini · BYOK         │
                └───────────────────────────────┘
```

**Stack:**

| Layer | Technology |
|------------------|------------------------------------------------|
| Backend          | FastAPI 0.111 + Python 3.11                    |
| Attention Router | Rust 1.75 + PyO3 + DashMap                     |
| Embeddings       | all-MiniLM-L6-v2 (22MB, CPU-friendly, 384-dim) |
| Semantic Memory  | ChromaDB (local persistent or remote)          |
| Relational       | SQLite (dev) → PostgreSQL (production)         |
| Real-time        | WebSockets (native FastAPI)                    |
| Frontend         | Next.js 14 + React + Tailwind                  |

Full technical specification: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

---

## REST API

```
POST   /api/v1/sessions                        Create session
GET    /api/v1/sessions/{id}                   Get session
PATCH  /api/v1/sessions/{id}                   Update session
DELETE /api/v1/sessions/{id}                   Delete session
POST   /api/v1/sessions/{id}/close             Graceful close (flush + distill)
GET    /api/v1/sessions/{id}/messages          Message history

POST   /api/v1/sessions/{id}/agents            Add agent to session
GET    /api/v1/sessions/{id}/agents            List agents with live state
DELETE /api/v1/sessions/{id}/agents/{aid}      Remove agent
POST   /api/v1/sessions/{id}/agents/{aid}/reset_state

GET    /api/v1/marketplace/agents              Browse public agents
POST   /api/v1/marketplace/agents              Publish agent
GET    /api/v1/marketplace/agents/{id}         Get agent profile
PATCH  /api/v1/marketplace/agents/{id}         Update agent
DELETE /api/v1/marketplace/agents/{id}         Delete agent

POST   /api/v1/marketplace/bundles/export      Export agents as bundle
POST   /api/v1/marketplace/bundles/validate    Validate bundle
POST   /api/v1/marketplace/bundles/import      Import bundle

GET    /api/v1/sessions/{id}/export/json       Export conversation (JSON)
GET    /api/v1/sessions/{id}/export/html       Export conversation (HTML)

WS     /ws/{session_id}                        Main chat WebSocket
GET    /health                                 Health check
GET    /ready                                  Readiness probe
GET    /info                                   Runtime diagnostics
```

---

## Docker

```bash
# Development (SQLite + local ChromaDB, hot-reload)
docker compose up omninode

# Full stack (PostgreSQL + ChromaDB)
docker compose --profile full up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Contributing

OmniNode has a specific architectural philosophy. Before contributing, read
[docs/EMERGENCE.md](docs/EMERGENCE.md) to understand the design principles,
and [CONTRIBUTING.md](CONTRIBUTING.md) for the full contribution guide.

The short version:

> Ask what structural rule would make the behavior arise naturally,
> rather than what text would produce it as output.

**High-value contributions:**
- New LLM adapters for OpenAI-compatible endpoints
- Marketplace agents with genuine, opinionated system prompts
- Improvements to the Rust router (performance benchmarks required)
- Memory system improvements (distillation scoring, decay tuning)
- Test coverage for orchestrator edge cases

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

You are free to use OmniNode in commercial products, modify it, and distribute
it. Attribution is required. See the license for the full terms.

---

## Acknowledgments

Built by **SHV Groups Eunoia Labs**.

The Rust attention router architecture is informed by the same design
principles as [Eunoia Raiden](https://shvgroups.com/research [upcoming]) — the
353.9M parameter ARC-AGI reasoning engine developed by the same team.
The insight that routing is a hard gate, not a soft preference, comes
directly from that work.

Lead Dev and architecture:- shaurya verma, founder SHV Groups

---

<div align="center">

**OmniNode** — because the best conversations happen when everyone is listening.

[⭐ Star this repo](https://github.com/shvgroups/omninode) · [🐛 Report a bug](https://github.com/shvgroups/omninode/issues) · [💬 Discuss](https://github.com/shvgroups/omninode/discussions)

</div>
