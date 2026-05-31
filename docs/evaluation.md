# Code Quality Platform - Evaluation Design

## 1. Purpose

The platform uses LLMs to assist code quality assessment. Because LLM outputs can vary and may hallucinate, evaluation must be a first-class component of the system.

This document defines how to evaluate:

1. The input evidence used by the LLM.
2. The LLM-generated quality report.
3. The final platform output shown to users.

The goal is to prevent the system from becoming an unverified LLM wrapper.

## 2. Core Evaluation Principles

The project should follow these principles:

```text
No evidence, no conclusion.
No validation, no report.
No versioning, no trust.
No deterministic baseline, no LLM judgment.
```

Practical implications:

- LLM must not be the sole source of code quality judgment.
- LLM conclusions must reference evidence.
- LLM output must follow a strict schema.
- LLM output must pass evaluator checks before acceptance.
- Reports must expose confidence and validation status.
- Prompt, rubric, model, scanner, and evaluator versions must be stored.
- The system must support fallback to static-only reports when LLM output is unreliable.

### Structured Output and Semantic Validation

Using Claude Bedrock tool use or structured output enforces schema validity at the API level. This reduces JSON formatting failures and eliminates most schema-related retry needs.

However, structured output does not replace semantic validation. The evaluator must still verify:

- Whether evidence references exist in the evidence package.
- Whether file paths, class names, and function names exist in the repo.
- Whether severity is consistent with evidence strength.
- Whether recommendations are specific enough to be actionable.
- Whether the LLM has cited anything outside the evidence package.

```text
Tool use reduces format failures.
Semantic validation and repair are still required.
```

## 3. What Needs to Be Evaluated

## 3.1 Static Evidence Quality

Before the LLM runs, the system should validate the evidence package.

Questions:

- Did scanners run successfully?
- Is repo structure available?
- Are source snippets available?
- Are findings normalized?
- Are file paths valid?
- Is the repository complete enough for assessment?
- Are there missing inputs, such as test coverage?

If evidence is weak, the LLM assessment should still be possible, but the report confidence should be lower and unknowns should be explicitly listed.

## 3.2 LLM Output Quality

The LLM output must be evaluated for:

- Schema correctness.
- Grounding.
- Factual consistency.
- Internal consistency.
- Actionability.
- Severity calibration.
- Stability.
- Boundary compliance.

## 3.3 Final Report Quality

The final report should be evaluated as a product output:

- Is it useful to engineers?
- Is it understandable to managers?
- Is it sufficiently grounded for architecture review?
- Does it separate objective facts from LLM judgment?
- Does it expose confidence limitations?

## 4. LLM Report Acceptance Criteria

A report can be accepted only if it satisfies the following minimum criteria.

## 4.1 Schema Validity

The LLM output must be valid JSON and must conform to the expected schema.

Required top-level fields:

```json
{
  "overall_assessment": {},
  "quality_themes": [],
  "recommended_backlog": [],
  "unknowns": [],
  "metadata": {}
}
```

Hard failures:

- Invalid JSON.
- Missing required fields.
- Invalid enum values.
- Scores outside expected range.
- Confidence outside 0 to 1.

## 4.2 Evidence Grounding

Every important claim must reference evidence.

Required:

- Each quality theme must include at least one evidence reference.
- High severity themes should include at least two independent evidence references.
- Evidence references must exist in the evidence package.
- Referenced files must exist in the repo tree.

Hard failures:

- Important theme has no evidence.
- LLM references a non-existent file/class/function.
- Evidence reference ID does not exist.

## 4.3 No Hallucinated References

The evaluator should reject or retry reports that mention files, classes, methods, modules, or scanner findings that are not present in the evidence package.

Examples of failure:

```text
The LLM recommends refactoring UserPaymentProcessor.java, but that file does not exist.
The LLM cites a CodeQL finding that was not in the input.
The LLM refers to test coverage results when no coverage data was provided.
```

## 4.4 Severity Consistency

Severity must match evidence strength.

Guidelines:

- High severity requires strong evidence and meaningful engineering impact.
- Medium severity applies to localized maintainability or design risks.
- Low severity applies to isolated readability or cleanup issues.

Inconsistency examples:

```text
risk_level = low, but the report contains five high severity themes.
overall_score = 92, but the summary says the repo has severe maintainability risk.
A naming issue is classified as high severity without broader impact.
```

## 4.5 Recommendation Actionability

Recommendations must be specific enough for engineers to act on.

Bad:

```text
Improve code quality.
Make the design cleaner.
Add better abstractions.
```

Acceptable:

```text
Move validation logic from PaymentService and PaymentController into PaymentValidationPolicy, then add unit tests for invalid payment requests.
```

