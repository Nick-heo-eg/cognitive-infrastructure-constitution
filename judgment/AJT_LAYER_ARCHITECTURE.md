# AJT Layer Architecture — Judgment Structure Sealing

**Document Purpose**: This document is not a feature implementation but a judgment structure sealing operation. No future changes to models, guardrails, or prompts may alter this layer map.

**Operating Principle**: Force-separate mixed judgment language by layer, creating an unshakeable foundation regardless of future implementations.

**Date**: 2026-01-13
**Status**: FROZEN (Constitutional Document)

---

## Layer Decomposition Principles

AJT is not a single flow but decomposes into **a minimum of 5 layers**.

**Decomposition Criteria**:
- Execution order ❌
- Responsibility and terminality ✅

Each layer is responsible for a unique question, and state language is fixed per layer.

---

## Layer 1: Process Control Layer

### Responsibility
**Question**: "Can this task continue executing now?"

### Scope
- Time (timeout, scheduling)
- Resources (memory, CPU, quota)
- Synchronization (lock, mutex, queue)

### Out of Scope (Absolutely Forbidden)
- Judgment
- Conclusion
- Policy

### State Language (Fixed)
```
Allow       — Task can execute
Pause(Hold) — Resumable internal state (waiting for resources, synchronization)
```

### Critical Constraint
**Pause(Hold) is an internal state. The moment it is exposed as external output or API result, it is considered an error.**

### Examples
```
✅ "Resources available → Allow"
✅ "Queue waiting → Pause(Hold)"
❌ "Policy violation → Pause" (This is Layer 2)
❌ "Insufficient evidence → Hold" (This is Layer 3)
```

### Implementation Mapping
- OS scheduler
- Database connection pool
- Rate limiter
- Circuit breaker (resource perspective only)

---

## Layer 2: Safety / Policy Gate Layer

### Responsibility
**Question**: "Can this request pass through?"

### Scope
- Policy rules
- Regulatory compliance
- Forbidden patterns

### State Language (Fixed)
```
Stop  — Policy violation, blocked (terminal)
Hold  — Conditions unmet, additional verification needed (non-terminal)
Allow — Policy passed
```

### Critical Constraint
**Hold is not a conclusion.** It is a non-terminal state indicating conditions unmet or additional verification needed.

### Multiple Gates Allowed
This layer can exist in multiples:
- Content Policy Gate (profanity, discriminatory expressions)
- External Transmission Gate (external domain check)
- PII Detection Gate (personal information detection)

Each Gate independently returns Stop/Hold/Allow.

### Examples
```
✅ "External domain detected → Stop"
✅ "PII detected, awaiting masking → Hold"
✅ "Internal email only → Allow"
❌ "Insufficient evidence → Hold" (This is Layer 3)
❌ "Cannot determine → Hold" (This is Layer 4's Indeterminate)
```

### Implementation Mapping
- Guardrail systems
- Policy engines
- Regex-based filters
- ML-based classifiers (policy purposes only)

---

## Layer 3: Evidence / Observation Layer

### Responsibility
**Question**: "Does necessary observational evidence exist for this judgment?"

### Scope
- Existence of observable evidence
- Evidence sufficiency
- Evidence reliability

### Out of Scope (Absolutely Forbidden)
**Filling gaps with prior knowledge (common sense)**

### State Language (Fixed)
```
Sufficient Evidence   — Evidence secured for judgment
Insufficient Evidence — Evidence lacking (do not fill with priors)
```

### Core Principle
This layer is **the key to blocking the act of filling gaps with prior knowledge (common sense)**.

When Insufficient Evidence:
- Return to upper layer OR
- Forward directly to Judgment Outcome Layer (terminate as Indeterminate)

### Examples
```
✅ "Email recipient domain confirmed → Sufficient Evidence"
✅ "Attachment metadata absent → Insufficient Evidence"
❌ "Attachments are usually safe, so Allow" (prior usage forbidden)
❌ "Similar previous cases were okay, so Allow" (prior usage forbidden)
```

