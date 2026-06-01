# Dynamic Harness System

> A `CLAUDE.md` for Claude Code that eliminates the #1 cause of bad multi-agent output — ambiguous natural language passed directly between agents.

> Claude Code용 `CLAUDE.md`. 멀티에이전트 출력 품질을 떨어뜨리는 가장 큰 원인 — 모호한 자연어가 에이전트에 그대로 전달되는 것 — 을 구조로 차단한다.

---

## 이걸 왜 써야 하나 / Why This Exists

Claude Code로 복잡한 작업을 시킬 때 이런 일이 생긴다.  
When running complex tasks with Claude Code, these problems appear.

**상황 1 — 에이전트가 서로 다른 걸 이해함**  
**Case 1 — Agents interpret the same input differently**

```
입력 / Input: "이 코드 성능 좋게 해줘" / "Make this code faster"

에이전트 A (진단) / Agent A (Diagnosis): 메모리 기준으로 분석 / analyzes by memory
에이전트 B (처방) / Agent B (Fix):      실행속도 기준으로 개선안 작성 / writes fix by execution speed
→ 진단과 처방이 서로 다른 기준 / Diagnosis and fix use different criteria
→ 결과가 어긋남 / Output is misaligned
```

**상황 2 — 아무도 안 하는 영역이 생김**  
**Case 2 — Blind spots appear between agents**

```
역할 / Roles: "분석가" / "Analyst", "개선 담당" / "Fixer"
→ 경계가 없으니 둘 다 분석만 함 / No boundary — both just analyze
→ 실제 개선안은 아무도 안 씀 / Nobody writes the actual fix
```

**상황 3 — 재시도가 반복됨**  
**Case 3 — Retries pile up**

```
모호한 입력 → 잘못된 방향 실행 → 재시도
Ambiguous input → wrong execution → retry
→ 토큰 2배, 시간 2배 / 2x tokens, 2x time
```

이 하네스는 이 세 가지를 구조로 차단한다.  
This harness blocks all three structurally.

---

## 어떻게 작동하나 / How It Works

자연어가 에이전트에 직접 들어가는 걸 막는다.  
Prevents natural language from entering agents directly.

```
기존 방식 / Before:
"코드 빠르게 해줘" → Agent A → Agent B → 결과
                      (각자 해석)  (각자 해석)
"Make it faster"  → Agent A → Agent B → output
                    (own interp.) (own interp.)

이 하네스 / This harness:
"코드 빠르게 해줘" / "Make it faster"
    ↓
PARSER: "빠르게" = 실행속도로 확정 / "faster" = execution speed, locked
    ↓
Agent A: intent=최적화, scope=실행속도 (고정값 / fixed value)
    ↓
Agent B: bottleneck=line23:O(n²) (필드:값 / field:value, not prose)
    ↓
결과 / Output
```

모든 에이전트가 동일한 구조화된 표현을 공유한다.  
All agents share the same structured representation.

---

## 실제로 뭐가 달라지나 / What Actually Changes

**역할 정의가 바뀐다 / Role definition changes**

```
기존 / Before:
  - "데이터 분석가" / "Data Analyst"
  - "개선 담당자" / "Fixer"

이 하네스 / This harness:
  - WHO : 데이터를 정적으로 읽는 분석가 / Analyst who reads data statically
  - WHAT: 패턴 식별만 / 개선안은 판단하지 않음
          Pattern identification only / Must NOT suggest fixes
  - OUT : severity: HIGH/MID/LOW, pattern: {field:value}
```

**에이전트 간 통신이 바뀐다 / Inter-agent communication changes**

```
기존 / Before:
  "외인 매도세가 강하게 나타나고 있으며 거래량도 높음"
  "Foreign selling pressure is strong with elevated volume"

이 하네스 / This harness:
  foreign_flow: -284.7B KRW, volume_ratio: 1.34, trend: DOWN
```

자연어 요약 대신 필드:값. 다음 에이전트가 재해석할 여지가 없다.  
Field:value instead of prose. No room for reinterpretation.

**불필요한 페이즈를 건너뛴다 / Unnecessary phases are skipped**

```
단순 질문 / Simple query  → PARSER 스킵 + VERIFIER 스킵 → 최소 경로 / minimal path
복잡한 작업 / Complex task → 풀 실행 / full execution
```

---

## 누가 쓰면 좋나 / Who Should Use This

Claude Code로 이런 작업을 반복하는 사람:  
Anyone using Claude Code for tasks like these:

- 코드베이스 분석 후 리팩토링 / Codebase analysis and refactoring
- 데이터 수집 → 분석 → 판단 파이프라인 / Data collection → analysis → decision pipeline
- 문서 파싱 → 구조화 → 출력 변환 / Document parsing → structuring → output transformation
- 여러 관점의 검토가 필요한 의사결정 / Decisions requiring multiple perspectives

한 번에 끝나는 단순 질문보다 **단계가 나뉘는 작업**에서 효과가 크다.  
Works best on **multi-step tasks**, not one-shot simple queries.

---

## 사용법 / Usage

프로젝트 루트에서 아래 명령어 한 줄로 설치한다.

Run this one-liner in your project root:

```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/ERUTCEL/dynamic-harness-system/main/CLAUDE.md
```

그 다음 / Then:

1. Claude Code 실행 / Run Claude Code
2. 평소처럼 자연어로 태스크 입력 / Enter your task in natural language as usual
3. 하네스가 자동으로 최적 경로 선택 후 실행 / The harness selects the optimal path and runs

별도 설정 없음. 파일 하나로 작동한다.  
No extra configuration. One file, plug and play.

---

## 설계 원칙 / Design Principles

| 원리 / Principle | 이유 / Reason |
|---|---|
| 먼저 구조화한다 / Structure first | 자연어는 단계마다 다르게 읽힌다. 의도를 한 번 고정하고 이후엔 그 값만 쓴다 / Natural language shifts at every step. Fix the intent once and use only that |
| 역할에 경계를 준다 / Bound every role | "안 할 것"이 없으면 역할들이 침범하거나 아무도 안 하는 영역이 생긴다 / Without must-NOT, roles overlap or leave blind spots |
| 필드로 대화한다 / Talk in fields | 자연어 요약은 전달 과정에서 의미를 잃는다. 숫자와 열거형은 잃지 않는다 / Prose loses meaning in transit. Numbers and enums don't |
| 막히면 접근을 바꾼다 / Change approach, not retry | 같은 방식으로 재시도하면 같은 결과가 나온다 / Retrying the same way produces the same result |
| 단순한 건 단순하게 / Simple stays simple | 구조를 위한 구조는 낭비다 / Structure for its own sake is waste |

---

## 관련 레포 / Related

- [revfactory/harness](https://github.com/revfactory/harness) — 패턴 기반 하네스 팩토리 / Pattern-based harness factory
- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — 하네스 엔지니어링 개념 / Harness engineering concepts

---

## 파일 구성 / Files

```
.
├── CLAUDE.md   — 하네스 전체 정의 / Full harness definition
└── README.md   — 이 파일 / This file
```

---

## 라이선스 / License

MIT
