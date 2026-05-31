# Code Quality Platform - Pipeline and Data Flow Details

## 1. Purpose

This document describes the end-to-end pipeline and data flow for the first demo version of the Code Quality Intelligence Platform.

The first version supports local repository upload and on-demand scanning. The architecture should remain compatible with future Bitbucket integration and AWS deployment.

## 2. Pipeline Overview

```text
User
  -> Upload Repo Archive
  -> Create Scan Run
  -> Prepare Workspace
  -> Run Static Analysis
  -> Normalize Findings
  -> EvidenceSelector (rank, select, enforce token budget)
  -> Build Evidence Package
  -> [Multi-Step LLM Pipeline]
       Step 1: ThemeExtractor (LLM call, structured output via tool use)
       Step 2: ThemeEvidenceValidator (deterministic)
       Step 3: RecommendationGenerator (LLM call, per validated theme)
       Step 4: ReportAssembler (deterministic, no LLM)
  -> LLM Output Evaluator (deterministic hard gates)
  -> Retry / Accept / Fallback
  -> Compute Final Score (deterministic)
  -> Generate Report
  -> Store Results
  -> Show Dashboard
```

### Design Principle

```text
The product is AI-assisted but engineering-controlled.

LLM interprets, synthesizes, and recommends.
Deterministic components validate, score, and assemble.
Every conclusion must be grounded in evidence and pass validation
before appearing in the final report.
```

## 3. Runtime Modes

## 3.1 Local CLI Mode

Used for developer testing and quick demo iteration.

Example:

```bash
code-quality scan --input ./repo.zip --repo-name payment-service --tenant demo
```

Output:

```text
output/
  scan_result.json
  findings.json
  evidence_package.json
  llm_report.json
  evaluation_result.json
  final_report.json
  report.md
```

## 3.2 Docker Mode

Used for reproducible local execution.

Example:

```bash
docker run \
  -v ./input:/input \
  -v ./output:/output \
  -e RUN_MODE=local \
  code-quality-platform:latest \
  scan --input /input/repo.zip --repo-name payment-service --tenant demo
```

## 3.3 API Mode

Used by dashboard or internal tools.

Endpoints:

```text
POST /api/repos/upload
POST /api/scans
GET  /api/scans/{scan_id}
GET  /api/scans/{scan_id}/report
GET  /api/repos/{repo_id}/history
```

## 3.4 Future AWS Mode

In AWS, Step Functions can orchestrate ECS tasks, but the scan engine remains the same executable logic.

```text
EventBridge / Manual API Trigger
  -> Step Functions
  -> ECS Fargate worker running code-quality scan --scan-id xxx
  -> S3 + RDS
```

## 4. Detailed Pipeline Stages

## Stage 1: Repository Upload

### Input

- `.zip` or `.tar.gz` repository archive.
- Tenant name.
- Repo name.
- Optional branch.
- Optional commit SHA.
- Optional description.

### Processing

- Validate file extension.
- Validate size limit.
- Store raw artifact.
- Create repository record if needed.
- Create scan run record.

### Output

```text
RepoArtifact
- artifact_id
- source_type = LOCAL_UPLOAD
- artifact_uri
- repo_name
- tenant_id
- branch optional
- commit_sha optional
```

### Notes

The first demo can use local filesystem storage. Future enterprise version should use S3.

## Stage 2: Scan Run Creation

### Input

- Repo artifact.
- User request.
- Scan config.

### Processing

Create `scan_run` record.

Initial status:

```text
CREATED
```

### Output

```text
scan_run_id
```

Recommended statuses:

```text
CREATED
SOURCE_FETCHED
WORKSPACE_PREPARED
STATIC_SCAN_STARTED
STATIC_SCAN_COMPLETED
EVIDENCE_BUILT
LLM_ASSESSMENT_STARTED
LLM_OUTPUT_RECEIVED
LLM_OUTPUT_EVALUATED
RETRY_REQUIRED
REPORT_GENERATED
COMPLETED
COMPLETED_WITH_WARNING
FAILED_FALLBACK_TO_STATIC_REPORT
FAILED
```

## Stage 3: Workspace Preparation

### Input

- Repo archive artifact.

### Processing

- Download or read artifact.
- Unzip into isolated workspace.
- Validate file count and size.
- Remove or ignore generated/binary folders.
- Detect languages.
- Build repo tree summary.

### Recommended Exclusions

```text
.git
node_modules
target
build
dist
.venv
__pycache__
.m2
.gradle
.idea
.vscode
```

