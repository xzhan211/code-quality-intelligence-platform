# Code Quality Intelligence Platform - Requirements and Architecture

## 1. Purpose

This project builds an internal code quality assessment application for company engineering teams. The first version is a demo-oriented platform that evaluates a repository on demand, generates structured code quality findings, uses LLM-based engineering judgment to synthesize risks and recommendations, and presents the result through a simple dashboard.

The application is not intended to replace existing enterprise scanners such as SonarQube, CodeQL, Checkmarx, Snyk, or Veracode. Instead, it provides an intelligence layer on top of existing/static analysis evidence and source code context.

The first version focuses on proving the core value:

- Upload or provide a code repository.
- Run code quality analysis for Java and Python repositories.
- Build an evidence package from static metrics and selected source snippets.
- Use LLM to produce evidence-grounded engineering assessment.
- Evaluate the LLM output before accepting it.
- Generate a dashboard/report for engineers, managers, and architecture reviewers.

## 2. Product Positioning

### 2.1 What This Platform Is

The platform is an internal Code Quality Intelligence system.

It provides:

- Repo-level code quality assessment.
- Module/file/function-level hotspots.
- Complexity, maintainability, duplication, readability, and refactoring signals.
- LLM-based synthesis of engineering risks.
- Evidence-grounded recommendations.
- Report confidence and evaluation metadata.
- A foundation for future Bitbucket integration and scheduled scans.

### 2.2 What This Platform Is Not

The demo version is not:

- A real-time PR review bot.
- A release gate.
- A replacement for SonarQube, CodeQL, Checkmarx, Snyk, or Veracode.
- A security/compliance approval system.
- An automatic refactoring agent.
- A full enterprise governance platform.

## 3. Target Users

The dashboard should be readable by multiple roles:

### Engineers

Engineers need actionable details:

- Which files/functions are problematic.
- Why they are problematic.
- What evidence supports the finding.
- How to refactor or improve the code.
- Estimated effort and suggested priority.

### Engineering Managers

Managers need a higher-level view:

- Overall repo quality score.
- Risk level.
- Quality trend, when history is available.
- Technical debt concentration.
- Recommended refactor backlog.
- Whether the repo needs planned engineering investment.

### Architecture Reviewers

Architecture reviewers need structural insights:

- Module boundaries.
- Coupling and responsibility concentration.
- Domain model duplication.
- Service layer design smells.
- Risky architecture patterns.
- Evidence supporting major refactor recommendations.

Security/compliance is explicitly out of scope for the first demo, but the architecture should leave room for future security/compliance feature integration.

## 4. Requirements

## 4.1 Functional Requirements

### FR-1: Repository Input

The first demo version must support local repository upload.

Supported input:

- `.zip`
- `.tar.gz`

The user should provide:

- Tenant name or team name.
- Repo name.
- Optional branch name.
- Optional commit SHA.
- Optional description.

Future versions should support:

- Bitbucket archive download.
- Bitbucket git clone.
- Scheduled scans from Bitbucket.

### FR-2: Supported Languages

The demo should focus on:

- Java
- Python

Language detection should be automatic when possible, but manual override is acceptable for the demo.

### FR-3: Static Metrics Extraction

The platform should extract objective quality signals before invoking the LLM.

Minimum demo metrics:

- File size.
- Function/method length.
- Class size.
- Cyclomatic or cognitive complexity, when available.
- Duplicated code or suspicious repeated patterns, when available.
- Basic maintainability hotspots.

Future integrations:

- SonarQube.
- CodeQL.
- Checkmarx.
- Snyk.
- Veracode.

### FR-4: LLM Assessment

The platform should use LLM to produce engineering judgment, not just summarize tool output.

LLM responsibilities:

- Group related findings into quality themes.
- Explain engineering impact.
- Identify cross-file maintainability risks.
- Detect likely architecture smells using evidence.
- Prioritize refactor recommendations.
- Generate role-aware summaries.

The LLM output must be structured and validated.

### FR-5: LLM Output Evaluation

