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
  -> Evidence Builder
  -> LLM Assessment
  -> LLM Output Evaluator
  -> Scoring Engine
  -> Report Generator
  -> Storage
  -> Dashboard/API
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

Demo scanner adapters:

- Local Java metrics adapter.
- Local Python metrics adapter.
- Optional Semgrep adapter.

Future scanner adapters:

- SonarQube result ingestion.
- CodeQL result ingestion.
- Checkmarx result ingestion.
- Snyk result ingestion.
- Veracode result ingestion.

## 8.4 Evidence Builder

Builds an evidence package from:

- Static metrics.
- Normalized findings.
- Repo structure.
- Selected code snippets.
- Hotspot files.
- Test file summary.
- Historical results, when available.

The evidence package is the only input that should be used for LLM quality assessment.

## 8.5 LLM Assessor

Responsible for:

- Building prompts from the evidence package.
- Calling the approved LLM provider.
- Requesting schema-constrained structured output.
- Returning raw and parsed LLM output.

The LLM should provide engineering judgment, not raw facts.

## 8.6 LLM Output Evaluator

Responsible for validating the LLM report.

Possible final statuses:

- `ACCEPTED`
- `ACCEPTED_WITH_WARNING`
- `RETRY_REQUIRED`
- `FAILED_FALLBACK_TO_STATIC_REPORT`

## 8.7 Scoring Engine

Computes deterministic score from metrics and accepted LLM assessment.

Recommended MVP approach:

- Static metrics dominate the score.
- LLM contributes a smaller advisory component.
- LLM must not be the sole source of the final score.

Example weighting:

```text
Overall Score =
  25% Maintainability
+ 20% Complexity
+ 15% Duplication
+ 15% Test Coverage or Test Presence Proxy
+ 15% Risk Signals
+ 10% LLM Architecture/Readability Assessment
```

For the first demo, if test coverage is unavailable, use a placeholder or test presence proxy and clearly mark confidence limitations.

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
      snippet_selector.py
      repo_summarizer.py

    llm/
      bedrock_client.py
      prompt_builder.py
      schemas.py
      assessor.py
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

## 12. Open Questions

- Which internal scanner result APIs are easiest to access first?
- Should SonarQube be integrated in the first company-internal PoC after local demo?
- Which LLM model and Bedrock endpoint are approved for source code input?
- What maximum repo size should the first demo support?
- Should uploaded source archives be persisted or deleted after scan completion?
- What is the expected dashboard authentication model in the company environment?