### Output

```text
Workspace
- workspace_id
- local_path
- detected_languages
- repo_tree
- file_inventory
```

## Stage 4: Static Analysis

### Input

- Workspace.
- Scan config.

### Demo Scanner Set

For MVP:

- Local Java metrics scanner.
- Local Python metrics scanner.
- Optional Semgrep scanner.

Future enterprise adapters:

- SonarQube.
- CodeQL.
- Checkmarx.
- Snyk.
- Veracode.

### Processing

Each scanner returns raw findings. Raw outputs are stored for traceability.

### Output

```text
RawScannerResult[]
```

Example:

```json
{
  "source_tool": "local_python",
  "raw_output_uri": "...",
  "status": "SUCCESS",
  "summary": {
    "files_scanned": 43,
    "findings_count": 25
  }
}
```

## Stage 5: Finding Normalization

### Input

- Raw scanner outputs.

### Processing

Convert scanner-specific outputs into a common finding schema.

### Normalized Finding Schema

```json
{
  "finding_id": "finding-001",
  "source_tool": "local_python",
  "category": "complexity",
  "severity": "medium",
  "file_path": "src/payment/service.py",
  "class_name": null,
  "function_name": "process_payment",
  "line_start": 45,
  "line_end": 130,
  "metric_name": "cyclomatic_complexity",
  "metric_value": 18,
  "message": "Function has high complexity",
  "evidence_ref": "finding:finding-001"
}
```

### Output

```text
NormalizedFinding[]
```

## Stage 6: Evidence Package Build (EvidenceSelector + Evidence Builder)

### Input

- Normalized findings.
- Workspace metadata.
- Repo tree.
- Scan config (including max_evidence_tokens).

### Stage 6a: EvidenceSelector

The EvidenceSelector ranks files and functions by importance and enforces the token budget before building the evidence package.

#### File Importance Scoring (MVP)

```text
file_importance_score =
  0.5 * normalized_complexity
+ 0.3 * normalized_finding_count
+ 0.2 * normalized_file_size
```

Each factor is normalized to [0, 1] across all files in the repo. The top-N files by this score are selected for snippet inclusion.

#### Token Budget Enforcement

A hard token limit is applied before the evidence package is sent to any LLM step:

```text
max_evidence_tokens = configurable (default: 80,000)
```

If the selected evidence exceeds the budget, lower-ranked findings and snippets are dropped until the package fits. The final token count is logged per scan.

#### Snippet Selection Rules

Prefer snippets from:

- Top-ranked files by importance score.
- Functions flagged by complexity findings.
- Files with multiple findings.

Avoid:

- Binary files.
- Generated files.
- Secrets or credentials when detected.
- Very large files without specific selection.

#### Phase 2 Enhancements (Deferred)

- File centrality from import/dependency graph.
- Churn × complexity signal from git history.
- Embedding-based evidence retrieval.

### Stage 6b: Evidence Builder

Combines EvidenceSelector output with repo metadata into the final evidence package.

### Evidence Package Schema

```json
{
  "evidence_package_id": "evidence-001",
  "scan_run_id": "scan-001",
  "repo": {
    "tenant": "demo",
    "repo_name": "payment-service",
    "languages": ["Java"],
    "branch": "main",
    "commit_sha": null
  },
  "repo_tree_summary": [],
  "metrics_summary": {},
  "findings": [],
  "snippets": [],
  "unknowns": [],
  "token_count": 42000,
  "versions": {
    "evidence_builder_version": "v1.0",
    "evidence_selector_version": "v1.0"
  }
}
```

### Output

```text
EvidencePackage (bounded, schema-valid, reproducible)
```

## Stage 7: LLM Assessment (Multi-Step Pipeline)

A single large LLM call is replaced by a 4-step pipeline. Each LLM step is smaller, focused, and validated before its output is used in the next step.

### Step 7a: ThemeExtractor (LLM Call)

#### Input

- Evidence package.
- ThemeExtractor prompt (versioned).
- Model config (Claude Bedrock, structured output via tool use).

#### Processing

The LLM receives the evidence package and identifies the top 3–5 quality themes. Structured output (tool use) is used to enforce schema at the API level.

LLM responsibilities in this step:
- Identify the top quality themes from the evidence.
- For each theme, provide a concise rationale citing supporting evidence.
- Each theme must reference at least one `evidence_ref` from the evidence package.

