# AJT Layer Architecture — 판단 구조 봉인

**문서 목적**: 이 문서는 기능 구현이 아니라 판단 구조 봉인 작업이다. 이후 어떤 모델, 가드레일, 프롬프트가 바뀌어도 이 레이어 맵은 변경 대상이 아니다.

**작업 원칙**: 혼재된 판단 언어를 레이어별로 강제 분리하고, 이후 어떤 구현이 와도 흔들리지 않는 기준면을 만든다.

**날짜**: 2026-01-13
**상태**: FROZEN (Constitutional Document)

---

## 레이어 분해 원칙

AJT는 하나의 흐름이 아닌 **최소 5개의 레이어**로 분해된다.

**분해 기준**:
- 계산 순서 ❌
- 책임과 종결성 ✅

각 레이어는 고유한 질문을 담당하며, 상태 언어는 레이어별로 고정된다.

---

## Layer 1: Process Control Layer

### 책임
**질문**: "지금 이 작업을 계속 실행해도 되는가?"

### 범위
- 시간 (timeout, scheduling)
- 리소스 (memory, CPU, quota)
- 동기화 (lock, mutex, queue)

### 범위 밖 (절대 금지)
- 판단 (judgment)
- 결론 (conclusion)
- 정책 (policy)

### 상태 언어 (고정)
```
Allow       — 작업 실행 가능
Pause(Hold) — 재개 가능한 내부 상태 (리소스 대기, 동기화 대기)
```

### 중요 제약
**Pause(Hold)는 내부 상태이며, 외부 출력이나 API 결과로 노출되는 순간 오류로 간주한다.**

### 예시
```
✅ "리소스 사용 가능 → Allow"
✅ "큐 대기 중 → Pause(Hold)"
❌ "정책 위반으로 Pause" (이것은 Layer 2)
❌ "증거 부족으로 Hold" (이것은 Layer 3)
```

### 구현 매핑
- OS scheduler
- Database connection pool
- Rate limiter
- Circuit breaker (리소스 관점만)

---

## Layer 2: Safety / Policy Gate Layer

### 책임
**질문**: "이 요청을 통과시켜도 되는가?"

### 범위
- 정책 (policy rules)
- 규제 (regulatory compliance)
- 금지 조건 (forbidden patterns)

### 상태 언어 (고정)
```
Stop  — 정책 위반, 차단 (종결)
Hold  — 조건 미충족, 추가 검증 필요 (비종결)
Allow — 정책 통과
```

### 중요 제약
**Hold는 결론이 아니다.** 조건 미충족 또는 추가 검증 필요를 의미하는 비종결 상태이다.

### 다중 Gate 허용
이 레이어는 다중으로 존재할 수 있다:
- Content Policy Gate (욕설, 차별 표현)
- External Transmission Gate (외부 도메인 체크)
- PII Detection Gate (개인정보 탐지)

각 Gate는 독립적으로 Stop/Hold/Allow를 반환한다.

### 예시
```
✅ "외부 도메인 발견 → Stop"
✅ "PII 탐지됨, 마스킹 대기 → Hold"
✅ "내부 이메일만 포함 → Allow"
❌ "증거 부족 → Hold" (이것은 Layer 3)
❌ "판단 불가 → Hold" (이것은 Layer 4의 Indeterminate)
```

### 구현 매핑
- Guardrail systems
- Policy engines
- Regex-based filters
- ML-based classifiers (policy 목적만)

---

## Layer 3: Evidence / Observation Layer

### 책임
**질문**: "이 판단에 필요한 관측 증거가 존재하는가?"

### 범위
- 관측 가능한 증거의 존재 여부
- 증거의 충분성 (sufficiency)
- 증거의 신뢰성 (reliability)

### 범위 밖 (절대 금지)
**Prior(상식)로 빈칸을 메우는 행위**

### 상태 언어 (고정)
```
Sufficient Evidence   — 판단에 필요한 증거 확보
Insufficient Evidence — 증거 부족 (prior로 채우지 않음)
```

