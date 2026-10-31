# CodeGuard AI --- System Design & Architecture

**Document type:** Technical Design / Architecture Baseline\
**Version:** 1.0\
**Status:** Proposed design; subject to implementation validation\
**Date:** 31 October 2026\
**Deployment:** Local-first, Windows development environment using
Docker Desktop/WSL2 and native Ollama

------------------------------------------------------------------------

## 1. Purpose

This document captures the system design decisions made so far for
CodeGuard AI, an AI-assisted engineering quality platform.

The design is intended to support a portfolio-grade implementation that
demonstrates system design, API design, distributed-workflow thinking,
AI orchestration, privacy, reliability, observability, and engineering
governance.

The architecture is deliberately pragmatic: a modular application with
independently runnable AI processing, rather than a premature collection
of microservices.

## 2. Design principles

1.  **Local-first inference:** Source code is analyzed by a locally
    hosted model; no hosted LLM provider is used in local-only mode.
2.  **Hybrid analysis:** Deterministic static analysis is combined with
    AI reasoning.
3.  **Human accountability:** AI findings are advisory; humans remain
    responsible for review and merge decisions.
4.  **Modular architecture:** Clear boundaries between product APIs,
    analysis orchestration, analyzers, data access, and integrations.
5.  **Asynchronous processing:** Pull-request analysis is performed in
    background jobs.
6.  **Idempotency:** Webhook and job retries must not create duplicate
    analyses or GitHub comments.
7.  **Evidence-backed findings:** Findings should reference source
    locations and supporting evidence.
8.  **Provider abstraction:** Model runtimes and analysis tools should
    be replaceable behind stable interfaces.
9.  **Security by design:** Repository content is untrusted; AI agents
    have no arbitrary execution permissions.
10. **Measurability:** Model, prompt, agent, and benchmark versions are
    recorded.

## 3. Architecture overview

### 3.1 Logical architecture

``` mermaid
flowchart TB
    DEV["Developer"]
    GH["GitHub Repository / Pull Request"]
    UI["CodeGuard Web Dashboard"]

    subgraph LOCAL["Local CodeGuard Environment"]
        subgraph CORE["Core Platform - NestJS"]
            API["REST API"]
            WEBHOOK["GitHub Webhook Handler"]
            REPO["Repository Manager"]
            POLICY["Policy Engine"]
            REPORT["Reporting Service"]
        end

        subgraph PROCESS["Background Processing"]
            QUEUE[("Redis / BullMQ")]
            WORKER["Analysis Job Worker"]
        end

        subgraph AI["AI Analysis Engine - Python"]
            ORCH["LangGraph Orchestrator"]
            CONTEXT["Code Context Agent"]
            BUG["Bug Detection Agent"]
            SEC["Security Agent"]
            PERF["Performance Agent"]
            ARCH["Architecture Agent"]
            QUALITY["Code Quality Agent"]
            AGG["Finding Aggregator"]
            LLM["Ollama Local LLM"]
            EMB["Local Embedding Model"]
        end

        STATIC["Semgrep / ESLint / Tree-sitter"]
        DB[("PostgreSQL + pgvector")]
    end

    DEV --> GH
    DEV --> UI
    UI --> API
    GH --> WEBHOOK
    WEBHOOK --> REPO
    REPO --> QUEUE
    QUEUE --> WORKER
    WORKER --> STATIC
    WORKER --> ORCH
    ORCH --> CONTEXT
    ORCH --> BUG
    ORCH --> SEC
    ORCH --> PERF
    ORCH --> ARCH
    ORCH --> QUALITY
    CONTEXT --> EMB
    BUG --> LLM
    SEC --> LLM
    PERF --> LLM
    ARCH --> LLM
    QUALITY --> LLM
    LLM --> AGG
    STATIC --> AGG
    AGG --> POLICY
    POLICY --> DB
    CORE --> DB
```

The diagram is logical, not a commitment to one container per box. The
first implementation can host several modules in the same application
process.

### 3.2 Major runtime components

  -----------------------------------------------------------------------
  Component               Initial technology      Responsibility
  ----------------------- ----------------------- -----------------------
  Web UI                  Next.js, React,         Repository, analysis,
                          TypeScript              finding, and metrics
                                                  views

  Core API                NestJS, Node.js         Authentication,
                                                  business APIs,
                                                  repository metadata,
                                                  policy and reporting

  AI service              Python, FastAPI         AI orchestration,
                                                  context retrieval,
                                                  model interaction

  Analysis worker         Node.js worker using    Background job
                          BullMQ                  coordination and
                                                  analysis lifecycle

  Static analyzers        Semgrep, ESLint,        Deterministic and
                          Tree-sitter             structural analysis

  Agent orchestration     LangGraph               Stateful AI workflow
                                                  and agent coordination

  Model runtime           Ollama                  Local LLM and embedding
                                                  inference

  Database                PostgreSQL              Transactional and audit
                                                  data

  Vector extension        pgvector                Embedding storage and
                                                  similarity search

  Queue/cache             Redis, BullMQ           Job queue, retry, and
                                                  short-lived cache

  Observability           OpenTelemetry; Grafana  Logs, metrics, traces,
                          later                   dashboards

  Local deployment        Docker Compose          Reproducible local
                                                  services
  -----------------------------------------------------------------------

