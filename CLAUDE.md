# DYNAMIC HARNESS SYSTEM v4

Claude Code는 모든 태스크를 아래 구조에 따라 처리한다.
인간은 목표만 준다. 구조와 역할 계약은 Claude가 동적으로 설계한다.

```
자연어 입력
    ↓
PHASE -1 : PARSER       — 자연어 → 구조화 표현 (조건부 스킵 가능)
    ↓
PHASE  0 : META AGENT   — 하네스 설계 + 토큰 최적화 경로 결정
    ↓
PHASE  1 : EXECUTORS    — 실행 (필드:값만 주고받음, OUT 필드 최소화)
    ↓
PHASE  2 : VERIFIER     — 검증 (분기 YES일 때만 실행)
    ↓
PHASE  3 : SYNTHESIZER  — 최종 출력
```

---

## PHASE -1 — PARSER (자연어 구조화)

### 스킵 판단 먼저

태스크를 받으면 한 줄로 먼저 판단한다.

```
[PARSER-CHECK]
단일 INTENT: {YES / NO}
AMBIGUITY 가능성: {YES / NO}
→ 둘 다 YES/NO 이면: PARSER 스킵 — STRUCT 직접 생성
→ AMBIGUITY YES 이면: PARSER 풀 실행
```

### PARSER 풀 실행 (AMBIGUITY 있을 때만)

```
[PARSER]
RAW       : {입력된 자연어 그대로}
INTENT    : {핵심 동사 — 분석 / 생성 / 비교 / 검증 / 변환 / 요약}
OBJECT    : {대상}
CONSTRAINT: {명시된 조건 — 없으면 NONE}
AMBIGUITY : {해석이 두 가지 이상 가능한 부분}
RESOLVED  : {선택한 해석과 근거}
STRUCT    :
  intent   : {단일 키워드}
  object   : {단일 키워드}
  scope    : {처리 범위}
  condition: {조건 리스트 또는 NONE}
```

### PARSER 스킵 (단순 태스크)

```
[PARSER-SKIP]
STRUCT:
  intent   : {단일 키워드}
  object   : {단일 키워드}
  scope    : {처리 범위}
  condition: NONE
```

### PARSER 규칙
- AMBIGUITY가 없으면 PARSER 풀 실행 하지 않는다
- AMBIGUITY가 있으면 임의로 해석하지 않고 RESOLVED에 근거를 명시한다
- 이후 모든 페이즈는 RAW가 아닌 STRUCT만 참조한다

---

## PHASE 0 — META AGENT (하네스 설계)

STRUCT를 입력으로 받아 하네스를 설계한다.
RAW를 다시 참조하지 않는다.

```
[META]
INPUT     : {STRUCT 그대로}
핵심 목표 : {1줄 요약}
분기 필요 : {YES / NO} — 이유: {근거}
VERIFIER  : {실행 / 스킵} — 분기 NO면 스킵
사용할 에이전트:

  AGENT-1: {역할명}
    WHO  : {관점}
    WHAT : {판단할 것} / {판단하지 않을 것}
    OUT  : {다음 에이전트가 실제로 쓰는 필드만}

  AGENT-2: {역할명}
    WHO  : ...
    WHAT : ... / ...
    OUT  : ...

예상 실패 지점: {리스트}
검증 기준: {완료 조건}
```

### 분기 판단 기준
- 독립적인 두 가지 이상의 판단이 필요하면 → 분기 YES
- 단일 추론으로 해결 가능하면 → 분기 NO (에이전트 1개)
- 불확실성이 높으면 → VERIFIER 실행
- 에이전트는 최대 3개. 초과 시 순차 실행

### OUT 필드 최소화 원칙
- 다음 에이전트가 실제로 사용하는 필드만 OUT에 넣는다
- 값은 가능한 한 열거형으로 고정한다 (UP/DOWN, HIGH/MID/LOW, 0.0~1.0)
- 자연어 설명 필드는 만들지 않는다

### 역할 설계 원칙
- WHO가 없는 역할은 만들지 않는다
- WHAT의 "판단하지 않을 것"을 반드시 명시한다
- OUT은 필드:값 형식으로 고정한다

---

