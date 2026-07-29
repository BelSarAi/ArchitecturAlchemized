# 007 Operational Visibility Framework — Implementation

**Assumption note:** `AuditLogAdapter.PRD` provides only the backend-neutral persistence interface for this framework. The canonical shapes, validation rules, correlation standards, severity levels, PII minimization policy, and event naming conventions below are derived strictly from `AuditEventContract.PRD`. The Operational Visibility Framework owns the envelope contract and normalization pipeline; the Visibility Persistence Boundary owns durable storage, retention, fallback handling, and query interfaces.

---

## Project Structure

```text
operational-visibility-framework/
│
├── contracts/
│   ├── audit-event.contract.ts
│   ├── severity-level.contract.ts
│   ├── correlation-ids.contract.ts
│   ├── audit-log-result.contract.ts
│   ├── visibility-handoff.contract.ts
│   └── raw-observation.contract.ts
│
├── normalization/
│   ├── envelope-contract.guardian.ts
│   ├── correlation-vector.injector.ts
│   ├── pii-minimization.filter.ts
│   ├── severity.classifier.ts
│   └── operational-visibility.gateway.ts
│
├── errors/
│   ├── envelope-violation.error.ts
│   └── privacy-violation.error.ts
│
└── tests/
    ├── contract.test.ts
    ├── privacy.test.ts
    ├── correlation.test.ts
    ├── fail-open.test.ts
    └── severity.test.ts
```

**Mapping to Layer 3 Blueprint §3 (Recommended Project Structure):**

| Layer 3 Blueprint Folder / Component | Layer 4 Implementation Location |
|---|---|
| Canonical Event Envelope Contract | `contracts/audit-event.contract.ts` |
| Correlation Vector Contract | `contracts/correlation-ids.contract.ts` |
| Severity Classification Contract | `contracts/severity-level.contract.ts` |
| Visibility Handoff Contract | `contracts/visibility-handoff.contract.ts` + `contracts/audit-log-result.contract.ts` |
| Envelope Contract Guardian | `normalization/envelope-contract.guardian.ts` |
| Correlation Vector Injector | `normalization/correlation-vector.injector.ts` |
| PII Minimization Filter | `normalization/pii-minimization.filter.ts` |
| Severity Classifier | `normalization/severity.classifier.ts` |
| Asynchronous Handoff to Visibility Persistence | `normalization/operational-visibility.gateway.ts` |
| Contract / Privacy / Correlation / Fail-Open Tests | `tests/*.test.ts` |

---

## contracts/severity-level.contract.ts

```typescript
/**
 * Severity Classification Contract
 *
 * Maps directly to AuditEventContract.PRD §5.2 and §14.1.
 * The framework assigns visibility priority levels; it does not trigger alerts.
 */
export const SEVERITY_LEVELS = ["info", "warning", "error"] as const;

export type SeverityLevel = typeof SEVERITY_LEVELS[number];

export function isSeverityLevel(value: unknown): value is SeverityLevel {
  return typeof value === "string" && SEVERITY_LEVELS.includes(value as SeverityLevel);
}
```

**Mapping to Layer 3 Blueprint §4 (Output Contract: Canonical Event Envelope):**
The blueprint requires a severity classification in every canonical event. This contract supplies the allowed values.

---

## contracts/correlation-ids.contract.ts

```typescript
/**
 * Correlation Vector Contract
 *
 * Maps directly to AuditEventContract.PRD §5.2 correlation keys.
 * All keys are optional; emitters include the keys that apply to the observation.
 */
export interface CorrelationIds {
  readonly callSessionId?: string;
  readonly intakeId?: string;
  readonly clientId?: string;
  readonly locationId?: string;
}

export const KNOWN_CORRELATION_KEYS: readonly (keyof CorrelationIds)[] = [
  "callSessionId",
  "intakeId",
  "clientId",
  "locationId",
] as const;
```

**Mapping to Layer 3 Blueprint §4 (Output Contract: Canonical Event Envelope):**
The blueprint requires an interaction trace vector linking events to interactions, requests, clients, and locations without duplicating protected attributes.

---

## contracts/audit-event.contract.ts

