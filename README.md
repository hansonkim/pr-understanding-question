# pr-understanding-question

An AI agent skill that runs an interactive PR understanding check — generates context-aware questions from a PR, presents them one at a time, waits for the developer's answers, then evaluates each answer against the PR facts.

## Why this exists

Standard PR review gives feedback on code quality. This skill does something different: it checks whether the **developer** actually understands their own change.

A PR can look clean, and the developer who wrote it may still be unable to answer:

- Why this design was chosen over alternatives
- What fails if the external dependency goes down
- Which callers are now unprotected after the validation moved layers
- Why the cache TTL value is what it is

Those gaps surface in production, not in review. This skill surfaces them before merge.

## How it works

The skill runs as a conversation, not a one-shot output.

```
1. Provide PR context: title, description, diff, known risks, design notes
2. The skill generates 3–7 targeted questions internally, keeping the answer rubric hidden
3. Questions are presented one at a time — developer answers before the next appears
4. After all answers are collected, each is evaluated against the PR facts
5. Evaluation gives a 0–4 score, specific gaps, and a focused follow-up question
```

### What the developer sees (per question)

```
## Question 2 / 5: Behavior when Redis is unavailable

### Context
The design notes explicitly state "no fallback logic when Redis fails."
get_history() calls self.redis.get() with no exception handling.

### Why this matters
Before this PR, a Redis failure had no impact on this API.
This PR makes Redis a hard dependency — Redis availability becomes the ceiling for API availability.

### Question
Before this PR, a Redis failure would not affect /v2/location/history.
After this PR, what happens when Redis goes down, and what was the reasoning behind accepting that impact?
```

The answer rubric ("Good answer should include") is hidden until evaluation. Revealing it before the developer answers defeats the purpose.

### What evaluation looks like

```
## Question 2 / 5: Behavior when Redis is unavailable

### Score
1 / 4

### Assessment
Correctly identified that HTTP 500 would be returned, but did not explain
the reasoning behind omitting a fallback, and did not mention any monitoring strategy.

### Supported by provided context
- services/location_service.py: self.redis.get() has no exception handling — confirmed

### Gaps
- No rationale given for omitting fallback (Redis SLA, operational cost, acceptable risk)
- No mention of how Redis failures would be detected or alerted

### Follow-up question
What was the basis for shipping without a fallback?
Did you verify that the Redis availability SLA is sufficient to meet this API's SLA?
```

## Question generation principles

Questions are generated from the gap between what the PR changes and what the developer must be able to explain:

- **Design decisions** — why this approach, what alternatives were rejected
- **Responsibility movement** — which callers are now affected, what bypass risk exists
- **Boundary changes** — failure behavior, retries, idempotency, consistency, observability
- **Risky details** — transactions, caching, migration, partial failure, silent failure
- **Test strategy** — what assumptions the tests verify, which failure paths are uncovered
- **Operational impact** — how operators detect failure, how to roll back

Generic questions ("What are the risks of this PR?") are never generated. Every question is anchored to a specific file, function, decision, or diff in the provided PR.

## Installation

### With APM (Agent Package Manager)

```sh
apm install hansonkim/pr-understanding-question
```

### Manual

Copy `SKILL.md` and `references/heuristics.md` into your skills directory.

**Claude Code**
```
~/.claude/skills/pr-understanding-question/
├── SKILL.md
└── references/
    └── heuristics.md
```

**Other runtimes** (Codex, Gemini CLI, OpenCode, etc.)
```
~/.agents/skills/pr-understanding-question/
├── SKILL.md
└── references/
    └── heuristics.md
```

## Usage

Trigger with natural language after providing PR context:

```
Review this PR for understanding gaps: [paste PR content]
```

```
Generate understanding questions for this change: [paste diff + description]
```

Provide as much context as available — PR title, description, code diff, design notes, known risks, reviewer comments. The more context, the more specific the questions.

## Evaluation scoring

| Score | Meaning |
|---|---|
| 0 | Incorrect or contradicts PR facts |
| 1 | Very shallow; mostly generic |
| 2 | Describes code behavior but misses design intent or risk |
| 3 | Explains behavior, intent, and major risks with reasonable accuracy |
| 4 | Explains intent, alternatives, trade-offs, risks, tests, and operational impact |

Scores are diagnostic, not judgmental. The goal is to surface gaps before production does.

## License

MIT
