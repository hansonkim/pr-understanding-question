---
name: pr-understanding-question
description: >
  Generates context-aware understanding questions from a PR, design notes, and code changes to verify
  whether the reviewer truly understands the why, trade-offs, risks, and behavior of the changes
  before approving. Runs as an interactive conversation: presents questions, waits for the reviewer's
  answers, then evaluates each answer against the PR facts.

  Use this skill whenever:
  - A reviewer wants to verify their own understanding of a PR before approving it
  - A user shares a PR (link, diff, description, or code changes) and wants understanding questions generated
  - A user says "check my understanding of this PR", "generate questions for this PR", "quiz me on this change"
  - A user wants to evaluate whether their answers about a PR are accurate and complete
  - A code review process needs structured questions to probe design decisions, risks, or trade-offs

  Do NOT trigger for: general code review feedback, summarizing a PR, writing a PR description.
---

# PR Understanding Question

This skill runs an interactive understanding check for a PR. It generates targeted questions,
waits for the reviewer's answers, then evaluates them against the PR facts.

The goal is not to summarize the PR.
The goal is to verify whether the reviewer understands the PR well enough to approve it:
- why the change exists
- what design decisions were made
- what risks were accepted
- what trade-offs were chosen
- what the code actually does
- what remains uncertain or weak

## Input Assumption

PR information is provided by the user.

May include: PR title, description, design notes, architectural intent, code diff,
changed files, test changes, known risks, reviewer comments, implementation constraints.

Use only the provided information. Do not fetch additional PR data.
Do not assume missing design intent. Do not invent code behavior not supported by the context.

If an important fact is missing, mark it as `Unverified from provided context`.

---

## Interactive Flow

This is the mandatory execution order. Do not skip or reorder steps.

### Step 1 — Generate questions (internal)

Internally prepare 3–7 questions. Each question has two parts:

**Developer-facing** (shown to the developer):
- Context: what part of the PR caused this question
- Why this matters: the risk, trade-off, or decision being tested
- Question: a self-contained question with 1–2 sentences of background so it reads clearly without Context

**Internal rubric** (kept hidden until evaluation):
- Good answer should include: bullet list of what a strong answer covers
- Evidence to check: specific files, functions, or design sections to verify against

### Step 2 — Present questions one at a time

Questions are asked one by one. Do NOT output all questions at once.

For each question, output ONLY the reviewer-facing parts:

```md
## Question N / <total>: <title>

### Context
...

### Why this matters
...

### Question
<1-2 sentence background>. <The actual question>?
```

After presenting each question, stop and wait for the reviewer's answer before showing the next one.
Do NOT output "Good answer should include" or "Evidence to check" — revealing the rubric defeats the purpose.

After the final question is answered, show the **Coverage** summary:

```md
## Coverage
- 설계 결정 / 책임 이동 / 위험·실패 케이스 / 테스트 전략 / 운영 영향 중 해당 항목 나열
```

And a **Potentially uncovered area** note only if the context hints at an important area
but lacks enough information to form a precise question.

### Step 3 — Collect answers one by one

After each question, wait for the reviewer's reply before moving to the next question.
When all questions are answered, proceed to evaluation.

### Step 4 — Evaluate answers (Mode B)

Once answers are received, evaluate each one using the internal rubric.
Now the "Good answer should include" criteria are used for scoring.

---

# Question Generation Guide

## Finding strong questions

Use `references/heuristics.md` for a categorized guide (design decisions, responsibility movement,
boundary changes, risky details, test strategy, operational impact).

Prioritize:
1. High-risk behavior
2. Architectural decisions
3. Non-obvious trade-offs
4. Hidden failure cases
5. Test gaps
6. Operational or rollback concerns

Do not generate questions for trivial changes unless they affect behavior or risk.

## Question quality rules

Good questions require the developer to explain:
- why this design was chosen vs. alternatives
- what can fail and how failure is detected or recovered
- what assumption the implementation relies on
- why the test strategy is sufficient or insufficient
- how the change affects other parts of the system

