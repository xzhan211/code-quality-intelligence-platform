# Code Quality Platform - Implementation Plan

## 1. Purpose

This document breaks the first demo version into implementation phases and features suitable for PR/commit tracking.

The goal is to help a small team build the project efficiently while keeping architecture, evaluation, and traceability under control.

## 2. Team Assumptions

Team size:

- 1 tech lead responsible for architecture, LLM pipeline, evaluation, and overall design decisions.
- 1 junior engineer responsible for workspace preparation, scanner implementation, and dashboard.
- A third engineer may be added once the development process is stable.

Project stage:

- First version is a demo.
- Initial repo input is local upload.
- First scan target is one medium/small service repository.
- Main language for MVP scanner: Java (preferred for bank demo) or Python (acceptable if team capacity requires).
- Scanner adapter interface is language-agnostic from day 1.
- Results are advisory only and do not block releases.

## 3. Workstream Ownership

### Tech Lead

Primary focus:

- Architecture and design decisions.
- Project skeleton and configuration system.
- FastAPI service and scan run lifecycle.
- EvidenceSelector and Evidence Builder.
- Multi-step LLM pipeline (ThemeExtractor, ThemeEvidenceValidator, RecommendationGenerator, ReportAssembler).
- LLM client abstraction (Claude Bedrock, structured output via tool use).
- LLM output evaluator (hard gate checks).
- Deterministic scoring engine.
- Report generator.
- Demo narrative and demo dataset design.
- PR review and scope control.

### Junior Engineer

Primary focus:

- Workspace preparation.
- File inventory and language detection.
- Scanner adapter implementation (one language).
- Finding normalization.
- Streamlit dashboard (upload, report, evaluation pages).
- Structured logging and observability basics.
- Dockerization.

## 4. Development Principles

- Keep each PR small and reviewable.
- Do not mix infrastructure, scanner logic, LLM logic, and dashboard changes in the same PR unless necessary.
- Add tests for core logic before dashboard polish.
- Treat evaluation as a core feature, not a later enhancement.
- Use interfaces and adapters from day 1, even if the first implementation is local-only.
- The scanner adapter interface must be language-agnostic. Adding a second language means adding a new scanner adapter, not changing the pipeline.
- Store raw outputs and metadata for traceability.
- The LLM does not compute the final score. All scoring is deterministic.
- Do not let invalid LLM output reach the final report. Every LLM output passes the hard gate evaluator.
- Tool use reduces format failures. Semantic validation is still required.
- The product is AI-assisted but engineering-controlled.

## 5. Phase 0: Project Foundation

Goal:

Create a clean project skeleton that supports local execution, tests, and future adapters.

### Feature 0.1: Repository Skeleton

Scope:

- Create project folder structure as defined in architecture.md section 10.
- Add README.
- Add package setup (pyproject.toml or setup.cfg).
- Add lint and test commands.

```text
PR-001: Initialize project structure and developer tooling
```

Acceptance criteria:

- Project installs locally.
- Test command runs.
- CLI placeholder works.

### Feature 0.2: Core Domain Models

Scope:

Create core models:

- Tenant.
- Repository.
- ScanRun (with all status codes from pipeline_details.md section 6).
- Finding (normalized finding schema).
- EvidencePackage (including `token_count` and `versions` fields).
- ValidatedTheme (output of ThemeEvidenceValidator).
- AssembledLLMReport (output of ReportAssembler).
- EvaluationResult (hard gate results and soft quality scores).
- LLMUsageMetadata (model_id, input_tokens, output_tokens per step).
- FinalReport (Code Quality Score + LLM Report Quality Score).

```text
PR-002: Add core domain models for scan, finding, evidence, LLM pipeline, and evaluation
```

Acceptance criteria:

- Models are serializable to JSON.
- Basic unit tests exist.
- No runtime-specific dependencies inside models.
- Code Quality Score and LLM Report Quality Score are separate fields.

### Feature 0.3: Configuration System

Scope:

