# Research Papers Reference

## Purpose

This document lists research papers that are directly relevant to the Code Quality Intelligence Platform. For each paper, it explains what the paper is about and why it matters for our design.

Papers are grouped by theme. Within each theme, they are ordered from most immediately actionable to more foundational.

> **Note:** Always verify exact titles, authors, publication venues, and years before citing formally. arXiv links are the most reliable source for preprints. Conference proceedings (NeurIPS, EMNLP, ICSE, MSR, FSE) are the most reliable source for peer-reviewed versions.

---

## Category 1: LLM Engineering and Structured Output

These papers directly inform the harness engineering decisions in the platform — how to make LLM outputs deterministic, structured, and robust.

---

### Paper 1.1

**Title:** Efficient Guided Generation for Large Language Models

**Authors:** Brandon T. Willard, Rémi Louf

**Year:** 2023

**Venue:** arXiv preprint (basis for the `outlines` library)

#### What it is

This paper introduces a method for constraining LLM generation so that the output always conforms to a specified grammar, regular expression, or JSON schema. The technique works by masking invalid tokens at each generation step, making it impossible for the model to produce output that violates the declared structure.

The `outlines` Python library is the practical implementation of this approach. A similar mechanism is used by Anthropic and OpenAI in their native structured output and tool use APIs.

#### Why it is good for us

Our platform requires that every LLM step (ThemeExtractor, RecommendationGenerator) produce schema-valid JSON. Without constrained generation, schema failures require retry loops.

This paper explains the theoretical foundation for why Claude Bedrock tool use with `tool_choice: {"type": "tool"}` eliminates schema format failures: the model is constrained at the token level to only produce tokens that form valid JSON matching the declared schema.

Reading this paper helps the team understand why structured output is a first-class reliability mechanism, not just a formatting convenience. It also explains why semantic validation (evidence references, file existence) is still necessary — constrained decoding enforces structure, not content correctness.

---

### Paper 1.2

**Title:** Chain-of-Thought Prompting Elicits Reasoning in Large Language Models

**Authors:** Jason Wei, Xuezhi Wang, Dale Schuurmans, Maarten Bosma, Brian Ichter, Fei Xia, Ed Chi, Quoc V. Le, Denny Zhou

**Year:** 2022

**Venue:** NeurIPS 2022

#### What it is

This paper shows that prompting an LLM to produce intermediate reasoning steps ("chain of thought") before a final answer significantly improves accuracy on complex tasks, especially tasks requiring multi-step reasoning or factual synthesis.

The key finding is that asking the model to show its reasoning, rather than jump directly to a conclusion, reduces errors — particularly hallucination and logical inconsistency.

#### Why it is good for us

Our ThemeExtractor prompt requires the LLM to produce a rationale for each quality theme before the theme conclusion. This is directly motivated by chain-of-thought research.

However, our design does not expose the full chain of thought to the user. Instead, the prompt requires the LLM to produce an `evidence_refs` list and a `rationale` summary per theme — the key outputs of the reasoning process. This gives us the reliability benefit (reduced hallucination, better evidence grounding) without exposing verbose internal reasoning in the user-facing report.

The paper also provides the academic justification for structuring the ThemeExtractor output as "evidence first, then conclusion" rather than asking the LLM to output a conclusion directly.

---

### Paper 1.3

**Title:** Large Language Models Are Zero-Shot Reasoners

**Authors:** Takeshi Kojima, Shixiang Shane Gu, Machel Reid, Yutaka Matsuo, Yusuke Iwasawa

**Year:** 2022

**Venue:** NeurIPS 2022

#### What it is

This paper introduces the "Let's think step by step" prompting technique. It shows that simply adding a step-by-step instruction to a prompt, without few-shot examples, significantly improves LLM performance on reasoning tasks.

The paper demonstrates that this zero-shot chain-of-thought approach generalizes across many tasks.

#### Why it is good for us

Useful as a lightweight complement to the Wei et al. CoT paper. The "step by step" instruction pattern can be incorporated into the ThemeExtractor and RecommendationGenerator prompts with minimal complexity.

For the demo, if prompt engineering reveals that the LLM produces too many hallucinated references, adding a step-by-step instruction ("First list the evidence you are relying on. Then explain the quality theme.") is a low-cost reliability improvement grounded in this research.

---

## Category 2: LLM Output Evaluation

