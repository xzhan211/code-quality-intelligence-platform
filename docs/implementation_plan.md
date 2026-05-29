# Code Quality Platform - Implementation Plan

## 1. Purpose

This document breaks the first demo version into implementation phases and small features suitable for PR/commit tracking.

The goal is to help a 3-engineer team develop the project efficiently while keeping architecture, evaluation, and traceability under control.

## 2. Team Assumptions

Team size:

- 3 software engineers.
- 1 manager/tech lead guiding architecture, scope, and review.

Project stage:

- First version is a demo.
- Initial repo input can be local upload.
- First scan target is one medium/small microservice.
- Main languages are Java and Python.
- Results are advisory only and do not block release.

## 3. Workstream Ownership

## Engineer 1: Platform / Runtime / API

Primary focus:

- Project skeleton.
- CLI entrypoint.
- FastAPI service.
- Upload handling.
- Scan run lifecycle.
- Local storage.
- Dockerization.

## Engineer 2: Static Analysis / Evidence

Primary focus:

- Workspace preparation.
- Java/Python file inventory.
- Local metrics extraction.
- Scanner adapter abstraction.
- Finding normalization.
- Evidence package builder.

## Engineer 3: LLM / Evaluation / Dashboard

Primary focus:

- LLM prompt/schema.
- LLM client abstraction.
- LLM output evaluator.
- Retry/fallback flow.
- Streamlit dashboard.
- Report rendering.

## Tech Lead / Manager

Primary focus:

- Scope control.
- Architecture review.
- Evaluation standard.
- PR review strategy.
- Demo narrative.
- Coordination with internal scanner/Bedrock/Bitbucket stakeholders.

## 4. Development Principles

- Keep each PR small and reviewable.
- Do not mix infrastructure, scanner logic, LLM logic, and dashboard changes in the same PR unless necessary.
- Add tests for core logic before dashboard polish.
- Treat evaluation as a core feature, not a later enhancement.
- Use interfaces/adapters even if the first implementation is local-only.
- Store raw outputs and metadata for traceability.
- Do not let invalid LLM output reach the final report.

## 5. Phase 0: Project Foundation

Goal:

Create a clean project skeleton that supports local execution, tests, and future adapters.

### Feature 0.1: Repository Skeleton

Scope:

- Create project folder structure.
- Add README.
- Add basic package setup.
- Add lint/test commands.

Deliverables:

```text
app/
cli/
dashboard/
tests/
docker/
README.md
```

Suggested PR:

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
- ScanRun.
- Finding.
- EvidencePackage.
- LLMReport.
- EvaluationResult.

Suggested PR:

```text
PR-002: Add core domain models for scan, finding, evidence, and evaluation
```

Acceptance criteria:

- Models are serializable.
- Basic unit tests exist.
- No runtime-specific dependencies inside models.

### Feature 0.3: Configuration System

Scope:

- App config.
- Scan limits.
- LLM config placeholders.
- Storage mode config.

Suggested PR:

```text
PR-003: Add configuration loading and scan defaults
```

Acceptance criteria:

- Local config loads from env/file.
- Defaults are documented.

## 6. Phase 1: Local Scan Engine MVP

Goal:

Run a local scan from a repo archive and produce normalized findings without LLM.

### Feature 1.1: CLI Scan Entrypoint

Scope:

Add command:

```bash
code-quality scan --input ./repo.zip --repo-name demo-service --tenant demo
```

Suggested PR:

```text
PR-004: Add CLI entrypoint for local scan execution
```

Acceptance criteria:

- CLI accepts input path, repo name, tenant.
- Creates a scan_run object.
- Writes basic scan metadata to output folder.

### Feature 1.2: Local Repo Source Adapter

Scope:

- Define `RepoSourceAdapter` interface.
- Implement `LocalUploadSourceAdapter`.
- Return normalized `RepoArtifact`.

Suggested PR:

```text
PR-005: Implement local repo source adapter
```

Acceptance criteria:

