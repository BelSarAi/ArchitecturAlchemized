Updated todo list

Here is the **005 Emergency Client-Contact Workflow — Implementation Architectural Blueprint**.

---

# 005 Emergency Client-Contact Workflow — Implementation Architectural Blueprint

## 1. Construction Overview

This blueprint translates the approved Emergency Client-Contact Workflow into buildable construction guidance. It defines the coordination boundary that receives verified emergency truth, executes the configured client contact path, and produces an immutable outcome envelope for downstream interpretation and messaging.

## 2. Build Scope

| Included | Deferred | Forbidden |
|---|---|---|
| Emergency workflow coordination gateway | Automatic retry policies | Emergency severity determination |
| Precondition validation | Multi-recipient escalation schedules | Service capability evaluation |
| Configuration-driven path selection | Rich authorization fact capture beyond MVP | Service area validation |
| Outbound contact orchestration | Outcome schema migration tooling | Intake record creation |
| Live transfer handoff coordination | | Direct telephone call placement |
| Fallback notification triggering | | Direct notification sending |
| Immutable outcome envelope compilation | | Caller-facing message rendering |
| Intake outcome field updates | | Retry logic |
| Audit event emission | | Hardcoded business behavior |

## 3. Recommended Project Structure

```text
emergency-client-contact-workflow/
│
├── Contracts
│   ├── Emergency Client Contact Outcome Contract
│   ├── Attempt Client Contact Adapter Contract
│   ├── Live Call Transfer Adapter Contract
│   └── Notification Request Contract
│
├── Coordination
│   ├── Emergency Client Contact Gateway
│   ├── Precondition Validator
│   └── Workflow Router
│
├── Execution
│   └── External Action Dispatcher
│
├── Outcomes
│   └── Outcome Record Builder
│
└── Tests
    ├── Contract Tests
    ├── Path Selection Tests
    ├── Outcome Tests
    ├── Fallback Tests
    └── Adapter Isolation Tests
```

## 4. Public Contracts

### Input Contract
The gateway consumes:
- Confirmed emergency handling decision
- Service area decision
- Capability decision
- Persisted intake record
- Location emergency contact configuration
- Current time context
- Optional transaction correlation keys

### Output Contract: Emergency Client Contact Outcome
A successful outcome carries:
- Contact mode followed
- Whether live contact was attempted
- Terminal workflow outcome state
- Transfer summary, when applicable
- Notification summary, when applicable
- Timeline authorization facts, when applicable
- Operational timestamps
- Correlation markers

A workflow error outcome carries:
- Error classification
- Internal diagnostic context
- Correlation markers

### Adapter Contracts
- **Attempt Client Contact Adapter Contract** — Handles outbound contact mechanics and returns structured contact results.
- **Live Call Transfer Adapter Contract** — Bridges an answered client line back to the active caller.
- **Notification Request Contract** — Requests notification compilation and dispatch through downstream boundaries.

## 5. Internal Components

| Component | Purpose | Forbidden |
|---|---|---|
| Emergency Client Contact Gateway | Entry point and orchestration | Determining emergency status; rendering caller messages |
| Precondition Validator | Confirms emergency authorization, serviceability, capability, and intake completeness | Modifying active records; selecting contact paths |
| Workflow Router | Selects the configured contact path and fallback based on capability flags | Placing calls; managing active connections |
| External Action Dispatcher | Coordinates outbound contact adapters, transfer adapters, and fallback notification requests | Writing persistent records; deciding retry rules |
| Outcome Record Builder | Compiles the immutable outcome envelope and updates the intake record's contact outcome fields | Composing notification templates; selecting recipients |

## 6. Runtime Construction Flow

```text
Verified Emergency Inputs
        |
        v
Precondition Validator
        |
        v
Workflow Router
        |
        v
+-------+-------+
|               |
v               v
Warm Transfer   Notification Only
Path
        |
        v
Attempt Client Contact
        |
        v
+------------+----------+-----------+
|            |          |           |
v            v          v           v
Answered  Declined  Voicemail  Unavailable
Accepted
        |
        v
Live Call Transfer
        |
        v
+-------+-------+
|               |
v               v
Transfer      Transfer
Completed       Failed
        |
        v
Fallback Notification Request
        |
        v
Outcome Record Builder
        |
        v
Immutable Outcome Envelope
```