- App config.
- Scan limits (max file size, max file count).
- Token budget config (`max_evidence_tokens`, default 80,000).
- LLM config (model ID, prompt version, rubric version).
- Storage mode config (local vs. future S3/RDS).
- Supported languages config flag.

```text
PR-003: Add configuration loading and scan defaults
```

Acceptance criteria:

- Local config loads from environment or file.
- Defaults are documented and match architecture.md.
- `max_evidence_tokens` is configurable.

## 6. Phase 1: Local Scan Engine MVP

Goal:

Run a local scan from a repo archive and produce normalized findings without LLM.

### Feature 1.1: CLI Scan Entrypoint

Scope:

```bash
code-quality scan --input ./repo.zip --repo-name demo-service --tenant demo
```

Optional flag for offline stability evaluation:

```bash
code-quality scan --input ./repo.zip --repo-name demo-service --tenant demo --stability-check --runs 3
```

```text
PR-004: Add CLI entrypoint for local scan execution
```

Acceptance criteria:

- CLI accepts input path, repo name, tenant.
- Creates a scan_run object and writes basic scan metadata.
- `--stability-check` flag is accepted but can be a stub for now.

### Feature 1.2: Local Repo Source Adapter

Scope:

- Define `RepoSourceAdapter` interface.
- Implement `LocalUploadSourceAdapter`.
- Return normalized `RepoArtifact`.

```text
PR-005: Implement local repo source adapter
```

Acceptance criteria:

- Zip and tar.gz paths are accepted.
- Invalid file type fails clearly.
- Adapter output is independent of runtime.

### Feature 1.3: Workspace Manager

Scope:

- Unpack archive.
- Validate size and file count limits.
- Build file inventory (path, extension, size, detected language).
- Exclude generated and binary folders (see architecture.md section 8.2).
- Detect supported languages (Java, Python).
- For unsupported or undetected languages, add an entry to the `unknowns` list rather than failing.

```text
PR-006: Add workspace preparation and file inventory
```

Acceptance criteria:

- Archive is extracted to isolated workspace.
- Excluded folders are ignored.
- File inventory includes path, extension, size, language.
- Unsupported language produces a warning in `unknowns`, not a scan failure.
- Temporary workspace can be cleaned up.

### Feature 1.4: Language-Agnostic Scanner Adapter Interface

Scope:

- Define `ScannerAdapter` interface: `scan(workspace) -> NormalizedFinding[]`.
- Define `NormalizedFinding` schema (finding_id, source_tool, category, severity, file_path, class_name, function_name, line_start, line_end, metric_name, metric_value, evidence_ref).
- Add `ScannerSuite` runner that runs one or more adapters and aggregates results.
- Individual scanner failure is captured and stored, not propagated as a fatal error.

```text
PR-007: Add language-agnostic scanner adapter interface and scanner suite runner
```

Acceptance criteria:

- Scanner interface does not contain any language-specific logic.
- Scanner suite can run multiple adapters.
- Individual scanner failure is captured with a clear reason.
- Adding a new language scanner requires only a new adapter class, no changes to the suite.

### Feature 1.5: MVP Language Scanner

Scope:

Implement the scanner adapter for the MVP language (Java or Python, as agreed by the team).

Java scanner (preferred):
- File size, class size, method length.
- Basic cyclomatic or cognitive complexity using a lightweight parser.
- Detect long methods and large classes.

Python scanner (alternative):
- File size, function length, class size.
- Cyclomatic complexity via `radon` or equivalent.
- Detect long functions and large files.

```text
PR-008: Add MVP language scanner (Java or Python)
```

Acceptance criteria:

- Produces normalized findings that conform to the NormalizedFinding schema.
- Handles parsing errors and syntax issues gracefully.
- Unit tests use small sample source files.
- Output is deterministic for the same input.

### Feature 1.6: Finding Normalization

Scope:

- Normalize all scanner outputs into the common NormalizedFinding schema.
- Assign categories (complexity, maintainability, duplication, size) and severity (low, medium, high).
- Generate stable evidence IDs (`finding:{finding_id}`).
- Severity assignment is deterministic (based on metric thresholds in config).

```text
PR-009: Normalize scanner findings into common schema
```