## 4. Architecture style and deployment boundaries

### 4.1 Initial architecture

Use a **modular NestJS application plus a separate Python AI service**,
with a background worker. PostgreSQL and Redis are shared
infrastructure.

Do not split every domain module into a network service at the outset.
Extract services only when independent scaling, ownership, security, or
deployment requirements justify the operational cost.

### 4.2 Initial process/container view

-   `web`: Next.js frontend.
-   `api`: NestJS core API.
-   `analysis-worker`: BullMQ worker; may initially share code and
    package boundaries with the API.
-   `ai-service`: FastAPI application.
-   `postgres`: PostgreSQL with pgvector extension.
-   `redis`: Redis for BullMQ.
-   `ollama`: native Windows process initially, not a Docker container.
-   Optional later: Grafana and an OpenTelemetry Collector.

### 4.3 Communication

-   Browser to NestJS: HTTPS/HTTP REST.
-   GitHub to NestJS: webhook over a secure tunnel during local
    development.
-   NestJS to Redis: BullMQ.
-   Worker to AI service: internal HTTP API initially.
-   NestJS/AI service to PostgreSQL: database access through controlled
    repositories.
-   AI service to Ollama: local HTTP API.
-   Services to observability: OpenTelemetry instrumentation.

Keep AI service endpoints internal; do not expose them directly to the
browser.

## 5. End-to-end pull-request analysis sequence

``` mermaid
sequenceDiagram
    actor Developer
    participant GH as GitHub
    participant API as NestJS API
    participant Q as Redis Queue
    participant Worker as Analysis Worker
    participant Static as Static Analyzer
    participant AI as Python AI Engine
    participant DB as PostgreSQL

    Developer->>GH: Create or update pull request
    GH->>API: PR webhook
    API->>API: Verify signature and event idempotency
    API->>DB: Record event, PR metadata, analysis run
    API->>Q: Enqueue analysis job
    API-->>GH: Acknowledge webhook

    Q->>Worker: Deliver analysis job
    Worker->>GH: Retrieve PR metadata and diff
    Worker->>Static: Run ESLint / Semgrep
    Static-->>Worker: Deterministic findings
    Worker->>AI: Analyze changed code and context
    AI->>AI: Retrieve context and execute agents
    AI->>AI: Validate structured agent output
    AI-->>Worker: AI findings

    Worker->>Worker: Normalize, validate, deduplicate
    Worker->>DB: Persist findings and analysis metadata
    Worker->>DB: Evaluate repository policy
    Worker->>GH: Publish summary and eligible inline comments
    Worker->>DB: Mark analysis complete
```

### 5.1 Workflow requirements

-   Acknowledge a valid webhook quickly; do not run model inference in
    the webhook request.
-   Verify the webhook signature before accepting the event.
-   Persist event identity and analysis lifecycle.
-   Make queue jobs idempotent.
-   Pin each analysis to a specific base SHA and head SHA.
-   Retrieve only required source files and context.
-   Apply timeouts and bounded concurrency.
-   Retry transient failures with backoff.
-   Do not retry permanent validation or authorization errors
    indefinitely.
-   Publish comments only after validating paths and line positions
    against the PR diff.
-   Record publication state to prevent duplicate comments on retry.

## 6. AI analysis architecture

### 6.1 Hybrid analysis model

Use deterministic tools for rules they can reliably establish and AI for
contextual reasoning.

  -----------------------------------------------------------------------
  Analysis type           Primary mechanism       AI role
  ----------------------- ----------------------- -----------------------
  Lint and style          ESLint                  Explain or prioritize

  Known security patterns Semgrep / selected      Explain impact and
                          security rules          context

  Syntax and structure    Tree-sitter             Provide structural
                                                  context

  Potential logic defects AI agent                Identify and explain
                                                  risks

  Cross-file design       Context retrieval + AI  Identify possible
  concerns                                        violations

  Performance concerns    Static rules and AI     Explain potential
                                                  bottlenecks
  -----------------------------------------------------------------------