**Avoid shallow questions.** Do not ask "What does this function do?" or "Did you test this?"
unless tied to a concrete context and risk.

**Avoid answer leakage.** The question may include context, but must not fully answer itself.

Bad:
> Since this change uses eventual consistency and may create duplicate events, explain the duplicate event scenario.

Better:
> This PR changes when the event is emitted relative to the persistence step. What inconsistent states can occur if one step succeeds and the other fails, and how does the design handle them?

**Make questions PR-specific.** Generic risk questions ("What are the risks?") are useless.
Anchor every question to a specific file, function, decision, or diff in the provided PR.

**Plain language.** Questions probe technical decisions, but the *language* of the question
must be accessible. Reviewers should not have to mentally translate jargon to understand
what is being asked.

Concrete identifiers (function names, file paths, table names, ADR IDs, command names,
specific Redis/SQL primitives the PR actually uses) stay verbatim — they are the anchors
that make a question PR-specific and clickable.

For *non-identifier* technical terms (architectural jargon, internal slang, domain abstractions),
do one of two things:
- Replace with everyday words when a simpler phrase fits.
  Example: `entry severity 승격` → "응답 로그 한 건 전체의 심각도를 ERROR로 끌어올리는 동작".
- Keep the term but inline a one-line plain-language gloss the first time it appears.
  Example: `proxy-only subnet (Internal LB 전용으로 분리한 서브넷)`.

Self-check before showing a question: "Could a teammate one or two years junior read this
and understand what is being asked, without looking up half the words?" If not, simplify.
The point of the question is to test the reviewer's understanding of the change, not their
fluency in the author's vocabulary.

---

# Mode B: Answer Evaluation

## Process

For each answer:
1. Compare against the internal rubric ("Good answer should include") and PR/design/code facts.
2. Check whether the answer explains the decision, not just the implementation.
3. Check whether risks and failure cases are concrete.
4. Check whether claims are supported by the provided context.
5. Identify misunderstanding, omission, or overconfidence.
6. Provide focused follow-up.

## Evaluation criteria

Score each answer from 0 to 4:

```
0 = Incorrect or contradicts provided facts
1 = Very shallow; mostly generic
2 = Describes code behavior but misses design intent or risk
3 = Explains behavior, intent, and major risks with reasonable accuracy
4 = Explains intent, alternatives, trade-offs, risks, tests, and operational impact
```

Use the score as a diagnostic tool, not as a punishment.

## Evaluation output format

```md
# PR Understanding Evaluation

## Question 1: <title>

### Score
2 / 4

### Assessment
What the answer got right and what it missed.

### Supported by provided context
- Specific code/design evidence supporting the assessment.

### Gaps
- Missing design rationale
- Missing failure scenario
- Missing test limitation

### Follow-up question
One focused question to help the developer fill the gap.
```

## Fact-based evaluation rules

**Distinguish fact from inference.**
- `Supported by provided context` — PR/design/code clearly supports this
- `Inferred risk` — logically plausible but not explicitly stated
- `Unverified from provided context` — may be true but cannot be checked from provided material

**Do not grade based on preference.** Evaluate whether the developer understands the chosen
design, its trade-offs, risks, constraints, and verification strategy.

**Do not invent missing facts.** If test details are absent, say
"The provided context does not show tests for this behavior" and ask where it is verified.

**Challenge vague answers.** "Safer", "more scalable", "handles errors", "covered by tests"
require specifics: safer against what? which dimension? what error path? which test?

**Prefer follow-up questions over full correction.** When incomplete, show what is missing,
why it matters, and ask one focused follow-up. Only give a full explanation if explicitly
requested or if evaluation is complete.

---

A good question does not come from a template. It comes from the gap between:
- what the PR changes
- what the design assumes
- what the code actually guarantees
- what can fail in production
- what the developer must be able to explain

A question without context is noise. A score without evidence is opinion.
An answer without risk awareness is not understanding.
