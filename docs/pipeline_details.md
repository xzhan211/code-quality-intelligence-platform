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
  -> Build Evidence Package
  -> Run LLM Assessment
  -> Evaluate LLM Output
  -> Retry / Accept / Fallback
  -> Compute Final Score
  -> Generate Report
  -> Store Results
  -> Show Dashboard
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

## Stage 6: Evidence Package Build

### Input

- Normalized findings.
- Workspace metadata.
- Repo tree.
- Selected source snippets.
- Optional historical scan data.

### Processing

Evidence builder selects and packages the information that the LLM is allowed to use.

Recommended inputs to LLM:

- Repo summary.
- Language/framework detection.
- Top complex files/functions.
- Top duplicated or suspicious repeated patterns.
- Top scanner findings.
- Selected code snippets.
- Test file summary.
- Unknowns and missing data.

### Snippet Selection Rules

Prefer snippets from:

- Top complexity findings.
- Repeated logic findings.
- Large service/controller/repository files.
- Files with multiple findings.
- Representative test files.

Avoid:

- Binary files.
- Generated files.
- Very large files without selection.
- Secrets or credentials when detected.

### Evidence Package Schema

```json
{
  "evidence_package_id": "evidence-001",
  "scan_run_id": "scan-001",
  "repo": {
    "tenant": "demo",
    "repo_name": "payment-service",
    "languages": ["Java", "Python"],
    "branch": "main",
    "commit_sha": null
  },
  "repo_tree_summary": [],
  "metrics_summary": {},
  "findings": [],
  "snippets": [],
  "unknowns": [],
  "versions": {
    "evidence_builder_version": "v1.0"
  }
}
```

### Output

```text
EvidencePackage
```

## Stage 7: LLM Assessment

### Input

- Evidence package.
- Prompt version.
- Rubric version.
- Model config.

### Processing

LLM assessor sends a schema-constrained prompt to the approved model.

### LLM Responsibilities

- Group findings into quality themes.
- Explain engineering impact.
- Identify cross-file maintainability risks.
- Prioritize refactor recommendations.
- Produce role-aware summary.
- Explicitly list unknowns.

### Output

```text
RawLLMOutput
ParsedLLMReport optional
```

## Stage 8: LLM Output Evaluation

### Input

- Raw/parsed LLM report.
- Evidence package.
- Repo tree.
- Evaluation config.

### Processing

Run evaluator checks:

- Schema validation.
- Evidence reference validation.
- File path validation.
- Hallucination detection.
- Severity consistency.
- Recommendation actionability.

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
  "failure_reasons": []
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

## Stage 10: Final Scoring

### Input

- Normalized findings.
- Static metrics.
- Accepted or warning-level LLM report.
- Evaluation result.

### Processing

Compute final score using deterministic formula.

Recommended MVP formula:

```text
Overall Score =
  25% Maintainability
+ 20% Complexity
+ 15% Duplication
+ 15% Test Coverage or Test Presence Proxy
+ 15% Risk Signals
+ 10% LLM Architecture/Readability Assessment
```

If LLM report is failed:

- Exclude LLM component.
- Mark score confidence lower.

### Output

```text
FinalScore
RiskLevel
ScoreBreakdown
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

Show both:

```text
Code Quality Score
LLM Report Quality / Confidence
```

Example:

```text
Quality Score: 72 / 100
Risk Level: Medium
LLM Report Confidence: High
Validation Status: Passed
Evidence Coverage: 91%
Retry Count: 0
```

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