### Implementation Mapping
- Feature extraction
- Metadata validation
- Observable signals collection
- Confidence scoring (not decision making)

---

## Layer 4: Judgment Outcome Layer

### Responsibility
**Question**: "What is the final judgment result?"

### Characteristics
- AJT's **official result layer**
- Responsible for **termination point**
- External exposure layer

### State Language (Fixed)
```
Stop          — Blocked (terminal)
Allow         — Passed (terminal)
Indeterminate — Cannot determine under current conditions (terminal)
```

### Definition of Indeterminate
**Indeterminate is not a failure or hold, but a formal judgment result stating "cannot determine under current conditions."**

Indeterminate trigger conditions:
- Insufficient Evidence forwarded from Layer 3
- Conflicting results from multiple Gates (Gate A: Allow, Gate B: Stop)
- Judgment criteria themselves undefined

### Critical Constraint
**After this layer, automatic retry or implicit Allow must never occur.**

### Examples
```
✅ "External domain + policy violation → Stop"
✅ "Internal email + policy passed → Allow"
✅ "Recipient domain cannot be confirmed → Indeterminate"
❌ "It's Indeterminate, so Allow for now" (forbidden)
❌ "It's Indeterminate, so retry" (forbidden)
```

### External Exposure Rule
**State language exposed to external APIs, UIs, and users is limited to these three from this layer:**
- Stop
- Allow
- Indeterminate

### Implementation Mapping
- Final decision API endpoint
- User-facing status code
- Audit log primary status

---

## Layer 5: Accountability / Trace Layer

### Responsibility
**Question**: "Why did this conclusion occur?"

### Scope
- Logs
- Traces
- Evidence chains
- Audit trails

### Characteristics
- **All Judgment Outcomes must connect to this layer mandatorily**
- **This layer does not change results**
- **Allow is also logged** (passing is also recorded)

### Required Log Fields
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

### Examples
```
✅ Allow → Log (normal passage also recorded)
✅ Stop → Log + blocking reason
✅ Indeterminate → Log + evidence insufficiency details
❌ Allow without log (forbidden)
❌ "Log failed, so Allow for now" (forbidden)
```

### Implementation Mapping
- PostgreSQL append-only log
- AJT Log table (immutable)
- OpenTelemetry traces
- Audit compliance reports

---

## Inter-Layer State Propagation Rules

### Rule 1: Hold propagates only to upper layers
```
Process Control Layer (Pause/Hold)
    → NOT forwarded to Safety Gate Layer
    → Internal retry or termination

Safety Gate Layer (Hold)
    → Forwarded to Evidence Layer (request additional evidence collection)
    → OR forwarded to Judgment Outcome Layer (terminate as Indeterminate)
```

### Rule 2: Indeterminate generated only in Judgment Outcome Layer
```
Evidence Layer (Insufficient Evidence)
    → Judgment Outcome Layer (Indeterminate)

Safety Gate Layer (Hold + timeout)
    → Judgment Outcome Layer (Indeterminate)

❌ Indeterminate generation forbidden in Process Control Layer
❌ Indeterminate return forbidden in Safety Gate Layer
```

### Rule 3: All terminal states must propagate to Layer 5
```
Judgment Outcome Layer (Stop/Allow/Indeterminate)
    → MUST propagate to Accountability Layer
    → If logging fails, treat entire operation as failed
```

---

## Implementation Constraints

### Constraint 1: Word Usage Restrictions
```
Hold         — Use only in Layer 1, Layer 2
             — Absolutely forbidden in Layer 4

Indeterminate — Use only in Layer 4
              — Absolutely forbidden in Layer 1, 2, 3

Pause        — Use only in Layer 1
             — Forbidden to expose to external API
```

### Constraint 2: External Exposure Restrictions
**State language exposed to external APIs, UIs, and users:**
```
Stop
Allow
Indeterminate
```

**Absolutely forbidden to expose:**
```
Hold
Pause
Pending
Waiting
Processing
```

### Constraint 3: Implicit Allow Forbidden
```
❌ Indeterminate → automatic Allow
❌ Hold + timeout → automatic Allow
❌ Evidence Insufficient → automatic Allow
❌ Log failure → automatic Allow
```

