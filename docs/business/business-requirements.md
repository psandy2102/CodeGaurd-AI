# CodeGuard AI --- Business & Product Requirements

**Document type:** Business Requirements Document (BRD)\
**Version:** 1.0\
**Status:** Proposed baseline\
**Date:** 31 October 2026\
**Product:** CodeGuard AI\
**Deployment philosophy:** Local-first, privacy-oriented AI engineering
quality platform

------------------------------------------------------------------------

## 1. Executive summary

CodeGuard AI is an AI-assisted engineering quality platform that
integrates with source-code repositories and pull-request workflows. It
combines deterministic static analysis with local AI reasoning and
repository context to identify potential defects, security concerns,
performance risks, maintainability issues, and architectural concerns.

The platform provides developers with actionable feedback close to the
point of change. It gives engineering managers and technical leaders
visibility into quality trends, recurring issues, policy compliance, and
the usefulness of AI-generated findings.

The initial product is designed for local deployment. Source code,
embeddings, analysis records, and model inference remain within the
locally operated environment. GitHub may remain an external
source-control system for repository and pull-request integration.

**Business objective:** Improve code-review effectiveness and
engineering quality while reducing repetitive manual review effort. AI
findings supplement human review and do not replace engineering
accountability.

## 2. Product vision

Create a practical engineering quality assistant that helps software
teams identify risks earlier, apply consistent standards, and make
informed decisions about code changes.

### 2.1 Target audiences

-   **Software developers:** Receive timely, understandable feedback on
    proposed code changes.
-   **Senior developers and reviewers:** Focus review effort on
    substantive risks rather than repetitive checks.
-   **Engineering managers:** Understand quality trends, recurring
    issues, and workflow performance.
-   **Staff engineers and technical architects:** Identify architectural
    risks and promote consistent engineering standards.
-   **Security engineers:** Review security-related findings and policy
    compliance.
-   **Platform administrators:** Configure repositories, users,
    policies, integrations, and retention.

## 3. Business problem statement

Engineering teams face several recurring challenges:

  -----------------------------------------------------------------------
  ID                      Problem                 Potential business
                                                  impact
  ----------------------- ----------------------- -----------------------
  BP-01                   Manual review effort    Slower feedback and
                          varies by reviewer and  inconsistent reviews
                          workload                

  BP-02                   Static analysis may     Relevant defects or
                          lack                    risks may be missed
                          application-specific    
                          context                 

  BP-03                   Architectural concerns  Technical debt and
                          span multiple files and costly rework
                          modules                 

  BP-04                   Security and            Higher remediation
                          performance problems    effort
                          may be identified late  

  BP-05                   Quality information is  Limited visibility for
                          spread across tools     technical leaders

  BP-06                   AI review tools can     Reduced developer trust
                          produce unsupported or  
                          repetitive findings     
  -----------------------------------------------------------------------

These are problem hypotheses to validate with users and pilot
measurements, not claims that every team experiences the same impact.

## 4. Business objectives and proposed success measures

Targets below are initial hypotheses. Establish a baseline during the
pilot and revise targets using measured results.

  ------------------------------------------------------------------------
  Objective             Measure                             Initial target
  --------------------- --------------------- ----------------------------
  Reduce review effort  Average human review                 30% reduction
                        time for comparable   
                        PRs                   

  Improve useful defect Valid seeded defects               Improve against
  detection             detected in benchmark         static-analysis-only
                                                                  baseline

  Reduce review         Average review cycles                25% reduction
  iterations            per PR                

  Improve               Recommendations                      70% or higher
  recommendation        accepted or confirmed 
  usefulness            by reviewers          

  Control false         Findings judged                          Below 15%
  positives             incorrect or          
                        irrelevant            

  Maintain              Standard PR analysis         95% within 2 minutes,
  responsiveness        duration                subject to hardware and PR
                                                                      size
  ------------------------------------------------------------------------

Do not claim these outcomes until they are measured. Report benchmark
methodology, sample size, model version, and hardware alongside results.

## 5. Scope

### 5.1 MVP scope

1.  GitHub repository integration.
2.  Pull-request event intake and validation.
3.  Retrieval of pull-request metadata and changed files.
4.  Initial language support for TypeScript/JavaScript.
5.  Deterministic analysis using ESLint and Semgrep.
6.  Local AI-assisted bug and security review.
7.  Repository-context retrieval for relevant files and symbols.
8.  Structured finding validation, normalization, and deduplication.
9.  Review summary and inline feedback on GitHub.
10. Basic dashboard for repositories, analyses, and findings.
11. Analysis history and operational status.
12. Local Docker-based deployment and setup documentation.

