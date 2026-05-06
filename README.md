# pr-understanding-question

A Claude Code skill that runs an interactive PR understanding check — generates context-aware questions from a PR, waits for the developer's answers one by one, then evaluates each answer against the PR facts.

## Why this exists

Standard PR review gives feedback on code quality. This skill does something different: it checks whether the **developer** actually understands their own change.

A PR can look clean and still be approved by someone who can't answer:
- Why this design was chosen over alternatives
- What fails if the external dependency goes down
- Which callers are now unprotected after the validation moved
- Why the TTL value is what it is

Those gaps surface in production, not in review. This skill surfaces them before merge.

## How it works

The skill runs as a conversation, not a one-shot output.

```
1. You share PR context (title, description, diff, known risks, design notes)
2. Skill generates 3–7 targeted questions internally, keeping the answer rubric hidden
3. Questions are presented one at a time — developer answers before the next appears
4. After all answers are collected, each is evaluated against the PR facts
5. Evaluation gives a 0–4 score, specific gaps, and a focused follow-up question
```

### What the developer sees (per question)

```
## Question 2 / 5: Redis 장애 시 서비스 동작

### Context
설계 메모에 "Redis 장애 시 fallback 로직 없음"이 명시되어 있다.
get_history()는 self.redis.get()을 예외 처리 없이 호출한다.

### Why this matters
이 PR 이전에는 Redis 의존성이 없었다. 이 PR은 Redis를 필수 의존성으로 만든다.
Redis 가용성이 이 API 가용성의 상한이 된다.

### Question
이 PR 이전에는 Redis 장애가 /v2/location/history API에 영향을 주지 않았습니다.
이 PR 이후 Redis가 다운되면 어떤 일이 발생하고, 그 영향을 수용한 이유는 무엇인가요?
```

The answer rubric ("Good answer should include") is hidden until evaluation — revealing it before the developer answers defeats the purpose.

### What evaluation looks like

```
## Question 2 / 5: Redis 장애 시 서비스 동작

### Score
1 / 4

### Assessment
Redis 장애 시 HTTP 500이 반환된다는 것은 맞지만, fallback을 생략한 설계 근거를
설명하지 않았고 모니터링 수단도 언급되지 않았습니다.

### Supported by provided context
- services/location_service.py: self.redis.get() 예외 처리 없음 — 확인됨

### Gaps
- fallback을 생략한 판단 근거 (Redis 안정성 기준, SLA, 운영 비용) 미언급
- Redis 장애를 감지하고 알림받는 수단 미언급

### Follow-up question
fallback 없이 배포하는 것이 허용된다고 판단한 근거는 무엇인가요?
Redis 가용성 SLA가 이 API의 SLA를 충족하는지 확인했나요?
```

## Question generation principles

Questions are generated from the gap between what the PR changes and what the developer must be able to explain:

- **Design decisions**: why this approach, what alternatives were rejected
- **Responsibility movement**: what callers are now affected, what bypass risk exists
- **Boundary changes**: failure behavior, retries, idempotency, observability
- **Risky details**: transactions, caching, migration, partial failure, silent failure
- **Test strategy**: what assumptions the tests verify, what failure paths are uncovered
- **Operational impact**: how operators know it works, how to rollback

Generic risk questions ("What are the risks?") are never generated. Every question is anchored to a specific file, function, decision, or diff in the provided PR.

## Installation

### With APM (Agent Package Manager)

```sh
apm install hansonkim/pr-understanding-question
```

### Manual (Claude Code)

Copy `SKILL.md` and `references/heuristics.md` into your `~/.claude/skills/pr-understanding-question/` directory.

```
~/.claude/skills/pr-understanding-question/
├── SKILL.md
└── references/
    └── heuristics.md
```

### Other runtimes

The skill works with any runtime that reads from `~/.agents/skills/`:

```sh
cp -r pr-understanding-question ~/.agents/skills/
```

Compatible with: Claude Code, Codex, Gemini CLI, OpenCode.

## Usage

Once installed, trigger the skill with natural language:

```
이 PR 이해도 확인해줘: [PR 내용]
```

```
이 PR 변경사항으로 개발자 이해도 질문 만들어줘
```

```
/pr-understanding-question [PR 내용 붙여넣기]
```

Provide as much context as available: PR title, description, code diff, design notes, known risks, reviewer comments. The more context, the more specific the questions.

## Evaluation scoring

| Score | Meaning |
|---|---|
| 0 | Incorrect or contradicts PR facts |
| 1 | Very shallow; mostly generic |
| 2 | Describes code behavior but misses design intent or risk |
| 3 | Explains behavior, intent, and major risks accurately |
| 4 | Explains intent, alternatives, trade-offs, risks, tests, and operational impact |

Scores are diagnostic, not judgmental. The goal is to surface gaps before production does.

## License

MIT