The application must evaluate LLM output before storing it as a final report.

Minimum evaluator checks:

- JSON schema validity.
- Required fields present.
- Evidence references exist.
- Referenced file paths exist.
- Severity and score consistency.
- Recommendation actionability.
- Hallucinated file/class/function detection when feasible.

### FR-6: Report Generation

The final report should include:

- Overall score.
- Risk level.
- Top quality themes.
- Top hotspot files.
- Refactor backlog.
- Evidence references.
- LLM report confidence.
- Validation status.
- Retry count.
- Prompt/rubric/model version.

### FR-7: Dashboard

The first version can use Streamlit.

Dashboard pages:

- Upload/trigger scan page.
- Scan status page.
- Repo overview page.
- Findings page.
- LLM assessment page.
- Evaluation/confidence page.
- Raw JSON/report download page.

### FR-8: Advisory Only

The result does not block releases or merges.

The platform provides observations and recommendations only.

## 4.2 Non-Functional Requirements

### NFR-1: Runtime Portability

The core scan engine must be runtime-agnostic.

It should run in:

- Local CLI mode.
- Docker container mode.
- FastAPI service mode.
- Future ECS/Step Functions mode.

Step Functions should not be required for local demo execution.

### NFR-2: Evidence Grounding

Every major LLM conclusion must reference evidence.

No evidence means no accepted conclusion.

### NFR-3: Reproducibility

Each scan must store:

- Repo metadata.
- Source type.
- Scan configuration.
- Scanner versions.
- Model ID.
- Prompt version.
- Rubric version.
- Evaluation version.

### NFR-4: Security and Data Handling

For the demo:

- Uploaded source code should be treated as sensitive.
- Local files should be cleaned up after scan completion.
- Secrets and large binary files should be excluded from LLM input where feasible.

For future enterprise deployment:

- Use approved AWS environment.
- Use KMS encryption.
- Store secrets in Secrets Manager.
- Use least privilege IAM roles.
- Keep source code and LLM calls within approved company boundaries.
- Add audit logs for scan execution and report access.

### NFR-5: Extensibility

The system must support pluggable adapters:

- Repo source adapters.
- Scanner adapters.
- Storage adapters.
- LLM providers.
- Runtime runners.

## 5. High-Level Architecture

```text
Repository Input
  -> Repo Source Adapter
  -> Workspace Preparation
  -> Static Analysis / Scanner Suite
  -> Finding Normalization
  -> Evidence Builder + EvidenceSelector
  -> [Multi-Step LLM Pipeline]
       ThemeExtractor (LLM)
       ThemeEvidenceValidator (deterministic)
       RecommendationGenerator (LLM)
       ReportAssembler (deterministic)
  -> LLM Output Evaluator (deterministic hard gates)
  -> Scoring Engine (deterministic)
  -> Report Generator
  -> Storage
  -> Dashboard/API
```

### Core Design Principle

```text
The product is AI-assisted but engineering-controlled.

LLM is allowed to interpret, synthesize, and recommend.
It must not be the sole source of truth.
Every important conclusion must be grounded in evidence
and pass deterministic validation before appearing in the final report.
```

## 6. Recommended Demo Architecture

For the first demo:

```text
Streamlit UI
  -> FastAPI API
  -> Local/S3-like artifact storage
  -> Scan Engine
  -> Local scanner adapters
  -> Bedrock/OpenAI-compatible LLM adapter
  -> LLM output evaluator
  -> SQLite/Postgres-compatible storage
  -> Streamlit dashboard
```

The demo can initially run locally with Docker and local filesystem storage. The architecture should still align with the future AWS deployment model.

## 7. Future AWS Architecture

```text
Bitbucket / Local Upload
  -> FastAPI Upload/Trigger API
  -> S3 Artifact Bucket
  -> Step Functions Orchestrator
  -> ECS Fargate Scan Worker
  -> Scanner Integrations
       - SonarQube
       - CodeQL
       - Checkmarx
       - Snyk
       - Veracode
  -> Bedrock LLM Assessment
  -> RDS PostgreSQL
  -> S3 Raw Results
  -> Streamlit/React Dashboard
```