Every path reaches exactly one terminal outcome state.

## 7. Dependency Map

```text
Downstream Interpretation and Messaging Modules
        ^
        |
Emergency Client Contact Gateway
        |
+----------------+--------------+
|                |              |
v                v              v
Precondition  Workflow      External Action
Validator     Router        Dispatcher
                     |
                     v
            Attempt Client Contact Adapter
                     |
                     v
            Live Call Transfer Adapter
                     |
                     v
            Notification Request Boundary
                     |
                     v
            Outcome Record Builder
```

## 8. Error Handling Strategy

| Classification | Source | Handling |
|---|---|---|
| Precondition block | Missing emergency authorization, unsupported service, unserviceable area, incomplete intake | Terminal; return error without updating intake |
| Contact timeout | Outbound client contact exceeds configured window | Recoverable through fallback notification if enabled |
| Transfer failure | Client accepted but bridge could not be established | Recoverable through fallback notification if enabled |
| Notification compilation failure | Downstream notification boundary cannot build payload | Terminal; return workflow error |
| Notification delivery failure | Fallback notification cannot be delivered | Terminal; record failed outcome |
| Persistence fault | Intake outcome update fails | Terminal; return workflow error |
| Unexpected workflow error | Unclassified adapter or state failure | Terminal; return workflow error |

Retry — Owned downstream. This module executes each configured step exactly once.

Propagation — The boundary returns a structured outcome envelope. Internal diagnostics remain internal; telephony specifics, adapter states, and raw contact details are not exposed to callers.

## 9. Testing Blueprint

- Contract Tests — valid and invalid outcome envelope shapes
- Path Selection Tests — warm transfer, notification-only, and capability-disabled scenarios
- Outcome Tests — all seven terminal outcome states
- Fallback Tests — voicemail, notification fallback, and disabled fallback paths
- Adapter Isolation Tests — mock contact, transfer, and notification boundaries
- Determinism Tests — identical inputs produce identical routing decisions
- Precondition Tests — authorization, serviceability, capability, and intake completeness enforcement
- Outcome Immutability Tests — downstream modules cannot modify the sealed envelope

## 10. Construction Checklist

✓ Emergency Client Contact Outcome Contract defined  
✓ Attempt Client Contact Adapter Contract defined  
✓ Live Call Transfer Adapter Contract defined  
✓ Notification Request Contract defined  
✓ Emergency Client Contact Gateway accepts verified emergency inputs  
✓ Precondition Validator confirms emergency authorization  
✓ Precondition Validator confirms service capability and service area eligibility  
✓ Precondition Validator confirms intake record completeness  
✓ Workflow Router selects path from configured contact mode  
✓ Workflow Router respects preference-capability separation  
✓ External Action Dispatcher coordinates outbound contact attempts  
✓ External Action Dispatcher coordinates live transfer when accepted  
✓ External Action Dispatcher requests fallback notification when configured  
✓ Outcome Record Builder compiles immutable outcome envelope  
✓ Outcome Record Builder updates intake record contact outcome fields  
✓ Seven terminal outcome states are mutually exclusive and exhaustive  
✓ Audit events emitted for contact attempted, completed, failed, transfer completed, and notification requested  
✓ No emergency determination or severity evaluation introduced  
✓ No direct call placement, transfer execution, or notification sending introduced  
✓ No caller-facing message rendering introduced  
✓ No retry logic introduced  
✓ Correlation keys carried through every step  
✓ Architectural ownership preserved: workflow coordinates; upstream modules decide eligibility; downstream modules interpret and communicate  

---

# End of Blueprint

**Emergency Client-Contact Workflow**

**Derived from:** 005_Emergency_Client_Contact_Workflow.md and EmergencyClientContact.PRD

**Architectural Role:**

**Coordination Boundary / Emergency Client Contact Workflow Orchestrator**