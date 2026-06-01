# Dynamic Harness System (동적 하네스 시스템)

> A CLAUDE.md-based multi-agent harness that converts natural language into structured representations before execution — reducing ambiguity loss and inter-agent boundary violations.

> 자연어를 구조화 표현으로 변환한 뒤 실행하는 CLAUDE.md 기반 멀티에이전트 하네스. 모호성 손실과 에이전트 간 경계 침범을 줄이는 것이 핵심 목표.

---

## 배경 / Background

대부분의 멀티에이전트 하네스는 두 가지 문제를 가지고 있다.

1. **자연어 통신** — 에이전트 간 결과를 자연어로 주고받을 때 의미 손실이 발생한다
2. **역할 정의 부재** — 역할 이름만 있고 경계가 없으면 에이전트가 서로 침범하거나 아무도 처리하지 않는 영역이 생긴다

Most multi-agent harnesses share two failure modes:

1. **Natural language handoff** — passing results between agents as prose causes semantic loss
2. **Undefined role boundaries** — naming a role without defining what it must NOT do leads to scope overlap or blind spots

---

## 구조 / Architecture

```
자연어 입력 / Natural Language Input
    ↓
PHASE -1 : PARSER       — 자연어 → 구조화 표현 / NL → Structured Representation
    ↓
PHASE  0 : META AGENT   — 하네스 동적 설계 / Dynamic Harness Design
    ↓
PHASE  1 : EXECUTORS    — 필드:값 통신 / Field:Value Communication Only
    ↓
PHASE  2 : VERIFIER     — 경계 침범 및 계약 위반 검증 / Boundary & Contract Validation
    ↓
PHASE  3 : SYNTHESIZER  — 최종 출력 / Final Output
```

---

## 핵심 설계 원칙 / Key Design Principles

### 1. PARSER — 자연어 모호성 제거 / Ambiguity Elimination

자연어 입력을 INTENT / OBJECT / CONSTRAINT / AMBIGUITY / STRUCT 필드로 분해한다. AMBIGUITY가 있으면 임의 해석 없이 RESOLVED에 근거를 명시한다. 이후 모든 페이즈는 RAW가 아닌 STRUCT만 참조한다.

Decomposes natural language into INTENT / OBJECT / CONSTRAINT / AMBIGUITY / STRUCT. When ambiguity exists, the chosen interpretation and its rationale are recorded in RESOLVED. All subsequent phases reference STRUCT only — never RAW.

### 2. META AGENT — 역할 계약 3분할 / Three-Part Role Contract

에이전트를 역할명으로만 정의하지 않는다. 모든 에이전트는 세 가지를 반드시 명시한다.

Agents are never defined by name alone. Every agent must declare three things:

- **WHO** : 이 에이전트가 가진 관점 / The perspective this agent holds
- **WHAT** : 판단할 수 있는 것 **/** 판단하지 않을 것 / What it will decide **/** What it must not decide
- **OUT** : 다음 에이전트에게 넘기는 필드와 형식 / Fields and format passed to the next agent

### 3. EXECUTORS — RAW 참조 금지 / No RAW Reference

PHASE 1 이후 원래 자연어(RAW)를 다시 참조하지 않는다. 입력은 이전 에이전트의 OUT 필드 또는 PARSER STRUCT만 허용한다. 이는 PARSER가 제거한 모호함의 재유입을 차단한다.

After PHASE -1, no agent may reference the original natural language. Only the previous agent's OUT fields or PARSER STRUCT are valid inputs. This prevents re-introduction of ambiguity that PARSER eliminated.

### 4. VERIFIER — 경계 침범 검증 / Boundary Violation Detection

기존 검증 레이어가 품질만 확인한다면, 이 하네스의 VERIFIER는 **경계 침범**과 **RAW 재참조**를 추가로 검증한다.

Unlike typical quality gates, this VERIFIER additionally checks for **scope boundary violations** and **unauthorized RAW references**.

---

## 사용법 / Usage

프로젝트 루트에서 아래 명령어 한 줄로 설치한다.

Run this one-liner in your project root:

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/ERUTCEL/dynamic-harness-system/main/CLAUDE.md
```

그 다음 / Then:

1. Claude Code를 실행한다 / Run Claude Code
2. 자연어로 태스크를 입력한다 / Enter your task in natural language
3. 하네스가 자동으로 PHASE -1 → 3을 실행한다 / The harness runs PHASE -1 through 3 automatically

---

## 기존 하네스와의 비교 / Comparison with Existing Harnesses

| | 기존 / Existing | 이 하네스 / This Harness |
|---|---|---|
| 자연어 처리 | 그대로 에이전트에 전달 | PARSER가 구조화 후 전달 |
| NL Handling | Passed directly to agents | Structured by PARSER first |
| 역할 정의 | 역할명 또는 패턴 선택 | WHO / WHAT / OUT 3분할 강제 |
| Role Definition | Name or pattern selection | Mandatory WHO / WHAT / OUT split |
| 에이전트 간 통신 | 자연어 요약 | 필드:값 구조화 형식 |
| Inter-agent Comms | Natural language summary | Field:value structured format |
| 검증 | 품질 / 코드 게이팅 | 품질 + 경계 침범 + RAW 재참조 |
| Verification | Quality / code gating | Quality + boundary + RAW re-reference |
| 하네스 구조 | 미리 정의된 패턴 선택 | 태스크별 동적 생성 |
| Harness Structure | Pre-defined pattern selection | Dynamically generated per task |

---

## 파일 / Files

```
.
├── CLAUDE.md   — 하네스 전체 정의 / Full harness definition
└── README.md   — 이 파일 / This file
```

---

## 관련 레포 / Related Repos

- [revfactory/harness](https://github.com/revfactory/harness) — 패턴 기반 하네스 팩토리 / Pattern-based harness factory
- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — 하네스 엔지니어링 개념 / Harness engineering concepts
- [dralgorhythm/claude-agentic-framework](https://github.com/dralgorhythm/claude-agentic-framework) — 병렬 에이전트 프레임워크 / Parallel agent framework

---

## 라이선스 / License

MIT