- Zip/tar.gz path is accepted.
- Invalid file type fails clearly.
- Adapter output is independent of runtime.

### Feature 1.3: Workspace Manager

Scope:

- Unpack archive.
- Validate size/file count.
- Build file inventory.
- Exclude generated/binary folders.
- Detect Java/Python files.

Suggested PR:

```text
PR-006: Add workspace preparation and file inventory
```

Acceptance criteria:

- Archive is extracted to isolated workspace.
- Excluded folders are ignored.
- File inventory includes path, extension, size, language.
- Temporary workspace can be cleaned up.

### Feature 1.4: Scanner Adapter Interface

Scope:

- Define `ScannerAdapter` interface.
- Define raw scanner result and normalized finding contracts.
- Add scanner suite runner.

Suggested PR:

```text
PR-007: Add scanner adapter interface and scanner suite runner
```

Acceptance criteria:

- Scanner suite can run multiple scanner adapters.
- Individual scanner failure is captured.
- No actual scanner implementation required beyond stub.

### Feature 1.5: Python Metrics Scanner

Scope:

- Basic Python file/function metrics.
- Detect long functions.
- Detect large files.
- Optional cyclomatic complexity if lightweight library is approved.

Suggested PR:

```text
PR-008: Add local Python metrics scanner
```

Acceptance criteria:

- Produces normalized findings.
- Handles syntax errors gracefully.
- Unit tests use small sample Python files.

### Feature 1.6: Java Metrics Scanner

Scope:

- Basic Java file/class/method metrics.
- Detect long files/classes/methods.
- Optional lightweight parsing approach if approved.

Suggested PR:

```text
PR-009: Add local Java metrics scanner
```

Acceptance criteria:

- Produces normalized findings.
- Handles parsing limitations gracefully.
- Unit tests use small sample Java files.

### Feature 1.7: Finding Normalization

Scope:

- Normalize all scanner outputs into common finding schema.
- Assign categories and severity.
- Generate evidence IDs.

Suggested PR:

```text
PR-010: Normalize scanner findings into common schema
```

Acceptance criteria:

- Findings are valid JSON.
- Each finding has evidence_ref.
- Severity/category mapping is deterministic.

## 7. Phase 2: Evidence Package and Static Report

Goal:

Build a structured evidence package and generate static-only report.

### Feature 2.1: Repo Summary Builder

Scope:

- Generate repo tree summary.
- Language breakdown.
- Top large files.
- Test file detection.

Suggested PR:

```text
PR-011: Add repository summary builder
```

Acceptance criteria:

- Summary works for Java/Python mixed repo.
- Output is deterministic.

### Feature 2.2: Snippet Selector

Scope:

- Select snippets from top findings.
- Limit snippet size.
- Preserve file path and line ranges.
- Avoid binary/generated files.

Suggested PR:

```text
PR-012: Add evidence snippet selector
```

Acceptance criteria:

- Snippets include valid file paths.
- Line ranges are valid.
- Total evidence size is bounded.

### Feature 2.3: Evidence Package Builder

Scope:

- Combine repo summary, findings, snippets, unknowns, and metadata.
- Store evidence package JSON.

Suggested PR:

```text
PR-013: Build evidence package for LLM assessment
```

Acceptance criteria:

- Evidence package is schema-valid.
- Every snippet/finding has stable evidence ID.
- Evidence package can be reproduced from scan result.

### Feature 2.4: Static Report Generator

Scope:

- Generate static-only JSON and Markdown report.
- Include top findings and simple score.

Suggested PR:

```text
PR-014: Generate static-only quality report
```

Acceptance criteria:

- Report exists even without LLM.
- Report includes score, findings, hotspots, unknowns.

## 8. Phase 3: LLM Assessment

Goal:

Use LLM to generate evidence-grounded code quality assessment.

### Feature 3.1: LLM Report Schema

Scope:

Define structured output schema:

- Overall assessment.
- Quality themes.
- Recommended backlog.
- Unknowns.
- Metadata.

Suggested PR:

```text
PR-015: Define LLM report schema and validation models
```