The LLM must not invent file names, class names, or function names that are not in the evidence package.

#### Output

```json
{
  "themes": [
    {
      "theme_id": "theme-001",
      "title": "High-complexity payment processing logic",
      "severity": "high",
      "evidence_refs": ["finding:finding-005", "snippet:snippet-002"],
      "rationale": "PaymentService.process_payment has cyclomatic complexity of 18 ..."
    }
  ]
}
```

### Step 7b: ThemeEvidenceValidator (Deterministic)

#### Processing

Validates each theme from the ThemeExtractor output:

- All `evidence_refs` cited in each theme exist in the evidence package.
- Referenced file paths exist in the workspace.
- Referenced class or function names exist in the normalized findings.
- No hallucinated identifiers are present.
- Required theme fields are present.

Invalid themes are rejected. A targeted grounding repair prompt is sent if any themes fail (up to max retry). If retry fails, the scan falls back to static-only report.

#### Output

```text
ValidatedThemes[]
```

### Step 7c: RecommendationGenerator (LLM Call)

#### Input

- Validated themes.
- RecommendationGenerator prompt (versioned).
- Model config.

#### Processing

For each validated theme, the LLM generates a specific engineering recommendation.

Requirements:
- Target a specific module, file, or function.
- Describe the specific change recommended.
- Explain the expected benefit.
- Do not introduce new identifiers not present in the validated theme evidence.

#### Output

```json
{
  "recommendations": [
    {
      "theme_id": "theme-001",
      "target": "PaymentService.process_payment",
      "recommended_change": "Extract validation logic into PaymentValidationPolicy ...",
      "expected_benefit": "Reduces cyclomatic complexity from 18 to under 10 ...",
      "priority": "high"
    }
  ]
}
```

### Step 7d: ReportAssembler (Deterministic)

Assembles the final structured LLM report from validated themes and recommendations. No LLM call is made in this step.

- Combines validated themes with their recommendations.
- Attaches evidence package metadata.
- Records prompt versions, model ID, and LLM usage metadata (input tokens, output tokens per step).
- Produces a structured report ready for the evaluator.

The LLM does not compute the final Code Quality Score. Score computation happens in Stage 10.

#### Output

```text
AssembledLLMReport (structured, deterministic)
```

## Stage 8: LLM Output Evaluation

### Input

- Assembled LLM report (from ReportAssembler).
- Evidence package.
- Repo workspace.
- Evaluation config.

### Processing: Hard Gate Checks (Deterministic — Must Pass)

These checks are implemented in code and must not use an LLM:

- Schema is valid and required fields are present.
- Evidence references cited in themes exist in the evidence package.
- Referenced file paths exist in the repo workspace.
- Referenced line ranges are valid when provided.
- No hallucinated file, class, or function names appear in the assembled report.
- Final assembled structure can be stored and rendered without error.

If any hard gate fails, a targeted repair prompt is triggered (up to max retry). If retries are exhausted, fall back to static-only report.

### Processing: Soft Quality Checks (Deterministic Heuristics)

These are computed and stored but do not block report acceptance:

- Grounding score: percentage of themes with at least one valid evidence reference.
- Consistency score: does risk level match severity distribution?
- Actionability score: deterministic check on recommendation field completeness (target module, specific change, expected benefit present).
- Evidence coverage: percentage of quality themes that are grounded.
- Unknowns coverage: are missing inputs explicitly listed?

### Note on Structured Output and Semantic Validation

Using Claude Bedrock tool use reduces JSON formatting failures. It does not eliminate the need for semantic validation. Evidence references, file path existence, and claim scope must still be validated by the hard gate evaluator.

```text
Tool use reduces format failures.
Semantic validation and repair are still required.
```

### Output Statuses

```text
ACCEPTED
ACCEPTED_WITH_WARNING
RETRY_REQUIRED
FAILED_FALLBACK_TO_STATIC_REPORT
```

### Output

```json
{
  "final_status": "ACCEPTED",
  "schema_valid": true,
  "grounding_score": 0.93,
  "consistency_score": 0.88,
  "actionability_score": 0.81,
  "hallucination_count": 0,
  "retry_count": 0,
  "failure_reasons": [],
  "llm_usage": [
    {"step": "theme_extractor", "model_id": "...", "input_tokens": 8200, "output_tokens": 420},
    {"step": "recommendation_generator", "model_id": "...", "input_tokens": 3100, "output_tokens": 680}
  ]
}
```

## Stage 9: Retry or Fallback