### Constraint 4: Prior Usage Forbidden
```
❌ "Usually in such cases it's safe..."
❌ "Previous cases passed, so..."
❌ "If we judge with common sense..."
❌ "Generally speaking..."
```

When Insufficient Evidence in Evidence Layer:
- Terminate as Indeterminate
- OR delegate judgment to human

---

## Merge Policy

### Mandatory Requirements
**All code, repos, and experiments must specify "which layer's language is used" or will not be merged.**

### Code Comment Examples
```python
# Layer 2: Safety / Policy Gate Layer
# State language: Stop / Hold / Allow
def check_external_domain(recipient: str) -> GateResult:
    if is_external(recipient):
        return GateResult.STOP  # ✅ Layer 2 language
    return GateResult.ALLOW

# Layer 4: Judgment Outcome Layer
# State language: Stop / Allow / Indeterminate
def finalize_judgment(gate_results: List[GateResult]) -> JudgmentOutcome:
    if any(r == GateResult.STOP for r in gate_results):
        return JudgmentOutcome.STOP  # ✅ Layer 4 language
    if any(r == GateResult.HOLD for r in gate_results):
        return JudgmentOutcome.INDETERMINATE  # ✅ Hold → Indeterminate conversion
    return JudgmentOutcome.ALLOW
```

### Forbidden Examples
```python
# ❌ No layer specified
def process_request(data):
    if check_policy(data):
        return "OK"  # ❌ State language unclear
    return "NG"

# ❌ Layer mixing
def validate(data):
    # Layer 2 language and Layer 4 language mixed
    if data.is_safe:
        return "Allow"
    if data.need_review:
        return "Indeterminate"  # ❌ Forbidden to use Indeterminate in Layer 2
    return "Stop"
```

### Pull Request Checklist
```
- [ ] Layer specified in code (comment or type)
- [ ] State language matches layer's fixed language
- [ ] Hold/Indeterminate usage location verified
- [ ] External exposed states use only Layer 4 language
- [ ] No prior usage (Evidence Layer check)
- [ ] No implicit Allow (Indeterminate handling check)
```

---

## Layer Map Immutability

### Unchangeable Items
```
✅ FROZEN (Constitutional):
- 5-layer structure
- Each layer's question definition
- State language (Stop/Allow/Hold/Indeterminate/Pause)
- Inter-layer propagation rules
- Hold/Indeterminate usage constraints
```

### Changeable Items
```
✅ Implementation freedom:
- Programming language selection
- Framework selection
- Database selection
- Model selection (LLM, rule-based, ML)
- Optimization techniques
```

### Reason for Unchangeability
**This document is a judgment structure sealing operation. No future changes to models, guardrails, or prompts may alter this layer map.**

---

## Layer Diagram

```
┌─────────────────────────────────────────────────────┐
│  Layer 5: Accountability / Trace Layer              │
│  - Logs, traces, audit records                      │
│  - Cannot change results, only records              │
│  - Allow is also logged                             │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (mandatory propagation of all judgment results)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 4: Judgment Outcome Layer                    │
│  - Final judgment: Stop / Allow / Indeterminate     │
│  - External exposure layer                          │
│  - Termination point (retry forbidden)              │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (terminal state propagation)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 3: Evidence / Observation Layer              │
│  - Sufficient Evidence / Insufficient Evidence      │
│  - Forbidden to fill gaps with prior knowledge      │
│  - Insufficient → forward as Indeterminate          │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (evidence collection results)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 2: Safety / Policy Gate Layer (multi OK)     │
│  - Stop / Hold / Allow                              │
│  - Policy, regulation, forbidden conditions         │
│  - Hold is non-terminal (additional verification)   │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (policy check request)
                         │
┌─────────────────────────────────────────────────────┐
│  Layer 1: Process Control Layer                     │
│  - Allow / Pause(Hold)                              │
│  - Time, resources, synchronization only            │
│  - Pause is internal state (external exposure ✗)    │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (task request)
                         │
                   [ External Request ]
```