Acceptance criteria:

- Schema supports evidence_refs.
- Unit tests validate good/bad examples.

### Feature 3.2: Prompt and Rubric v1

Scope:

- Write system prompt.
- Write assessment rubric.
- Define severity rules.
- Define recommendation format.

Suggested PR:

```text
PR-016: Add prompt and rubric v1 for evidence-grounded assessment
```

Acceptance criteria:

- Prompt instructs model to use only provided evidence.
- Prompt requires structured JSON.
- Rubric version is included in output metadata.

### Feature 3.3: LLM Client Interface

Scope:

- Define `LLMClient` interface.
- Add mock LLM client for tests.
- Add Bedrock-compatible client placeholder or implementation depending on environment access.

Suggested PR:

```text
PR-017: Add LLM client abstraction and mock client
```

Acceptance criteria:

- Core code can run with mock LLM.
- Provider-specific code is isolated.

### Feature 3.4: LLM Assessor

Scope:

- Build prompt from evidence package.
- Call LLM client.
- Parse output.
- Store raw output.

Suggested PR:

```text
PR-018: Implement LLM assessor using evidence package
```

Acceptance criteria:

- Assessor returns raw and parsed report.
- Prompt/model/rubric versions are stored.
- Works with mock client in tests.

## 9. Phase 4: LLM Evaluation and Retry

Goal:

Prevent invalid LLM output from becoming accepted report.

### Feature 4.1: Schema Validator

Scope:

- Validate JSON parse.
- Validate required fields.
- Validate enum values.
- Validate score/confidence ranges.

Suggested PR:

```text
PR-019: Add LLM output schema evaluator
```

Acceptance criteria:

- Invalid JSON fails.
- Missing fields fail.
- Unit tests cover failure cases.

### Feature 4.2: Evidence Reference Validator

Scope:

- Validate evidence_refs exist.
- Validate referenced file paths exist.
- Validate line ranges when available.

Suggested PR:

```text
PR-020: Add evidence grounding evaluator
```

Acceptance criteria:

- Missing evidence refs fail.
- Non-existent files fail.
- Valid report passes.

### Feature 4.3: Consistency Evaluator

Scope:

- Check risk level vs severity distribution.
- Check score vs summary severity.
- Check high severity evidence requirements.

Suggested PR:

```text
PR-021: Add severity and score consistency evaluator
```

Acceptance criteria:

- Obvious inconsistencies fail or warn.
- Rules are documented.

### Feature 4.4: Actionability Evaluator

Scope:

- Score recommendation specificity.
- Detect vague recommendations.
- Mark report warning if actionability is low.

Suggested PR:

```text
PR-022: Add recommendation actionability evaluator
```

Acceptance criteria:

- Vague recommendations are detected.
- Actionability score is stored.

### Feature 4.5: Retry and Repair Prompt Flow

Scope:

- Add max retry config.
- Add targeted repair prompts.
- Store retry attempts and reasons.
- Fallback to static-only report if retries fail.

Suggested PR:

```text
PR-023: Add LLM retry and fallback workflow
```

Acceptance criteria:

- Failed schema triggers schema repair.
- Failed grounding triggers grounding repair.
- Max retry is enforced.
- Static report is returned if LLM fails.

### Feature 4.6: Evaluation Metadata Storage

Scope:

- Store evaluation result with scan.
- Include final status, scores, retry count, failure reasons.

Suggested PR:

```text
PR-024: Persist LLM evaluation metadata
```

Acceptance criteria:

- Evaluation metadata appears in final report.
- Dashboard can read evaluation metadata.

## 10. Phase 5: Final Scoring and Report

Goal:

Combine static metrics, LLM assessment, and evaluation status into final user-facing report.

### Feature 5.1: Scoring Engine v1

Scope:

- Implement deterministic score formula.
- Compute score breakdown.
- Handle missing metrics.
- Reduce confidence when LLM failed or evidence incomplete.

Suggested PR:

```text
PR-025: Add deterministic code quality scoring engine v1
```