These papers directly inform the LLM evaluation component of the platform — how to assess whether LLM outputs are grounded, consistent, and actionable.

---

### Paper 2.1

**Title:** Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena

**Authors:** Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, Siyuan Zhuang, Zhanghao Wu, Yonghao Zhuang, Zi Lin, Zhuohan Li, Dacheng Li, Eric P. Xing, Hao Zhang, Joseph E. Gonzalez, Ion Stoica

**Year:** 2023

**Venue:** NeurIPS 2023

#### What it is

This paper introduces and evaluates the pattern of using a strong LLM (GPT-4 in the original paper) to judge the quality of outputs from other LLMs. It introduces MT-Bench (a multi-turn benchmark) and Chatbot Arena (a human preference collection system) and compares LLM judgments against human ratings.

Key findings:
- LLM-as-judge correlates well with human judgment at the aggregate level.
- LLM-as-judge has known failure modes: position bias, verbosity bias, and self-enhancement bias.
- These biases can be mitigated with carefully structured evaluation prompts and rubrics.

#### Why it is good for us

The LLM-as-judge pattern is the basis for our Phase 2 soft evaluator (actionability scoring, clarity scoring, recommendation usefulness scoring). These qualitative dimensions are hard to score with simple heuristics but can be evaluated by a lightweight LLM call (e.g., Claude Haiku) using a structured rubric.

Understanding the known biases (verbosity bias, position bias) helps us design better evaluation prompts when we implement LLM-as-judge in Phase 2. For example: we should not simply ask "is this recommendation good?" but use a structured rubric with specific criteria.

The paper also explains why we deferred LLM-as-judge to Phase 2. The pattern is valuable but requires careful rubric design to avoid biases. For MVP, deterministic heuristics are safer and more predictable.

---

### Paper 2.2

**Title:** G-Eval: NLG Evaluation Using GPT-4 with Better Human Alignment

**Authors:** Yang Liu, Dan Iter, Yichong Xu, Shuohang Wang, Ruochen Xu, Chenguang Zhu

**Year:** 2023

**Venue:** ACL 2023

#### What it is

This paper proposes G-Eval, a framework for evaluating natural language generation (NLG) outputs using GPT-4 as an evaluator. The key innovation is using chain-of-thought to generate evaluation criteria before scoring, and then scoring using a structured rubric.

G-Eval achieves significantly higher correlation with human judgments than previous automatic NLG evaluation metrics.

#### Why it is good for us

G-Eval directly models how our LLM Report Quality Score should be computed when the LLM-as-judge approach is implemented in Phase 2.

The G-Eval pattern:
1. Define evaluation criteria (grounding, actionability, consistency, clarity).
2. Ask the evaluator LLM to reason about the criteria before scoring.
3. Produce a structured score per dimension.

This maps directly to our LLM Report Quality Score formula:

```text
30% Grounding
20% Schema Correctness
20% Consistency
20% Actionability
10% Conciseness / Clarity
```

The paper also shows that having the evaluator produce a reasoning step per dimension before the score reduces scoring errors. This informs how we should structure the Phase 2 LLM evaluator prompt.

---

## Category 3: Code Quality and Technical Debt

These papers provide the academic foundation for what we measure, why we measure it, and how to explain the value of code quality analysis to leadership.

---

### Paper 3.1

**Title:** An Empirical Model of Technical Debt and Interest

**Authors:** Ariadi Nugroho, Joost Visser, Tobias Kuipers

**Year:** 2011

**Venue:** MTD 2011 (Workshop on Managing Technical Debt, co-located with ICSE)

#### What it is

This paper introduces an empirical model for measuring technical debt in terms of two quantities:
- **Principal**: the estimated effort required to bring the codebase to a clean state.
- **Interest**: the ongoing extra cost per development cycle caused by the existing debt.

The model is derived from analysis of real software systems and provides concrete formulas for estimating debt and interest from code quality metrics (complexity, duplication, unit size, test coverage).

This research is the academic foundation behind the SIG/Softagram technical debt measurement approach and is referenced extensively in enterprise software quality research.

#### Why it is good for us

This paper gives us the academic vocabulary and justification for what we are measuring. When presenting to leadership, framing findings as "technical debt principal and interest" is more persuasive than "high cyclomatic complexity."

It also justifies our scoring formula components (complexity, duplication, unit size, test coverage) — these are the same dimensions the paper identifies as the primary drivers of technical debt accumulation.