### Retry Conditions

Retry when:

- JSON parse fails.
- Schema validation fails.
- Evidence references are missing.
- Referenced files do not exist.
- Risk/severity/score conflict.
- Report is empty or unusable.

### Retry Strategy

Use targeted repair prompt:

- Schema repair.
- Grounding repair.
- Consistency repair.
- Actionability repair.

### Max Retry

```text
max_retry = 2
```

### Fallback

If LLM output remains invalid:

- Generate static-only report.
- Mark LLM report as failed.
- Store failure reasons.
- Scan should still complete if static analysis succeeded.

## Stage 10: Final Scoring (Deterministic)

The scoring engine computes two separate scores. Both are fully deterministic. The LLM does not compute either score.

### Input

- Normalized findings.
- Static metrics.
- Validation result from Stage 8.
- LLM report (accepted or fallback).

### Code Quality Score

Measures the quality and risk level of the repository.

```text
Code Quality Score =
  25% Maintainability (from static metrics)
+ 20% Complexity (from scanner findings)
+ 15% Duplication (from scanner findings)
+ 15% Test Coverage or Test Presence Proxy
+ 15% Risk Signals (from normalized findings)
+ 10% Architecture/Design Signal (from validated LLM themes, advisory only)
```

If the LLM report failed, exclude the Architecture/Design Signal component and reduce score confidence accordingly.

If test coverage is unavailable, use a test presence proxy and mark the limitation explicitly in the report.

### LLM Report Quality Score

Measures how trustworthy the LLM-generated report is.

```text
LLM Report Quality Score =
  30% Grounding
+ 20% Schema Correctness
+ 20% Consistency
+ 20% Actionability
+ 10% Conciseness / Clarity
```

### Output

```text
FinalScore (Code Quality Score)
LLMReportQualityScore
RiskLevel
ScoreBreakdown
ScoreConfidence
```

## Stage 11: Report Generation

### Input

- Final score.
- Findings.
- LLM report.
- Evaluation result.
- Scan metadata.

### Output Files

```text
final_report.json
report.md
report.html optional
```

### Report Sections

- Executive summary.
- Score breakdown.
- Risk level.
- Top quality themes.
- Hotspot files.
- Detailed findings.
- Recommended refactor backlog.
- Evidence references.
- Unknowns and limitations.
- LLM evaluation status.
- Prompt/rubric/model versions.

## Stage 12: Dashboard

### Input

- Stored scan results.

### Pages

```text
Upload / Trigger Scan
Scan History
Repo Overview
Quality Themes
Findings
Recommended Backlog
LLM Evaluation
Raw Results
```

### Important Dashboard Signals

Show both scores separately on every report page:

```text
Code Quality Score: 72 / 100
Risk Level: Medium
---
LLM Report Quality Score: 88 / 100
LLM Report Confidence: High
Validation Status: Passed
Evidence Coverage: 91%
Retry Count: 0
Prompt Version: v1.0
Model: claude-3-5-sonnet
```

### Story-Mode Demo Flow (Leadership Demo)

The dashboard should support a guided demo narrative for leadership presentations. The demo should show the following sequence using a synthetic repo with intentional quality problems (e.g., a Java or Python payment-service):

1. Upload the repo archive and trigger a scan.
2. Show static findings and metrics (complexity hotspots, large files, findings count).
3. Show the EvidenceSelector output — which files were selected and why (importance scores).
4. Show the ThemeExtractor output — what quality themes the LLM identified, with evidence refs.
5. Show the evaluator in action — demonstrate a rejected output: for example, the LLM hallucinated a class name that does not exist. Show that the evaluator caught it and triggered a repair prompt.
6. Show the accepted report after repair: themes, recommendations with evidence refs, score breakdown.
7. Show both Code Quality Score and LLM Report Quality Score separately.
8. Show a recommendation with its evidence trail — click from recommendation to the finding to the source snippet.

The goal is to show leadership that this is not a black-box AI wrapper. It is an evidence-grounded engineering quality platform where every recommendation can be traced back to the source code and every LLM output passes deterministic validation.

## 5. Data Flow by Artifact

## 5.1 Repo Archive

```text
User Upload -> Artifact Storage -> Workspace Manager -> Scanner Suite
```

Persistence:

- Demo: local filesystem.
- Future: S3.

Retention:

- Demo decision required.
- Future should support configurable retention.

## 5.2 Raw Scanner Output

```text
Scanner Adapter -> Raw Result Storage -> Normalizer
```

