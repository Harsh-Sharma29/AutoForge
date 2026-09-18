# AutoForge
> An autonomous multi-agent software engineering system that plans, writes, validates, executes, and self-heals Python code inside secure Docker sandboxes.

## Demo / Links
- 🎥 [Watch Demo](#) <!-- Add Demo Link Here -->
- 💻 [GitHub Repository](https://github.com/Harsh-Sharma29/AutoForge)
- 🌐 [Live Demo](#) <!-- Add Live URL if applicable -->
- 📄 [Portfolio](https://harsh-sharma-portfolio12.netlify.app)

## Overview
AutoForge is an advanced ReAct-based multi-agent system designed to act as an autonomous software engineer. It solves the problem of unreliable AI code generation by enforcing a strict pipeline: plan, generate, statically validate, execute in a secure sandbox, and self-heal based on real runtime tracebacks. 

The architecture is built on LangGraph for stateful multi-agent orchestration, FastAPI for the backend API and real-time streaming, and Next.js for a professional IDE-like frontend. By combining AST-based static analysis with ephemeral Docker container execution, AutoForge ensures that AI-generated code is both safe to run and iteratively perfected without human intervention.

## Key Engineering Highlights
- **Multi-Agent Orchestration:** Powered by LangGraph, featuring specialized agents (Planner, Coder, Validator, Terminal, Debugger, Research, Knowledge) governed by deterministic intent routing.
- **Sandboxed Execution:** AI-generated code is executed safely in ephemeral, read-only `python:3.11-slim` Docker containers with strict resource limits (256MB memory, 60s timeout).
- **Self-Healing Loop:** Runtime tracebacks from the Docker sandbox are caught and routed to a dedicated Debugger agent, enabling autonomous, closed-loop error resolution.
- **AST-Powered Static Validation:** Pre-execution static analysis intercepts dangerous operations (e.g., `os.remove`, `eval`) and enforces safe coding practices.
- **Real-Time Streaming:** Built on FastAPI WebSockets and PubSub to stream sandbox stdout/stderr and pipeline state directly to the Next.js frontend in real time.
- **Dynamic Multi-Provider LLMs:** A robust factory pattern supports seamless switching between Groq, Gemini, OpenAI, and Anthropic Claude models with fallback capabilities.
- **Persistent Memory:** Utilizes PostgreSQL-backed checkpointers for conversation state management and context injection.

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
    Docker -.->|stdout/stderr| PubSub(PubSub / WebSockets)
    PubSub -.-> UI
    
    subgraph Data Layer
        DB[(PostgreSQL)] -.->|State/Checkpoints| Router
        Memory[RAG/Memory] -.-> Coder
    end
```

| Component | Responsibility |
|---|---|
| **Next.js Frontend** | Provides an IDE-like interface, real-time terminal output, and pipeline visualization. |
| **FastAPI Backend** | Exposes REST APIs, manages WebSockets for real-time streaming, and orchestrates the LangGraph pipeline. |
| **LangGraph Orchestrator** | Manages the state machine, routes user intent, and orchestrates agent transitions. |
| **AST Validator** | Statically analyzes generated code to prevent unsafe operations before execution. |
| **Docker Sandbox** | Runs generated code in an isolated, resource-constrained container and captures stdout/stderr. |
| **PostgreSQL** | Persists conversation checkpoints and agent state for seamless session recovery. |

## How It Works
1. **User Submits Request:** The user provides a prompt via the Next.js frontend.
2. **Intent Classification:** The Router agent classifies the request as `coding`, `research`, or `generic`.
3. **Planning & Generation:** For coding tasks, the Planner outlines a mission brief. The Coder (powered by the selected LLM and ReAct tooling) synthesizes the required Python code.
4. **Static Validation:** The Validator uses `ast.NodeVisitor` to inspect the code for hardcoded secrets or unauthorized system calls.
5. **Sandboxed Execution:** Validated code is dispatched to the Terminal agent, which spins up a secure Docker container, mounts the code, and executes it.
6. **Self-Healing (If Needed):** If execution fails, the Terminal captures the traceback and routes it to the Debugger. The Debugger synthesizes a fix and re-submits it to the Validator.
7. **Real-Time Delivery:** Throughout the process, state changes and sandbox logs are streamed via WebSockets to the user's IDE.

## Core Features
| Feature | Explanation | Implementation |
|---|---|---|
| **Autonomous Self-Healing** | The system automatically fixes its own runtime errors up to a configured retry limit. | Tracebacks are captured from Docker and fed into a closed LangGraph loop targeting the Debugger agent. |
| **Secure Sandboxing** | Code execution cannot harm the host machine or access unauthorized network resources. | Uses the `docker` Python SDK to deploy `python:3.11-slim` containers with `read_only=True` and `mem_limit="256m"`. |
| **AST Validation** | Prevents execution of malicious or destructive AI-generated code. | Custom Python `ast` parser intercepts functions like `exec`, `eval`, and specific `os` methods. |
| **Live Terminal Streaming** | Users see code execution logs exactly as if they were running them locally. | Backend `PubSub` module and WebSockets stream Docker `logs(stream=True)` directly to the UI. |
| **Multi-Model LLM Support** | Flexibility to use the most appropriate or cost-effective AI model. | Implemented via `llm_factory.py` supporting Groq, Gemini, OpenAI, and Anthropic APIs. |
| **Contextual Memory** | Conversations and context persist across sessions. | Integrated LangGraph checkpointer utilizing PostgreSQL. |

## AI / Agent Architecture
AutoForge relies on a strictly defined ReAct workflow orchestrated by LangGraph:
- **Router:** The entry point. Uses LLM classification to route the prompt to the appropriate subsystem, preventing unnecessary and expensive Docker invocations for simple Q&A.
- **Planner:** Breaks down complex user requests into actionable steps and artifact targets.
- **Coder:** The primary synthesis engine. It has access to tool nodes (e.g., PyGithub for repository navigation, Tavily for web search).
- **Validator:** A programmatic (non-LLM) node that enforces security policies.
- **Terminal:** The execution bridge interacting with the Docker daemon.
- **Debugger:** An LLM node specifically prompted to analyze tracebacks and patch existing code without rewriting from scratch.

*Workflow:* Plan → Generate → Validate → Execute → Observe → Debug → Repair → Re-execute

## Security
- **Sandboxing:** Docker containers are run in `read_only` mode with restricted PIDs (`pids_limit=50`) and CPU/memory quotas to prevent fork bombs and resource exhaustion.
- **Input Validation (AST):** Prevents the Coder agent from accidentally (or maliciously) generating code that compromises the environment.
- **Container Timeouts:** A strict 60-second execution limit ensures infinite loops in generated code do not hang the system.

## Tech Stack
| Category | Technologies |
|---|---|
| **Language** | Python 3.11, TypeScript |
| **AI / Orchestration** | LangGraph, LangChain, Groq, Google Gemini, OpenAI, Anthropic |
| **Backend** | FastAPI, Uvicorn, WebSockets |
| **Frontend** | Next.js 15, React, Tailwind CSS |
| **Databases** | PostgreSQL |
| **Infrastructure** | Docker, Docker Compose |
| **External APIs** | Tavily (Search), PyGithub |

## Project Structure
```text
AutoForge/
├── backend/
│   ├── src/
│   │   ├── agents/      # LangGraph agent definitions (Coder, Debugger, etc.)
│   │   ├── api/         # FastAPI routes and WebSocket handlers
│   │   ├── core/        # Memory, Checkpointer, LLM Factories, PubSub
│   │   ├── graph/       # LangGraph state machine orchestration
│   │   ├── sandbox/     # Docker execution manager
│   │   └── tools/       # ReAct tools (File Ops, Web Search, GitHub)
│   ├── main.py          # CLI entry point
│   └── requirements.txt 
├── frontend/
│   ├── src/             # Next.js App Router, Components, Hooks
│   ├── package.json     
│   └── next.config.ts   
├── docker-compose.yml   # Full stack deployment configuration
└── README.md
```

## Local Setup

### Prerequisites
- Python 3.9+
- Node.js 18+
- Docker Desktop (Must be running for sandboxed execution)
- API Keys for your preferred LLM

### 1. Clone & Install
```bash
git clone https://github.com/Harsh-Sharma29/AutoForge.git
cd AutoForge

# Backend Setup
python -m venv .venv
source .venv/bin/activate  # On Windows use: .venv\Scripts\activate
pip install -r backend/requirements.txt

# Frontend Setup
cd frontend
npm install
cd ..
```

### 2. Environment Configuration
Create a `.env` file in the root directory (refer to `.env.example` if available).
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
Start PostgreSQL (and optionally the full stack) via Docker Compose:
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
Navigate to `http://localhost:3000` (or `3005` based on your frontend port config) to access the UI.

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
| `POST` | `/api/v1/execute` | Trigger the LangGraph pipeline |
| `WS` | `/api/v1/ws/terminal/{thread_id}` | WebSocket connection for real-time sandbox streaming |

## Screenshots
<!-- Add screenshots here to demonstrate the Next.js UI, Live Terminal, and Graph State -->

## Demo
🎥 [Watch the Demo](#)
> Watch AutoForge autonomously plan, write, and debug a complex Python script in real-time.

## Engineering Decisions
- **LangGraph over LangChain Chains:** Chose LangGraph to support cyclical workflows (like the Debugger-Validator loop) and strict state management, which are impossible with linear DAG chains.
- **Docker over Local Execution:** Executing AI-generated code directly on the host is a severe security risk. Docker provides a fast, ephemeral, and strictly resource-constrained sandbox.
- **AST Parsing for Validation:** Relying on an LLM to "check its own work" for security flaws is unreliable. Programmatic AST parsing guarantees that specific dangerous system calls are caught 100% of the time.
- **PostgreSQL for Checkpointing:** Enables robust, long-term persistence of conversation graphs, allowing users to resume complex software engineering sessions across page reloads.

## Reliability / Error Handling
- **Deterministic Routing:** The Router agent prevents infinite loops by categorizing tasks before they reach the Coder.
- **Timeouts:** A hard 60-second timeout on Docker execution prevents the system from hanging on infinite `while` loops generated by the AI.
- **Fallback Logic:** `llm_fallback.py` implements routing to backup LLMs if the primary provider experiences downtime or rate limits.
- **State Recovery:** LangGraph checkpoints allow the system to recover the exact graph state if the backend restarts.

## Current Status
- ✅ **Implemented:** Multi-agent pipeline, Docker sandboxing, AST validation, WebSockets streaming, Next.js UI, PostgreSQL Checkpointing.
- 🚧 **Deployment:** Currently optimized for local development and demonstration via Docker Compose.
- ⚠️ **Limitations:** The sandbox currently only supports Python environments natively. Persistent storage across Docker sandbox runs is not supported by design (ephemeral).

## Future Improvements
- [ ] Support for Node.js/TypeScript sandboxing
- [ ] Integration with CI/CD pipelines (e.g., auto-fixing GitHub Issues)
- [ ] Enhanced RAG over local codebases using pgvector
- [ ] Persistent workspace volumes for multi-file projects spanning days

## Author
**Harsh Sharma**
- GitHub: [https://github.com/Harsh-Sharma29](https://github.com/Harsh-Sharma29)
- LinkedIn: [https://www.linkedin.com/in/harsh-sharma029](https://www.linkedin.com/in/harsh-sharma029)
- Portfolio: [https://harsh-sharma-portfolio12.netlify.app](https://harsh-sharma-portfolio12.netlify.app)
