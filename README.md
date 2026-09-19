# AutoForge
An autonomous multi-agent software engineering system that plans, writes, validates, executes, and self-heals Python code inside secure Docker sandboxes.

## 🎥 Demo & Links
- 🎥 [Watch Demo](#) <!-- Add Demo Link Here -->
- 💻 [GitHub Repository](https://github.com/Harsh-Sharma29/AutoForge)
- 📄 [Portfolio](https://harsh-sharma-portfolio12.netlify.app)

## Overview
AutoForge is a ReAct-based multi-agent system designed to autonomously generate and iteratively refine code. It enforces a strict execution pipeline: plan, generate, statically validate, execute in a secure sandbox, and self-heal based on real runtime tracebacks.

The architecture uses LangGraph for stateful multi-agent orchestration, FastAPI for the backend API and streaming, and Next.js for the frontend. By combining AST-based static analysis with ephemeral Docker container execution, AutoForge ensures generated code runs securely and is iteratively corrected based on deterministic execution feedback.

## Key Engineering Highlights
- **Multi-Agent Orchestration:** Powered by LangGraph, featuring specialized agents (Planner, Coder, Validator, Terminal, Debugger, Research, Knowledge) governed by deterministic intent routing.
- **Sandboxed Execution:** Code is executed in ephemeral, read-only `python:3.11-slim` Docker containers with strict resource limits (256MB memory, 60s timeout, restricted CPU and PIDs).
- **Self-Healing Loop:** Runtime tracebacks from the Docker sandbox are caught and routed to a dedicated Debugger agent, enabling a closed-loop error resolution process.
- **AST-Powered Static Validation:** Pre-execution static analysis intercepts configured dangerous operations (e.g., `os.remove`, `eval`) before they reach the runtime environment.
- **Real-Time Streaming:** Uses FastAPI Server-Sent Events (SSE) to stream pipeline state and WebSockets to stream Docker execution logs directly to the Next.js frontend.
- **Dynamic Multi-Provider LLMs:** A factory pattern supports switching between Groq, Gemini, OpenAI, and Anthropic Claude models with fallback capabilities.
- **Persistent Memory & RAG:** Utilizes PostgreSQL-backed checkpointers for conversation state and ChromaDB for workspace indexing and Retrieval-Augmented Generation (RAG).
- **CI/CD Automation:** Implemented GitHub Actions workflows to automate Docker builds, EC2 deployment via SSH, and post-deployment health checks for frontend and backend services.

## Architecture
```mermaid
graph TD
    User([User]) -->|Prompt| UI(Next.js IDE)
    UI -->|API Request| Router(LangGraph Router)
    
    subgraph Multi-Agent Orchestration
        Router -->|coding| Planner(Planner Agent)
        Router -->|research| Research(Research Agent)
        Router -->|generic| Knowledge(Knowledge Agent)
        
        Planner --> Coder(Coder Agent)
        Coder --> Validator(AST Validator)
        Validator -->|Failed| Coder
        Validator -->|Passed| Terminal(Terminal Agent)
        
        Terminal -->|Traceback| Debugger(Debugger Agent)
        Debugger --> Validator
    end

    Terminal -.->|Executes| Docker[Ephemeral Docker Sandbox]
    Docker -.->|stdout/stderr| PubSub(WebSockets)
    Router -.->|Pipeline State| SSE(Server-Sent Events)
    PubSub -.-> UI
    SSE -.-> UI
    
    subgraph Data Layer
        DB[(PostgreSQL)] -.->|State/Checkpoints| Router
        Memory[(ChromaDB)] -.->|Workspace Vectors| Coder
    end
```

| Component | Responsibility | Implementation |
|---|---|---|
| **Next.js Frontend** | User interface and real-time visualization. | React 19, Tailwind CSS. |
| **FastAPI Backend** | REST APIs, WebSockets, SSE, and pipeline orchestration. | `src.api.server:app` running via Uvicorn. |
| **LangGraph Orchestrator** | State machine management and agent transitions. | LangGraph checkpointer utilizing PostgreSQL. |
| **AST Validator** | Static analysis to block unsafe operations. | Python `ast.NodeVisitor`. |
| **Docker Sandbox** | Executes generated code in an isolated environment. | Docker Python SDK managing `python:3.11-slim`. |

## How It Works
1. **User Submits Request:** The prompt is sent via the Next.js frontend.
2. **Intent Classification:** The Router agent classifies the request as `coding`, `research`, or `generic`.
3. **Planning & Generation:** For coding tasks, the Planner outlines steps. The Coder (utilizing ReAct tooling) synthesizes the Python code.
4. **Static Validation:** The Validator uses `ast.NodeVisitor` to inspect the code for hardcoded secrets or explicitly unauthorized system calls.
5. **Sandboxed Execution:** Validated code is dispatched to the Terminal agent, which spins up a Docker container, mounts the code, and executes it.
6. **Self-Healing (If Needed):** Execution failures capture tracebacks, routing them to the Debugger. The Debugger synthesizes a patch and re-submits it to the Validator.
7. **Real-Time Delivery:** State changes are streamed via SSE, while sandbox logs are streamed via WebSockets to the UI.

## Core Features
| Feature | Explanation | Implementation |
|---|---|---|
| **Autonomous Self-Healing** | Automatically fixes runtime errors up to a configured retry limit. | Tracebacks feed a closed LangGraph loop targeting the Debugger agent. |
| **Sandboxed Execution** | Code runs in constrained ephemeral containers. | Docker Python SDK deploying `python:3.11-slim` with `read_only=True` and `mem_limit="256m"`. |
| **AST Validation** | AST-based validation provides deterministic checks for configured unsafe operations before execution. | Python `ast` parser intercepts functions like `exec`, `eval`, and specific `os` methods. |
| **Live Terminal Streaming** | Execution logs are visible in real-time. | Backend `PubSub` module and WebSockets stream Docker logs directly to the UI. |
| **Multi-Model LLM Support** | Flexibility across LLM providers. | `llm_factory.py` supporting Groq, Gemini, OpenAI, and Anthropic. |
| **Workspace RAG & Memory** | Persistent context across sessions and workspace search. | LangGraph Postgres checkpointer and ChromaDB vector indexing. |

## AI / Agent Architecture
AutoForge relies on a strictly defined ReAct workflow orchestrated by LangGraph:
- **Router:** The entry point. Uses LLM classification to route the prompt to the appropriate subsystem.
- **Planner:** Breaks down complex user requests into actionable steps and artifact targets.
- **Coder:** The primary synthesis engine. It accesses tool nodes (e.g., PyGithub, Tavily).
- **Validator:** A programmatic node that enforces security policies.
- **Terminal:** The execution bridge interacting with the Docker daemon.
- **Debugger:** An LLM node specifically prompted to analyze tracebacks and patch existing code.
- **Research & Knowledge:** Dedicated agents for documentation retrieval and general queries.

*Workflow:* Plan → Generate → Validate → Execute → Observe → Debug → Repair → Re-execute

## Security
- **Sandboxing:** Docker containers run in `read_only` mode with restricted PIDs (`pids_limit=50`) and memory limits (`256m`) to prevent fork bombs and resource exhaustion.
- **Input Validation (AST):** Prevents the Coder agent from generating code containing specific unauthorized system calls.
- **Container Timeouts:** A strict 60-second execution limit ensures infinite loops do not hang the system.

## Tech Stack
| Category | Technologies |
|---|---|
| **Language** | Python 3.10+ (Backend), Python 3.11 (Sandbox), TypeScript |
| **AI / Orchestration** | LangGraph, LangChain, Groq, Google Gemini, OpenAI, Anthropic |
| **Backend** | FastAPI, Uvicorn, WebSockets, SSE |
| **Frontend** | Next.js, React 19, Tailwind CSS |
| **Databases** | PostgreSQL, ChromaDB |
| **Infrastructure** | Docker, Docker Compose |
| **External APIs** | Tavily (Search), PyGithub |

## Project Structure
```text
AutoForge/
├── backend/
│   ├── src/
│   │   ├── agents/      # LangGraph agent definitions (Coder, Debugger, etc.)
│   │   ├── api/         # FastAPI routes, WebSocket and SSE handlers
│   │   ├── core/        # Checkpointer, LLM Factories, RAG, PubSub
│   │   ├── sandbox/     # Docker execution manager
│   │   └── tools/       # ReAct tools (File Ops, Web Search, GitHub)
│   ├── main.py          # CLI entry point
│   ├── Dockerfile
│   └── requirements.txt 
├── frontend/
│   ├── src/             # Next.js App Router, Components, Hooks
│   ├── package.json     
│   ├── next.config.ts   
│   └── Dockerfile
├── nginx/
│   └── Dockerfile
├── docker-compose.yml   # Full stack deployment configuration
└── README.md
```

## Local Setup

### Prerequisites
- Python 3.10+
- Node.js 18+
- Docker Desktop (Required for sandboxed execution)
- API Keys for your preferred LLM

### 1. Clone & Install
```bash
git clone https://github.com/Harsh-Sharma29/AutoForge.git
cd AutoForge

# Backend Setup
cd backend
python -m venv .venv
source .venv/bin/activate  # On Windows use: .venv\Scripts\activate
pip install -r requirements.txt
cd ..

# Frontend Setup
cd frontend
npm install
cd ..
```

### 2. Environment Configuration
Create a `.env` file in the root directory.
```env
# Example .env configuration
LLM_PROVIDER=gemini
LLM_MODEL=gemini-2.5-flash
GEMINI_API_KEY=your_api_key_here

# Database
DATABASE_URL=postgresql://autoforge:autoforge@localhost:5432/autoforge?sslmode=disable
AUTOFORGE_API_URL=http://localhost:8005
NEXT_PUBLIC_API_URL=http://localhost:8005
```

### 3. Start Infrastructure
Start the full stack (PostgreSQL, Backend, Frontend, Nginx) via Docker Compose:
```bash
docker compose up -d --build
```

### 4. Manual Startup (Development)
If not running the backend/frontend via Docker Compose, start them manually:

**Backend:**
```bash
cd backend
python -m uvicorn src.api.server:app --host 127.0.0.1 --port 8005
```

**Frontend:**
```bash
cd frontend
npm run dev
```
Navigate to `http://localhost:3005` to access the UI.

## Environment Variables
| Variable | Purpose | Required |
|---|---|---|
| `LLM_PROVIDER` | Determines which LLM factory to use (`groq`, `gemini`, `openai`, `anthropic`). | Yes |
| `LLM_MODEL` | Specific model string (e.g., `gemini-2.5-flash`). | Yes |
| `GEMINI_API_KEY` | Authentication for Google Gemini API. | If provider is gemini |
| `GROQ_API_KEY` | Authentication for Groq API. | If provider is groq |
| `DATABASE_URL` | PostgreSQL connection string for LangGraph checkpointer. | Yes |
| `TAVILY_API_KEY` | Enables the Tavily web search tool for the Coder agent. | No |
| `GITHUB_ACCESS_TOKEN` | Enables PyGithub repository navigation tools. | No |

## API / Service Endpoints
| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/v1/health` | System health check and telemetry |
| `POST` | `/api/v1/session/new` | Generate a new session `thread_id` |
| `GET` | `/api/v1/history/{thread_id}` | Retrieve conversation history for a specific session |
| `POST` | `/api/v1/execute` | Execute LangGraph pipeline and stream state via SSE |
| `WS` | `/api/v1/ws/terminal/{thread_id}` | WebSocket connection for real-time sandbox logs |

## Screenshots
<!-- Add screenshots here to demonstrate the Next.js UI, Live Terminal, and Graph State -->

## Engineering Decisions
- **LangGraph over Linear Chains:** Chose LangGraph to support cyclical workflows (like the Debugger-Validator loop) and strict state management.
- **Docker over Local Execution:** Executing generated code in Docker provides a fast, ephemeral, and resource-constrained sandbox.
- **AST Parsing for Validation:** Programmatic AST parsing provides deterministic rule enforcement for catching specific unsafe method calls.
- **PostgreSQL for Checkpointing:** Enables persistence of conversation graphs, allowing users to resume sessions across page reloads.
- **Dual Streaming Mechanisms:** WebSockets handle raw high-throughput Docker logs, while SSE manages structured pipeline state updates.

## Reliability / Error Handling
- **Deterministic Routing:** The Router agent categorizes tasks to prevent unnecessary execution loops.
- **Timeouts:** A hard 60-second timeout on Docker execution prevents the system from hanging on infinite loops.
- **Fallback Logic:** `llm_fallback.py` implements routing to backup LLMs if the primary provider experiences rate limits.
- **State Recovery:** LangGraph checkpoints allow the system to recover graph state if the backend restarts.

## Current Status
> Status: Actively developed; optimized for local development and demonstration.

- ✅ **Implemented:** Multi-agent pipeline, Docker sandboxing, AST validation, SSE/WebSockets streaming, Next.js UI, PostgreSQL Checkpointing, RAG.
- ⚠️ **Limitations:** The sandbox natively supports Python environments. Persistent storage across Docker sandbox runs is not supported by design.

## Future Improvements
- [ ] Support for Node.js/TypeScript sandboxing
- [ ] Persistent workspace volumes for multi-file projects

## Author
**Harsh Sharma**
- GitHub: [https://github.com/Harsh-Sharma29](https://github.com/Harsh-Sharma29)
- LinkedIn: [https://www.linkedin.com/in/harsh-sharma029](https://www.linkedin.com/in/harsh-sharma029)
- Portfolio: [https://harsh-sharma-portfolio12.netlify.app](https://harsh-sharma-portfolio12.netlify.app)