The paper's distinction between principal (one-time fix cost) and interest (ongoing overhead) maps well to the difference between "refactor backlog" items (reduce principal) and "quality themes" (reduce ongoing interest). This framing can strengthen the demo narrative for management and architecture reviewers.

---

### Paper 3.2

**Title:** Your Code as a Crime Scene / Software Design X-Rays (books + associated research)

**Author:** Adam Tornhill

**Year:** 2015 (first book), 2018 (second book), associated research papers ongoing

**Venue:** Pragmatic Bookshelf (books); CodeScene blog and conference talks (research)

#### What it is

Adam Tornhill's work introduces behavioral code analysis: combining static code metrics (complexity, code smells) with version control history (git churn — how frequently a file changes) to identify the highest-risk areas of a codebase.

The core insight: a complex file that is never touched is low risk. A complex file that is changed every sprint is the highest-risk area for bugs, regressions, and maintenance overhead.

Key metrics introduced:
- **Hotspot**: files that are both complex and frequently changed.
- **Temporal coupling**: files that tend to be changed together (co-change patterns), indicating hidden dependencies.
- **Code health degradation**: trend of complexity increasing over time in specific modules.

This work is the research foundation for CodeScene's product.

#### Why it is good for us

This is the primary justification for adding churn × complexity temporal signal in Phase 2. For the demo, we use static complexity only. But explaining to leadership that Phase 2 will add "hotspot analysis combining complexity with change frequency" is a compelling roadmap item.

The temporal coupling idea is also relevant for identifying hidden architecture dependencies that static analysis alone misses. When a set of files are always changed together despite having no explicit dependency, it suggests a hidden coupling — a prime refactor candidate.

The distinction between "complex but stable" and "complex and frequently changed" is a concrete example of why our EvidenceSelector needs to evolve beyond static importance scoring.

---

### Paper 3.3

**Title:** On the Diffuseness and the Impact on Maintainability of Code Smells: A Large Scale Empirical Study

**Authors:** Fabio Palomba, Gabriele Bavota, Massimiliano Di Penta, Fausto Vivanco, Rocco Oliveto, Andrea De Lucia

**Year:** 2018

**Venue:** IEEE Transactions on Software Engineering (TSE)

> **Verify details:** The exact title and year should be confirmed before formal citation. Fabio Palomba has published multiple papers on code smells in TSE; this description matches the content of one of the major studies in that series.

#### What it is

This paper studies which code smells (long method, god class, feature envy, data class, etc.) are most common in real Java projects and which smells have the greatest measurable impact on external quality attributes such as change-proneness, bug introduction rate, and developer productivity.

Key findings:
- Not all code smells are equally harmful. A small subset (long method, god class, feature envy) causes the majority of maintenance problems.
- God class and feature envy smells are particularly correlated with higher bug density and change-proneness.
- Many smells co-occur, and co-occurrence multiplies negative impact.

#### Why it is good for us

This paper informs which complexity and design metrics should be weighted most heavily in our EvidenceSelector importance scoring and in our scoring formula.