### 5.2 Later releases

**Release 2** - Performance, architecture, and maintainability agents. -
Repository engineering policies and configurable severity actions. -
Expanded language support. - Developer feedback and AI evaluation
workflows. - Organization-level quality analytics.

**Release 3** - Optional remediation proposals with explicit human
approval. - Additional source-control providers. - Advanced
technical-debt trend analysis. - More sophisticated repository
dependency and architecture mapping. - Optional enterprise deployment
and governance features.

### 5.3 Out of scope for MVP

-   Automatic pull-request merging.
-   Autonomous code modification or commit creation.
-   Arbitrary shell execution by AI agents.
-   Guaranteed detection of all defects or vulnerabilities.
-   Replacement of human code review or security sign-off.
-   Cloud-hosted AI inference.
-   Multi-region hosting.
-   Full enterprise identity federation.

## 6. Functional requirements

Priority definitions: **Must** = required for MVP; **Should** = planned
enhancement; **Could** = future capability.

### FR-01 --- Repository integration and management (Must)

-   Register and remove supported GitHub repositories.
-   Authenticate through a GitHub App using minimum required
    permissions.
-   Receive pull-request webhook events.
-   Validate webhook signatures before processing.
-   Identify repository, pull request, changed files, base SHA, and head
    SHA.
-   Support opened, reopened, and synchronized pull-request events.
-   Allow repository analysis to be enabled or disabled.
-   Store repository metadata without storing unnecessary credentials.

**Acceptance criteria** - A configured repository can generate an
analysis job from a supported pull-request event. - Invalid webhook
signatures are rejected. - Duplicate delivery of the same event does not
create duplicate analysis work.

### FR-02 --- Automated code analysis (Must)

-   Analyze changed TypeScript/JavaScript files.
-   Run configured static-analysis tools.
-   Detect supported code-quality and security rules.
-   Identify potential bugs, unsafe patterns, and maintainability
    concerns.
-   Normalize tool output into a common finding format.
-   Preserve rule identifiers and evidence from deterministic tools.
-   Support severity categories: Critical, High, Medium, Low, and
    Informational.

Every finding should include, where applicable: - Finding category and
severity. - Title and explanation. - Repository-relative file path and
line or range. - Evidence or code reference. - Rule or agent
identifier. - Confidence estimate for AI-generated findings. -
Recommended remediation. - Source classification: deterministic tool,
AI, or combined.

**Acceptance criteria** - Static-analysis findings are persisted and
linked to the correct analysis run. - Findings reference valid
repository-relative paths. - Unsupported or malformed tool output is
handled safely.

### FR-03 --- AI-powered contextual review (Must for MVP bug/security subset)

-   Use a local model runtime for inference.
-   Analyze changed code with relevant repository context.
-   Identify potential logic defects and security risks.
-   Explain why a finding may matter.
-   Suggest an actionable remediation.
-   Distinguish confirmed deterministic rule violations from potential
    AI findings.
-   Return structured output conforming to a validated schema.
-   Record model, prompt, and agent versions.

**Acceptance criteria** - AI analysis can be run without sending source
code to a hosted LLM provider. - Invalid model output is rejected or
safely retried. - AI findings are not treated as proven defects solely
because the model reported them.

### FR-04 --- Repository context and indexing (Must, basic form)

-   Identify supported source files.
-   Exclude generated files, dependencies, binaries, and configured
    paths.
-   Parse source structure where supported.
-   Split code into meaningful chunks.
-   Generate embeddings using a local embedding model.
-   Store embeddings and metadata in PostgreSQL with pgvector.
-   Retrieve relevant context for a changed file or symbol.
-   Re-index changed content incrementally where practical.
-   Support deletion of indexed repository content.

**Acceptance criteria** - Context retrieval returns repository-relative
source references. - Indexing is repeatable and does not create
uncontrolled duplicates. - Configured exclusions are respected.

### FR-05 --- Findings aggregation and validation (Must)

-   Normalize findings from static tools and AI agents.
-   Deduplicate overlapping findings.
-   Validate file paths and line references against the analyzed commit.
-   Preserve evidence and source attribution.
-   Apply consistent severity mapping.
-   Calculate a stable finding fingerprint where possible.
-   Retain findings per analysis run for historical comparison.

**Acceptance criteria** - Duplicate webhook processing does not create
duplicate review publication. - A finding cannot reference a path
outside the repository workspace. - Findings from separate analysis runs
remain auditable.