Step Functions should only orchestrate execution. The core business logic remains inside the scan engine and can be executed locally.

## 8. Core Components

## 8.1 Repo Source Adapter

Responsible for obtaining source code and returning a normalized repository artifact.

Implementations:

- `LocalUploadSourceAdapter` for demo.
- `BitbucketArchiveSourceAdapter` for future V2.
- `BitbucketCloneSourceAdapter` for future V3.

Output:

- Local workspace path.
- Repo metadata.
- Source artifact metadata.

## 8.2 Workspace Manager

Responsible for:

- Unzipping archives.
- Validating file size and format.
- Filtering ignored folders.
- Removing binary/generated files from analysis scope.
- Preparing clean workspace for scanners.

Recommended excluded folders:

- `.git`
- `node_modules`
- `target`
- `build`
- `dist`
- `.venv`
- `__pycache__`
- `.m2`
- `.gradle`
- `.idea`
- `.vscode`

## 8.3 Scanner Suite

Runs one or more scanner adapters and normalizes their outputs.

The scanner adapter interface is language-agnostic:

```text
ScannerAdapter.scan(workspace) -> NormalizedFinding[]
```

This interface does not change when a new language is added. Adding a new language means implementing a new scanner adapter.

MVP scanner adapter:

- One language scanner for the first demo: Java or Python, depending on implementation capacity. Java is preferred for a bank engineering demo because it is the dominant enterprise language. Python is acceptable if team capacity requires it.
- Optional Semgrep adapter if available.

Language detection in the workspace manager must handle all supported and unsupported languages. If an unsupported language is detected, the report should include a clear note such as "Language detected but not yet supported."

Future scanner adapters:

- SonarQube result ingestion.
- CodeQL result ingestion.
- Checkmarx result ingestion.
- Snyk result ingestion.
- Veracode result ingestion.

## 8.4 Evidence Builder and EvidenceSelector

The Evidence Builder and EvidenceSelector together produce the evidence package, which is the only input allowed for LLM quality assessment.

### 8.4.1 EvidenceSelector

The EvidenceSelector is a formal component responsible for:

- Ranking files and functions by importance.
- Selecting top-N findings and snippets.
- Enforcing a token budget.
- Validating that every selected evidence item has a stable evidence reference.
- Producing a bounded, deterministic, reproducible evidence package.

#### File Importance Scoring Formula (MVP)

```text
file_importance_score =
  0.5 * normalized_complexity
+ 0.3 * normalized_finding_count
+ 0.2 * normalized_file_size
```

Each component is normalized to the range [0, 1] across all files in the repo.

If test coverage data is available in a future version, the formula extends to:

```text
file_importance_score =
  0.4 * normalized_complexity
+ 0.3 * normalized_finding_count
+ 0.2 * normalized_file_size
+ 0.1 * normalized_test_gap
```

`test_gap` measures the degree to which high-complexity or high-risk code lacks test coverage. It is not required for MVP because coverage reports may not be available in the first demo.

#### Token Budget Management (MVP)

The evidence package must be bounded. A hard token limit is applied:

```text
max_evidence_tokens = configurable (default suggested: 80,000 tokens)
```

Suggested token allocation:

```text
repo structure summary: small fixed allocation
top findings: main allocation
selected snippets: main allocation
reserved output budget: minimum fixed reservation
```

The allocation does not need to be precisely optimized in MVP. The key requirement is that the evidence package is bounded before the LLM call, and the actual token count is logged per scan.

#### Phase 2 Enhancements (Do Not Implement in MVP)

The following enhancements are deferred:

- File centrality and dependency graph analysis.
- Churn × complexity temporal signal (requires git history or Bitbucket integration).
- Embedding-based evidence retrieval.

These will be documented in a Phase 2 specification.

### 8.4.2 Evidence Builder

Combines the outputs of the EvidenceSelector with repo metadata to produce the final evidence package:

- Repo summary (language breakdown, top-level structure).
- Selected normalized findings with evidence refs.
- Selected code snippets with file path and line ranges.
- Test file summary if available.
- Explicit list of unknowns (missing coverage, unsupported language, incomplete scanner output).
- Versions (evidence builder version, scanner versions).

## 8.5 LLM Assessment Pipeline (Multi-Step)

The LLM assessment uses a multi-step pipeline instead of a single large LLM call. Each step is smaller, focused, and validated before passing its output to the next step.

### Step 1: ThemeExtractor (LLM)

The LLM receives the evidence package and identifies the top 3–5 quality themes.

Requirements:
- Each theme must reference at least one evidence ID from the evidence package.
- The LLM must provide a concise rationale summary for each theme, citing the supporting evidence.
- Output must be schema-valid (use Claude Bedrock tool use or structured output to enforce this at the API level).

Note: Tool use or structured output reduces JSON formatting failures. It does not eliminate the need for semantic validation. The content of the output (evidence references, file paths, claim scope) must still be validated in Step 2.

### Step 2: ThemeEvidenceValidator (Deterministic)

A deterministic validator checks the ThemeExtractor output:

- All evidence references cited in themes exist in the evidence package.
- Referenced file paths exist in the workspace.
- Referenced class or function names exist in the findings.
- No hallucinated identifiers appear in the theme output.
- Theme output passes required field checks.

If validation fails, the theme is rejected and a targeted repair prompt is sent back to the ThemeExtractor (up to max retry count). If retry fails, the scan falls back to a static-only report.

### Step 3: RecommendationGenerator (LLM)

For each validated theme, the LLM generates a specific engineering recommendation.

Requirements:
- Recommendations must target a specific module, file, or function.
- Recommendations must include the specific change recommended.
- Recommendations must include the expected benefit.
- The LLM must not introduce new file, class, or function names that were not in the validated themes.

### Step 4: ReportAssembler (Deterministic)

A deterministic component assembles the final report:

- Combines validated themes and recommendations.
- Computes the final Code Quality Score using the deterministic scoring engine.
- Computes the LLM Report Quality Score.
- Attaches evaluation metadata.
- Writes the structured final report JSON.

The LLM does not compute the final Code Quality Score. The score is always computed deterministically.

A role-specific SummaryGenerator LLM step may be added in Phase 2 but is not required for MVP.

## 8.6 LLM Output Evaluator

The evaluator enforces hard gates before any LLM output is accepted as part of the final report.

### Hard Gate Checks (Deterministic, Always Run)

These checks are implemented in code, not by an LLM:

- Schema is valid and required fields are present.
- Evidence references exist in the evidence package.
- Referenced file paths exist in the repo workspace.
- Referenced line ranges are valid when provided.
- No hallucinated file, class, or function names appear in the report.
- Final score is within valid range.
- Report can be stored and rendered.

Failure in any hard gate check triggers a targeted repair prompt or fallback to static report.

### Soft Quality Checks (MVP)

These are recorded as scores but do not block the report:

- Actionability: are recommendations specific enough for engineers to act on?
- Unknowns coverage: are missing inputs explicitly listed?
- Evidence coverage: what percentage of claims are grounded?
- Confidence level: based on scanner completeness and evaluation results.

### Soft Quality Checks via LLM-as-Judge (Phase 2)

LLM-as-judge evaluation for qualitative dimensions (actionability, clarity, recommendation usefulness, role-specific summary quality) may be added in Phase 2. In MVP, soft checks should use deterministic heuristics (keyword patterns, field completeness, reference counts).

Possible final statuses:

- `ACCEPTED`
- `ACCEPTED_WITH_WARNING`
- `RETRY_REQUIRED`
- `FAILED_FALLBACK_TO_STATIC_REPORT`

## 8.7 Scoring Engine (Deterministic)

The Scoring Engine computes two separate scores deterministically.

### Code Quality Score

Measures how healthy or risky the repository is.

Inputs: static metrics and normalized findings from the Scanner Suite.

The LLM does not compute the Code Quality Score. The LLM may contribute qualitative assessments that inform the `risk_signals` component, but the score formula is always deterministic.