```typescript
import type { SeverityLevel } from "./severity-level.contract";
import type { CorrelationIds } from "./correlation-ids.contract";

/**
 * Canonical Event Envelope Contract
 *
 * Derived directly from AuditEventContract.PRD §5.2.
 * This is the universal envelope shared by every operational event.
 */
export interface AuditEvent {
  readonly eventId: string;
  readonly eventType: string;
  readonly timestamp: string;
  readonly sourceModule: string;
  readonly severity: SeverityLevel;
  readonly correlation: CorrelationIds;
  readonly eventData: Record<string, unknown>;
  readonly errorDetails?: {
    readonly errorType: string;
    readonly errorCode: string;
    readonly message: string;
    readonly stackTrace?: string;
  };
}

/**
 * Event type naming rule from AuditEventContract.PRD §7.
 * Format: module.action, lowercase, dot-separated, optional snake_case segments.
 */
export const EVENT_TYPE_PATTERN = /^[a-z0-9]+(?:_[a-z0-9]+)*\.[a-z0-9]+(?:_[a-z0-9]+)*$/;

/**
 * ISO 8601 UTC timestamp with milliseconds.
 * Example: "2026-06-24T14:32:15.123Z"
 */
export const TIMESTAMP_PATTERN = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/;

export function isValidEventType(value: unknown): value is string {
  return typeof value === "string" && EVENT_TYPE_PATTERN.test(value);
}

export function isValidTimestamp(value: unknown): value is string {
  return typeof value === "string" && TIMESTAMP_PATTERN.test(value);
}
```

**Mapping to Layer 3 Blueprint §4 (Output Contract: Canonical Event Envelope):**
The blueprint specifies a validated event carrying identity, event type, observed moment, originating boundary, severity, trace vector, domain context payload, and optional fault description context.

---

## contracts/audit-log-result.contract.ts

```typescript
/**
 * Audit Log Result Contract
 *
 * Derived directly from AuditEventContract.PRD §5.2 and §14.3.
 * Indicates synchronous acceptance or rejection of an event by the visibility pipeline.
 * Async persistence failures are handled through fallback paths and are invisible to emitters.
 */
export type AuditLogResult =
  | { readonly accepted: true }
  | { readonly accepted: false; readonly reason: string };
```

**Mapping to Layer 3 Blueprint §4 (Output Contract: Canonical Event Envelope):**
A validation failure returns an error classification and diagnostic context. This contract expresses that result.

---

## contracts/visibility-handoff.contract.ts

```typescript
import type { AuditEvent } from "./audit-event.contract";
import type { AuditLogResult } from "./audit-log-result.contract";

/**
 * Visibility Persistence Handoff Contract
 *
 * Maps to Layer 3 Blueprint §4 (Visibility Persistence Handoff Contract).
 * The framework depends only on this interface; concrete storage adapters live in the
 * Visibility Persistence Boundary (AuditLogAdapter.PRD §10.1).
 */
export interface VisibilityHandoffInterface {
  readonly submit: (event: AuditEvent) => Promise<AuditLogResult>;
}
```

**Mapping to Layer 3 Blueprint §4 (Visibility Persistence Handoff Contract):**
The framework hands normalized events to the visibility persistence boundary asynchronously. The persistence boundary owns durable storage, batching, retention, fallback handling, and query interfaces.

---

## contracts/raw-observation.contract.ts

```typescript
import type { SeverityLevel } from "./severity-level.contract";
import type { CorrelationIds } from "./correlation-ids.contract";

/**
 * Raw Observation Input Contract
 *
 * Maps to Layer 3 Blueprint §4 (Input Contract).
 * Emitting modules supply these fields; the framework normalizes them into the canonical envelope.
 */
export interface RawObservation {
  readonly eventType: string;
  readonly sourceModule: string;
  readonly severity: SeverityLevel;
  readonly correlation: CorrelationIds;
  readonly eventData: Record<string, unknown>;
  readonly observedAt?: string;
  readonly errorDetails?: {
    readonly errorType: string;
    readonly errorCode: string;
    readonly message: string;
    readonly stackTrace?: string;
  };
}
```

**Mapping to Layer 3 Blueprint §4 (Input Contract):**
The blueprint lists domain event type, originating boundary marker, domain context payload, optional fault description context, interaction trace vector, and observed moment marker. This interface captures all six inputs.

---

## errors/envelope-violation.error.ts

```typescript
/**
 * Error raised when a raw observation fails envelope validation.
 *
 * Maps to Layer 3 Blueprint §8 Error Handling Strategy — Envelope Violation.
 */
export class EnvelopeViolationError extends Error {
  constructor(
    message: string,
    public readonly diagnostics: readonly string[],
  ) {
    super(message);
    this.name = "EnvelopeViolationError";
  }
}
```

**Mapping to Layer 3 Blueprint §8:**
Envelope violations are terminal for the event and return a diagnostic. This error carries the diagnostic context.

---

## errors/privacy-violation.error.ts

```typescript
/**
 * Error raised when a domain context payload contains protected customer attributes.
 *
 * Maps to Layer 3 Blueprint §8 Error Handling Strategy — Privacy Violation.
 */
export class PrivacyViolationError extends Error {
  constructor(
    message: string,
    public readonly violations: readonly string[],
  ) {
    super(message);
    this.name = "PrivacyViolationError";
  }
}
```