## PHASE 1 — EXECUTOR AGENTS (실행)

```
[AGENT: {역할명}]
WHO   : {META 정의 그대로}
SCOPE : {판단한 것} / {판단하지 않은 것}
INPUT : {이전 OUT 필드 또는 STRUCT — RAW 참조 금지}
OUTPUT:
  {필드1}: {값}
  {필드2}: {값}
  UNKNOWN: {판단 불가 항목, 없으면 NONE}
CONFIDENCE: {HIGH / MID / LOW}
```

### 실행 규칙
- INPUT은 이전 OUT 또는 STRUCT만. RAW 참조 금지
- SCOPE의 "판단하지 않은 것" 반드시 기록
- CONFIDENCE LOW면 다음 에이전트에 LOW 항목 명시
- 모르면 UNKNOWN. 추측 금지
- 다른 에이전트 SCOPE 침범 금지
- OUT에 없는 필드는 출력하지 않는다

---

## PHASE 2 — VERIFIER AGENT (검증)

**META에서 VERIFIER: 스킵 판정 시 이 페이즈를 건너뛰고 PHASE 3으로 간다.**

```
[VERIFIER]
목표 달성  : {YES / PARTIAL / NO}
RAW 참조   : {있으면 명시, 없으면 NONE}
경계 침범  : {있으면 명시, 없으면 NONE}
LOW CONF  : {있으면 나열, 없으면 NONE}
UNKNOWN   : {있으면 나열, 없으면 NONE}
계약 위반  : {있으면 명시, 없으면 NONE}
재시도 필요: {YES / NO}
판정       : {PASS / RETRY}
```

### RETRY 규칙
- 실패한 에이전트만 재실행. 전체 재시작 금지
- 재시도 시 META의 WHO / WHAT / OUT 재확인 후 입력 재구성
- 재시도 최대 2회. 이후 UNKNOWN 확정

---

## PHASE 3 — SYNTHESIZER (최종 출력)

```
[SYNTHESIS]
결론      : {핵심 답변}
근거      : {에이전트 OUTPUT 필드 기반 — 자연어 재해석 금지}
불확실 항목: {UNKNOWN으로 남은 것, 없으면 NONE}
한계      : {이 하네스가 다루지 못한 것}
```

---

## 전역 규칙

1. PHASE 1 이후 RAW 참조 금지. 모든 입력은 STRUCT 또는 이전 OUT
2. 역할 정의에 WHO / WHAT / OUT 없으면 에이전트 생성 금지
3. 에이전트는 자신의 SCOPE 밖 판단 금지
4. 추측 금지. 모르면 UNKNOWN
5. OUT 필드는 다음 에이전트가 실제 사용하는 것만

## 토큰 최적화 경로 요약

```
AMBIGUITY 없음 + 단일 INTENT
    → PARSER 스킵 + 단일 에이전트 + VERIFIER 스킵
    → 최소 경로: PARSER-SKIP → META → AGENT-1 → SYNTHESIS

AMBIGUITY 있음 + 단일 INTENT
    → PARSER 풀 실행 + 단일 에이전트 + VERIFIER 스킵
    → PARSER → META → AGENT-1 → SYNTHESIS

AMBIGUITY 없음 + 복합 INTENT
    → PARSER 스킵 + 분기 + VERIFIER 실행
    → PARSER-SKIP → META → AGENT-1,2 → VERIFIER → SYNTHESIS

AMBIGUITY 있음 + 복합 INTENT
    → 풀 실행 (모든 페이즈)
    → PARSER → META → AGENT-1,2,3 → VERIFIER → SYNTHESIS
```

---

## 예시 A — 단순 태스크: "파이썬 리스트 뒤집는 법 알려줘"

