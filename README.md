# Cognitive Infrastructure Constitution

**This system does not generate decisions.
It maintains the conditions under which humans can decide.**

---

## What This Document Is

This is not a product specification.
This is not a technical manual.
This is not a research paper.

This is a **constitutional declaration**.

A set of irrevocable rules defining what *Cognitive Infrastructure* is —
and what it can **never** become.

---

## The Problem

AI systems in high-stakes domains (medical, legal, financial) encounter a recurring failure mode:

The more useful the AI becomes, the less clear **who decided**.

AI generates recommendations → Humans approve → Responsibility blurs
Audit logs exist → Accountability does not
Organizations respond by weakening AI — or abandoning it entirely

This is not a model problem.
It is a **structural** problem.

---

## The Solution (Structural, Not Algorithmic)

We do not make AI smarter.
We fix **where responsibility lives**.

**Core principle:**

> **Decisions are prohibited.
> Readiness is enforced.**

This system:

❌ Does **not** generate decisions
❌ Does **not** recommend final answers
❌ Does **not** optimize for choices

✅ Prepares information, context, and options
✅ Preserves human judgment as the only deciding authority
✅ Records **who decided**, not *what the system preferred*

---

## Invariant Rules (Irrevocable)

### 1. Decision Generation Prohibition

**Rule:** The system MUST NOT generate decision objects.

**Permitted outputs:**

* Normalized input
* Context frames
* Option sets
* Risk signals
* Readiness states

**Prohibited outputs:**

* Decisions
* Final answers
* Optimal choices
* Judgment-bearing recommendations

```python
def process(input):
    analysis = analyze(input)
    options = generate_options(analysis)

    # System stops here
    # decision = select_best(options)  # PROHIBITED

    return {
        "type": "OptionSet",
        "options": options,
        "unresolved": True
    }
```

---

### 2. STOP as the Default State

**Rule:** All workflows begin in a STOPPED state.

**Flow:**

```
INIT → STOPPED → (Human Signal) → READY → EXECUTE
```

**Prohibited:**

* Auto-transition from STOPPED
* Timeout-based execution
* Default-to-execute behavior

```python
state = STOPPED  # Mandatory default

if human_readiness_signal_received():
    state = READY
```

---

### 3. Trace as a Precondition

**Rule:** Execution cannot proceed without trace.

**Minimum trace fields:**

* `actor` (who judged)
* `context_snapshot`
* `option_set_hash`
* `timestamp`

```python
def execute(action):
    if not trace_slot_exists():
        abort("Trace required before execution")

    record_trace(actor, context, timestamp)
    proceed(action)
```

---

### 4. Responsibility Anchoring

**Rule:** Every judgment must have a named human owner.

**Required:**

* `decision_maker`: Named individual (never "system" or "AI")
* `decision_time`: Explicit timestamp
* `human_judgment`: Recorded independently of AI analysis

```json
{
  "decision_maker": "Dr. Sarah Chen",
  "decision_time": "2025-12-22T10:30:00Z",
  "human_judgment": "APPROVE",
  "ai_analysis": {...},
  "ai_recommendation": null
}
```

---

### 5. Forbidden Vocabulary

The following terms are prohibited in code, prompts, and documentation:

❌ `decision` (as system output)
❌ `final answer`
❌ `optimal choice`
❌ `best option`
❌ `auto-decide`

**Permitted replacements:**

✅ `option set`
✅ `unresolved state`
✅ `readiness`
✅ `prepared choices`

---

## Failure Conditions

The system is considered **FAILED**, regardless of performance, if:

* A decision object is generated automatically
* Execution proceeds without trace
* STOP is bypassed
* Forbidden vocabulary appears in production
* `decision_maker` is "AI" or "System"

**Priority order:**
Structural integrity → Responsibility → Speed → Accuracy

---

## What This Enables

**Before (Decision Automation):**
AI judges → Human approves → Responsibility unclear

**After (Cognitive Infrastructure):**
AI analyzes → System stops → Human judges → Responsibility fixed

**Measurable outcomes:**

* Audit logs prove **who decided**
* Liability attaches to a named individual
* Human judgment cannot be bypassed

---

## What This Is Not

❌ A bias mitigation framework
❌ An explainability tool
❌ A compliance checklist
❌ An AI safety mechanism

This is a **structural pattern** that fixes where responsibility lives.

---

## Governance

**Amendment process:** None.

This constitution may only be:

* Enforced
* Violated (system failure)
* Abandoned (different system)

Removing these constraints changes the system category.

---

## One Question

> **In a lawsuit, can you prove who made the decision?**

| Without This Constitution | With This Constitution |
| ------------------------- | ---------------------- |
| Responsibility unclear    | Responsibility fixed   |

---

## Additional Resources

**Explanatory notes (non-normative):** [ajt-explanatory-docs](https://github.com/Nick-heo-eg/ajt-explanatory-docs)

---

**Reference implementation:** [judgment-externalization](https://github.com/Nick-heo-eg/judgment-externalization)
**Status:** LOCKED
**Effective:** 2025-12-22
**Permanence:** Irrevocable

**Decisions are prohibited.
Readiness is enforced.**