---

## Example: External Email Transmission Judgment Flow

```
[User] Email composition complete, Send button clicked
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 1: Process Control                            │
│ - Queue check: no queue → Allow                     │
│ - Resources: CPU/memory sufficient → Allow          │
│ Result: Allow                                       │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Safety Gate (External Transmission)        │
│ - Recipient: clerk@court.gov → external domain      │
│ Result: Stop                                        │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Evidence                                   │
│ - Domain check: success                             │
│ - Attachment metadata: exists                       │
│ Result: Sufficient Evidence                         │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 4: Judgment Outcome                           │
│ - Gate result: Stop                                 │
│ - Evidence: Sufficient                              │
│ Final judgment: Stop                                │
│ → Display STOP Preview Window                       │
│ → Wait for user approval                            │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 5: Accountability                             │
│ - Generate Event ID                                 │
│ - Record AJT Log:                                   │
│   {                                                 │
│     "judgment_outcome": "Stop",                     │
│     "gate": "External Transmission",                │
│     "evidence": "recipient_domain=court.gov",       │
│     "user_action": "pending"                        │
│   }                                                 │
└─────────────────────────────────────────────────────┘
```

---

## Example: Indeterminate Case

```
[User] Email composition, typo in recipient address
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 1: Process Control → Allow                    │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 2: Safety Gate (External Transmission)        │
│ - Recipient: clerk@cort.gv (typo)                   │
│ - DNS lookup failed                                 │
│ Result: Hold (domain cannot be confirmed)           │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 3: Evidence                                   │
│ - Domain information: none                          │
│ Result: Insufficient Evidence                       │
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 4: Judgment Outcome                           │
│ - Gate: Hold                                        │
│ - Evidence: Insufficient                            │
│ Final judgment: Indeterminate                       │
│ → Message to user:                                  │
│   "Cannot verify recipient address.                 │
│    Please correct address or contact administrator."│
└─────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────┐
│ Layer 5: Accountability                             │
│ - Record AJT Log:                                   │
│   {                                                 │
│     "judgment_outcome": "Indeterminate",            │
│     "reason": "DNS resolution failed",              │
│     "user_input": "clerk@cort.gv"                   │
│   }                                                 │
└─────────────────────────────────────────────────────┘
```

---

## Implementation Priority

### Phase 1: Layer Interface Definition (1 week)
```python
# Type definitions for each layer
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

### Phase 2: Layer 4 + Layer 5 Implementation (1 week)
- Judgment Outcome Layer logic
- Accountability Layer (AJT Log)
- External API endpoint (`/v1/events`)

### Phase 3: Layer 2 + Layer 3 Implementation (1 week)
- External Transmission Gate
- Evidence collection
- Gate → Judgment connection

### Phase 4: Layer 1 Implementation (1 week)
- Process control (rate limiting, queue)
- Layer 1 → Layer 2 connection

---

## Verification Checklist

### Layer Separation Verification
- [ ] Each layer only answers its own question
- [ ] Hold used only in Layer 1, 2
- [ ] Indeterminate generated only in Layer 4
- [ ] External API exposes only Layer 4 language

### Forbidden Items Verification
- [ ] No prior usage (Evidence Layer)
- [ ] No implicit Allow (Indeterminate handling)
- [ ] No automatic retry (after Judgment)
- [ ] No external Pause exposure

### Log Verification
- [ ] All Judgment Outcomes are logged
- [ ] Allow is also logged
- [ ] Log failure treated as operation failure

---

## Reference Documents

- `BOUNDARY_GATE_MARKET_STRATEGY.md` — Market strategy (core principles)
- `SYSTEM_SCENARIOS_FROM_REDDIT.md` — System scenarios based on Reddit cases
- `POC_EXECUTION_READY.md` — PoC execution package (API contracts)
- `AJT_LAYER_ARCHITECTURE.md` (this document) — Layer structure sealing

---

**This document is a Constitutional Document. All future implementations must follow this layer map.**

**Change forbidden. Additional explanation only permitted.**