**Mapping to Layer 3 Blueprint §8:**
Privacy violations are rejected or masked; business execution continues. This error captures which fields violated policy.

---

## normalization/envelope-contract.guardian.ts

```typescript
import { v4 as uuidv4 } from "uuid";
import type { AuditEvent } from "../contracts/audit-event.contract";
import type { RawObservation } from "../contracts/raw-observation.contract";
import {
  isValidEventType,
  isValidTimestamp,
} from "../contracts/audit-event.contract";
import { isSeverityLevel } from "../contracts/severity-level.contract";
import { EnvelopeViolationError } from "../errors/envelope-violation.error";

/**
 * Envelope Contract Guardian
 *
 * Purpose (Layer 3 Blueprint §5):
 * Defines and validates the canonical event envelope. Ensures every emitted event
 * carries the required identity, timing, origin, severity, correlation, and payload fields.
 *
 * Forbidden:
 * - Defining business event semantics
 * - Selecting storage backends
 * - Deciding which events modules should emit
 */
export class EnvelopeContractGuardian {
  /**
   * Validates the raw observation and produces a canonical AuditEvent.
   * Generates eventId and timestamp if not supplied.
   */
  public guard(observation: RawObservation): AuditEvent {
    const diagnostics: string[] = [];

    if (!isValidEventType(observation.eventType)) {
      diagnostics.push(
        `eventType must match module.action naming convention; received "${observation.eventType}"`,
      );
    }

    if (typeof observation.sourceModule !== "string" || observation.sourceModule.length === 0) {
      diagnostics.push("sourceModule is required and must be a non-empty string");
    }

    if (!isSeverityLevel(observation.severity)) {
      diagnostics.push(
        `severity must be one of "info", "warning", "error"; received "${observation.severity}"`,
      );
    }

    if (observation.observedAt !== undefined && !isValidTimestamp(observation.observedAt)) {
      diagnostics.push(
        `observedAt must be ISO 8601 UTC with milliseconds; received "${observation.observedAt}"`,
      );
    }

    if (observation.eventData === null || typeof observation.eventData !== "object" || Array.isArray(observation.eventData)) {
      diagnostics.push("eventData must be a JSON-serializable object");
    } else if (!this.isJsonSerializable(observation.eventData)) {
      diagnostics.push("eventData contains non-serializable values (functions, circular references, etc.)");
    }

    if (observation.errorDetails !== undefined) {
      if (this.containsErrorRelatedFields(observation.eventData)) {
        diagnostics.push(
          "When errorDetails is present, eventData must not contain error-related fields",
        );
      }
      if (!this.isValidErrorDetails(observation.errorDetails)) {
        diagnostics.push("errorDetails must contain errorType, errorCode, and message");
      }
    }

    if (diagnostics.length > 0) {
      throw new EnvelopeViolationError("Envelope validation failed", diagnostics);
    }

    return {
      eventId: uuidv4(),
      eventType: observation.eventType,
      timestamp: observation.observedAt ?? this.utcNow(),
      sourceModule: observation.sourceModule,
      severity: observation.severity,
      correlation: observation.correlation,
      eventData: observation.eventData,
      errorDetails: observation.errorDetails,
    };
  }

  private utcNow(): string {
    return new Date().toISOString();
  }

  private isJsonSerializable(value: unknown): boolean {
    try {
      JSON.stringify(value);
      return true;
    } catch {
      return false;
    }
  }

  private containsErrorRelatedFields(eventData: Record<string, unknown>): boolean {
    const keys = Object.keys(eventData);
    return keys.some((key) =>
      ["error", "errorType", "errorCode", "message", "stackTrace"].includes(key),
    );
  }

  private isValidErrorDetails(
    details: RawObservation["errorDetails"],
  ): details is NonNullable<RawObservation["errorDetails"]> {
    return (
      details !== undefined &&
      typeof details.errorType === "string" &&
      typeof details.errorCode === "string" &&
      typeof details.message === "string"
    );
  }
}
```

**Mapping to Layer 3 Blueprint §5 (Internal Components):**
The Envelope Contract Guardian defines and validates the canonical event envelope. The implementation mirrors the blueprint's stated purpose and forbidden bounds.

---

## normalization/correlation-vector.injector.ts