```text
Code Quality Score =
  25% Maintainability
+ 20% Complexity
+ 15% Duplication
+ 15% Test Coverage or Test Presence Proxy
+ 15% Risk Signals
+ 10% Architecture/Design Signal (from validated LLM themes, advisory only)
```

If test coverage is unavailable, use a test presence proxy and mark the confidence limitation in the report.

### LLM Report Quality Score

Measures how trustworthy the LLM-generated report is.

Computed by the evaluator after hard gate checks:

```text
LLM Report Quality Score =
  30% Grounding (evidence reference coverage)
+ 20% Schema Correctness
+ 20% Consistency (severity vs. risk level vs. score)
+ 20% Actionability (recommendation specificity)
+ 10% Conciseness / Clarity
```

Both scores must be displayed separately in the dashboard.

## 8.8 Report Generator

Generates:

- JSON report.
- Optional Markdown report.
- Dashboard-readable summary.

## 8.9 Storage Layer

Demo:

- SQLite or local JSON files.
- Local filesystem for raw artifacts and reports.

Future:

- S3 for raw artifacts and raw scanner outputs.
- RDS PostgreSQL for structured scan metadata and findings.

## 9. Data Model

## 9.1 Tenant

```text
tenant
- tenant_id
- name
- owner_group
- created_at
```

## 9.2 Repository

```text
repository
- repo_id
- tenant_id
- repo_name
- source_type
- default_branch
- language_stack
- scan_enabled
- created_at
```

## 9.3 Scan Run

```text
scan_run
- scan_run_id
- repo_id
- source_type
- branch
- commit_sha
- status
- triggered_by
- started_at
- completed_at
- overall_score
- risk_level
- report_quality_status
- retry_count
```

## 9.4 Finding

```text
finding
- finding_id
- scan_run_id
- category
- severity
- file_path
- class_name
- function_name
- line_start
- line_end
- metric_name
- metric_value
- evidence_ref
- recommendation
- source_tool
```

## 9.5 LLM Report

```text
llm_report
- llm_report_id
- scan_run_id
- model_id
- prompt_version
- rubric_version
- raw_output_uri
- parsed_output_json
- confidence
- status
```

## 9.6 LLM Report Evaluation

```text
llm_report_evaluation
- evaluation_id
- scan_run_id
- llm_report_id
- evaluator_version
- schema_valid
- grounding_score
- consistency_score
- actionability_score
- hallucination_count
- retry_count
- final_status
- failure_reasons
```

## 10. Suggested Repository Structure

```text
code-quality-platform/
  app/
    api/
      main.py
      routes/
        uploads.py
        scans.py
        reports.py

    core/
      scan_engine.py
      scoring_engine.py
      report_generator.py

    sources/
      base.py
      local_upload.py
      bitbucket_archive.py
      bitbucket_clone.py

    scanners/
      base.py
      local_java.py
      local_python.py
      semgrep.py
      sonarqube.py
      codeql.py
      checkmarx.py
      snyk.py
      veracode.py

    evidence/
      builder.py
      selector.py
      snippet_selector.py
      repo_summarizer.py
      token_budget.py

    llm/
      bedrock_client.py
      prompt_builder.py
      schemas.py
      theme_extractor.py
      recommendation_generator.py
      report_assembler.py
      evaluator.py

    storage/
      base.py
      local_storage.py
      sqlite.py
      s3_storage.py
      postgres.py

    runtime/
      local_runner.py
      api_runner.py
      ecs_runner.py
      stepfunctions_runner.py

    models/
      scan.py
      finding.py
      report.py
      evaluation.py

  cli/
    main.py

  dashboard/
    streamlit_app.py

  docker/
    Dockerfile

  tests/
```

## 11. Architecture Decisions for Demo

### ADR-001: MVP Uses Local Repo Upload

Decision:

The first demo supports manual upload of repository archives.

Rationale:

This avoids early dependency on Bitbucket network access, credentials, enterprise approval, and repository permission integration. It allows the team to validate the core assessment pipeline first.