Acceptance criteria:

- All findings are valid JSON.
- Each finding has an `evidence_ref`.
- Severity and category assignment is deterministic and documented.
- Unit tests cover edge cases (zero findings, very large file, syntax errors).

## 7. Phase 2: Evidence Package and Static Report

Goal:

Build a bounded, reproducible evidence package and generate a static-only report that works without LLM.

### Feature 2.1: Repo Summary Builder

Scope:

- Generate repo tree summary.
- Language breakdown (count by language).
- Top large files list.
- Test file detection (files with `test`, `spec`, or `_test` naming patterns).
- Summary is deterministic for the same input.

```text
PR-010: Add repository summary builder
```

Acceptance criteria:

- Summary works for single-language and mixed-language repos.
- Output is deterministic.
- Output includes explicit `unknowns` for missing inputs (no coverage, no git history, etc.).

### Feature 2.2: EvidenceSelector

Scope:

Implement the EvidenceSelector component as specified in architecture.md section 8.4.1.

File importance scoring formula:

```text
file_importance_score =
  0.5 * normalized_complexity
+ 0.3 * normalized_finding_count
+ 0.2 * normalized_file_size
```

Each factor is normalized to [0, 1] across all files in the repo.

Responsibilities:

- Rank all files by importance score.
- Select top-N files and their associated findings and snippets.
- Enforce token budget (`max_evidence_tokens` from config).
- If selection exceeds budget, drop lower-ranked items until within budget.
- Assign a stable `evidence_ref` to every selected item.
- Log actual token count per scan.

```text
PR-011: Implement EvidenceSelector with importance scoring and token budget
```

Acceptance criteria:

- Evidence selection is deterministic for the same input and config.
- Token count is logged.
- Evidence package never exceeds `max_evidence_tokens`.
- Every selected item has a stable `evidence_ref`.
- Unit tests cover: normal case, budget exceeded (truncation), empty findings.

### Feature 2.3: Snippet Selector

Scope:

- Select code snippets from top-ranked files.
- Limit snippet size (configurable max lines per snippet).
- Preserve file path and line ranges.
- Exclude binary and generated files.
- Exclude likely secrets or credentials.

```text
PR-012: Add evidence snippet selector
```

Acceptance criteria:

- Snippets include valid file paths and line ranges.
- Total snippet contribution stays within token budget allocation.
- Secrets-like patterns (API keys, passwords) are redacted before inclusion.

### Feature 2.4: Evidence Package Builder

Scope:

- Combine repo summary, EvidenceSelector output, snippets, unknowns, and metadata.
- Output conforms to the EvidencePackage schema from pipeline_details.md section 6.
- Include `token_count` and `versions` fields.
- Store evidence package as a reproducible artifact.

```text
PR-013: Build and persist evidence package
```

Acceptance criteria:

- Evidence package is schema-valid.
- Every finding and snippet has a stable evidence ref.
- Evidence package can be reproduced from scan inputs and config.
- `token_count` is recorded.

### Feature 2.5: Static Report Generator

Scope:

- Generate a static-only JSON and Markdown report from findings and metrics.
- Include top findings, hotspot files, basic score, unknowns, and limitations.
- This report must work without any LLM. It is the fallback baseline.

```text
PR-014: Generate static-only quality report (no LLM)
```

Acceptance criteria:

- Report is produced even when LLM is not configured or fails.
- Report includes score, top findings, hotspots, unknowns, and limitations.
- Score is deterministic.

## 8. Phase 3: Multi-Step LLM Assessment Pipeline

Goal:

Use LLM to generate evidence-grounded quality themes and recommendations via a validated multi-step pipeline. Every LLM step uses structured output (tool use) and is followed by deterministic validation.

### Feature 3.1: LLM Output Schemas

Scope:

Define schemas for each LLM step:

- `ThemeExtractorOutput`: list of themes, each with theme_id, title, severity, evidence_refs, rationale.
- `RecommendationGeneratorOutput`: list of recommendations, each with theme_id, target, recommended_change, expected_benefit, priority.
- `AssembledLLMReport`: combined structure passed to the evaluator.

