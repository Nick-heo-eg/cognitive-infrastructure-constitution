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

**Core Principle**:
**Each layer answers exactly one question. No layer may answer another layer's question.**

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
Allow                   — Task can execute
Pause (Execution Hold)  — Resumable internal state (waiting for resources, synchronization)
```

**Note**: Pause is a technical execution state unrelated to judgment, policy, or conclusion.

### Critical Constraint
**Pause (Execution Hold) is an internal state. The moment it is exposed as external output or API result, it is considered an error.**

### Examples
```
✅ "Resources available → Allow"
✅ "Queue waiting → Pause (Execution Hold)"
❌ "Policy violation → Pause" (This is Layer 2)
❌ "Insufficient evidence → Hold" (This is Layer 3)
```

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
Stop                        — Policy violation, blocked (terminal)
Hold (Policy Non-Terminal)  — Conditions unmet, additional verification needed (non-terminal)
Allow                       — Policy passed
```

**Note**: Hold is a policy condition state, not a judgment result. It indicates policy conditions are unmet, not that judgment cannot be made.

### Critical Constraint
**Hold (Policy Non-Terminal) is not a conclusion.** It is a non-terminal state indicating conditions unmet or additional verification needed.

### Multiple Gates Allowed
This layer can exist in multiples:
- Content Policy Gate (profanity, discriminatory expressions)
- External Transmission Gate (external domain check)
- PII Detection Gate (personal information detection)

Each Gate independently returns Stop / Hold (Policy Non-Terminal) / Allow.

### Examples
```
✅ "External domain detected → Stop"
✅ "PII detected, awaiting masking → Hold"
✅ "Internal email only → Allow"
❌ "Insufficient evidence → Hold" (This is Layer 3)
❌ "Cannot determine → Hold" (This is Layer 4's Indeterminate)
```

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

**This layer never produces a judgment outcome.** It only assesses evidence sufficiency. Judgment outcomes (Stop/Allow/Indeterminate) are generated exclusively in Layer 4.

When Insufficient Evidence:
- Return to upper layer OR
- Forward directly to Judgment Outcome Layer (Layer 4 will terminate as Indeterminate)

### Examples
```
✅ "Domain confirmed → Sufficient Evidence"
✅ "Metadata absent → Insufficient Evidence"
❌ "Usually safe, so Allow" (prior usage forbidden)
❌ "Previous cases okay, so Allow" (prior usage forbidden)
```

---

## Layer 4: Judgment Outcome Layer

### Responsibility
**Question**: "What is the final judgment result?"

### Characteristics
- AJT's **official result layer**
- Responsible for **termination point**
- External exposure layer

**This is the only layer allowed to emit Stop / Allow / Indeterminate.** No other layer may produce these judgment outcomes.

### State Language (Fixed)
```
Stop          — Blocked (terminal)
Allow         — Passed (terminal)
Indeterminate — Cannot determine under current conditions (terminal)
```

### Definition of Indeterminate
**Indeterminate is not a failure or hold, but a formal judgment result stating "cannot determine under current conditions."**

**Indeterminate is a terminal judgment outcome, not a hold, not a failure, and not an escalation trigger.**

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
✅ "Domain cannot be confirmed → Indeterminate"
❌ "It's Indeterminate, so Allow for now" (forbidden)
❌ "It's Indeterminate, so retry" (forbidden)
```

### External Exposure Rule
**State language exposed to external APIs, UIs, and users is limited to these three from this layer:**
- Stop
- Allow
- Indeterminate

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

**This layer cannot alter, override, or reinterpret outcomes from Layer 4.** It only records what occurred.

### Required Log Fields
```json
{
  "event_id": "evt_...",
  "judgment_outcome": "Stop | Allow | Indeterminate",
  "timestamp": "ISO8601",
  "layers_executed": [
    {"layer": "Process Control Layer", "result": "Allow"},
    {"layer": "Safety Gate Layer", "result": "Stop"},
    {"layer": "Evidence Layer", "result": "Sufficient Evidence"},
    {"layer": "Judgment Outcome Layer", "result": "Stop"}
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

---

## Inter-Layer State Propagation Rules

### Rule 1: Hold propagates only to upper layers
```
Process Control Layer (Pause / Execution Hold)
    → NOT forwarded to Safety Gate Layer
    → Internal retry or termination

Safety Gate Layer (Hold / Policy Non-Terminal)
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
│  - Allow / Pause (Execution Hold)                   │
│  - Time, resources, synchronization only            │
│  - Pause is internal state (external exposure ✗)    │
└─────────────────────────────────────────────────────┘
                         ▲
                         │ (task request)
                         │
                   [ External Request ]
```

---

## Illustrative Example (Non-Normative)

The following simplified flow demonstrates layer interaction. This example is illustrative only and does not define or constrain the constitutional rules.

```
[Request] → Layer 1 (Allow) → Layer 2 (Stop: external domain)
         → Layer 3 (Sufficient Evidence) → Layer 4 (Stop) → Layer 5 (Log)
```

**For detailed case studies**, see [judgment-experiment-logs/cases](https://github.com/Nick-heo-eg/judgment-experiment-logs/tree/main/cases)

---

## Non-Normative Reference Materials

- `BOUNDARY_GATE_MARKET_STRATEGY.md` — Market strategy (core principles)
- `SYSTEM_SCENARIOS_FROM_REDDIT.md` — System scenarios based on cases
- `POC_EXECUTION_READY.md` — PoC execution package (API contracts)
- `AJT_LAYER_ARCHITECTURE.md` (this document) — Layer structure sealing

**Note**: These documents are provided for empirical and contextual background only. They do not define, modify, or constrain the constitutional rules specified in this document.

---

**This document is a Constitutional Document. All future implementations must follow this layer map.**

**Change forbidden. Additional explanation only permitted.**