Recommended scoring:

```text
0 = vague or not actionable
1 = partially actionable
2 = specific and actionable
```

## 4.6 Boundary Compliance

The LLM should not exceed the product boundary.

The demo product is advisory only.

Not allowed:

```text
This repo must be blocked from release.
This is a compliance violation.
This security issue is confirmed exploitable.
```

Allowed:

```text
This repo has high maintainability risk and should be reviewed before major feature expansion.
This finding may require security team review if confirmed by the approved scanner.
```

## 5. Evaluation Categories

## 5.1 Format Evaluation

Checks:

- JSON parse succeeds.
- Required fields exist.
- Field types are correct.
- Enum values are legal.
- Score ranges are valid.

Failure action:

- Retry with format repair prompt.

## 5.2 Grounding Evaluation

Checks:

- All evidence references exist.
- Referenced file paths exist.
- Referenced line ranges are valid when available.
- Claims do not cite missing scanner results.
- LLM does not invent classes, functions, or modules.

Failure action:

- Retry with grounding repair prompt.
- If repeated failure, fallback to static-only report.

## 5.3 Consistency Evaluation

Checks:

- Risk level matches severity distribution.
- Score matches summary tone.
- Recommendations match identified themes.
- Unknowns do not contradict conclusions.

Failure action:

- Retry with consistency repair prompt.

## 5.4 Actionability Evaluation

Checks:

- Recommendations include target module/file when possible.
- Recommendations include specific change.
- Recommendations include expected benefit.
- Recommendations include priority or effort.

Failure action:

- Retry if actionability is too low.
- Otherwise accept with warning.

## 5.5 Stability Evaluation

Stability evaluation runs the same scan multiple times and checks for consistency. It is not part of the normal scan path.

For MVP, stability evaluation is available only as an optional CLI flag:

```text
code-quality scan --stability-check --runs 3
```

This flag should not be enabled by default because it increases cost and latency.

Checks:

- Same input evidence should produce similar top themes.
- Risk level should not fluctuate significantly.
- Score drift should remain within threshold.
- Top refactor recommendations should overlap.

Suggested thresholds:

```text
overall_score drift <= 5 points
top 3 quality theme overlap >= 2
hallucination_count = 0
schema pass rate = 100%
grounding_score >= 0.85
```

## 6. Retry Policy

## 6.1 Retry-Required Failures

Retry when:

- JSON parse fails.
- Schema validation fails.
- Required fields are missing.
- Evidence references are missing.
- Referenced files/classes/functions do not exist.
- Severity/score/risk level is inconsistent.
- Output is empty or too short.
- LLM refuses to assess despite valid evidence.

## 6.2 Accept With Warning

Accept with warning when:

- Some recommendations are partially actionable but not ideal.
- Evidence package is incomplete.
- Test coverage data is missing.
- Confidence is medium or low.
- Unknowns are significant but clearly reported.

## 6.3 Fallback

Fallback to static-only report when:

- LLM fails after max retry count.
- LLM repeatedly hallucinates.
- LLM output cannot be parsed or validated.
- LLM provider is unavailable.

The scan itself should still succeed if static analysis succeeded.

## 6.4 Max Retry Count

Recommended MVP setting:

```text
max_retry = 2
```

All retries must store:

- Failure reason.
- Retry prompt type.
- Attempt number.
- Raw failed output URI.

## 7. Repair Prompt Strategy

Retry should not simply resend the same prompt.

Use targeted repair prompts.

## 7.1 Schema Repair

Use when JSON/schema validation fails.

Prompt intent:

```text
Your previous output failed schema validation. Regenerate the response using exactly the required JSON schema. Do not include prose outside JSON.
```

## 7.2 Grounding Repair

Use when evidence references are missing or invalid.

Prompt intent:

```text
Your previous output referenced unsupported conclusions. Regenerate the report using only the provided evidence_refs. Every quality theme must include valid evidence_refs. Do not mention files, classes, functions, or scanner findings that are not present in the evidence package.
```

## 7.3 Consistency Repair

Use when severity, risk level, and score conflict.

Prompt intent:

```text
Your previous output has inconsistent severity, risk level, and score. Re-evaluate using the rubric. High severity requires strong evidence and meaningful engineering impact.
```

## 7.4 Actionability Repair

Use when recommendations are too vague.

Prompt intent:

```text
Your previous recommendations were too vague. Rewrite each recommendation with target module/file, specific change, expected benefit, and estimated effort.
```

## 8. Evaluation Metadata

Evaluation result must be stored with every scan.