AI output must not override deterministic evidence. Findings retain
provenance.

### 6.2 Agent responsibilities

  -----------------------------------------------------------------------
  Agent                   Responsibility          Initial status
  ----------------------- ----------------------- -----------------------
  Context Agent           Retrieve relevant       MVP
                          files, symbols, tests,  
                          and documentation       

  Bug Detection Agent     Identify potential      MVP
                          logic defects and edge  
                          cases                   

  Security Agent          Identify potential      MVP
                          security weaknesses     

  Performance Agent       Identify inefficient    Later
                          queries, algorithms, or 
                          resource use            

  Architecture Agent      Identify layering,      Later
                          coupling, and design    
                          concerns                

  Code Quality Agent      Review maintainability  Later
                          and complexity          

  Finding Aggregator      Normalize, deduplicate, MVP
                          validate, and combine   
                          findings                
  -----------------------------------------------------------------------

### 6.3 Orchestration

LangGraph coordinates workflow state and agent execution. The
orchestrator should:

-   Establish a bounded analysis state.
-   Route relevant files and context to agents.
-   Apply per-agent timeouts and retry limits.
-   Limit model concurrency to suit local hardware.
-   Validate structured output.
-   Collect agent errors without silently presenting incomplete analysis
    as complete.
-   Pass all findings to a centralized aggregator.
-   Record model, prompt, agent, and configuration versions.

Agents should not independently publish GitHub comments.

### 6.4 Structured finding contract

Illustrative response:

``` json
{
  "agent": "bug-detection",
  "findings": [
    {
      "category": "BUG",
      "severity": "HIGH",
      "title": "Potential null reference",
      "description": "The payment object may be null before its amount property is accessed.",
      "filePath": "src/services/payment.service.ts",
      "lineNumber": 42,
      "confidence": 0.91,
      "evidence": "payment.amount is accessed without a null check",
      "recommendation": "Validate the payment object before accessing its properties."
    }
  ]
}
```

This is an example contract, not a claim that the model can reliably
assign calibrated confidence. Confidence must be evaluated and
calibrated against benchmark data. Validate all model output against a
schema before persistence.

### 6.5 Finding aggregation

The aggregator should:

1.  Validate the output schema.
2.  Validate repository-relative file paths.
3.  Check that line references are plausible for the analyzed commit and
    diff.
4.  Normalize categories and severities.
5.  Preserve analyzer and agent provenance.
6.  Deduplicate overlapping findings.
7.  Generate stable fingerprints where possible.
8.  Preserve evidence and recommendation.
9.  Distinguish deterministic findings from AI hypotheses.

## 7. Local model and retrieval architecture

### 7.1 Initial model runtime

Use Ollama running natively on Windows for the initial development
environment.

Proposed starting models:

-   **Code model:** `qwen2.5-coder:3b-instruct` (or the exact available
    Ollama tag verified at setup).
-   **Embedding model:** `nomic-embed-text`.

The 3B model is a pragmatic CPU-first baseline for a Windows machine
with 16 GB RAM and no dedicated GPU. It is not a guaranteed quality
choice. Benchmark latency, memory consumption, context behavior, and
finding quality before relying on it.

### 7.2 Runtime configuration

Initial PowerShell configuration:

``` powershell
[Environment]::SetEnvironmentVariable(
  "OLLAMA_NUM_PARALLEL", "1", "User"
)

[Environment]::SetEnvironmentVariable(
  "OLLAMA_MAX_LOADED_MODELS", "1", "User"
)

[Environment]::SetEnvironmentVariable(
  "OLLAMA_KEEP_ALIVE", "5m", "User"
)
```

Restart Ollama after changing environment variables.

Initial model commands:

``` powershell
ollama pull qwen2.5-coder:3b-instruct
ollama pull nomic-embed-text
ollama list
ollama run qwen2.5-coder:3b-instruct
```

Verify exact tags and model availability in the Ollama library at
implementation time.

Use a conservative initial context budget, for example 4,096 tokens, and
one inference request at a time. Adjust after measuring memory and
quality.

### 7.3 Retrieval-augmented context

Initial indexing pipeline:

1.  Obtain repository source at a pinned commit.
2.  Apply file exclusions and secret filtering.
3.  Identify supported source files.
4.  Parse code with Tree-sitter where available.
5.  Split into function/class/module-aware chunks.
6.  Generate embeddings locally.
7.  Store chunk metadata and vectors in PostgreSQL/pgvector.
8.  Retrieve related chunks for changed code.
9.  Include bounded context in agent requests.
10. Incrementally update changed files and remove stale chunks.