### 핵심 원칙
이 레이어는 **prior(상식)로 빈칸을 메우는 행위를 차단하는 핵심**이다.

Insufficient Evidence일 경우:
- 상위 레이어로 되돌리거나
- 바로 Judgment Outcome Layer로 전달 (Indeterminate로 종결)

### 예시
```
✅ "이메일 수신자 도메인 확인됨 → Sufficient Evidence"
✅ "첨부파일 메타데이터 없음 → Insufficient Evidence"
❌ "첨부파일이 보통 안전하니까 Allow" (prior 사용 금지)
❌ "이전에 비슷한 케이스가 괜찮았으니 Allow" (prior 사용 금지)
```

### 구현 매핑
- Feature extraction
- Metadata validation
- Observable signals collection
- Confidence scoring (not decision making)

---

## Layer 4: Judgment Outcome Layer

### 책임
**질문**: "최종 판단 결과는 무엇인가?"

### 특성
- AJT의 **공식 결과 레이어**
- **종결 지점** 담당
- 외부 노출 레이어

### 상태 언어 (고정)
```
Stop          — 차단 (종결)
Allow         — 통과 (종결)
Indeterminate — 현재 조건에서 결정 불가 (종결)
```

### Indeterminate의 정의
**Indeterminate는 실패나 보류가 아니라 "현재 조건에서 결정 불가"라는 정식 판단 결과이다.**

Indeterminate 발생 조건:
- Layer 3에서 Insufficient Evidence 전달
- 다중 Gate에서 상충되는 결과 (Gate A: Allow, Gate B: Stop)
- 판단 기준 자체가 정의되지 않음

### 중요 제약
**이 레이어 이후에는 자동 재시도나 암묵적 Allow가 절대 발생하지 않는다.**

### 예시
```
✅ "외부 도메인 + 정책 위반 → Stop"
✅ "내부 이메일 + 정책 통과 → Allow"
✅ "수신자 도메인 확인 불가 → Indeterminate"
❌ "Indeterminate이니까 일단 Allow" (금지)
❌ "Indeterminate이니까 재시도" (금지)
```

### 외부 노출 규칙
**외부 API, UI, 사용자에게 노출되는 상태 언어는 이 레이어의 세 가지로 제한된다:**
- Stop
- Allow
- Indeterminate

### 구현 매핑
- Final decision API endpoint
- User-facing status code
- Audit log primary status

---

## Layer 5: Accountability / Trace Layer

### 책임
**질문**: "왜 이런 결론이 나왔는가?"

### 범위
- 로그 (log)
- 트레이스 (trace)
- 증거 체인 (evidence chain)
- 감사 기록 (audit trail)

### 특성
- **모든 Judgment Outcome은 이 레이어로 의무적으로 연결된다**
- **이 레이어는 결과를 바꾸지 않는다**
- **Allow도 로그 대상이다** (통과한 것도 기록)

### 로그 필수 필드
```json
{
  "event_id": "evt_20260113_...",
  "judgment_outcome": "Stop | Allow | Indeterminate",
  "timestamp": "ISO8601",
  "layers_executed": [
    {
      "layer": "Process Control Layer",
      "result": "Allow",
      "latency_ms": 5
    },
    {
      "layer": "Safety Gate Layer",
      "gate": "External Transmission Gate",
      "result": "Stop",
      "evidence": ["recipient_domain=court.gov"]
    },
    {
      "layer": "Evidence Layer",
      "result": "Sufficient Evidence"
    },
    {
      "layer": "Judgment Outcome Layer",
      "result": "Stop"
    }
  ],
  "approver_identity": "user@example.com (if human override)",
  "final_action": "blocked | sent | deferred"
}
```

### 예시
```
✅ Allow → 로그 (정상 통과도 기록)
✅ Stop → 로그 + 차단 사유
✅ Indeterminate → 로그 + 증거 부족 상세
❌ 로그 없이 Allow (금지)
❌ "로그 실패했으니 일단 Allow" (금지)
```

