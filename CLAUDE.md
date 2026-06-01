# DYNAMIC HARNESS SYSTEM v3

Claude Code는 모든 태스크를 아래 구조에 따라 처리한다.
인간은 목표만 준다. 구조와 역할 계약은 Claude가 동적으로 설계한다.

```
자연어 입력
    ↓
PHASE -1 : PARSER       — 자연어 → 구조화 표현
    ↓
PHASE  0 : META AGENT   — 하네스 설계 (구조화 표현 기반)
    ↓
PHASE  1 : EXECUTORS    — 실행 (필드:값만 주고받음)
    ↓
PHASE  2 : VERIFIER     — 검증
    ↓
PHASE  3 : SYNTHESIZER  — 최종 출력
```

---

## PHASE -1 — PARSER (자연어 구조화)

태스크를 받으면 가장 먼저 실행한다.
자연어의 모호함을 제거하고 이후 모든 페이즈가 공유할 구조화 표현을 만든다.

```
[PARSER]
RAW       : {입력된 자연어 그대로}
INTENT    : {핵심 동사 — 분석 / 생성 / 비교 / 검증 / 변환 / 요약}
OBJECT    : {대상 — 무엇에 대한 태스크인가}
CONSTRAINT: {명시된 조건 — 없으면 NONE}
AMBIGUITY : {해석이 두 가지 이상 가능한 부분 — 없으면 NONE}
RESOLVED  : {AMBIGUITY가 있으면 선택한 해석과 근거, 없으면 NONE}
STRUCT    :
  intent   : {단일 키워드}
  object   : {단일 키워드}
  scope    : {처리 범위}
  condition: {조건 리스트 또는 NONE}
```

### PARSER 규칙
- RAW를 그대로 처리하지 않는다. 반드시 STRUCT로 변환한다
- AMBIGUITY가 있으면 임의로 해석하지 않고 RESOLVED에 근거를 명시한다
- 이후 모든 페이즈는 RAW가 아닌 STRUCT를 입력으로 사용한다
- 단순 질문도 PARSER는 실행한다. STRUCT가 단순하면 PHASE 0에서 단일 실행 판단

---

## PHASE 0 — META AGENT (하네스 설계)

PARSER의 STRUCT를 입력으로 받아 하네스를 설계한다.
자연어(RAW)를 다시 참조하지 않는다.

```
[META]
INPUT     : {PARSER STRUCT 그대로}
핵심 목표 : {STRUCT 기반 1줄 요약}
분기 필요 : {YES / NO} — 이유: {독립적 판단이 N개 존재 / 단일 추론으로 충분}
사용할 에이전트:

  AGENT-1: {역할명}
    WHO  : {이 에이전트가 가진 관점 — 무엇을 아는 존재인가}
    WHAT : {판단할 수 있는 것} / {판단하지 않을 것}
    OUT  : {다음 에이전트에게 넘기는 필드와 형식}

  AGENT-2: {역할명}
    WHO  : ...
    WHAT : ... / ...
    OUT  : ...

예상 실패 지점: {리스트}
검증 기준: {완료 조건}
```

### 분기 판단 기준
- 독립적인 두 가지 이상의 판단이 필요하면 → 분기
- 단일 추론으로 해결 가능하면 → 단일 실행 (에이전트 1개)
- 불확실성이 높으면 → VERIFIER 에이전트 추가
- 에이전트는 최대 3개. 초과 시 순차 실행으로 처리

### 역할 설계 원칙
- WHO가 없는 역할은 만들지 않는다 — 관점 없는 에이전트는 범위를 스스로 해석해 오염된다
- WHAT의 "판단하지 않을 것"을 반드시 명시한다 — 경계가 없으면 에이전트 간 침범이 생긴다
- OUT은 자연어가 아닌 필드:값 형식으로 고정한다 — 요약 손실 방지

---

## PHASE 1 — EXECUTOR AGENTS (실행)

META에서 정의한 계약대로 각 에이전트가 순서대로 실행된다.

```
[AGENT: {역할명}]
WHO   : {이 에이전트의 관점 — META 정의 그대로}
SCOPE : {이번 실행에서 판단한 것} / {판단하지 않은 것}
INPUT : {이전 에이전트 OUT 필드 또는 PARSER STRUCT}
OUTPUT:
  {필드1}: {값}
  {필드2}: {값}
  UNKNOWN: {판단 불가 항목, 없으면 NONE}
CONFIDENCE: {HIGH / MID / LOW}
```

### 실행 규칙
- INPUT은 이전 에이전트의 OUT 필드 또는 PARSER STRUCT만 사용한다. RAW 참조 금지
- SCOPE의 "판단하지 않은 것"을 반드시 기록한다
- CONFIDENCE LOW이면 다음 에이전트에게 LOW 항목을 명시한다
- 모르는 것은 UNKNOWN으로 표기한다. 추측하지 않는다
- 다른 에이전트의 SCOPE를 침범하지 않는다

---

## PHASE 2 — VERIFIER AGENT (검증)

모든 EXECUTOR 출력이 끝나면 실행된다.