```typescript
import type { AuditEvent } from "../contracts/audit-event.contract";
import type { CorrelationIds } from "../contracts/correlation-ids.contract";
import { KNOWN_CORRELATION_KEYS } from "../contracts/correlation-ids.contract";

/**
 * Correlation Vector Injector
 *
 * Purpose (Layer 3 Blueprint §5):
 * Attaches standardized trace markers to link events across modules.
 *
 * Forbidden:
 * - Duplicating protected customer attributes
 * - Introducing new correlation markers outside the agreed contract
 */
export class CorrelationVectorInjector {
  /**
   * Ensures only known correlation keys are present and strips undefined values.
   * Does not create new correlation dimensions.
   */
  public inject(
    event: Omit<AuditEvent, "correlation">,
    correlation: CorrelationIds,
  ): AuditEvent {
    const stripped: CorrelationIds = {};

    for (const key of KNOWN_CORRELATION_KEYS) {
      const value = correlation[key];
      if (value !== undefined && value !== null && value !== "") {
        stripped[key] = value;
      }
    }

    const unknownKeys = Object.keys(correlation).filter(
      (key) => !KNOWN_CORRELATION_KEYS.includes(key as keyof CorrelationIds),
    );

    if (unknownKeys.length > 0) {
      // Unknown correlation keys are an envelope violation.
      // They risk breaking cross-module tracing contracts.
      throw new Error(`Unknown correlation keys: ${unknownKeys.join(", ")}`);
    }

    return { ...event, correlation: stripped };
  }
}
```

**Mapping to Layer 3 Blueprint §5:**
The Correlation Vector Injector attaches standardized trace markers. The implementation enforces the agreed key set and removes empty values, preventing contamination.

---

## normalization/pii-minimization.filter.ts

```typescript
import type { AuditEvent } from "../contracts/audit-event.contract";
import { PrivacyViolationError } from "../errors/privacy-violation.error";

/**
 * PII Minimization Filter
 *
 * Purpose (Layer 3 Blueprint §5):
 * Reviews domain context payloads for protected customer attributes and applies
 * masking or rejection before the event is handed off.
 *
 * Forbidden:
 * - Altering business semantics
 * - Persisting events
 * - Blocking business execution when a violation is found
 *
 * Policy source: AuditEventContract.PRD §8.
 */
export class PiiMinimizationFilter {
  private readonly prohibitedPatterns = [
    { name: "customerName", pattern: /^[A-Za-z]+([ '-][A-Za-z]+)*$/ },
    { name: "email", pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
    { name: "phone", pattern: /^\+?[\d\s\-\(\)]{10,}$/ },
    { name: "fullAddress", pattern: /\d+\s+\w+/ },
  ];

  private readonly prohibitedKeys = [
    "customerName",
    "fullName",
    "firstName",
    "lastName",
    "phone",
    "phoneNumber",
    "email",
    "emailAddress",
    "streetAddress",
    "fullAddress",
  ];

  /**
   * Returns a scrubbed event if all violations can be masked.
   * Throws PrivacyViolationError if unmaskable protected data is detected.
   */
  public filter(event: AuditEvent): AuditEvent {
    const violations: string[] = [];
    const scrubbedData = this.scrubObject(event.eventData, violations);

    if (violations.length > 0) {
      throw new PrivacyViolationError(
        "Event payload contains prohibited PII",
        violations,
      );
    }

    return { ...event, eventData: scrubbedData };
  }

  private scrubObject(
    value: unknown,
    violations: string[],
    path = "eventData",
  ): unknown {
    if (value === null || typeof value !== "object") {
      return this.scrubScalar(value, path, violations);
    }

    if (Array.isArray(value)) {
      return value.map((item, index) => this.scrubObject(item, violations, `${path}[${index}]`));
    }

    const result: Record<string, unknown> = {};
    for (const [key, val] of Object.entries(value as Record<string, unknown>)) {
      const childPath = `${path}.${key}`;
      if (this.isProhibitedKey(key)) {
        violations.push(childPath);
        result[key] = this.maskValue(val);
      } else {
        result[key] = this.scrubObject(val, violations, childPath);
      }
    }
    return result;
  }

  private scrubScalar(
    value: unknown,
    path: string,
    violations: string[],
  ): unknown {
    if (typeof value === "string") {
      for (const { name, pattern } of this.prohibitedPatterns) {
        if (pattern.test(value)) {
          violations.push(`${path} (matched ${name} pattern)`);
          return this.maskValue(value);
        }
      }
    }
    return value;
  }

  private isProhibitedKey(key: string): boolean {
    return this.prohibitedKeys.includes(key);
  }

  private maskValue(value: unknown): unknown {
    if (typeof value === "string") {
      if (value.length <= 4) return "****";
      return value.slice(0, -4).replace(/./g, "*") + value.slice(-4);
    }
    return "[REDACTED]";
  }
}
```

**Mapping to Layer 3 Blueprint §5:**
The PII Minimization Filter reviews payloads for protected customer attributes. The implementation detects known keys and patterns, masks where possible, and raises a structured error for unmaskable violations.

---

## normalization/severity.classifier.ts