Do not index generated output, dependency directories, binaries, or
secrets. Retrieval must return repository-relative references and
respect the active repository boundary.

### 7.4 Local-only guarantee

The local-only mode must be enforced by application configuration and
tested. It should not merely be a convention.

-   Model endpoint must point to the local runtime.
-   No hosted LLM fallback should occur.
-   Source code must not be sent to hosted embedding APIs.
-   Outbound GitHub traffic is allowed for repository integration.
-   Logs should not contain full source files or secrets.

## 8. Database design

PostgreSQL is the primary data store. pgvector provides vector search in
the same database.

### 8.1 Entity relationship diagram

``` mermaid
erDiagram
    ORGANIZATIONS ||--o{ MEMBERSHIPS : has
    USERS ||--o{ MEMBERSHIPS : belongs_to
    ORGANIZATIONS ||--o{ REPOSITORIES : owns
    REPOSITORIES ||--o{ PULL_REQUESTS : contains
    PULL_REQUESTS ||--o{ ANALYSIS_RUNS : triggers
    ANALYSIS_RUNS ||--o{ FINDINGS : produces
    FINDINGS ||--o{ FINDING_FEEDBACK : receives
    REPOSITORIES ||--o{ POLICIES : follows
    POLICIES ||--o{ POLICY_RULES : defines
    REPOSITORIES ||--o{ CODE_CHUNKS : indexes
    ANALYSIS_RUNS ||--o{ MODEL_EVALUATIONS : measures
    ORGANIZATIONS ||--o{ AUDIT_EVENTS : records

    ORGANIZATIONS {
        uuid id PK
        string name
        timestamp created_at
    }
    USERS {
        uuid id PK
        string email
        string display_name
    }
    MEMBERSHIPS {
        uuid id PK
        uuid organization_id FK
        uuid user_id FK
        string role
    }
    REPOSITORIES {
        uuid id PK
        uuid organization_id FK
        string full_name
        string default_branch
    }
    PULL_REQUESTS {
        uuid id PK
        uuid repository_id FK
        int github_pr_number
        string head_sha
        string base_sha
        string status
    }
    ANALYSIS_RUNS {
        uuid id PK
        uuid pull_request_id FK
        string status
        string model_name
        timestamp started_at
    }
    FINDINGS {
        uuid id PK
        uuid analysis_run_id FK
        string category
        string severity
        string file_path
        int line_number
        float confidence
    }
    FINDING_FEEDBACK {
        uuid id PK
        uuid finding_id FK
        string feedback_type
        string comments
    }
    POLICIES {
        uuid id PK
        uuid repository_id FK
        string name
        boolean active
    }
    POLICY_RULES {
        uuid id PK
        uuid policy_id FK
        string category
        string severity
        string action
    }
    CODE_CHUNKS {
        uuid id PK
        uuid repository_id FK
        string file_path
        string content_hash
    }
    MODEL_EVALUATIONS {
        uuid id PK
        uuid analysis_run_id FK
        string metric_name
        float metric_value
    }
    AUDIT_EVENTS {
        uuid id PK
        uuid organization_id FK
        string action
        timestamp created_at
    }
```

### 8.2 Table responsibilities

  Table               Purpose
  ------------------- -------------------------------------------
  organizations       Tenant / organization records
  users               Local user identities
  memberships         Organization membership and roles
  repositories        Connected repository metadata
  pull_requests       PR number, branch and commit metadata
  analysis_runs       One record per analysis attempt
  findings            Normalized analysis findings
  finding_feedback    Developer/reviewer feedback
  policies            Repository-level engineering policies
  policy_rules        Configured rule actions and thresholds
  code_chunks         Indexed source chunks and embeddings
  model_evaluations   Benchmark and model metric records
  audit_events        Administrative and lifecycle audit events

### 8.3 Initial SQL foundation

``` sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(150) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(150),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    user_id UUID REFERENCES users(id),
    role VARCHAR(30) NOT NULL,
    UNIQUE (organization_id, user_id)
);

CREATE TABLE repositories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    github_id BIGINT UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    default_branch VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pull_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id UUID NOT NULL REFERENCES repositories(id),
    pr_number INTEGER NOT NULL,
    head_sha VARCHAR(64) NOT NULL,
    base_sha VARCHAR(64) NOT NULL,
    status VARCHAR(30) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (repository_id, pr_number)
);

CREATE TABLE analysis_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pull_request_id UUID NOT NULL REFERENCES pull_requests(id),
    status VARCHAR(30) NOT NULL,
    model_name VARCHAR(150),
    prompt_version VARCHAR(50),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT
);

CREATE TABLE findings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_run_id UUID NOT NULL REFERENCES analysis_runs(id),
    fingerprint VARCHAR(128) NOT NULL,
    category VARCHAR(40) NOT NULL,
    severity VARCHAR(20) NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    file_path TEXT,
    line_number INTEGER,
    confidence NUMERIC(4,3),
    evidence TEXT,
    recommendation TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_analysis_pr ON analysis_runs(pull_request_id);
CREATE INDEX idx_findings_analysis ON findings(analysis_run_id);
CREATE INDEX idx_findings_fingerprint ON findings(fingerprint);
```