Purpose:

- Debugging.
- Traceability.
- Re-running normalization without scanning again.

## 5.3 Normalized Findings

```text
Normalizer -> Database -> Evidence Builder -> Dashboard
```

Purpose:

- Unified view across scanners.
- Score calculation.
- LLM input.

## 5.4 Evidence Package

```text
Evidence Builder -> LLM Assessor -> Evaluator
```

Purpose:

- Controlled LLM input.
- Grounding validation.
- Reproducibility.

## 5.5 LLM Raw Output

```text
LLM Assessor -> Raw Output Storage -> Parser/Evaluator
```

Purpose:

- Debugging.
- Audit.
- Prompt improvement.

## 5.6 Evaluation Result

```text
Evaluator -> Database -> Dashboard
```

Purpose:

- Trust indicator.
- Retry/fallback decision.
- Regression analysis.

## 5.7 Final Report

```text
Report Generator -> Storage -> Dashboard/API
```

Purpose:

- User-facing output.
- Exportable review artifact.

## 6. State Management

Recommended scan statuses:

```text
CREATED
SOURCE_FETCHED
WORKSPACE_PREPARED
STATIC_SCAN_STARTED
STATIC_SCAN_COMPLETED
NORMALIZATION_COMPLETED
EVIDENCE_BUILT
LLM_ASSESSMENT_STARTED
LLM_OUTPUT_RECEIVED
LLM_OUTPUT_EVALUATED
LLM_RETRY_STARTED
LLM_ACCEPTED
LLM_ACCEPTED_WITH_WARNING
LLM_FAILED
SCORING_COMPLETED
REPORT_GENERATED
COMPLETED
COMPLETED_WITH_WARNING
FAILED_FALLBACK_TO_STATIC_REPORT
FAILED
```

## 7. Error Handling

## 7.1 Upload Failure

Examples:

- Unsupported format.
- File too large.
- Archive cannot be opened.

User result:

- Scan not created or marked failed early.

## 7.2 Workspace Failure

Examples:

- Archive corrupt.
- Too many files.
- Workspace preparation timeout.

User result:

- Scan failed with clear reason.

## 7.3 Scanner Failure

Examples:

- Scanner command fails.
- Language unsupported.
- Timeout.

User result:

- If all scanners fail, scan fails.
- If some scanners fail, continue with warning.

## 7.4 LLM Failure

Examples:

- Provider unavailable.
- Invalid output.
- Hallucinated references.
- Repeated validation failure.

User result:

- Fallback to static-only report.
- Mark LLM status as failed.

## 8. Observability

Minimum logs:

- scan_run_id.
- stage name.
- duration.
- status.
- error reason.
- scanner versions.
- LLM model/prompt/rubric version.
- retry count.

Minimum metrics:

- Scan success rate.
- Scanner failure rate.
- LLM validation failure rate.
- Average scan duration.
- Average LLM latency.
- Retry count distribution.
- Fallback rate.

## 9. Security and Data Controls

For demo:

- Avoid logging source code.
- Avoid logging full prompts if source code is included, unless explicitly configured.
- Redact likely secrets from snippets before LLM input.
- Clean up temporary workspaces.

For future enterprise deployment:

- Use KMS encryption for artifacts and results.
- Use Secrets Manager for Bitbucket and scanner credentials.
- Use least privilege IAM.
- Use VPC endpoints/PrivateLink where required.
- Add audit log for scan trigger and report access.

## 10. Future Bitbucket Data Flow

### V2: Bitbucket Archive Download

```text
User selects Bitbucket repo/branch
  -> BitbucketArchiveSourceAdapter downloads archive
  -> Store archive artifact
  -> Same workspace/scanner/evidence/LLM pipeline
```

This is the easiest transition from local upload because both paths produce a repo archive.

### V3: Bitbucket Git Clone

```text
User selects repo/branch/commit
  -> BitbucketCloneSourceAdapter performs shallow clone
  -> Workspace includes git metadata
  -> Future diff scan support
```

This is better for future PR/diff analysis.

## 11. Demo Completion Criteria

The pipeline demo is complete when:

- A user can upload one Java/Python repo archive.
- A scan run is created and tracked.
- Static findings are produced.
- Evidence package is generated.
- LLM report is generated.
- LLM output is evaluated.
- Invalid LLM output can trigger retry/fallback.
- Final report is displayed in dashboard.
- Raw JSON artifacts are downloadable.