Recommended table:

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
- stability_score (Phase 2, optional)
- hallucination_count
- retry_count
- final_status
- failure_reasons
- created_at
```

Model usage metadata must also be stored per scan to support cost analysis:

```text
llm_usage_metadata
- scan_run_id
- llm_step (theme_extractor, recommendation_generator, etc.)
- model_id
- input_tokens
- output_tokens
- retry_count
- estimated_cost_when_available
```

Cost should be measured and recorded, not estimated in the design.

## 9. Report Quality Score

The platform should separate code quality score from LLM report quality score.

### 9.1 Code Quality Score

Meaning:

- How healthy is the repo?

Inputs:

- Static metrics.
- Findings.
- LLM assessment, with small weight.

### 9.2 LLM Report Quality Score

Meaning:

- How trustworthy is the generated LLM assessment?

Suggested formula:

```text
LLM Report Quality Score =
  30% Grounding
+ 20% Schema Correctness
+ 20% Consistency
+ 20% Actionability
+ 10% Conciseness / Clarity
```

This score should be displayed in the dashboard separately.

## 10. Dashboard Evaluation Indicators

The dashboard should show:

```text
LLM Report Confidence: High / Medium / Low
Validation Status: Passed / Warning / Failed
Evidence Coverage: percentage
Retry Count: number
Prompt Version: version
Rubric Version: version
Evaluator Version: version
```

Example display:

```text
Quality Score: 72 / 100
Risk Level: Medium
LLM Report Confidence: High
Validation Status: Passed
Evidence Coverage: 91%
Retry Count: 0
Prompt Version: v1.0
Rubric Version: v1.0
```

## 11. Offline Regression Evaluation

The team should maintain a small evaluation dataset.

Recommended initial dataset:

```text
Repo Snapshot A: clean service
Repo Snapshot B: high complexity service
Repo Snapshot C: duplicated logic
Repo Snapshot D: poor test coverage
Repo Snapshot E: mixed Java/Python service
```

For each snapshot, store:

- Static scanner output.
- Evidence package.
- Expected high-level risk themes.
- Human-reviewed notes.

Run regression evaluation when changing:

- Prompt.
- Rubric.
- Model.
- Evidence selection logic.
- Scoring formula.
- Evaluator logic.

Regression checks:

- Schema pass rate.
- Hallucination count.
- Grounding score.
- Top theme overlap.
- Score drift.
- Recommendation actionability.

## 12. Human Review Loop

For the demo and early adoption phase, human review is required to calibrate the system.

Recommended process:

- Each week, review 3 to 5 generated reports.
- Ask senior engineers/architects to rate:
  - Accuracy.
  - Usefulness.
  - Actionability.
  - Missing important issues.
  - Overstated issues.
- Feed review results back into rubric and prompt improvements.

Human review should not block every scan, but it should guide system calibration.

## 13. MVP Evaluation Scope

The first demo should implement:

### Hard Gate Checks (Deterministic — Always Run)

Hard gate checks are implemented in code. They must not use an LLM.

- JSON schema validation and required field check.
- Evidence reference validation (all evidence_refs cited by LLM exist in the evidence package).
- File path validation (referenced file paths exist in the repo workspace).
- Line range validation when line ranges are provided.
- Hallucination detection: referenced class, function, or module names must exist in the workspace or findings.
- Final score is within valid range.
- Report can be stored and rendered without error.

Failure in any hard gate check triggers a targeted repair prompt or fallback.

### Soft Quality Checks (Deterministic Heuristics — MVP)

These are computed and recorded but do not block report acceptance:

- Actionability score: deterministic heuristic based on recommendation field completeness (target module present, specific change described, expected benefit stated).
- Unknowns count: how many missing inputs are explicitly listed.
- Evidence coverage: what percentage of quality themes have at least one valid evidence reference.
- Confidence level: derived from scanner completeness and evaluation pass rate.

### Soft Quality Checks via LLM-as-Judge (Phase 2 — Defer)

LLM-as-judge evaluation for qualitative dimensions (actionability nuance, clarity, recommendation usefulness, role-specific summary quality) is a Phase 2 enhancement. It should not replace deterministic hard gates. It should supplement them with qualitative scoring.

### Retry and Fallback

- Max 2 retries.
- Targeted repair prompts (schema repair, grounding repair, consistency repair).
- Fallback to static-only report if all retries fail.
- Store retry count, failure reasons, and raw failed outputs for audit.

## 14. Success Criteria

The evaluation system is successful when:

- Invalid JSON never reaches the final dashboard.
- Hallucinated files/classes/functions are caught.
- Every major claim has evidence.
- Users can see confidence and validation status.
- LLM failures do not break the whole scan.
- Prompt/model/rubric changes can be evaluated through regression runs.
- Engineers can understand why the system made a recommendation.