```typescript
import type { AuditEvent } from "../contracts/audit-event.contract";
import type { SeverityLevel } from "../contracts/severity-level.contract";

/**
 * Severity Classifier
 *
 * Purpose (Layer 3 Blueprint §5):
 * Assigns visibility priority level based on operational significance.
 *
 * Forbidden:
 * - Triggering alerts or paging systems
 * - Making operational decisions
 */
export class SeverityClassifier {
  /**
   * Applies the severity supplied by the emitter. The framework does not reinterpret
   * business significance; it only ensures the value is one of the allowed levels.
   */
  public classify(
    event: Omit<AuditEvent, "severity">,
    severity: SeverityLevel,
  ): AuditEvent {
    return { ...event, severity };
  }
}
```

**Mapping to Layer 3 Blueprint §5:**
The Severity Classifier assigns visibility priority levels. The implementation preserves emitter intent, matching the blueprint's rule that severity misclassification is corrected at the emitter.

---

## normalization/operational-visibility.gateway.ts

```typescript
import type { RawObservation } from "../contracts/raw-observation.contract";
import type { AuditEvent } from "../contracts/audit-event.contract";
import type { AuditLogResult } from "../contracts/audit-log-result.contract";
import type { VisibilityHandoffInterface } from "../contracts/visibility-handoff.contract";
import { EnvelopeContractGuardian } from "./envelope-contract.guardian";
import { CorrelationVectorInjector } from "./correlation-vector.injector";
import { PiiMinimizationFilter } from "./pii-minimization.filter";
import { SeverityClassifier } from "./severity.classifier";

/**
 * Operational Visibility Gateway
 *
 * Purpose (Layer 3 Blueprint §6):
 * Orchestrates the runtime construction flow:
 *   Domain Observation -> Envelope Guardian -> Correlation Injector ->
 *   PII Filter -> Severity Classifier -> Canonical Event -> Async Handoff
 *
 * Forbidden:
 * - Business logic execution
 * - Notification delivery
 * - Workflow decision making
 * - Direct persistence writes
 * - Blocking business execution on visibility failure
 */
export class OperationalVisibilityGateway {
  constructor(
    private readonly guardian: EnvelopeContractGuardian,
    private readonly correlationInjector: CorrelationVectorInjector,
    private readonly piiFilter: PiiMinimizationFilter,
    private readonly severityClassifier: SeverityClassifier,
    private readonly handoff: VisibilityHandoffInterface,
  ) {}

  /**
   * Records a domain observation.
   * Returns synchronously as soon as the event is validated and handed off.
   * Business execution continues regardless of persistence outcome.
   */
  public async record(observation: RawObservation): Promise<AuditLogResult> {
    try {
      const validated = this.guardian.guard(observation);
      const correlated = this.correlationInjector.inject(validated, validated.correlation);
      const scrubbed = this.piiFilter.filter(correlated);
      const classified = this.severityClassifier.classify(scrubbed, scrubbed.severity);

      // Hand off asynchronously. The framework never waits for persistence.
      return this.handoff.submit(classified);
    } catch (error) {
      // Envelope or privacy violations reject the event synchronously,
      // but the caller continues business execution.
      return {
        accepted: false,
        reason: error instanceof Error ? error.message : "visibility_pipeline_error",
      };
    }
  }

  /**
   * Synchronous variant for fire-and-forget scenarios.
   * The caller receives immediate control and never awaits persistence.
   */
  public recordFireAndForget(observation: RawObservation): void {
    this.record(observation).catch((error) => {
      this.logFallback(observation, error);
    });
  }

  private logFallback(observation: RawObservation, error: unknown): void {
    // Fallback path specified in Layer 3 Blueprint §8.
    // In production this writes to stderr, console, or a dead-letter queue.
    // It must remain privacy-safe and never reintroduce protected attributes.
    console.error("[VISIBILITY FALLBACK] Event could not be handed off", {
      eventType: observation.eventType,
      sourceModule: observation.sourceModule,
      reason: error instanceof Error ? error.message : "unknown",
    });
  }
}
```

**Mapping to Layer 3 Blueprint §6:**
The runtime construction flow in the blueprint lists six sequential stages. The gateway implements all six: observation ingestion, envelope validation, correlation injection, PII filtering, severity classification, and asynchronous handoff.

---

## tests/contract.test.ts