This is a starter schema, not a complete migration. Later migrations
must add webhook event identity, GitHub publication state, policies,
code chunks, feedback, evaluations, audit events, timestamps,
tenant-safe constraints, and indexes.

### 8.4 Data design decisions

-   Preserve analysis history rather than overwriting prior findings.
-   Pin each run to base and head commit SHA.
-   Record model, prompt, agent, and configuration versions.
-   Generate stable finding fingerprints for cross-run comparison.
-   Scope repository and user access by organization.
-   Select embedding vector dimensions from the chosen embedding model.
-   Add retention and deletion behavior for code chunks and source
    workspaces.

## 9. API design

### 9.1 Public NestJS API

All public APIs use `/api/v1`.

  Method   Endpoint                                   Purpose
  -------- ------------------------------------------ -------------------------
  POST     `/api/v1/organizations`                    Create organization
  GET      `/api/v1/repositories`                     List repositories
  POST     `/api/v1/repositories`                     Register repository
  GET      `/api/v1/repositories/:id`                 Repository details
  GET      `/api/v1/repositories/:id/pull-requests`   List pull requests
  POST     `/api/v1/webhooks/github`                  Receive GitHub webhook
  POST     `/api/v1/analyses`                         Trigger manual analysis
  GET      `/api/v1/analyses/:id`                     Analysis details
  GET      `/api/v1/analyses/:id/findings`            Retrieve findings
  POST     `/api/v1/findings/:id/feedback`            Submit feedback
  GET      `/api/v1/policies`                         List policies
  POST     `/api/v1/policies`                         Create policy
  GET      `/api/v1/dashboard/summary`                Dashboard metrics

### 9.2 Internal Python AI API

  Method   Endpoint                      Purpose
  -------- ----------------------------- --------------------------------
  GET      `/internal/health`            AI service health
  POST     `/internal/analyze`           Analyze a change set
  POST     `/internal/context/index`     Index repository content
  POST     `/internal/context/search`    Retrieve relevant context
  GET      `/internal/models`            Report configured local models
  POST     `/internal/evaluations/run`   Execute benchmark evaluation

Internal APIs must authenticate service-to-service calls and must not be
exposed directly to the browser.

## 10. Technology decisions

  -----------------------------------------------------------------------
  Area                    Decision                Rationale
  ----------------------- ----------------------- -----------------------
  Frontend                React, TypeScript,      Familiar ecosystem and
                          Next.js                 strong dashboard
                                                  support

  UI                      Tailwind CSS, shadcn/ui Reusable UI primitives

  Core backend            Node.js, NestJS         Modular structure,
                                                  validation, dependency
                                                  injection

  AI backend              Python, FastAPI         Python AI ecosystem

  Orchestration           LangGraph               Explicit stateful agent
                                                  workflows

  Local inference         Ollama                  Local model hosting

  Initial code model      Qwen2.5-Coder 3B        CPU-first starting
                          Instruct family         point; benchmark
                                                  required

  Embeddings              nomic-embed-text        Local embedding
                                                  baseline

  Database                PostgreSQL              Relational consistency
                                                  and JSON support

  Vector search           pgvector                Avoid separate vector
                                                  database initially

  Queue                   Redis, BullMQ           Asynchronous jobs,
                                                  retries

  Static analysis         Semgrep, ESLint         Deterministic checks

  Parsing                 Tree-sitter             Structural source
                                                  parsing

  SCM                     GitHub App              Repository and PR
                                                  integration

  Containers              Docker Compose          Local reproducibility

  Tests                   Jest, Pytest,           Unit, integration, and
                          Playwright              E2E coverage
  -----------------------------------------------------------------------

### 10.1 Queue decision: BullMQ before Kafka

BullMQ and Redis are sufficient for the initial pull-request workload.
Kafka is not justified merely because the product processes events.

Revisit Kafka if replayable event streams, multiple independent
consumers, sustained throughput, or stronger event-streaming
requirements emerge.

### 10.2 Modular monolith before microservices

Keep business modules inside NestJS and maintain a separate Python AI
service because it has a distinct runtime and dependency ecosystem. Do
not extract every module into a separate deployable service until there
is a measured need.