Acceptance criteria:

- Score is reproducible.
- Score breakdown is included.
- LLM does not dominate score.

### Feature 5.2: Final Report Generator

Scope:

- Combine static report, LLM report, score, and evaluation result.
- Generate final JSON and Markdown.

Suggested PR:

```text
PR-026: Generate final quality report with evaluation status
```

Acceptance criteria:

- Report includes code quality score.
- Report includes LLM report quality/confidence.
- Report includes evidence references.
- Report includes unknowns and limitations.

### Feature 5.3: Report Export

Scope:

- Allow report download from local output/API.
- Include raw JSON artifacts.

Suggested PR:

```text
PR-027: Add report export and raw artifact access
```

Acceptance criteria:

- User can inspect final report and raw artifacts.

## 11. Phase 6: API and Dashboard

Goal:

Provide a simple user interface for demo.

### Feature 6.1: FastAPI Scan API

Scope:

- Upload endpoint.
- Trigger scan endpoint.
- Get scan status endpoint.
- Get report endpoint.

Suggested PR:

```text
PR-028: Add FastAPI endpoints for upload, scan, and report retrieval
```

Acceptance criteria:

- Upload creates scan run.
- Scan can be triggered.
- Report can be retrieved.

### Feature 6.2: Background Scan Execution

Scope:

- API triggers scan asynchronously or as controlled background task.
- Status is updated.

Suggested PR:

```text
PR-029: Add background scan execution and status tracking
```

Acceptance criteria:

- UI/API can observe scan progress.
- Failed stages show clear error.

### Feature 6.3: Streamlit Upload Page

Scope:

- Upload repo archive.
- Enter tenant/repo metadata.
- Trigger scan.

Suggested PR:

```text
PR-030: Add Streamlit upload and scan trigger page
```

Acceptance criteria:

- User can start scan from UI.

### Feature 6.4: Streamlit Report Page

Scope:

- Display score and risk level.
- Show top quality themes.
- Show findings.
- Show refactor backlog.
- Show LLM evaluation status.

Suggested PR:

```text
PR-031: Add Streamlit quality report dashboard
```

Acceptance criteria:

- Dashboard is useful for engineer/manager/architecture reviewer.
- Evaluation status is visible.

### Feature 6.5: Scan History Page

Scope:

- List previous scans.
- Show scan status and created time.
- Link to report.

Suggested PR:

```text
PR-032: Add scan history dashboard page
```

Acceptance criteria:

- User can view prior scan results.

## 12. Phase 7: Containerization and Demo Hardening

Goal:

Make the demo easy to run and review.

### Feature 7.1: Dockerfile

Scope:

- Build application container.
- Include CLI/API runtime.
- Document local volume usage.

Suggested PR:

```text
PR-033: Add Dockerfile for local container execution
```

Acceptance criteria:

- Container can run CLI scan.
- Container can run API server.

### Feature 7.2: Docker Compose

Scope:

- Run API + dashboard + local storage.
- Optional SQLite volume.

Suggested PR:

```text
PR-034: Add docker-compose for local demo
```

Acceptance criteria:

- One command starts local demo.

### Feature 7.3: Demo Dataset

Scope:

- Add small sample Java/Python repos or synthetic examples.
- Include examples for high complexity, duplication, and clean code.

Suggested PR:

```text
PR-035: Add demo input repos and expected report examples
```

Acceptance criteria:

- Demo can run without company code.
- Evaluation behavior can be shown.

### Feature 7.4: Observability Basics

Scope:

- Structured logs.
- Stage duration.
- Error reasons.
- Retry count.

Suggested PR:

```text
PR-036: Add structured logging and scan stage timing
```

Acceptance criteria:

- Each scan stage logs start/end/status.
- Failures are diagnosable.

## 13. Phase 8: Enterprise Integration Preparation

Goal:

Prepare for company environment without blocking demo.

### Feature 8.1: SonarQube Adapter Stub

Scope:

- Add interface and config for SonarQube ingestion.
- Provide stub implementation or sample JSON parser.