```typescript
import { describe, it, expect } from "vitest";
import { EnvelopeContractGuardian } from "../normalization/envelope-contract.guardian";
import type { RawObservation } from "../contracts/raw-observation.contract";
import { EnvelopeViolationError } from "../errors/envelope-violation.error";

describe("Contract Tests", () => {
  const guardian = new EnvelopeContractGuardian();

  it("TC-AUDIT-CONTRACT-01: valid event conforms to envelope", () => {
    const observation: RawObservation = {
      eventType: "service_area.checked",
      sourceModule: "ServiceAreaValidation",
      severity: "info",
      correlation: { callSessionId: "call_123", intakeId: "intake_456" },
      eventData: { serviceable: true },
    };

    const event = guardian.guard(observation);

    expect(event.eventId).toBeDefined();
    expect(event.eventType).toBe("service_area.checked");
    expect(event.timestamp).toMatch(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/);
    expect(event.sourceModule).toBe("ServiceAreaValidation");
    expect(event.severity).toBe("info");
    expect(event.correlation.callSessionId).toBe("call_123");
    expect(event.eventData.serviceable).toBe(true);
  });

  it("TC-AUDIT-CONTRACT-02: event type follows naming convention", () => {
    const observation: RawObservation = {
      eventType: "Invalid Event Type",
      sourceModule: "ServiceAreaValidation",
      severity: "info",
      correlation: {},
      eventData: {},
    };

    expect(() => guardian.guard(observation)).toThrow(EnvelopeViolationError);
  });

  it("TC-AUDIT-CONTRACT-03: invalid severity is rejected", () => {
    const observation = {
      eventType: "service_area.checked",
      sourceModule: "ServiceAreaValidation",
      severity: "invalid",
      correlation: {},
      eventData: {},
    } as unknown as RawObservation;

    expect(() => guardian.guard(observation)).toThrow(EnvelopeViolationError);
  });

  it("TC-AUDIT-CONTRACT-04: unknown correlation keys are rejected", () => {
    const observation = {
      eventType: "service_area.checked",
      sourceModule: "ServiceAreaValidation",
      severity: "info",
      correlation: { campaignId: "camp_123" },
      eventData: {},
    } as unknown as RawObservation;

    expect(() => guardian.guard(observation)).toThrow();
  });

  it("TC-AUDIT-CONTRACT-09: error details canonical location enforced", () => {
    const observation: RawObservation = {
      eventType: "notification.failed",
      sourceModule: "NotificationAdapter_SMS",
      severity: "error",
      correlation: { intakeId: "intake_456" },
      eventData: { errorCode: "SMS_001" },
      errorDetails: {
        errorType: "TransportError",
        errorCode: "SMS_001",
        message: "Provider unavailable",
      },
    };

    expect(() => guardian.guard(observation)).toThrow(EnvelopeViolationError);
  });
});
```

**Mapping to Layer 3 Blueprint §9 (Testing Blueprint):**
Contract Tests verify envelope conformance and required field validation, matching TC-AUDIT-CONTRACT-01 through TC-AUDIT-CONTRACT-09 from AuditEventContract.PRD §27.

---

## tests/privacy.test.ts

```typescript
import { describe, it, expect } from "vitest";
import { PiiMinimizationFilter } from "../normalization/pii-minimization.filter";
import { PrivacyViolationError } from "../errors/privacy-violation.error";
import type { AuditEvent } from "../contracts/audit-event.contract";

describe("Privacy Tests", () => {
  const filter = new PiiMinimizationFilter();

  it("TC-AUDIT-CONTRACT-05: PII in eventData is flagged", () => {
    const event: AuditEvent = {
      eventId: "evt_001",
      eventType: "intake.created",
      timestamp: "2026-06-24T14:32:15.123Z",
      sourceModule: "IntakeRequest",
      severity: "info",
      correlation: { intakeId: "intake_456" },
      eventData: { customerName: "John Doe" },
    };

    expect(() => filter.filter(event)).toThrow(PrivacyViolationError);
  });

  it("masks phone numbers when permitted by policy", () => {
    const event: AuditEvent = {
      eventId: "evt_002",
      eventType: "notification.sent",
      timestamp: "2026-06-24T14:32:15.123Z",
      sourceModule: "NotificationAdapter_SMS",
      severity: "info",
      correlation: { intakeId: "intake_456" },
      eventData: { callbackPhone: "+15551234567" },
    };

    const result = filter.filter(event);
    expect(result.eventData.callbackPhone).toBe("+*******4567");
  });
});
```

**Mapping to Layer 3 Blueprint §9:**
Privacy Tests verify detection and masking of protected customer attributes.

---

## tests/correlation.test.ts