### FR-06 --- Pull-request feedback (Must)

-   Publish a review summary to the GitHub pull request.
-   Publish inline comments only when a valid changed-line location
    exists.
-   Group findings by severity and category.
-   Link to the local CodeGuard analysis view where accessible.
-   Avoid repeated comments when the same analysis event is retried.
-   Make clear that AI findings are advisory.

**Acceptance criteria** - A completed analysis publishes one coherent
summary. - Invalid line locations fall back to summary-only reporting. -
Retry behavior does not spam duplicate comments.

### FR-07 --- Engineering standards and policy management (Should)

-   Define organization-wide and repository-specific policies.
-   Configure severity thresholds and actions.
-   Configure rules by category and tool.
-   Support exceptions with rationale, owner, and expiry where
    appropriate.
-   Version policy changes.
-   Evaluate findings against the active policy.

For MVP, policy evaluation may be limited to a small set of hard-coded
or configuration-driven rules.

### FR-08 --- Engineering quality dashboard (Must, basic form)

-   List configured repositories.
-   Show pull requests and analysis status.
-   Display findings by severity and category.
-   Show analysis duration and model information.
-   Provide analysis history.
-   Filter findings by repository, pull request, category, and severity.

Later releases may add organization comparisons, trend charts, and
team-level metrics.

### FR-09 --- AI evaluation and feedback (Should)

-   Allow reviewers to mark findings as useful, incorrect, duplicate, or
    not applicable.
-   Record optional explanations.
-   Maintain a curated benchmark repository containing known defects and
    security issues.
-   Measure precision, recall on seeded cases, severity classification,
    and false-positive rate.
-   Compare model and prompt versions.
-   Record inference duration and resource usage where available.

### FR-10 --- Administration, audit, and operations (Must basic controls; advanced features Should)

-   Provide basic user authentication and role-based access.
-   Record administrative actions and analysis lifecycle events.
-   Track job failures and retry outcomes.
-   Expose health checks for application services.
-   Support configurable data retention and deletion.
-   Avoid logging secrets or unnecessary source code.

## 7. User workflows

### 7.1 Pull-request analysis workflow

1.  Developer opens or updates a pull request.
2.  GitHub sends a webhook.
3.  CodeGuard validates the signature and event.
4.  The platform checks event idempotency and records the request.
5.  A background job is enqueued.
6.  The worker retrieves PR metadata and changed files.
7.  Static analysis executes.
8.  Repository context is retrieved or indexed.
9.  Local AI agents analyze relevant changes.
10. The aggregator validates, normalizes, and deduplicates findings.
11. The policy engine evaluates configured rules.
12. Findings and analysis metadata are persisted.
13. A review summary and eligible inline comments are published.
14. The dashboard displays the analysis.
15. Developer feedback may be captured for evaluation.

### 7.2 Repository onboarding workflow

1.  Authorized user registers a repository.
2.  GitHub App permissions are checked.
3.  Repository metadata and default branch are stored.
4.  Analysis configuration and exclusions are applied.
5.  Optional initial indexing begins.
6.  Repository becomes available for pull-request analysis.

## 8. Non-functional requirements

  -----------------------------------------------------------------------
  ID                      Area                    Requirement
  ----------------------- ----------------------- -----------------------
  NFR-01                  Performance             Target 95% of standard
                                                  PR analyses within 2
                                                  minutes; validate
                                                  against defined
                                                  hardware and PR size

  NFR-02                  Scalability             Queue-based processing
                                                  and bounded concurrency

  NFR-03                  Availability            Local service health
                                                  checks and
                                                  restart/recovery
                                                  behavior

  NFR-04                  Security                Validate webhooks,
                                                  enforce authorization,
                                                  protect credentials

  NFR-05                  Privacy                 No source-code
                                                  transmission to hosted
                                                  AI providers in
                                                  local-only mode

  NFR-06                  Reliability             Retry recoverable jobs
                                                  and make processing
                                                  idempotent

  NFR-07                  Extensibility           Provider abstraction
                                                  for model runtime and
                                                  analyzers

  NFR-08                  Observability           Structured logs,
                                                  metrics, traces, and
                                                  correlation IDs

  NFR-09                  Maintainability         Modular code, automated
                                                  tests, documented
                                                  contracts

  NFR-10                  Tenant isolation        Organization-scoped
                                                  access and data
                                                  ownership

  NFR-11                  Resource control        Configurable model
                                                  concurrency, queue
                                                  limits, and analysis
                                                  timeouts

  NFR-12                  Data lifecycle          Explicit retention and
                                                  deletion for source
                                                  checkouts, chunks,
                                                  embeddings, and
                                                  analysis records
  -----------------------------------------------------------------------