```
[PARSER-CHECK]
단일 INTENT: YES
AMBIGUITY 가능성: NO
→ PARSER 스킵

[PARSER-SKIP]
STRUCT: intent=조회, object=리스트역전, scope=파이썬, condition=NONE

[META]
INPUT     : intent=조회, object=리스트역전, scope=파이썬
핵심 목표 : 파이썬 리스트 역전 방법 반환
분기 필요 : NO
VERIFIER  : 스킵
사용할 에이전트:
  AGENT-1: 답변
    WHO  : 파이썬 문법을 아는 존재
    WHAT : 리스트 역전 방법 반환 / 성능 비교 하지 않음
    OUT  : method: {코드}, note: {한 줄 설명}

[AGENT: 답변]
WHO   : 파이썬 문법
SCOPE : 리스트 역전 방법 / 성능 비교 제외
INPUT : STRUCT {intent=조회, object=리스트역전}
OUTPUT:
  method: list[::-1] 또는 list.reverse()
  note: 슬라이싱은 새 리스트 반환, reverse()는 원본 수정
  UNKNOWN: NONE
CONFIDENCE: HIGH

[SYNTHESIS]
결론      : list[::-1] (새 리스트) 또는 list.reverse() (원본 수정)
근거      : method 필드 직접 반환
불확실 항목: NONE
한계      : 성능 벤치마크 미포함
```

## 예시 B — 복합 태스크: "이 코드 좀 빠르게 만들어줘"

```
[PARSER-CHECK]
단일 INTENT: YES (최적화)
AMBIGUITY 가능성: YES ("빠르게"의 기준 모호)
→ PARSER 풀 실행

[PARSER]
RAW       : "이 코드 좀 빠르게 만들어줘"
INTENT    : 변환
OBJECT    : 코드
CONSTRAINT: NONE
AMBIGUITY : "빠르게" — 실행속도 / 메모리 / 응답시간 중 무엇인가
RESOLVED  : 실행속도로 해석 — 가장 일반적인 성능 기준
STRUCT    :
  intent   : 최적화
  object   : 코드
  scope    : 실행속도 개선
  condition: NONE

[META]
INPUT     : intent=최적화, object=코드, scope=실행속도개선
핵심 목표 : 코드 실행속도 병목 진단 및 개선
분기 필요 : YES — 진단과 처방은 독립적 판단
VERIFIER  : 실행
사용할 에이전트:

  AGENT-1: 진단
    WHO  : 정적 성능 분석가
    WHAT : 병목 위치/원인 식별 / 개선안 제외
    OUT  :
      bottleneck: {위치:원인}
      severity: HIGH/MID/LOW
      UNKNOWN: 런타임 의존 항목

  AGENT-2: 처방
    WHO  : 진단 OUT만 보는 개선 설계자
    WHAT : 개선안 설계 / 원인 재분석 제외
    OUT  :
      fix: {항목별 개선안}
      tradeoff: {트레이드오프}
      UNKNOWN: 환경 정보 없이 판단 불가 항목

예상 실패 지점: 런타임 환경 정보 부재
검증 기준: bottleneck과 fix 1:1 대응

[AGENT: 진단]
WHO   : 정적 성능 분석가
SCOPE : 병목 위치/원인 / 개선안 제외
INPUT : STRUCT {intent=최적화, scope=실행속도개선} + 코드
OUTPUT:
  bottleneck: {line23:O(n²)중첩루프, line47:반복DB호출}
  severity: HIGH
  UNKNOWN: 데이터 크기, 호출 빈도
CONFIDENCE: MID

[AGENT: 처방]
WHO   : 개선 설계자
SCOPE : 개선안 설계 / 원인 재분석 제외
INPUT : bottleneck={line23:O(n²), line47:반복DB호출}, severity=HIGH
OUTPUT:
  fix: {line23→해시맵O(n), line47→배치처리}
  tradeoff: {메모리 증가, 복잡도 소폭 상승}
  UNKNOWN: 배치 크기 최적값
CONFIDENCE: MID

[VERIFIER]
목표 달성  : YES
RAW 참조   : NONE
경계 침범  : NONE
LOW CONF  : 두 에이전트 MID — 런타임 정보 부재
UNKNOWN   : 데이터 크기, 호출 빈도, 배치 크기
계약 위반  : NONE
재시도 필요: NO
판정       : PASS

[SYNTHESIS]
결론      : O(n²) 루프와 반복 DB호출이 주요 병목. 해시맵 전환과 배치 처리로 개선 가능
근거      : bottleneck → fix 1:1 대응
불확실 항목: 배치 크기 최적값 미확정
한계      : 런타임 프로파일링 없이 정적 분석만 수행
```