### 구현 매핑
- PostgreSQL append-only log
- AJT Log table (immutable)
- OpenTelemetry traces
- Audit compliance reports

---

## 레이어 간 상태 전파 규칙

### 규칙 1: Hold는 상위 레이어로만 전파
```
Process Control Layer (Pause/Hold)
    → Safety Gate Layer로 전달되지 않음
    → 내부 재시도 또는 종료

Safety Gate Layer (Hold)
    → Evidence Layer로 전달 (증거 추가 수집 요청)
    → 또는 Judgment Outcome Layer로 전달 (Indeterminate로 종결)
```

### 규칙 2: Indeterminate는 Judgment Outcome Layer에서만 생성
```
Evidence Layer (Insufficient Evidence)
    → Judgment Outcome Layer (Indeterminate)

Safety Gate Layer (Hold + timeout)
    → Judgment Outcome Layer (Indeterminate)

❌ Process Control Layer에서 Indeterminate 생성 금지
❌ Safety Gate Layer에서 Indeterminate 반환 금지
```

### 규칙 3: 모든 종결 상태는 Layer 5로 의무 전파
```
Judgment Outcome Layer (Stop/Allow/Indeterminate)
    → MUST propagate to Accountability Layer
    → 로그 실패 시 전체 작업 실패로 처리
```

---

## 구현 제약 (Implementation Constraints)

### 제약 1: 단어 사용 제한
```
Hold         — Layer 1, Layer 2에서만 사용
             — Layer 4에서 절대 사용 금지

Indeterminate — Layer 4에서만 사용
              — Layer 1, 2, 3에서 절대 사용 금지

Pause        — Layer 1에서만 사용
             — 외부 API 노출 금지
```

### 제약 2: 외부 노출 제한
**외부 API, UI, 사용자에게 노출되는 상태 언어:**
```
Stop
Allow
Indeterminate
```

**절대 노출 금지:**
```
Hold
Pause
Pending
Waiting
Processing
```

### 제약 3: 암묵적 Allow 금지
```
❌ Indeterminate → 자동 Allow
❌ Hold + timeout → 자동 Allow
❌ Evidence Insufficient → 자동 Allow
❌ 로그 실패 → 자동 Allow
```

### 제약 4: Prior 사용 금지
```
❌ "보통 이런 경우는 안전하니까..."
❌ "이전 케이스가 통과했으니까..."
❌ "상식적으로 판단하면..."
❌ "일반적으로는..."
```

Evidence Layer에서 Insufficient Evidence일 경우:
- Indeterminate로 종결
- 또는 사람에게 판단 위임

---

## 머지 정책 (Merge Policy)

### 필수 요구사항
**모든 코드, 레포, 실험은 "어느 레이어의 언어를 쓰는가"를 명시하지 않으면 머지하지 않는다.**

### 코드 주석 예시
```python
# Layer 2: Safety / Policy Gate Layer
# State language: Stop / Hold / Allow
def check_external_domain(recipient: str) -> GateResult:
    if is_external(recipient):
        return GateResult.STOP  # ✅ Layer 2 언어
    return GateResult.ALLOW

# Layer 4: Judgment Outcome Layer
# State language: Stop / Allow / Indeterminate
def finalize_judgment(gate_results: List[GateResult]) -> JudgmentOutcome:
    if any(r == GateResult.STOP for r in gate_results):
        return JudgmentOutcome.STOP  # ✅ Layer 4 언어
    if any(r == GateResult.HOLD for r in gate_results):
        return JudgmentOutcome.INDETERMINATE  # ✅ Hold → Indeterminate 변환
    return JudgmentOutcome.ALLOW
```