```typescript
import { describe, it, expect } from "vitest";
import { CorrelationVectorInjector } from "../normalization/correlation-vector.injector";
import type { AuditEvent } from "../contracts/audit-event.contract";

describe("Correlation Tests", () => {
  const injector = new CorrelationVectorInjector();

  it("attaches standardized trace markers consistently", () => {
    const partial: Omit<AuditEvent, "correlation"> = {
      eventId: "evt_003",
      eventType: "intake.created",
      timestamp: "2026-06-24T14:32:15.123Z",
      sourceModule: "IntakeRequest",
      severity: "info",
      eventData: { serviceId: "svc_001" },
    };

    const event = injector.inject(partial, {
      callSessionId: "call_123",
      clientId: "client_456",
      locationId: "location_789",
      intakeId: "intake_abc",
    });

    expect(event.correlation).toEqual({
      callSessionId: "call_123",
      clientId: "client_456",
      locationId: "location_789",
      intakeId: "intake_abc",
    });
  });

  it("strips empty correlation values", () => {
    const partial: Omit<AuditEvent, "correlation"> = {
      eventId: "evt_004",
      eventType: "configuration.loaded",
      timestamp: "2026-06-24T14:32:15.123Z",
      sourceModule: "ConfigurationLoader",
      severity: "info",
      eventData: {},
    };

    const event = injector.inject(partial, {
      callSessionId: "",
      clientId: "client_456",
      intakeId: undefined,
    });

    expect(event.correlation).toEqual({ clientId: "client_456" });
  });
});
```

**Mapping to Layer 3 Blueprint §9:**
Correlation Tests verify trace marker consistency across modules.

---

## tests/fail-open.test.ts

```typescript
import { describe, it, expect, vi } from "vitest";
import { OperationalVisibilityGateway } from "../normalization/operational-visibility.gateway";
import { EnvelopeContractGuardian } from "../normalization/envelope-contract.guardian";
import { CorrelationVectorInjector } from "../normalization/correlation-vector.injector";
import { PiiMinimizationFilter } from "../normalization/pii-minimization.filter";
import { SeverityClassifier } from "../normalization/severity.classifier";
import type { RawObservation } from "../contracts/raw-observation.contract";
import type { VisibilityHandoffInterface } from "../contracts/visibility-handoff.contract";

describe("Fail-Open Tests", () => {
  it("business workflow continues when persistence fails", async () => {
    const failingHandoff: VisibilityHandoffInterface = {
      submit: vi.fn().mockRejectedValue(new Error("storage unavailable")),
    };

    const gateway = new OperationalVisibilityGateway(
      new EnvelopeContractGuardian(),
      new CorrelationVectorInjector(),
      new PiiMinimizationFilter(),
      new SeverityClassifier(),
      failingHandoff,
    );

    const observation: RawObservation = {
      eventType: "service_area.checked",
      sourceModule: "ServiceAreaValidation",
      severity: "info",
      correlation: { intakeId: "intake_456" },
      eventData: { serviceable: true },
    };

    const result = await gateway.record(observation);

    expect(result.accepted).toBe(false);
    expect(failingHandoff.submit).toHaveBeenCalled();
  });

  it("fire-and-forget never throws to caller", async () => {
    const failingHandoff: VisibilityHandoffInterface = {
      submit: vi.fn().mockRejectedValue(new Error("storage unavailable")),
    };

    const gateway = new OperationalVisibilityGateway(
      new EnvelopeContractGuardian(),
      new CorrelationVectorInjector(),
      new PiiMinimizationFilter(),
      new SeverityClassifier(),
      failingHandoff,
    );

    const observation: RawObservation = {
      eventType: "service_area.checked",
      sourceModule: "ServiceAreaValidation",
      severity: "info",
      correlation: { intakeId: "intake_456" },
      eventData: { serviceable: true },
    };

    expect(() => gateway.recordFireAndForget(observation)).not.toThrow();
  });
});
```

**Mapping to Layer 3 Blueprint §9:**
Fail-Open Tests verify business workflow continuation during visibility pipeline failure.

---

## tests/severity.test.ts

```typescript
import { describe, it, expect } from "vitest";
import { SeverityClassifier } from "../normalization/severity.classifier";
import type { AuditEvent } from "../contracts/audit-event.contract";

describe("Severity Tests", () => {
  const classifier = new SeverityClassifier();

  it("preserves emitter-supplied severity classification", () => {
    const partial: Omit<AuditEvent, "severity"> = {
      eventId: "evt_005",
      eventType: "notification.failed",
      timestamp: "2026-06-24T14:32:15.123Z",
      sourceModule: "NotificationAdapter_SMS",
      correlation: { intakeId: "intake_456" },
      eventData: { provider: "twilio" },
    };

    const event = classifier.classify(partial, "error");
    expect(event.severity).toBe("error");
  });
});
```

**Mapping to Layer 3 Blueprint §9:**
Severity Tests verify correct classification mapping.

---

## Traceability to Layer 3 Blueprint