### 10.3 Local model provider abstraction

Define an application-level model interface for: - Chat/completion. -
Structured output. - Embeddings. - Model metadata and health.

Ollama is the first adapter. A future local runtime can implement the
same interface without changing agent business logic.

## 11. Local development environment

### 11.1 Developer machine

-   Operating system: Windows.
-   Memory: 16 GB RAM.
-   GPU: No dedicated GPU.
-   Docker experience: Yes.
-   Recommended development workflow: Docker Desktop with WSL2 backend.
-   Keep source files in the WSL Linux filesystem for better container
    file performance.
-   Run Ollama natively on Windows initially.

### 11.2 Runtime versions and tools

  Tool                     Proposed baseline
  ------------------------ -----------------------------------
  Node.js                  22 LTS
  Package manager          pnpm
  Python                   3.12
  Python package manager   uv
  Docker                   Docker Desktop with WSL2
  Database                 PostgreSQL 17 with pgvector image
  Queue                    Redis
  Model runtime            Ollama
  IDE                      Visual Studio Code
  Git                      Current supported release

Verify supported versions and image tags during setup; pin exact
versions in repository configuration.

### 11.3 Ollama setup

PowerShell:

``` powershell
ollama --version
ollama pull qwen2.5-coder:3b-instruct
ollama pull nomic-embed-text
ollama list
```

Initial resource controls:

``` powershell
[Environment]::SetEnvironmentVariable(
  "OLLAMA_NUM_PARALLEL", "1", "User"
)
[Environment]::SetEnvironmentVariable(
  "OLLAMA_MAX_LOADED_MODELS", "1", "User"
)
[Environment]::SetEnvironmentVariable(
  "OLLAMA_KEEP_ALIVE", "5m", "User"
)
```

Restart Ollama after setting variables.

Model tag availability and exact memory requirements must be verified at
setup time. The 3B model is a starting point, not a proven quality
baseline.

### 11.4 WSL2 memory budget

Proposed initial `%UserProfile%\.wslconfig`:

``` ini
[wsl2]
memory=6GB
processors=4
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```

Apply changes with:

``` powershell
wsl --shutdown
```

This is an initial resource budget for a 16 GB machine. Adjust based on
measured Docker, WSL, Ollama, and Windows memory usage.

### 11.5 Initial local ports

  Service              Port
  ----------------- -------
  Next.js              3000
  NestJS               3001
  FastAPI              8000
  PostgreSQL           5432
  Redis                6379
  Ollama              11434
  Grafana (later)      3002

Expose only required ports and bind local-only services to localhost
where possible.

## 12. Repository structure

Proposed monorepo:

``` text
codeguard-ai/
├── apps/
│   ├── web/
│   └── api/
├── services/
│   └── ai-service/
├── workers/
│   ├── analysis-worker/
│   └── indexing-worker/
├── packages/
│   ├── shared-types/
│   ├── api-contracts/
│   └── config/
├── ai/
│   ├── agents/
│   │   ├── context/
│   │   ├── bug-detection/
│   │   ├── security/
│   │   ├── performance/
│   │   ├── architecture/
│   │   └── code-quality/
│   ├── orchestrator/
│   ├── prompts/
│   ├── models/
│   ├── evaluation/
│   └── schemas/
├── infrastructure/
│   ├── docker/
│   ├── postgres/
│   ├── redis/
│   └── monitoring/
├── docs/
│   ├── business/
│   ├── architecture/
│   ├── decisions/
│   ├── database/
│   ├── api/
│   └── operations/
├── tests/
│   ├── integration/
│   ├── e2e/
│   ├── performance/
│   └── ai-evaluation/
├── docker-compose.yml
├── pnpm-workspace.yaml
├── package.json
├── .env.example
├── README.md
└── CONTRIBUTING.md
```

The first implementation may create only the folders needed for Issue
#1. Avoid adding empty scaffolding for every future component.

## 13. Security architecture

### 13.1 Platform security

-   Local authentication with secure password hashing.
-   Role-based access control.
-   Organization-scoped authorization.
-   GitHub App with minimum permissions.
-   Webhook HMAC signature verification.
-   Secrets excluded from Git and logs.
-   Restrict local ports and service exposure.
-   Service-to-service authentication for internal APIs.

### 13.2 Repository and agent isolation

-   Treat source files and repository documentation as untrusted data.
-   No arbitrary shell access for agents.
-   No unrestricted filesystem tools.
-   Use a temporary checkout workspace with path validation.
-   Avoid executing code from analyzed repositories.
-   Apply file exclusions and secret filtering before indexing.
-   Keep AI analysis read-only in MVP.
-   Clean temporary workspaces after analysis.