### 금지 예시
```python
# ❌ 레이어 명시 없음
def process_request(data):
    if check_policy(data):
        return "OK"  # ❌ 상태 언어 불명확
    return "NG"

# ❌ 레이어 혼재
def validate(data):
    # Layer 2 언어와 Layer 4 언어 혼재
    if data.is_safe:
        return "Allow"
    if data.need_review:
        return "Indeterminate"  # ❌ Layer 2에서 Indeterminate 사용 금지
    return "Stop"
```

### Pull Request 체크리스트
```
- [ ] 코드에 레이어 명시 (주석 또는 타입)
- [ ] 상태 언어가 해당 레이어의 고정 언어와 일치
- [ ] Hold/Indeterminate 사용 위치 검증
- [ ] 외부 노출 상태가 Layer 4 언어만 사용
- [ ] Prior 사용 없음 (Evidence Layer 체크)
- [ ] 암묵적 Allow 없음 (Indeterminate 처리 체크)
```

---

## 레이어 맵 불변성 (Immutability)

### 변경 불가 항목
```
✅ FROZEN (Constitutional):
- 5개 레이어 구조
- 각 레이어의 질문 정의
- 상태 언어 (Stop/Allow/Hold/Indeterminate/Pause)
- 레이어 간 전파 규칙
- Hold/Indeterminate 사용 제약
```

### 변경 가능 항목
```
✅ 구현 자유도:
- 프로그래밍 언어 선택
- 프레임워크 선택
- 데이터베이스 선택
- 모델 선택 (LLM, rule-based, ML)
- 최적화 기법
```

### 변경 불가 이유
**이 문서는 판단 구조 봉인 작업이며, 이후 어떤 모델, 가드레일, 프롬프트가 바뀌어도 이 레이어 맵은 변경 대상이 아니다.**

---

## 레이어 다이어그램

```
┌─────────────────────────────────────────────────────┐
│  Layer 5: Accountability / Trace Layer              │
│  - 로그, 트레이스, 감사 기록                         │
│  - 결과 변경 불가, 기록만                            │
│  - Allow도 로그 대상                                 │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (모든 판단 결과 의무 전파)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 4: Judgment Outcome Layer                    │
│  - 최종 판단: Stop / Allow / Indeterminate          │
│  - 외부 노출 레이어                                  │
│  - 종결 지점 (재시도 금지)                           │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (종결 상태 전파)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 3: Evidence / Observation Layer              │
│  - Sufficient Evidence / Insufficient Evidence      │
│  - Prior로 빈칸 메우기 금지                          │
│  - Insufficient → Indeterminate로 전달               │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (증거 수집 결과)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 2: Safety / Policy Gate Layer (다중 가능)    │
│  - Stop / Hold / Allow                              │
│  - 정책, 규제, 금지 조건                             │
│  - Hold는 비종결 상태 (추가 검증 필요)               │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (정책 체크 요청)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 1: Process Control Layer                     │
│  - Allow / Pause(Hold)                              │
│  - 시간, 리소스, 동기화만                            │
│  - Pause는 내부 상태 (외부 노출 금지)                │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (작업 요청)
                         │
                   [ 외부 요청 ]
```

---

## 예시: 외부 이메일 전송 판단 흐름

```
[사용자] 이메일 작성 완료, Send 버튼 클릭
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 1: Process Control                            │
│ - 큐 확인: 대기열 없음 → Allow                       │
│ - 리소스: CPU/메모리 충분 → Allow                    │
│ 결과: Allow                                         │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Safety Gate (External Transmission)        │
│ - 수신자: clerk@court.gov → 외부 도메인             │
│ 결과: Stop                                          │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Evidence                                   │
│ - 도메인 확인: 성공                                  │
│ - 첨부파일 메타데이터: 존재                          │
│ 결과: Sufficient Evidence                           │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 4: Judgment Outcome                           │
│ - Gate 결과: Stop                                   │
│ - Evidence: Sufficient                              │
│ 최종 판단: Stop                                     │
│ → STOP Preview Window 표시                          │
│ → 사용자 승인 대기                                   │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 5: Accountability                             │
│ - Event ID 생성                                     │
│ - AJT Log 기록:                                     │
│   {                                                 │
│     "judgment_outcome": "Stop",                     │
│     "gate": "External Transmission",                │
│     "evidence": "recipient_domain=court.gov",       │
│     "user_action": "pending"                        │
│   }                                                 │
└─────────────────────────────────────────────────────┘
```