Suggested PR:

```text
PR-037: Add SonarQube scanner adapter skeleton
```

Acceptance criteria:

- Adapter can parse sample SonarQube-like JSON.
- Real API integration can be added later.

### Feature 8.2: Enterprise Scanner Adapter Skeletons

Scope:

Add placeholders for:

- CodeQL.
- Checkmarx.
- Snyk.
- Veracode.

Suggested PR:

```text
PR-038: Add enterprise scanner adapter skeletons
```

Acceptance criteria:

- Adapter contracts are consistent.
- No fake results are mixed into real reports unless explicitly marked.

### Feature 8.3: Bitbucket Source Adapter Skeleton

Scope:

- Define Bitbucket archive/clone adapter config.
- Add placeholder implementation.

Suggested PR:

```text
PR-039: Add Bitbucket source adapter skeleton
```

Acceptance criteria:

- Future integration path is clear.
- Core scan engine does not need changes.

## 14. Suggested Timeline for First Demo

This is a rough plan for a focused team.

## Week 1: Local Static Scan Foundation

Target:

- CLI scan works.
- Local upload adapter works.
- Workspace manager works.
- Java/Python metrics produce findings.

Key PRs:

```text
PR-001 to PR-010
```

Demo at end of week:

- Run CLI on sample repo.
- Generate findings.json.

## Week 2: Evidence + LLM + Evaluation

Target:

- Evidence package built.
- LLM report generated.
- Evaluator validates output.
- Retry/fallback works.

Key PRs:

```text
PR-011 to PR-024
```

Demo at end of week:

- Run full scan with LLM.
- Show accepted report and failed/retry example.

## Week 3: API + Dashboard + Scoring

Target:

- FastAPI upload/scan/report endpoints.
- Streamlit dashboard.
- Final scoring/report.

Key PRs:

```text
PR-025 to PR-032
```

Demo at end of week:

- Upload repo through UI.
- View report in dashboard.

## Week 4: Containerization + Hardening + Review Prep

Target:

- Dockerized demo.
- Sample datasets.
- Logging.
- Enterprise adapter skeletons.
- Documentation cleanup.

Key PRs:

```text
PR-033 to PR-039
```

Demo at end of week:

- One-command local demo.
- Architecture review materials ready.

## 15. Definition of Done for Demo

The demo is done when:

- A user can upload a repo archive.
- The platform can run a scan for Java/Python repo.
- Static findings are normalized.
- Evidence package is generated.
- LLM report is produced.
- LLM output is evaluated.
- Retry/fallback is implemented.
- Final report shows code quality and LLM report confidence separately.
- Dashboard displays actionable recommendations.
- The app can run locally or in Docker.
- Core scan engine is not tied to Step Functions.

## 16. PR Review Checklist

Every PR should answer:

- Is this feature small and reviewable?
- Does it preserve adapter boundaries?
- Does it add or update tests?
- Does it avoid logging source code unnecessarily?
- Does it store version/metadata where relevant?
- Does it keep LLM output separate from validated report?
- Does it fail safely?
- Does it keep local execution working?

## 17. Technical Debt to Avoid in Demo

Avoid:

- Hardcoding local paths deeply in core logic.
- Mixing dashboard code with scan engine logic.
- Letting LLM directly decide final score without deterministic baseline.
- Ignoring invalid LLM output.
- Building Bitbucket integration before the core scan pipeline works.
- Adding too many open-source tools before scanner abstraction is stable.
- Designing for EKS/Step Functions before local container execution works.

## 18. Future Roadmap After Demo

After the demo, likely next steps:

- Integrate SonarQube first.
- Add Bitbucket archive download.
- Add scheduled scan.
- Add scan history and trends.
- Add team/repo multi-tenancy.
- Add CodeQL/Checkmarx/Snyk/Veracode result ingestion.
- Add architecture review workflow.
- Add Jira ticket generation for accepted refactor backlog.
- Add offline regression evaluation dataset.
- Add human feedback loop.