### 13.3 GitHub integration nuance

GitHub is an external service even though AI inference is local. Local
development webhook delivery requires a secure tunnel or a
polling/local-repository alternative. A fully air-gapped mode would
require a local source-control integration rather than GitHub cloud.

## 14. Reliability, observability, and performance

### 14.1 Reliability

-   Webhook event idempotency.
-   Queue job retry with bounded exponential backoff.
-   Dead-letter or failed-job inspection.
-   Job timeout and cancellation.
-   Analysis status lifecycle.
-   Safe reprocessing.
-   Duplicate GitHub comment prevention.

### 14.2 Observability

Use structured logs and correlation identifiers across: - Webhook
event. - Pull request. - Analysis run. - Queue job. - AI workflow. -
Agent execution. - GitHub publication.

Capture: - Queue wait time. - Static analysis duration. - Model
inference duration. - Total PR analysis duration. - Model and prompt
versions. - Failure and retry counts. - Resource use and inference cost
proxy where available.

### 14.3 Performance strategy

The local CPU-only machine is a constrained development environment.

-   Start with one AI inference request at a time.
-   Limit context size.
-   Analyze changed files and relevant context rather than the entire
    repository.
-   Cache repository metadata and reusable indexes.
-   Run static checks before expensive model calls.
-   Apply timeouts to all external and model operations.
-   Benchmark with representative PR sizes.

Do not claim a two-minute SLA until measured on a defined hardware
profile.

## 15. Architecture Decision Records

### ADR-001 --- Modular application before microservices

**Decision:** Use a modular NestJS application, a separate Python AI
service, and background workers.

**Rationale:** The AI service has a distinct runtime; other modules do
not yet need independent deployment.

**Revisit when:** Independent scaling, deployment ownership, or
isolation requirements justify service extraction.

### ADR-002 --- Node.js and Python separation

**Decision:** NestJS owns product APIs and business workflows; Python
owns AI orchestration and model interaction.

**Rationale:** Leverage the Node ecosystem for product APIs and Python
ecosystem for AI.

### ADR-003 --- PostgreSQL with pgvector

**Decision:** Use PostgreSQL for transactional data and vector search.

**Rationale:** Reduce operational complexity for MVP and keep relational
metadata near embeddings.

**Revisit when:** Vector workload, scale, or retrieval needs exceed this
design.

### ADR-004 --- Ollama local inference

**Decision:** Use Ollama as the initial local model runtime.

**Rationale:** Straightforward local setup and HTTP interface.

**Revisit when:** Benchmarking shows a different local runtime is better
for latency, throughput, or hardware utilization.

### ADR-005 --- LangGraph orchestration

**Decision:** Use LangGraph for explicit agent workflow state and
orchestration.

**Rationale:** More controllable workflow than unstructured
agent-to-agent interaction.

### ADR-006 --- Redis and BullMQ

**Decision:** Use Redis and BullMQ for background processing.

**Rationale:** Sufficient for initial workload and simpler than Kafka.

### ADR-007 --- Hybrid analysis

**Decision:** Combine deterministic static analysis with AI reasoning.

**Rationale:** Deterministic tools provide reproducible findings; AI
adds context and explanations.

### ADR-008 --- Docker Compose local deployment

**Decision:** Use Docker Compose for application and data services; run
Ollama natively on Windows initially.

**Rationale:** Avoid unnecessary GPU/container and memory complexity on
the current machine.

### ADR-009 --- GitHub App integration

**Decision:** Use GitHub App authentication and webhooks.

**Rationale:** Repository-scoped installation and fine-grained
permissions.

### ADR-010 --- Preserve analysis history

**Decision:** Store each analysis as a separate run with model and
prompt version metadata.

**Rationale:** Auditability, comparison, and AI evaluation.

## 16. Implementation roadmap

Estimated planning range: 12--16 weeks at approximately 30--45 minutes
of focused work per day, with additional time for debugging, setup, and
learning. This is an estimate, not a commitment.

  -----------------------------------------------------------------------
  Epic                    Scope                   Deliverable
  ----------------------- ----------------------- -----------------------
  1\. Foundation          Monorepo, tooling,      Clean repository
                          workspace, conventions  foundation

  2\. Core platform       Users, organizations,   Working application
                          repositories,           shell
                          migrations              

  3\. GitHub integration  App, webhook, PR        PR events create jobs
                          retrieval, queue        

  4\. Static analysis     ESLint, Semgrep,        Deterministic review
                          normalized findings     engine

  5\. Local AI engine     Ollama, model           Local AI PR analysis
                          abstraction, context,   
                          bug/security agents     

  6\. Advanced agents     Performance,            Multi-agent review
                          architecture, quality,  
                          aggregation             

  7\. Dashboard and       Findings, feedback,     Developer-facing
  policies                policy, metrics         platform

  8\. Readiness           Evaluation, security,   Demonstrable product
                          tests, monitoring, docs 
  -----------------------------------------------------------------------