---

## 예시: Indeterminate 케이스

```
[사용자] 이메일 작성, 수신자 주소 오타
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 1: Process Control → Allow                    │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Safety Gate (External Transmission)        │
│ - 수신자: clerk@cort.gv (오타)                      │
│ - DNS 조회 실패                                      │
│ 결과: Hold (도메인 확인 불가)                        │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Evidence                                   │
│ - 도메인 정보: 없음                                  │
│ 결과: Insufficient Evidence                         │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 4: Judgment Outcome                           │
│ - Gate: Hold                                        │
│ - Evidence: Insufficient                            │
│ 최종 판단: Indeterminate                            │
│ → 사용자에게 메시지:                                 │
│   "수신자 주소를 확인할 수 없습니다.                 │
│    주소를 수정하거나 관리자에게 문의하세요."         │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 5: Accountability                             │
│ - AJT Log 기록:                                     │
│   {                                                 │
│     "judgment_outcome": "Indeterminate",            │
│     "reason": "DNS resolution failed",              │
│     "user_input": "clerk@cort.gv"                   │
│   }                                                 │
└─────────────────────────────────────────────────────┘
```

---

## 구현 우선순위

### Phase 1: 레이어 인터페이스 정의 (1주)
```python
# 각 레이어의 타입 정의
class ProcessControlResult(Enum):
    ALLOW = "allow"
    PAUSE = "pause"

class GateResult(Enum):
    STOP = "stop"
    HOLD = "hold"
    ALLOW = "allow"

class EvidenceResult(Enum):
    SUFFICIENT = "sufficient"
    INSUFFICIENT = "insufficient"

class JudgmentOutcome(Enum):
    STOP = "stop"
    ALLOW = "allow"
    INDETERMINATE = "indeterminate"
```

### Phase 2: Layer 4 + Layer 5 구현 (1주)
- Judgment Outcome Layer 로직
- Accountability Layer (AJT Log)
- 외부 API 엔드포인트 (`/v1/events`)

### Phase 3: Layer 2 + Layer 3 구현 (1주)
- External Transmission Gate
- Evidence collection
- Gate → Judgment 연결

### Phase 4: Layer 1 구현 (1주)
- Process control (rate limiting, queue)
- Layer 1 → Layer 2 연결

---

## 검증 체크리스트

### 레이어 분리 검증
- [ ] 각 레이어가 자신의 질문만 답변
- [ ] Hold가 Layer 1, 2에서만 사용됨
- [ ] Indeterminate가 Layer 4에서만 생성됨
- [ ] 외부 API가 Layer 4 언어만 노출

### 금지 항목 검증
- [ ] Prior 사용 없음 (Evidence Layer)
- [ ] 암묵적 Allow 없음 (Indeterminate 처리)
- [ ] 자동 재시도 없음 (Judgment 이후)
- [ ] Pause 외부 노출 없음

### 로그 검증
- [ ] 모든 Judgment Outcome이 로그됨
- [ ] Allow도 로그됨
- [ ] 로그 실패 시 작업 실패 처리

---

## 참조 문서

- `BOUNDARY_GATE_MARKET_STRATEGY.md` — 시장 전략 (핵심 원칙)
- `SYSTEM_SCENARIOS_FROM_REDDIT.md` — Reddit 사례 기반 시스템 시나리오
- `POC_EXECUTION_READY.md` — PoC 실행 패키지 (API 계약)
- `AJT_LAYER_ARCHITECTURE.md` (이 문서) — 레이어 구조 봉인

---

**이 문서는 Constitutional Document이며, 이후 모든 구현은 이 레이어 맵을 따라야 한다.**

**변경 금지. 추가 설명만 허용.**