## 9. Security and privacy requirements

-   Treat repository content, comments, documentation, and diffs as
    untrusted input.
-   Never interpret source-code content as system instructions.
-   Do not grant AI agents arbitrary shell execution or unrestricted
    filesystem access.
-   Use temporary, isolated working directories for repository
    checkouts.
-   Restrict file access to the active repository workspace.
-   Exclude secrets and sensitive files from indexing and prompts where
    feasible.
-   Use minimum GitHub App permissions.
-   Verify webhook HMAC signatures.
-   Keep tokens and secrets out of source control and logs.
-   Bind local service ports to localhost unless external access is
    explicitly needed.
-   Use explicit retention and deletion controls.
-   Make the local-only AI provider policy enforceable through
    configuration and tests.

## 10. Risks and mitigations

  -----------------------------------------------------------------------
  Risk                                Mitigation
  ----------------------------------- -----------------------------------
  AI hallucinations                   Evidence requirements, confidence
                                      calibration, reviewer feedback

  False positives                     Benchmark evaluation and
                                      configurable thresholds

  Prompt injection in repository      Treat retrieved content as data;
  content                             restrict agent tools

  CPU inference latency               Small quantized model, bounded
                                      concurrency, asynchronous jobs

  Limited RAM                         Load one model at a time; control
                                      Docker/WSL memory

  GitHub rate limits                  Backoff, retry, and
                                      rate-limit-aware scheduling

  Large repositories                  Incremental indexing and context
                                      budgets

  Duplicate webhook delivery          Event idempotency and publication
                                      deduplication

  Source-code exposure                Local inference, temporary
                                      workspaces, retention controls

  Model regression                    Versioned benchmark and
                                      prompt/model evaluation
  -----------------------------------------------------------------------

## 11. MVP acceptance criteria

The MVP is ready for a portfolio demonstration when:

1.  A user can register a GitHub repository.
2.  A pull-request event is received and signature-validated.
3.  Duplicate events are handled idempotently.
4.  The platform retrieves the relevant PR diff.
5.  Static analysis runs against supported files.
6.  Local AI analysis runs without hosted model inference.
7.  Findings are validated and stored.
8.  The GitHub PR receives a summary and valid inline comments.
9.  The dashboard displays analysis results and history.
10. Failed jobs can be retried safely.
11. Automated tests cover core workflows and failure cases.
12. Setup, architecture, limitations, and evaluation methodology are
    documented.

## 12. Assumptions and constraints

-   Initial source-control integration is GitHub.
-   Initial language support is TypeScript/JavaScript.
-   The local development environment is Windows with 16 GB RAM and no
    dedicated GPU.
-   Ollama is the initial local inference runtime.
-   A small quantized coder model will be benchmarked for CPU inference.
-   PostgreSQL with pgvector is the initial relational and vector store.
-   Redis with BullMQ is the initial job queue.
-   Docker Compose is the initial deployment approach.
-   GitHub remains an external service; local model inference and
    analysis data remain local.
-   The initial product is single-machine oriented, though the data
    model will be organization-aware.

## 13. Proposed release roadmap

  -----------------------------------------------------------------------
  Release                             Main capabilities
  ----------------------------------- -----------------------------------
  MVP                                 GitHub, TypeScript, static
                                      analysis, local bug/security
                                      review, PR feedback, basic
                                      dashboard

  Release 2                           Additional agents, repository
                                      policies, feedback, evaluation,
                                      analytics

  Release 3                           Human-approved remediation, more
                                      SCM providers, advanced
                                      architecture and technical-debt
                                      insights
  -----------------------------------------------------------------------

## 14. Open decisions to resolve during implementation

-   Exact local coder model and quantization after hardware
    benchmarking.
-   Exact embedding model and vector dimension.
-   Authentication approach for the first local deployment.
-   GitHub App permissions and local webhook tunnel setup.
-   Supported Semgrep rulesets and CodeQL usage/licensing constraints.
-   Retention periods and repository exclusion defaults.
-   Finding fingerprint strategy and cross-run identity behavior.
-   Definition of the standard PR benchmark used for performance
    targets.

------------------------------------------------------------------------

**Document ownership:** CodeGuard AI project\
**Next design artifact:** System architecture, component design, data
model, and implementation plan.