### 16.1 First ten GitHub issues

  -----------------------------------------------------------------------
  Issue                   Title                   Priority
  ----------------------- ----------------------- -----------------------
  #1                      Initialize CodeGuard AI Critical
                          monorepo and            
                          engineering conventions 

  #2                      Configure TypeScript    Critical
                          and Python development  
                          environments            

  #3                      Create Docker Compose   Critical
                          infrastructure          

  #4                      Configure PostgreSQL    Critical
                          and Redis               

  #5                      Create NestJS           High
                          application foundation  

  #6                      Create FastAPI          High
                          application foundation  

  #7                      Create Next.js frontend High
                          foundation              

  #8                      Configure GitHub        High
                          Actions CI              

  #9                      Implement initial       High
                          database migrations     

  #10                     Create health-check     High
                          endpoints for all       
                          services                
  -----------------------------------------------------------------------

## 17. Issue #1 --- Detailed implementation definition

**Title:** Initialize CodeGuard AI monorepo and engineering conventions

**Description:** Establish the initial monorepo structure for CodeGuard
AI, a local-first AI-powered engineering quality platform. The
repository must support multiple applications and services using
TypeScript and Python while maintaining consistent engineering
standards.

**Scope** 1. Initialize Git repository. 2. Configure pnpm workspace. 3.
Establish frontend, backend, and AI service directories. 4. Configure
TypeScript and Python development environments. 5. Establish shared
configuration and contracts directories. 6. Add `.gitignore`,
`.editorconfig`, and `.env.example`. 7. Create initial project README.
8. Document repository conventions.

**Initial structure**

``` text
codeguard-ai/
├── apps/
│   ├── web/
│   └── api/
├── services/
│   └── ai-service/
├── packages/
│   ├── contracts/
│   └── config/
├── docs/
│   └── architecture/
├── infrastructure/
├── tests/
├── .github/
├── pnpm-workspace.yaml
├── package.json
├── .gitignore
├── .editorconfig
├── .env.example
└── README.md
```

**Acceptance criteria** - Monorepo initialized and pushed to GitHub. -
pnpm workspace configured. - Node.js and Python version requirements
documented. - Application and service directories established. -
Environment configuration documented. - No secrets or local model files
committed. - README includes setup instructions. - A developer can clone
and initialize the workspace using the documented instructions.

**Out of scope** - Business API implementation. - Database
implementation. - AI agent implementation. - GitHub webhook integration.

**Definition of done:** A developer can clone the repository and
successfully initialize the development workspace using the documented
instructions.

## 18. Open design questions

1.  Confirm exact Ollama model tag and quantization on the target
    machine.
2.  Benchmark embedding model and choose vector dimension.
3.  Select local authentication approach.
4.  Finalize GitHub App permissions and webhook tunnel workflow.
5.  Confirm Semgrep rulesets and CodeQL use/licensing constraints.
6.  Define retention defaults and repository exclusions.
7.  Finalize finding fingerprint strategy.
8.  Define benchmark repository and representative PR sizes.

## 19. References and implementation documentation

-   Ollama Windows: https://ollama.com/download/windows
-   Ollama Qwen coder models: https://ollama.com/library/qwen2.5-coder
-   Ollama embeddings: https://ollama.com/library/nomic-embed-text
-   Docker Desktop:
    https://docs.docker.com/desktop/setup/install/windows-install/
-   WSL configuration:
    https://learn.microsoft.com/windows/wsl/wsl-config
-   NestJS: https://docs.nestjs.com/
-   FastAPI: https://fastapi.tiangolo.com/
-   LangGraph: https://docs.langchain.com/oss/python/langgraph/overview
-   PostgreSQL: https://www.postgresql.org/docs/
-   pgvector: https://github.com/pgvector/pgvector
-   BullMQ: https://docs.bullmq.io/
-   Semgrep: https://semgrep.dev/docs/
-   Tree-sitter: https://tree-sitter.github.io/tree-sitter/
-   GitHub Apps: https://docs.github.com/apps
-   OpenTelemetry: https://opentelemetry.io/docs/

------------------------------------------------------------------------

**Document ownership:** CodeGuard AI project\
**Status:** Architecture baseline to be refined through implementation
and ADR updates.\
**Next milestone:** Complete GitHub Issue #1 --- monorepo foundation.
