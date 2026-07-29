# 003 Business Decision Framework — Implementation Architectural Blueprint

## 1. Construction Overview

This blueprint translates the approved Business Decision Framework into buildable construction guidance. It defines the pure-logic capability gate that evaluates customer requests against a validated business capability catalog, returning either a supported match, a bounded clarifying question, or a structured unsupported outcome.

## 2. Build Scope

| Included | Deferred | Forbidden |
|---|---|---|
| Capability evaluation gateway | AI-based intent classification | Scheduling or booking actions |
| Two-pass matching engine | Configurable per-service matching weights | Persistence writes |
| Confidence tier assignment | Fuzzy matching for typos | Service area validation |
| Clarification budget enforcement | Multi-channel input adapters | Emergency qualification |
| Decline context compiler | ML model training | Template rendering |
| Structured decision contract | | Customer data collection |
| Correlation markers for observability | | Hardcoded business behavior |

## 3. Recommended Project Structure

```text
business-decision-framework/
│
├── Contracts
│   ├── Capability Decision Contract
│   ├── Capability Match Descriptor Contract
│   ├── Clarifying Question Contract
│   └── Decline Context Contract
│
├── Evaluation
│   ├── Intent Normalizer
│   ├── Double-Pass Matrix Matcher
│   ├── Confidence Arbiter
│   └── Clarification Budget Tracker
│
├── Outcomes
│   ├── Supported Outcome Compiler
│   └── Decline Envelope Compiler
│
└── Tests
    ├── Contract Tests
    ├── Matching Tests
    ├── Clarification Tests
    ├── Decline Tests
    └── Determinism Tests
```

## 4. Public Contracts

### Input Contract
The framework consumes:
- The caller's service request description
- Optional priority context markers
- The validated business capability catalog
- The operational scope profile
- Optional transaction correlation key

### Output Contract: Capability Decision

A successful outcome carries:
- Supported flag
- Capability match descriptor, including matched service identity, confidence tier, and match reasoning

A clarifying outcome carries:
- Clarifying question text
- Question identifier
- Remaining question budget

An unsupported outcome carries:
- Unsupported reason
- Original request summary
- Tag-based category summary
- Operational scope display name
- Count of clarifying questions already asked

### Catalog Consumption Contract
The framework reads an already-validated capability catalog. It does not fetch, validate, or modify the catalog.

## 5. Internal Components

| Component | Purpose | Forbidden |
|---|---|---|
| Intent Normalizer | Prepares caller text for matching by standardizing case and spacing | Interpreting meaning; running classification models |
| Double-Pass Matrix Matcher | Scans the capability catalog and assigns confidence tiers | Executing downstream workflows; modifying catalog entries |
| Confidence Arbiter | Maps match scores to confidence tiers and selects the next action | Making business decisions beyond supported or unsupported |
| Clarification Budget Tracker | Tracks questions asked and enforces the maximum limit | Generating final customer messages; storing persistent state |
| Supported Outcome Compiler | Builds the supported match descriptor | Rendering messages; triggering actions |
| Decline Envelope Compiler | Builds the structured unsupported context | Hardcoding decline scripts; invoking communication drivers |

## 6. Runtime Construction Flow

```text
Caller Request
        |
        v
Intent Normalizer
        |
        v
Pass 1 Matrix Matcher
        |
        v
Confidence Check
        |
   +----+----+
   |         |
  High/    Low or None
Medium
   |         |
   v         v
Supported  Clarifying Question
Outcome       |
              v
           Caller Response
              |
              v
         Pass 2 Matcher
              |
              v
       Confidence Check
              |
        +-----+-----+
        |           |
   High/Medium   Budget Exhausted
        |           |
        v           v
   Supported    Unsupported
    Outcome       Outcome
```

## 7. Dependency Map

```text
Downstream Workflow Coordinators
        ^
        |
Capability Decision Gateway
        |
+-------+-------+-------+
|       |       |       |
v       v       v       v
Intent  Double-Pass  Clarification  Outcome
Normalizer  Matcher    Budget Tracker  Compilers
        |
        v
Validated Business Capability Catalog
        |
        v
Configuration Management Layer
```

## 8. Error Handling Strategy

| Classification | Source | Handling |
|---|---|---|
| Unsupported input | Empty description, empty catalog, missing required fields | Terminal; return unsupported outcome |
| Ambiguous intent | Low-confidence match | Recoverable through clarifying question until budget is exhausted |
| Exhausted clarification | Ambiguity persists after maximum questions | Terminal; return unsupported outcome |
| Integration misuse | Consumer violates output contract | Prevented through contract validation and documentation |
| Upstream failure | Invalid or unavailable catalog | Terminal; treat as unsupported to protect downstream boundaries |

Retry — Owned downstream.

Propagation — The boundary returns structured decision outcomes. It never exposes internal catalog identifiers, raw scores, or matching internals.

## 9. Testing Blueprint

- Contract Tests — valid and invalid decision shapes
- Matching Tests — exact, strong, weak, and no-match scenarios
- Clarification Tests — budget enforcement and resolution paths
- Decline Tests — category summarization and unsupported outcomes
- Determinism Tests — identical inputs always produce identical decisions
- Regression Tests — all match strategies and confidence tiers covered

## 10. Construction Checklist

✓ Capability Decision Contract defined  
✓ Capability Match Descriptor Contract defined  
✓ Clarifying Question Contract defined  
✓ Decline Context Contract defined  
✓ Intent Normalizer standardizes input text without interpreting meaning  
✓ Double-Pass Matrix Matcher evaluates catalog across two passes  
✓ Confidence Arbiter maps scores to high, medium, low, or none  
✓ Clarification Budget Tracker enforces the maximum question limit  
✓ Supported Outcome Compiler builds match descriptors only  
✓ Decline Envelope Compiler builds structured unsupported contexts only  
✓ No scheduling, booking, intake creation, or notification dispatch introduced  
✓ No service area validation or emergency qualification introduced  
✓ No template rendering or user-facing string generation introduced  
✓ No persistence writes or audit events introduced  
✓ Inputs treated as read-only within the framework  
✓ Deterministic matching verified through regression tests  
✓ Architectural ownership preserved: the framework evaluates, downstream modules act  

---

# End of Blueprint

**Business Decision Framework**

**Derived from:** 003_Business_Decision_Framework.md and ServiceCapabilityDecision.PRD

**Architectural Role:**

**Decision Boundary / Pure-Logic Capability Matcher**