```text
PR-015: Define LLM step schemas and validation models
```

Acceptance criteria:

- Schemas support evidence_refs.
- Pydantic or equivalent validation models exist.
- Unit tests validate good and bad examples for each schema.

### Feature 3.2: LLM Client Interface and Claude Bedrock Adapter

Scope:

- Define `LLMClient` interface: `call(prompt, tool_definition, tool_choice) -> structured_output`.
- Implement `BedrockLLMClient` using the AWS Bedrock API with Claude.
- Use `tool_choice: {"type": "tool", "name": "<tool_name>"}` to enforce structured output on every LLM call.
- Add `MockLLMClient` for unit tests.
- Store model ID, input tokens, and output tokens per call in `LLMUsageMetadata`.

```text
PR-016: Add LLM client interface and Claude Bedrock adapter with structured output
```

Acceptance criteria:

- Core pipeline can run with MockLLMClient without Bedrock access.
- BedrockLLMClient enforces tool use on every call.
- LLMUsageMetadata is populated after every real call.
- Provider-specific code is isolated in the adapter.

### Feature 3.3: Prompt and Rubric v1

Scope:

- Write ThemeExtractor system prompt and rubric.
- Write RecommendationGenerator system prompt.
- Prompts must instruct the LLM to use only provided evidence refs.
- Prompts must require the LLM to provide a concise rationale citing evidence, not hidden chain-of-thought.
- Rubric defines severity levels with evidence requirements (high severity requires strong evidence and meaningful engineering impact).
- Each prompt is versioned (v1.0).

```text
PR-017: Add prompt and rubric v1 for ThemeExtractor and RecommendationGenerator
```

Acceptance criteria:

- Prompts instruct LLM to cite only evidence_refs from the evidence package.
- Prompts require rationale per conclusion.
- Prompt and rubric versions are stored in every scan result.
- Prompts are tested with MockLLMClient.

### Feature 3.4: ThemeExtractor

Scope:

- Build ThemeExtractor prompt from evidence package.
- Call LLM client with `ThemeExtractorOutput` tool definition.
- Parse and return structured output.
- Store raw LLM output for audit.

```text
PR-018: Implement ThemeExtractor LLM step
```

Acceptance criteria:

- ThemeExtractor output conforms to `ThemeExtractorOutput` schema.
- Raw output is stored with prompt version and model ID.
- Works with MockLLMClient in tests.

### Feature 3.5: ThemeEvidenceValidator

Scope:

Deterministic validator for ThemeExtractor output:

- All evidence_refs cited in each theme exist in the evidence package.
- Referenced file paths exist in the workspace.
- Referenced class or function names exist in the normalized findings.
- Required theme fields are present.
- No hallucinated identifiers.

If any theme fails validation, it is flagged for repair. Valid themes continue to the next step.

```text
PR-019: Implement ThemeEvidenceValidator (deterministic)
```

Acceptance criteria:

- Missing evidence_refs are detected and reported.
- Non-existent file paths are detected and reported.
- Hallucinated class/function names are detected.
- Unit tests cover: all-valid case, one invalid theme, all invalid themes.

### Feature 3.6: RecommendationGenerator

Scope:

- For each validated theme, build RecommendationGenerator prompt.
- Call LLM client with `RecommendationGeneratorOutput` tool definition.
- Recommendations must reference the theme's validated evidence.
- Store raw output.

```text
PR-020: Implement RecommendationGenerator LLM step
```

Acceptance criteria:

- Each recommendation references its parent theme_id.
- Recommendation fields include target, recommended_change, expected_benefit, priority.
- Works with MockLLMClient in tests.

### Feature 3.7: ReportAssembler

Scope:

Deterministic component that assembles the final LLM report from validated themes and recommendations. No LLM call.

- Combine validated themes with their recommendations.
- Attach evidence package version, prompt versions, model IDs, and LLM usage metadata.
- Produce a structured `AssembledLLMReport` for the evaluator.

```text
PR-021: Implement deterministic ReportAssembler
```