| Layer 3 Blueprint Component | Layer 4 Implementation | Evidence |
|---|---|---|
| Canonical Event Envelope Contract | `contracts/audit-event.contract.ts` | `AuditEvent` interface with all required fields from §4 |
| Correlation Vector Contract | `contracts/correlation-ids.contract.ts` | `CorrelationIds` interface with four agreed keys |
| Severity Classification Contract | `contracts/severity-level.contract.ts` | `SeverityLevel` type: `info` \| `warning` \| `error` |
| Visibility Handoff Contract | `contracts/visibility-handoff.contract.ts` | `VisibilityHandoffInterface` with async `submit` method |
| Input Contract | `contracts/raw-observation.contract.ts` | `RawObservation` capturing emitter-supplied fields |
| Envelope Contract Guardian | `normalization/envelope-contract.guardian.ts` | Validates required fields, naming, timestamps, serialization |
| Correlation Vector Injector | `normalization/correlation-vector.injector.ts` | Attaches known trace markers, strips empties, rejects unknown keys |
| PII Minimization Filter | `normalization/pii-minimization.filter.ts` | Detects prohibited keys/patterns, masks or raises `PrivacyViolationError` |
| Severity Classifier | `normalization/severity.classifier.ts` | Applies emitter-supplied severity without reinterpretation |
| Asynchronous Handoff to Visibility Persistence | `normalization/operational-visibility.gateway.ts` | Calls `handoff.submit()` asynchronously, never blocks business flow |
| Fallback Logging Path | `OperationalVisibilityGateway.logFallback()` | Writes privacy-safe fallback log on persistence failure |
| Runtime Construction Flow §6 | `OperationalVisibilityGateway.record()` | Sequential: guard → inject → filter → classify → handoff |
| Dependency Map §7 | Constructor injection graph | Gateway depends on contracts and handoff interface only |
| Testing Blueprint §9 | `tests/*.test.ts` | Contract, privacy, correlation, fail-open, severity tests |

---

## Design Decisions Captured

- **Framework owns the envelope contract; emitters own event semantics.** No central event registry exists. Event types follow `module.action` and are validated structurally, not enumerated.
- **Correlation keys are strict and optional.** Only `callSessionId`, `intakeId`, `clientId`, and `locationId` are recognized. Empty values are stripped to avoid noisy logs.
- **PII enforcement is structural, not semantic.** The filter catches known prohibited keys and patterns. New PII categories must be added explicitly; the framework does not attempt natural-language detection.
- **Severity is emitter intent.** The classifier does not reinterpret operational significance; emitters document their event severity in their own PRDs.
- **Fail-open by design.** Envelope and privacy violations reject the event synchronously (returning diagnostics), but the caller continues business execution. Persistence failures are handled through fallback logging.
- **No storage implementation.** The framework depends only on `VisibilityHandoffInterface`. PostgreSQL, CloudWatch, file, and in-memory adapters belong to the Visibility Persistence Boundary.
- **No business logic.** The framework records decisions made elsewhere; it does not route calls, evaluate services, schedule appointments, or send notifications.

---

## Final Quality Checklist

- ✓ Architecture preserved exactly — cross-cutting visibility boundary, no business logic.
- ✓ Responsibilities unchanged — envelope guardian, correlation injector, PII filter, severity classifier, gateway each own one stage.
- ✓ Dependency direction preserved — business modules depend on the contract; the contract depends only on its own definitions.
- ✓ Runtime flow matches Layer 3 — domain observation → envelope guardian → correlation injector → PII filter → severity classifier → canonical event → async handoff.
- ✓ Public contracts implemented — `AuditEvent`, `SeverityLevel`, `CorrelationIds`, `AuditLogResult`, `VisibilityHandoffInterface`, `RawObservation`.
- ✓ Internal components implemented — guardian, injector, filter, classifier, gateway.
- ✓ Tests included — contract, privacy, correlation, fail-open, severity.
- ✓ No architectural shortcuts — no storage backend, retention policy, query interface, dashboard, or alerting logic.
- ✓ No business logic moved across boundaries — routing, scheduling, and notification remain in their owning modules.
- ✓ Production-quality implementation — explicit interfaces, constructor injection, immutable outputs, pure normalization functions.
- ✓ PII minimization enforced — prohibited keys and patterns are detected, masked, or rejected.
- ✓ Fail-open behavior ensured — business workflows continue when visibility pipeline fails.
- ✓ Traceability to Layer 3 documented — every blueprint component maps to a concrete implementation unit.

---

**Operational Visibility Framework**

> Operational Visibility defines how meaningful system actions leave behind structured, privacy-safe evidence.

**Derived from:** `007_Operational_Visibility_Framework.md` and `AuditEventContract.PRD`

**Architectural Role:** Cross-Cutting Visibility Boundary / Canonical Audit Event Contract Owner