```
[VERIFIER]
목표 달성  : {YES / PARTIAL / NO}
RAW 참조   : {PHASE 1에서 RAW를 참조한 에이전트 — 있으면 명시, 없으면 NONE}
경계 침범  : {에이전트 간 SCOPE 침범 여부 — 있으면 명시, 없으면 NONE}
LOW CONF  : {LOW CONFIDENCE 항목 — 있으면 나열, 없으면 NONE}
UNKNOWN   : {있으면 나열, 없으면 NONE}
계약 위반  : {OUT 형식 미준수 여부 — 있으면 명시, 없으면 NONE}
재시도 필요: {YES / NO}
판정       : {PASS / RETRY}
```

### RETRY 규칙
- 실패한 에이전트만 재실행한다. 전체 재시작 금지
- 재시도 시 META의 WHO / WHAT / OUT을 재확인하고 입력을 재구성한다
- 재시도는 최대 2회. 이후엔 UNKNOWN으로 확정한다

---

## PHASE 3 — SYNTHESIZER (최종 출력)

```
[SYNTHESIS]
결론      : {핵심 답변}
근거      : {각 에이전트 OUTPUT 필드 기반 — 자연어 재해석 금지}
불확실 항목: {UNKNOWN으로 남은 것}
한계      : {이 하네스가 다루지 못한 것}
```

---

## 전역 규칙

1. PHASE -1 없이 실행하지 않는다
2. PHASE 1 이후 RAW를 참조하지 않는다. 모든 입력은 STRUCT 또는 이전 OUT
3. 역할 정의에 WHO / WHAT / OUT이 없으면 에이전트를 만들지 않는다
4. 에이전트는 자신의 SCOPE 밖을 판단하지 않는다
5. 추측하지 않는다. 모르면 UNKNOWN
6. 단순 질문도 PARSER는 실행한다. STRUCT가 단순하면 단일 실행으로 처리

---

## 예시 — 태스크: "이 코드 좀 빠르게 만들어줘"

```
[PARSER]
RAW       : "이 코드 좀 빠르게 만들어줘"
INTENT    : 변환
OBJECT    : 코드
CONSTRAINT: NONE
AMBIGUITY : "빠르게"의 기준이 모호 — 실행속도 / 메모리 / 응답시간 중 무엇인가
RESOLVED  : 실행속도로 해석 — 가장 일반적인 성능 기준
STRUCT    :
  intent   : 최적화
  object   : 코드
  scope    : 실행속도 개선
  condition: NONE

[META]
INPUT     : intent=최적화, object=코드, scope=실행속도개선
핵심 목표 : 코드 실행속도 병목 진단 및 개선
분기 필요 : YES — 이유: 진단(현상)과 처방(개선)은 독립적 판단
사용할 에이전트:

  AGENT-1: 진단
    WHO  : 코드를 정적으로 읽는 성능 분석가
           실행속도 패턴과 안티패턴을 아는 존재
    WHAT : 병목 위치와 원인 식별 /
           개선 방법은 판단하지 않음
    OUT  :
      bottleneck: {위치: 원인} 형식
      severity: HIGH / MID / LOW
      UNKNOWN: 런타임 환경 의존 항목

  AGENT-2: 처방
    WHO  : 진단 OUT만 보는 개선 설계자
           원래 코드를 직접 보지 않음
    WHAT : 실행속도 개선안 설계 /
           원인 재분석 하지 않음
    OUT  :
      fix: {항목별 구체 개선안}
      tradeoff: {개선 시 발생하는 트레이드오프}
      UNKNOWN: 환경 정보 없이 판단 불가 항목

예상 실패 지점: 런타임 환경 정보 부재
검증 기준: bottleneck과 fix 1:1 대응 확인

[AGENT: 진단]
WHO   : 정적 성능 분석가
SCOPE : 병목 위치/원인 식별 / 개선안 제외
INPUT : STRUCT {intent=최적화, scope=실행속도개선} + 코드
OUTPUT:
  bottleneck: {line 23: O(n²) 중첩루프, line 47: 반복 DB호출}
  severity: HIGH
  UNKNOWN: 실제 데이터 크기, 호출 빈도
CONFIDENCE: MID

[AGENT: 처방]
WHO   : 개선 설계자
SCOPE : 실행속도 개선안 설계 / 원인 재분석 제외
INPUT : bottleneck={line23:O(n²), line47:반복DB호출}, severity=HIGH
OUTPUT:
  fix: {line23→해시맵으로 O(n)변환, line47→배치처리}
  tradeoff: {메모리 증가, 코드 복잡도 소폭 상승}
  UNKNOWN: 배치 크기 최적값 (데이터 크기 모름)
CONFIDENCE: MID

[VERIFIER]
목표 달성  : YES
RAW 참조   : NONE
경계 침범  : NONE
LOW CONF  : 두 에이전트 MID — 런타임 정보 부재
UNKNOWN   : 데이터 크기, 호출 빈도, 배치 크기 최적값
계약 위반  : NONE
재시도 필요: NO
판정       : PASS

[SYNTHESIS]
결론      : O(n²) 루프와 반복 DB호출이 주요 병목. 해시맵 전환과 배치 처리로 개선 가능
근거      : 진단 bottleneck → 처방 fix 1:1 대응
불확실 항목: 데이터 크기/빈도 없어 배치 크기 최적값 미확정
한계      : 런타임 프로파일링 없이 정적 분석만 수행
```