For Java repositories (our preferred demo target), the findings justify:
- Detecting "god class" patterns (large classes with many responsibilities) as high-severity findings.
- Treating feature envy (a class that uses another class's data more than its own) as a cross-file architecture smell.
- Prioritizing long-method findings as higher-severity than other size metrics when they co-occur with high complexity.

For leadership, this paper provides evidence that what we are detecting (the specific code smells we flag) has been empirically validated as a predictor of real maintenance cost — not just theoretical code hygiene.

---

## Category 4: LLM for Code Analysis

These papers situate our product in the broader research landscape of applying LLMs to software engineering tasks.

---

### Paper 4.1

**Title:** CodeBERT: A Pre-Trained Model for Programming and Natural Languages

**Authors:** Zhangyin Feng, Daya Guo, Duyu Tang, Nan Duan, Xiaocheng Feng, Ming Gong, Linjun Shou, Bing Qin, Ting Liu, Daxin Jiang, Ming Zhou

**Year:** 2020

**Venue:** EMNLP 2020 (Findings)

#### What it is

CodeBERT is one of the foundational pre-trained models for joint understanding of natural language and programming language (NL-PL). It is trained on bimodal data (code with docstrings) across six programming languages including Java and Python.

CodeBERT established the NL-PL pre-training paradigm that later influenced Copilot, Code LLaMA, and the current generation of code-capable LLMs.

#### Why it is good for us

This paper provides historical context for why general-purpose LLMs like Claude are capable of high-quality code understanding. Understanding the NL-PL pre-training foundation helps explain to skeptical audiences why the LLM can reason about code quality from source snippets — it is not simply pattern-matching, but genuine semantic understanding developed through pre-training.

It also establishes that Java and Python are among the best-covered languages in LLM training data, which supports the claim that our evidence-grounded LLM assessment will be more reliable for Java/Python than for less-common languages.

---

### Paper 4.2

**Title:** Large Language Models for Software Engineering: A Systematic Literature Review

> **Note:** There are several systematic literature reviews in this space (2023–2025). Search for "LLM software engineering survey" or "LLM code review survey" on arXiv to find the most current comprehensive review. The specific title and authors should be verified before citation.

#### What it is

Systematic literature reviews in this space survey how LLMs are applied to software engineering tasks: code generation, code review, bug detection, test generation, code summarization, refactoring recommendation, and quality assessment.

These reviews typically identify: which tasks LLMs perform well on, where hallucination is most problematic (code generation vs. analysis vs. review), and what evaluation methodologies the research community uses.

#### Why it is good for us

A systematic review provides a neutral academic summary that can be cited in leadership presentations to show that LLM-assisted code quality analysis is an active and credible research area — not a speculative concept.

It also surfaces evaluation methodologies that we can adopt or reference for our own LLM evaluation design (the evaluation.md document).

Suggested search: arXiv 2023–2025, keyword "LLM code review" or "large language models software quality."

---

### Paper 4.3

**Title:** An Empirical Study of Deep Learning Models for Code Smell Detection

> **Note:** Multiple papers exist on this topic with varying authors and years (2021–2024). A reliable starting point is searching "code smell detection deep learning" on Google Scholar. The general finding across papers is consistent and citable.

#### What it is

These papers study whether machine learning and deep learning models (and more recently LLMs) can detect code smells and quality issues with accuracy comparable to or better than rule-based tools like SonarQube.

Consistent findings across papers:
- ML/DL models can detect certain code smells (god class, long method, feature envy) with high accuracy.
- Models trained on real-world data often outperform threshold-based tools on project-specific patterns.
- Combining static metrics with code structure (AST, call graph) improves detection accuracy.

#### Why it is good for us

This body of research validates the design premise of our platform: static rule-based scanners have limitations that machine learning and LLM-based approaches can complement. Our platform does not replace SonarQube but uses LLM synthesis on top of static metrics — which is exactly the combination these papers suggest is most effective.

For the leadership demo, citing that this approach has been validated in peer-reviewed research adds credibility beyond "we built an AI wrapper."

---

## Summary Table

| # | Paper / Work | Category | Phase Relevance | Effort to Apply |
|---|---|---|---|---|
| 1.1 | Willard & Louf — Guided Generation | Harness engineering | MVP (foundation of tool use) | Low — already applied via Bedrock tool use |
| 1.2 | Wei et al. — Chain-of-Thought | Harness engineering | MVP (prompt design) | Low — applied in ThemeExtractor prompt design |
| 1.3 | Kojima et al. — Zero-Shot CoT | Harness engineering | MVP (prompt tuning) | Low — simple prompt addition |
| 2.1 | Zheng et al. — LLM-as-Judge | LLM evaluation | Phase 2 (soft evaluator) | Medium — requires rubric design |
| 2.2 | Liu et al. — G-Eval | LLM evaluation | Phase 2 (LLM Report Quality Score) | Medium — rubric prompt design |
| 3.1 | Nugroho et al. — Technical Debt Model | Code quality | MVP (scoring formula justification) | Low — informs scoring weights |
| 3.2 | Tornhill — Behavioral Code Analysis | Code quality | Phase 2 (churn × complexity) | High — requires git history integration |
| 3.3 | Palomba et al. — Code Smell Impact | Code quality | MVP (severity weighting) | Low — informs finding severity rules |
| 4.1 | Feng et al. — CodeBERT | LLM for code | Background context | None — reference only |
| 4.2 | LLM SE Survey (various) | LLM for code | Background context | None — reference only |
| 4.3 | ML Code Smell Detection (various) | LLM for code | Background context | None — reference only |