### ADR-002: Source Acquisition Is Pluggable

Decision:

The scan engine consumes a normalized repo artifact and does not care whether the source came from local upload, Bitbucket archive, or git clone.

Rationale:

This avoids rewriting the scanner pipeline when Bitbucket integration is added.

### ADR-003: LLM Is Evidence-Grounded

Decision:

LLM assessment must be based on evidence packages produced by the system. Major conclusions must reference evidence.

Rationale:

This improves consistency, explainability, and trust.

### ADR-004: LLM Output Must Be Evaluated

Decision:

LLM output must pass evaluator checks before becoming part of the final accepted report.

Rationale:

The platform should be engineering-controlled, not black-box AI output.

### ADR-005: Step Functions Is Optional Orchestration

Decision:

Step Functions may be used in AWS production, but the core scan engine must be executable locally and inside a standalone container.

Rationale:

This enables local testing, faster iteration, and simpler demo development.

### ADR-006: Advisory Only

Decision:

The platform does not block releases or merges in the demo version.

Rationale:

The assessment system must establish trust before being used in any gating workflow.

### ADR-007: AI-Assisted but Engineering-Controlled

Decision:

The LLM is allowed to interpret, synthesize, and recommend. It must not be the sole source of truth. Every important conclusion must be grounded in evidence and pass deterministic validation before appearing in the final report.

Rationale:

This keeps the system trustworthy for a regulated enterprise audience. It also means the system degrades gracefully — if the LLM fails or is unavailable, a static-only report is still produced.

### ADR-008: Multi-Step LLM Pipeline Over Single Large Call

Decision:

The LLM assessment uses a 4-step pipeline: ThemeExtractor (LLM) → ThemeEvidenceValidator (deterministic) → RecommendationGenerator (LLM) → ReportAssembler (deterministic). The final report is assembled by a deterministic component.

Rationale:

A single large LLM call that extracts themes, generates recommendations, writes summaries, and outputs final JSON has too many failure modes. Breaking the pipeline into validated steps reduces hallucination scope, makes repair prompts more targeted, and enables deterministic assembly of the final report.

### ADR-009: Structured Output to Reduce Format Failures

Decision:

Use Claude Bedrock tool use or structured output (e.g., `tool_choice: {"type": "tool"}`) for each LLM step to enforce schema at the API level.

Rationale:

Tool use ensures that each LLM step output is structurally valid JSON that matches the declared schema. This reduces format-related retry failures. It does not eliminate the need for semantic validation. Evidence references, file path existence, and claim scope must still be validated by the deterministic evaluator.

### ADR-010: MVP Language Scope

Decision:

The first demo scanner implements one language: Java (preferred for a bank engineering demo) or Python (acceptable if team capacity requires it). The scanner adapter interface is language-agnostic from day 1.

Rationale:

Implementing one language first reduces complexity and allows the team to complete the end-to-end pipeline. The adapter interface ensures that adding the second language requires only a new scanner implementation, not changes to the evidence builder, LLM pipeline, evaluator, or dashboard.

### ADR-011: Deferred Enhancements

The following capabilities are out of scope for MVP and documented as Phase 2 targets:

- Churn × complexity temporal signal (requires git history or Bitbucket integration).
- File centrality and dependency graph analysis.
- Embedding-based evidence retrieval.
- Self-consistency sampling (available as `--stability-check` flag in CLI, not default).
- LLM-as-judge soft evaluation.
- Role-specific SummaryGenerator LLM step.
- SonarQube, CodeQL, Checkmarx, Snyk, Veracode scanner adapters.
- Portfolio-level multi-repo analysis.

## 12. Open Questions

- Which internal scanner result APIs are easiest to access first?
- Should SonarQube be integrated in the first company-internal PoC after local demo?
- Which LLM model and Bedrock endpoint are approved for source code input?
- What maximum repo size should the first demo support?
- Should uploaded source archives be persisted or deleted after scan completion?
- What is the expected dashboard authentication model in the company environment?
