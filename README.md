# CodeGaurd AI

CodeGaurd AI is a local-first engineering quality platform that combines
deterministic static analysis with privacy-preserving, locally hosted AI
review. It integrates with GitHub pull requests to identify potential bugs,
security risks, and code-quality issues while keeping source code, embeddings,
analysis records, and model inference in the locally operated environment.

AI findings are advisory. Human reviewers remain accountable for code,
security, and merge decisions.

## Product goals

- Reduce repetitive manual review effort without replacing human review.
- Improve detection of defects and security risks with repository context.
- Provide evidence-backed findings with actionable remediation guidance.
- Make analysis history, quality trends, and operational status visible.
- Keep local-only deployments free from hosted LLM inference.

Initial success hypotheses are a 30% reduction in comparable review time, a
25% reduction in review iterations, at least 70% accepted or confirmed
recommendations, fewer than 15% incorrect or irrelevant findings, and 95% of
standard pull-request analyses completing within two minutes. These targets
must be validated against a documented benchmark, model version, hardware
profile, and sample size before being treated as results.

## MVP scope

The MVP focuses on:

1. GitHub App repository registration and signed pull-request webhooks.
2. Idempotent background analysis of TypeScript and JavaScript changes.
3. ESLint and Semgrep deterministic checks with normalized findings.
4. Local Ollama-based AI review for bug and security risks.
5. Local repository indexing and context retrieval with PostgreSQL and
   pgvector.
6. Finding validation, severity mapping, deduplication, and audit history.
7. Pull-request summaries and valid inline comments, with retry-safe
   publication.
8. A basic dashboard for repositories, pull requests, findings, and analysis
   history.
9. Health checks, safe retries, retention controls, and local Docker setup.

Automatic merging, autonomous code changes, arbitrary AI shell execution,
hosted AI inference, and additional source-control providers are outside the
MVP.

## Architecture

The initial design is a modular application rather than a collection of
premature microservices:

- **Web:** Next.js and React dashboard.
- **Core API:** NestJS/Node.js for authentication, repositories, policies,
  reporting, and GitHub webhooks.
- **Analysis worker:** BullMQ over Redis for asynchronous, bounded processing.
- **AI service:** Python/FastAPI with LangGraph orchestration and a provider
  abstraction.
- **Analyzers:** ESLint, Semgrep, and Tree-sitter where structural analysis is
  useful.
- **Model runtime:** Ollama for local LLM and embedding inference.
- **Data:** PostgreSQL with pgvector for transactional records and retrieval.
- **Deployment:** Docker Compose for local services; Ollama initially runs
  natively on Windows/WSL2.

See [business requirements](docs/business/business-requirements.md) for the
product baseline and [architecture design](docs/architechture/architecture-design.md)
for runtime boundaries, data design, API contracts, security controls, and
implementation decisions.

## Pull-request workflow

1. GitHub sends an opened, reopened, or synchronized pull-request webhook.
2. The API validates the signature and event idempotency key.
3. A background job retrieves metadata, the diff, and changed files.
4. Static analyzers run against supported files.
5. The AI service retrieves relevant repository context and performs local
   bug/security review.
6. Findings are validated, normalized, deduplicated, and policy-evaluated.
7. Results are persisted and published as one coherent pull-request summary,
   with inline comments only for valid changed-line locations.
8. The dashboard exposes status, findings, evidence, model metadata, and
   analysis history.

## Delivery roadmap

| Release | Focus |
| --- | --- |
| MVP | GitHub integration, TypeScript/JavaScript analysis, local bug/security review, PR feedback, and basic dashboard |
| Release 2 | Additional agents, configurable policies, reviewer feedback, evaluation, and organization analytics |
| Release 3 | Human-approved remediation, more SCM providers, architecture mapping, and technical-debt insights |

The implementation backlog is managed in the **CodeGaurd AI — MVP Delivery**
GitHub Project. Work is organized into Foundation, Core Platform, GitHub
Integration, Static Analysis, Local AI, Findings and Feedback, Dashboard and
Policy, and Readiness epics. Each story includes acceptance criteria and links
back to the relevant business or non-functional requirement.

## Local development prerequisites

- Windows with Docker Desktop/WSL2 and approximately 16 GB RAM.
- Node.js and pnpm for the web, API, worker, and shared packages.
- Python for the AI service.
- Docker Compose, PostgreSQL with pgvector, and Redis.
- Ollama with a small quantized coder model and embedding model.
- A GitHub App configured with the minimum permissions required for repository
  metadata, pull requests, and webhooks.

Detailed setup commands will be added as the application skeleton is
implemented. Do not commit credentials, model files, repository checkouts, or
generated embeddings. Use environment variables and the eventual
`.env.example` contract.

## Repository documentation

- [Business and product requirements](docs/business/business-requirements.md)
- [System design and architecture](docs/architechture/architecture-design.md)

## Project conventions

- Treat repository content as untrusted input and never as system
  instructions.
- Keep AI agents unable to execute arbitrary shell commands or access files
  outside the active repository workspace.
- Record model, prompt, agent, and benchmark versions with analysis results.
- Preserve deterministic evidence and clearly label AI findings as advisory.
- Make webhook, job, and comment publication paths idempotent.
- Add automated tests for core workflows, invalid input, retries, and security
  boundaries.