Acceptance criteria:

- ReportAssembler output is deterministic given the same validated themes and recommendations.
- LLM usage metadata (tokens per step) is included.
- No LLM is called.

## 9. Phase 4: LLM Evaluation and Retry

Goal:

Prevent invalid LLM output from reaching the final report. Hard gate checks are deterministic. Soft quality checks are heuristic.

### Feature 4.1: Hard Gate Evaluator

Scope:

Implement deterministic hard gate checks:

- Schema is valid and required fields are present.
- Evidence references cited in themes exist in the evidence package.
- Referenced file paths exist in the repo workspace.
- Referenced line ranges are valid when provided.
- No hallucinated file, class, or function names (check against workspace file tree and normalized findings).
- Assembled report can be serialized and rendered.

Note: These checks are implemented in code only. No LLM is used for hard gate validation.

```text
PR-022: Add hard gate evaluator (deterministic)
```

Acceptance criteria:

- Each check is independently testable.
- All hard gate checks are covered by unit tests.
- Failures produce clear, structured failure reasons.
- Hard gate never calls the LLM.

### Feature 4.2: Soft Quality Scorer

Scope:

Implement deterministic heuristics for soft quality checks:

- Grounding score: percentage of themes with at least one valid evidence reference.
- Consistency score: does risk level match severity distribution?
- Actionability score: are target, recommended_change, and expected_benefit fields populated in each recommendation?
- Evidence coverage: percentage of themes that are evidence-backed.
- Unknowns coverage: are missing inputs (no coverage, unsupported language) explicitly listed?

These scores do not block acceptance. They feed into the LLM Report Quality Score.

```text
PR-023: Add soft quality scorer (deterministic heuristics)
```

Acceptance criteria:

- Each score is a float in [0, 1].
- Scores are reproducible for the same input.
- Scores are included in the evaluation result and final report.

### Feature 4.3: Retry and Repair Prompt Flow

Scope:

- Implement repair prompts for ThemeExtractor failures: schema repair, grounding repair, consistency repair.
- Max retry count from config (default: 2).
- Store failure reason, repair prompt type, attempt number, and raw failed output for each retry.
- If all retries fail, fall back to static-only report and mark LLM status as FAILED.

```text
PR-024: Add LLM retry and repair prompt flow
```

Acceptance criteria:

- Schema failure triggers schema repair prompt.
- Grounding failure triggers grounding repair prompt.
- Max retry is enforced.
- Static report is returned on LLM failure.
- All retry attempts are stored.

### Feature 4.4: Evaluation Metadata Storage

Scope:

- Persist evaluation result with every scan.
- Include hard gate results, soft quality scores, LLM usage metadata, retry count, and failure reasons.

```text
PR-025: Persist LLM evaluation metadata and LLM usage metadata
```

Acceptance criteria:

- Evaluation metadata appears in the final report.
- Dashboard can read and display evaluation metadata.
- LLM usage metadata (tokens per step) is stored.

## 10. Phase 5: Final Scoring and Report

Goal:

Combine static metrics and validated LLM assessment into two separate scores and a final user-facing report.

### Feature 5.1: Deterministic Scoring Engine

Scope:

Compute two separate scores:

Code Quality Score:

```text
Code Quality Score =
  25% Maintainability
+ 20% Complexity
+ 15% Duplication
+ 15% Test Coverage or Test Presence Proxy
+ 15% Risk Signals
+ 10% Architecture/Design Signal (from validated LLM themes, advisory only)
```

LLM Report Quality Score:

```text
LLM Report Quality Score =
  30% Grounding
+ 20% Schema Correctness
+ 20% Consistency
+ 20% Actionability
+ 10% Conciseness / Clarity
```

If LLM report failed: exclude Architecture/Design Signal from Code Quality Score and mark score confidence lower. LLM Report Quality Score reflects the failure.

The LLM does not compute either score.

```text
PR-026: Add deterministic scoring engine for both Code Quality Score and LLM Report Quality Score
```

Acceptance criteria:

- Both scores are reproducible.
- Score breakdowns are included.
- LLM does not contribute to score computation.
- Missing metrics produce explicit limitations in the report.

### Feature 5.2: Final Report Generator

Scope:

- Combine static report, validated LLM report, both scores, and evaluation result.
- Generate final JSON report and Markdown report.
- Report sections follow pipeline_details.md Stage 11.

```text
PR-027: Generate final quality report with both scores and evaluation status
```

Acceptance criteria:

- Report includes Code Quality Score and LLM Report Quality Score separately.
- Report includes evidence references for all major themes.
- Report includes unknowns and limitations section.
- Report includes prompt, rubric, model, and evaluator versions.

### Feature 5.3: Report Export

Scope:

- Allow report download from local output or API.
- Include raw JSON artifacts (findings, evidence package, LLM step outputs, evaluation result).

```text
PR-028: Add report export and raw artifact access
```

Acceptance criteria:

- User can inspect the final report and every intermediate artifact.

## 11. Phase 6: API and Dashboard

Goal:

Provide a simple user interface for the demo. Dashboard supports both standard report view and story-mode demo flow.

### Feature 6.1: FastAPI Scan API

Scope:

- POST /api/repos/upload
- POST /api/scans
- GET /api/scans/{scan_id}
- GET /api/scans/{scan_id}/report

```text
PR-029: Add FastAPI endpoints for upload, scan trigger, status, and report retrieval
```

Acceptance criteria:

- Upload creates scan run.
- Scan can be triggered.
- Status can be polled.
- Report can be retrieved.

### Feature 6.2: Background Scan Execution

Scope:

- API triggers scan as a background task.
- Scan status is updated at each pipeline stage.
- Failed stages produce a clear error message with stage name and reason.

```text
PR-030: Add background scan execution and stage-level status tracking
```

Acceptance criteria:

- UI or API can observe scan progress by stage.
- Failed stage is identifiable and diagnosable.

### Feature 6.3: Streamlit Upload Page

Scope:

- Upload repo archive (zip or tar.gz).
- Enter tenant and repo metadata.
- Select scan config (optional language override).
- Trigger scan.
- Show scan status with stage progress.

```text
PR-031: Add Streamlit upload and scan trigger page
```

Acceptance criteria:

- User can start scan from UI without CLI.

### Feature 6.4: Streamlit Report Page

Scope:

- Display both scores prominently and separately:
  - Code Quality Score: X / 100, Risk Level: Y
  - LLM Report Quality Score: X / 100, Validation Status: Y
- Show top quality themes with evidence refs.
- Show recommended backlog with links to source evidence.
- Show evaluation indicators (validation status, evidence coverage, retry count, prompt/model/rubric versions).
- Allow clicking from a recommendation through to the evidence finding and source snippet.

```text
PR-032: Add Streamlit quality report dashboard
```

Acceptance criteria:

- Both scores are displayed separately.
- Evidence trail from recommendation to source snippet is navigable.
- Evaluation status is visible.
- Page is usable by an engineer, a manager, and an architecture reviewer.

### Feature 6.5: Story-Mode Demo Flow

Scope:

Add a guided demo mode to the dashboard for the leadership presentation. The demo mode walks through the following steps using a preloaded demo scan result:

1. Repo uploaded and scan triggered.
2. Static findings and metrics shown.
3. EvidenceSelector output shown: which files were selected and why (importance scores).
4. ThemeExtractor output shown: quality themes with evidence refs and rationale.
5. Evaluator catching a rejected output: show a hallucinated class name being detected and the repair prompt being triggered.
6. Accepted report after repair: themes, recommendations, evidence trail.
7. Both Code Quality Score and LLM Report Quality Score displayed separately.
8. Click-through from recommendation to evidence to source snippet.

```text
PR-033: Add story-mode demo flow to dashboard
```

Acceptance criteria:

- Demo mode can run without a live scan (from preloaded demo data).
- The evaluator rejection and repair scenario is demonstrable.
- Both scores are shown separately with a clear explanation.
- Demo is understandable to a non-engineer leadership audience.

### Feature 6.6: Scan History Page

Scope:

- List previous scans with status, timestamp, and Code Quality Score.
- Link to report.

```text
PR-034: Add scan history dashboard page
```

Acceptance criteria:

- User can view prior scan results.

## 12. Phase 7: Containerization and Demo Hardening

Goal:

Make the demo easy to run reproducibly.

### Feature 7.1: Dockerfile

Scope:

- Build application container.
- Include CLI and API runtime.
- Document local volume usage for input and output.

```text
PR-035: Add Dockerfile for local container execution
```

Acceptance criteria:

- Container can run CLI scan.
- Container can run API server.

### Feature 7.2: Docker Compose

Scope:

- Run API plus dashboard plus local storage in one command.
- Optional SQLite volume.

```text
PR-036: Add docker-compose for local demo
```

Acceptance criteria:

- `docker compose up` starts the full demo stack.

### Feature 7.3: Demo Dataset

Scope:

- Create a synthetic repo with intentional quality problems.
- Java Spring Boot payment-service is the preferred demo target (realistic for a bank audience, no sensitive code).
- Python alternative if team requires it.
- Include examples for: high-complexity service, duplicated logic, poor test presence, clean module for contrast.
- Include a preloaded demo scan result that can be used for the story-mode demo without re-running a live scan.

```text
PR-037: Add demo input repo and preloaded demo scan result
```

Acceptance criteria:

- Demo can run without company source code.
- Story-mode demo can run from preloaded data without a live LLM call.
- EvidenceSelector behavior and evaluator rejection scenario are demonstrable.

### Feature 7.4: Observability Basics

Scope:

- Structured logs per scan stage: scan_run_id, stage name, duration, status, error reason.
- LLM usage metadata logged per step: model_id, input_tokens, output_tokens, retry_count.
- Scanner versions logged.

```text
PR-038: Add structured logging and scan stage timing
```

Acceptance criteria:

- Each scan stage logs start, end, and status.
- Failures are diagnosable from logs.
- LLM usage is logged without including raw source code in log output.

## 13. Phase 8: Enterprise Integration Preparation

Goal:

Prepare for company environment without blocking demo.

### Feature 8.1: SonarQube Adapter Stub

Scope:

- Add interface and config for SonarQube result ingestion.
- Provide stub implementation or sample JSON parser.
- Adapter must conform to the language-agnostic `ScannerAdapter` interface.

```text
PR-039: Add SonarQube scanner adapter skeleton
```

Acceptance criteria:

- Adapter can parse sample SonarQube-format JSON.
- Real API integration can be added later with no pipeline changes.

### Feature 8.2: Enterprise Scanner Adapter Skeletons

Scope:

Add interface-conforming stubs for: CodeQL, Checkmarx, Snyk, Veracode.

```text
PR-040: Add enterprise scanner adapter skeletons
```

Acceptance criteria:

- Adapter contracts are consistent with the language-agnostic scanner interface.
- No fake results are mixed into real reports unless explicitly marked as demo data.

### Feature 8.3: Second Language Scanner

Scope:

- Add scanner for the second language (Java if Python was first, Python if Java was first).
- No pipeline changes are needed.

```text
PR-041: Add second language scanner adapter
```

Acceptance criteria:

- End-to-end scan works for both languages.
- No changes to EvidenceSelector, LLM pipeline, evaluator, or dashboard.

### Feature 8.4: Bitbucket Source Adapter Skeleton

Scope:

- Define Bitbucket archive and clone adapter config.
- Add placeholder implementation conforming to `RepoSourceAdapter` interface.

```text
PR-042: Add Bitbucket source adapter skeleton
```

Acceptance criteria:

- Future integration path is clear.
- Core scan engine does not require changes.

## 14. Suggested Timeline

This is a rough plan for a 2-person team (tech lead + junior engineer).

### Weeks 1–2: Local Static Scan Foundation

Target:

- CLI scan works.
- Local upload adapter works.
- Workspace manager works.
- One language scanner produces normalized findings.
- Finding normalization complete.

Key PRs:

```text
PR-001 to PR-009
```

End of week 2 demo:

- Run CLI on sample repo.
- Produce findings.json with normalized findings.

### Weeks 3–4: Evidence Package + LLM Pipeline

Target:

- Evidence package built with EvidenceSelector and token budget.
- ThemeExtractor, ThemeEvidenceValidator, and RecommendationGenerator work.
- ReportAssembler produces structured report.
- Hard gate evaluator catches invalid output.
- Retry/fallback flow works.

Key PRs:

```text
PR-010 to PR-025
```

End of week 4 demo:

- Run full scan with LLM.
- Show accepted report and rejected/retry scenario.
- Evidence package token count logged.

### Weeks 5–6: API + Dashboard + Scoring

Target:

- FastAPI endpoints.
- Streamlit upload and report pages.
- Both scores computed and displayed.
- Story-mode demo flow working.

Key PRs:

```text
PR-026 to PR-034
```

End of week 6 demo:

- Upload repo through UI.
- View report with both scores.
- Walk through story-mode demo narrative.

### Weeks 7–8: Containerization + Demo Hardening

Target:

- Dockerized demo.
- Demo dataset (synthetic Java or Python repo).
- Preloaded demo scan for story mode.
- Structured logging.
- Enterprise adapter skeletons.

Key PRs:

```text
PR-035 to PR-042
```

End of week 8 demo:

- One-command local demo.
- Leadership demo narrative is rehearsed and presentable.

## 15. Definition of Done for Demo

The demo is complete when:

- A user can upload a repo archive.
- The platform runs a scan for the MVP language.
- Static findings are normalized.
- EvidenceSelector selects and ranks evidence within the token budget.
- Evidence package is bounded and reproducible.
- ThemeExtractor identifies quality themes with evidence refs.
- ThemeEvidenceValidator catches invalid references.
- RecommendationGenerator produces specific, targeted recommendations.
- ReportAssembler assembles the final report deterministically.
- Hard gate evaluator rejects hallucinated references and triggers repair.
- Retry and fallback are functional.
- Final report shows Code Quality Score and LLM Report Quality Score separately.
- Dashboard displays the story-mode demo flow.
- Evidence trail from recommendation to source snippet is navigable.
- The app runs locally or in Docker.
- Core scan engine is not tied to Step Functions.

## 16. PR Review Checklist

Every PR should answer:

- Is this feature small and reviewable?
- Does it preserve adapter boundaries (scanner, source, storage, LLM)?
- Does it add or update tests?
- Does it avoid logging source code unnecessarily?
- Does it store version and metadata where relevant?
- Does it keep LLM output separate from the validated report?
- Does it fail safely and produce a diagnosable error?
- Does it keep local execution working?
- If it touches scoring: is the score still deterministic?
- If it touches the LLM pipeline: does it go through the hard gate evaluator?

## 17. Technical Debt to Avoid in Demo

Avoid:

- Hardcoding local paths in core logic.
- Mixing dashboard code with scan engine logic.
- Letting LLM compute the final score.
- Accepting LLM output without passing the hard gate evaluator.
- Using LLM for evidence reference or file path checks.
- Building Bitbucket integration before the core scan pipeline is stable.
- Adding a second language scanner before the scanner adapter interface is proven.
- Designing for EKS or Step Functions before local container execution works.
- Estimating LLM cost in design documents — measure it instead.

## 18. Deferred Enhancements (Phase 2)

After the demo, the following are the prioritized next steps:

- Integrate SonarQube as the first enterprise scanner adapter.
- Add Bitbucket archive download source adapter.
- Add second language scanner.
- Add scheduled scan support.
- Add churn × complexity temporal signal (requires Bitbucket or git history).
- Add LLM-as-judge soft evaluator for actionability and clarity.
- Add role-specific SummaryGenerator LLM step.
- Add scan history trend analysis.
- Add CodeQL, Checkmarx, Snyk, Veracode adapters.
- Add embedding-based evidence retrieval.
- Add portfolio-level multi-repo analysis.
- Add Jira ticket generation for accepted refactor backlog.
- Add offline regression evaluation dataset and human feedback loop.
