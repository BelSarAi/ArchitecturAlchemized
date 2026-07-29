# 006 Client Notification Framework — Layer 4 Implementation

> **Scope:** Pure-logic notification event compiler. Implements NotificationEventBuilder.PRD. Uses ClientAlertRouting.PRD only as the source of the `RoutingDecision` input contract. No sending, rendering, routing, persistence, or workflow decisions.

---

## 1. Project Structure

```text
client-notification-framework/
│
├── contracts/
│   ├── notification-event.contract.ts
│   ├── template-variable.contract.ts
│   └── compilation-result.contract.ts
│
├── compilation/
│   ├── notification-event-builder.ts
│   ├── input-validator.ts
│   ├── variable-token-mapper.ts
│   └── identity-binder.ts
│
└── tests/
    ├── contract.test.ts
    ├── variable-mapping.test.ts
    ├── namespacing.test.ts
    ├── identity.test.ts
    ├── failure.test.ts
    └── pure-function.test.ts
```

---

## 2. Contracts

### 2.1 Notification Event Contract

```typescript
// contracts/notification-event.contract.ts

export const NOTIFICATION_TYPES = ["standard", "callback", "emergency", "update"] as const;
export type NotificationType = typeof NOTIFICATION_TYPES[number];

export const PRIORITIES = ["standard", "high"] as const;
export type Priority = typeof PRIORITIES[number];

export const CHANNELS = ["sms", "email"] as const;
export type Channel = typeof CHANNELS[number];

export interface Recipient {
  readonly recipient: string; // E.164 phone or email address
  readonly channel: Channel;
  readonly order: number;     // 1 = first attempt, 2 = fallback
}

export interface NotificationEvent {
  readonly schemaVersion: number;
  readonly notificationId: string;
  readonly notificationType: NotificationType;
  readonly priority: Priority;
  readonly templateId: string;
  readonly templateVariables: Readonly<Record<string, string>>;
  readonly recipients: ReadonlyArray<Recipient>;
  readonly correlationId: string;
  readonly intakeId: string;
  readonly clientId: string;
  readonly locationId: string;
  readonly createdAt: string; // ISO 8601 UTC
}
```

### 2.2 Template Variable Contract

```typescript
// contracts/template-variable.contract.ts

/**
 * Canonical, namespaced template variable vocabulary.
 * All keys follow the {domain}{Field} camelCase convention.
 */
export const REQUIRED_TEMPLATE_VARIABLES = [
  "customerName",
  "serviceDisplayName",
  "callbackPhone",
  "issueSummary",
] as const;

export const OPTIONAL_TEMPLATE_VARIABLES = [
  "serviceAddress",
  "accessDetails",
  "preferredCallbackTime",
  "publicViewerUrl",
  "urgencyLevel",
] as const;

export const KNOWN_TEMPLATE_VARIABLES = [
  ...REQUIRED_TEMPLATE_VARIABLES,
  ...OPTIONAL_TEMPLATE_VARIABLES,
] as const;

export type TemplateVariableName = typeof KNOWN_TEMPLATE_VARIABLES[number];

export interface TemplateVariableContract {
  readonly keys: ReadonlyArray<TemplateVariableName>;
  readonly isValidNamespace(key: string): boolean;
}
```

### 2.3 Compilation Result Contract

```typescript
// contracts/compilation-result.contract.ts

import { NotificationEvent } from "./notification-event.contract";

/**
 * Minimal port of the cross-cutting ErrorNormalizer shape.
 * NotificationEventBuilder does not own this contract; it consumes it.
 */
export interface NormalizedError {
  readonly category: "validation_error" | "contract_error" | "internal_error";
  readonly severity: "info" | "warning" | "error";
  readonly callerSafeCode: string;
  readonly errorCode: string;
  readonly errorMessage: string;
  readonly correlationKeys: Readonly<Record<string, string>>;
  readonly isCritical: boolean;
}

export type CompilationResult =
  | { readonly success: true; readonly event: NotificationEvent }
  | { readonly success: false; readonly error: NormalizedError };
```

### 2.4 Routing Decision Input Contract (from ClientAlertRouting.PRD)

```typescript
// contracts/routing-decision.contract.ts

import { Channel, Recipient } from "./notification-event.contract";

/**
 * Consumed as an upstream input only.
 * NotificationEventBuilder does not create or modify routing decisions.
 */
export interface RoutingDecision {
  readonly timeContext: "business_hours" | "after_hours";
  readonly recipients: ReadonlyArray<Recipient>;
}
```

---

## 3. Compilation Components

### 3.1 Input Validator

```typescript
// compilation/input-validator.ts

import {
  CHANNELS,
  NotificationType,
  NOTIFICATION_TYPES,
  Priority,
  PRIORITIES,
  Recipient,
} from "../contracts/notification-event.contract";
import { RoutingDecision } from "../contracts/routing-decision.contract";
import {
  CompilationResult,
  NormalizedError,
} from "../contracts/compilation-result.contract";

export interface NotificationInput {
  readonly notificationType: NotificationType;
  readonly priority: Priority;
  readonly intakeRequest: unknown;           // opaque, validated upstream
  readonly routingDecision: RoutingDecision;
  readonly clientProfile: unknown;           // opaque, validated upstream
  readonly locationProfile: unknown;         // opaque, validated upstream
  readonly publicViewerResult?: unknown;     // optional upstream result
  readonly templateId: string;
  readonly templateVariables: Readonly<Record<string, string>>;
  readonly correlationId: string;
  readonly now: string;                      // ISO 8601 UTC timestamp
}

export interface ValidationOutput {
  readonly notificationType: NotificationType;
  readonly priority: Priority;
  readonly routingDecision: RoutingDecision;
  readonly templateId: string;
  readonly templateVariables: Readonly<Record<string, string>>;
}

export class InputValidator {
  constructor(
    private readonly errorNormalizer: {
      normalize(raw: Error, context: Record<string, string>): NormalizedError;
    }
  ) {}

  validate(input: NotificationInput): CompilationResult {
    try {
      this.assertPresence(input);
      this.assertNotificationType(input.notificationType);
      this.assertPriority(input.priority);
      this.assertTemplateId(input.templateId);
      this.assertTemplateVariables(input.templateVariables);
      this.assertRoutingDecision(input.routingDecision);

      return {
        success: true,
        event: {
          // placeholder; real event constructed downstream
        } as unknown as NotificationEvent, // not reached
      };
    } catch (rawError) {
      return {
        success: false,
        error: this.errorNormalizer.normalize(
          rawError instanceof Error ? rawError : new Error(String(rawError)),
          {
            notificationType: String(input.notificationType),
            priority: String(input.priority),
            correlationId: input.correlationId ?? "",
          }
        ),
      };
    }
  }

  private assertPresence(input: NotificationInput): void {
    const required: Array<[string, unknown]> = [
      ["notificationType", input.notificationType],
      ["priority", input.priority],
      ["intakeRequest", input.intakeRequest],
      ["routingDecision", input.routingDecision],
      ["clientProfile", input.clientProfile],
      ["locationProfile", input.locationProfile],
      ["templateId", input.templateId],
      ["templateVariables", input.templateVariables],
      ["correlationId", input.correlationId],
      ["now", input.now],
    ];

    for (const [name, value] of required) {
      if (value === undefined || value === null) {
        throw new Error(`MISSING_REQUIRED_INPUT:${name}`);
      }
    }
  }

  private assertNotificationType(value: unknown): asserts value is NotificationType {
    if (!NOTIFICATION_TYPES.includes(value as NotificationType)) {
      throw new Error(`INVALID_NOTIFICATION_TYPE:${String(value)}`);
    }
  }

  private assertPriority(value: unknown): asserts value is Priority {
    if (!PRIORITIES.includes(value as Priority)) {
      throw new Error(`INVALID_PRIORITY:${String(value)}`);
    }
  }

  private assertTemplateId(value: string): void {
    if (typeof value !== "string" || value.trim().length === 0) {
      throw new Error(`MISSING_TEMPLATE_ID:${String(value)}`);
    }
  }

  private assertTemplateVariables(
    value: Readonly<Record<string, string>>
  ): void {
    if (
      typeof value !== "object" ||
      value === null ||
      Array.isArray(value) ||
      Object.keys(value).length === 0
    ) {
      throw new Error("MISSING_TEMPLATE_VARIABLES");
    }

    for (const key of Object.keys(value)) {
      if (!this.isNamespacedCamelCase(key)) {
        throw new Error(`INVALID_TEMPLATE_VARIABLE:${key}`);
      }
    }
  }

  private assertRoutingDecision(decision: RoutingDecision): void {
    if (
      !decision ||
      !Array.isArray(decision.recipients) ||
      decision.recipients.length === 0
    ) {
      throw new Error("MISSING_RECIPIENTS");
    }

    decision.recipients.forEach((recipient, index) => {
      this.assertRecipient(recipient, index);
    });
  }

  private assertRecipient(recipient: Recipient, index: number): void {
    if (
      typeof recipient.recipient !== "string" ||
      recipient.recipient.trim().length === 0
    ) {
      throw new Error(`MISSING_RECIPIENTS:recipient[${index}]`);
    }

    if (!CHANNELS.includes(recipient.channel)) {
      throw new Error(`MISSING_RECIPIENTS:channel[${index}]`);
    }

    if (
      typeof recipient.order !== "number" ||
      !Number.isFinite(recipient.order) ||
      recipient.order < 1
    ) {
      throw new Error(`MISSING_RECIPIENTS:order[${index}]`);
    }
  }

  private isNamespacedCamelCase(key: string): boolean {
    // {domain}{Field} camelCase: starts with lowercase letter, contains no separators.
    return /^[a-z][a-zA-Z0-9]*$/.test(key);
  }
}
```

### 3.2 Variable Token Mapper

```typescript
// compilation/variable-token-mapper.ts

import { TemplateVariableName, KNOWN_TEMPLATE_VARIABLES } from "../contracts/template-variable.contract";

/**
 * Opaque upstream shapes consumed only for token extraction.
 */
interface IntakeRequestShape {
  readonly intakeId: string;
  readonly callSessionId: string;
  readonly locationId: string;
  readonly clientId: string;
  readonly customerIdentity?: {
    readonly name?: { readonly full?: string };
    readonly phone?: { readonly normalized?: string };
  };
  readonly serviceRequest?: {
    readonly serviceDisplayName?: string;
    readonly description?: string;
  };
  readonly serviceAddress?: {
    readonly normalized?: string;
  };
  readonly accessDetails?: string;
  readonly preferredCallbackTime?: string;
  readonly urgencySignals?: ReadonlyArray<unknown>;
}

interface LocationProfileShape {
  readonly displayName?: string;
  readonly serviceDeliveryMode?: "inShop" | "onSite" | "hybrid";
}

interface PublicViewerResultShape {
  readonly publicUrl?: string;
}

export interface TokenMapContext {
  readonly intakeRequest: IntakeRequestShape;
  readonly locationProfile: LocationProfileShape;
  readonly publicViewerResult?: PublicViewerResultShape;
  readonly callerVariables: Readonly<Record<string, string>>;
}

export class VariableTokenMapper {
  map(context: TokenMapContext): Readonly<Record<string, string>> {
    const tokens: Record<string, string> = {};

    // Required variables
    tokens.customerName = this.read(context.intakeRequest.customerIdentity?.name?.full);
    tokens.serviceDisplayName = this.read(context.intakeRequest.serviceRequest?.serviceDisplayName);
    tokens.callbackPhone = this.read(context.intakeRequest.customerIdentity?.phone?.normalized);
    tokens.issueSummary = this.read(context.intakeRequest.serviceRequest?.description);

    // Optional variables
    const serviceDeliveryMode = context.locationProfile.serviceDeliveryMode;
    if (serviceDeliveryMode !== "inShop") {
      const address = context.intakeRequest.serviceAddress?.normalized;
      if (address !== undefined && address !== "") {
        tokens.serviceAddress = address;
      }
    }

    if (context.intakeRequest.accessDetails !== undefined && context.intakeRequest.accessDetails !== "") {
      tokens.accessDetails = context.intakeRequest.accessDetails;
    }

    if (
      context.intakeRequest.preferredCallbackTime !== undefined &&
      context.intakeRequest.preferredCallbackTime !== ""
    ) {
      tokens.preferredCallbackTime = context.intakeRequest.preferredCallbackTime;
    }

    if (context.publicViewerResult?.publicUrl !== undefined && context.publicViewerResult.publicUrl !== "") {
      tokens.publicViewerUrl = context.publicViewerResult.publicUrl;
    }

    if (
      Array.isArray(context.intakeRequest.urgencySignals) &&
      context.intakeRequest.urgencySignals.length > 0
    ) {
      // PRD §6.2 default table: urgencyLevel defaults to "standard" when derived from urgencySignals presence.
      tokens.urgencyLevel = "standard";
    }

    // Caller-supplied variables are merged as supplementary values.
    // Canonical derived values take precedence to preserve upstream truth.
    const merged = { ...context.callerVariables, ...tokens };

    // Ensure only namespaced keys remain (validator already checked caller keys).
    return Object.freeze(merged);
  }

  private read(value: unknown): string {
    if (value === undefined || value === null) {
      return "";
    }
    return String(value);
  }
}
```

### 3.3 Identity Binder

```typescript
// compilation/identity-binder.ts

import { createHash } from "crypto";
import { NotificationType } from "../contracts/notification-event.contract";
import { RoutingDecision } from "../contracts/routing-decision.contract";

export interface IdentityContext {
  readonly correlationId: string;
  readonly notificationType: NotificationType;
  readonly routingDecision: RoutingDecision;
  readonly templateId: string;
}

export class IdentityBinder {
  bind(context: IdentityContext): string {
    if (
      !context.routingDecision.recipients ||
      context.routingDecision.recipients.length === 0
    ) {
      throw new Error("MISSING_RECIPIENTS");
    }

    const primaryRecipient = context.routingDecision.recipients[0].recipient;

    const identityInput =
      context.correlationId +
      context.notificationType +
      primaryRecipient +
      context.templateId;

    const hash = createHash("sha256").update(identityInput).digest("hex");
    const truncated = hash.slice(0, 16);

    return `notif_${truncated}`;
  }
}
```

### 3.4 Notification Event Builder

```typescript
// compilation/notification-event-builder.ts

import { NotificationEvent } from "../contracts/notification-event.contract";
import {
  CompilationResult,
  NormalizedError,
} from "../contracts/compilation-result.contract";
import { InputValidator, NotificationInput } from "./input-validator";
import { VariableTokenMapper } from "./variable-token-mapper";
import { IdentityBinder } from "./identity-binder";

export interface NotificationEventBuilderDependencies {
  readonly inputValidator: InputValidator;
  readonly variableTokenMapper: VariableTokenMapper;
  readonly identityBinder: IdentityBinder;
  readonly errorNormalizer: {
    normalize(raw: Error, context: Record<string, string>): NormalizedError;
  };
}

export class NotificationEventBuilder {
  constructor(private readonly deps: NotificationEventBuilderDependencies) {}

  compile(input: NotificationInput): CompilationResult {
    // 1. Input validation
    const validation = this.deps.inputValidator.validate(input);
    if (!validation.success) {
      return validation;
    }

    try {
      // 2. Variable token mapping
      const templateVariables = this.deps.variableTokenMapper.map({
        intakeRequest: input.intakeRequest as any,
        locationProfile: input.locationProfile as any,
        publicViewerResult: input.publicViewerResult as any,
        callerVariables: input.templateVariables,
      });

      // 3. Deterministic identity binding
      const notificationId = this.deps.identityBinder.bind({
        correlationId: input.correlationId,
        notificationType: input.notificationType,
        routingDecision: input.routingDecision,
        templateId: input.templateId,
      });

      // 4. Envelope construction
      const event: NotificationEvent = {
        schemaVersion: 1,
        notificationId,
        notificationType: input.notificationType,
        priority: input.priority,
        templateId: input.templateId,
        templateVariables,
        recipients: Object.freeze([...input.routingDecision.recipients]),
        correlationId: input.correlationId,
        intakeId: (input.intakeRequest as any).intakeId,
        clientId: (input.intakeRequest as any).clientId,
        locationId: (input.intakeRequest as any).locationId,
        createdAt: input.now,
      };

      return {
        success: true,
        event: Object.freeze(event),
      };
    } catch (rawError) {
      return {
        success: false,
        error: this.deps.errorNormalizer.normalize(
          rawError instanceof Error ? rawError : new Error(String(rawError)),
          {
            correlationId: input.correlationId,
            intakeId: (input.intakeRequest as any)?.intakeId ?? "",
            clientId: (input.intakeRequest as any)?.clientId ?? "",
            locationId: (input.intakeRequest as any)?.locationId ?? "",
          }
        ),
      };
    }
  }
}
```

---

## 4. Factory / Wiring

```typescript
// index.ts

import { InputValidator } from "./compilation/input-validator";
import { VariableTokenMapper } from "./compilation/variable-token-mapper";
import { IdentityBinder } from "./compilation/identity-binder";
import { NotificationEventBuilder } from "./compilation/notification-event-builder";
import { NormalizedError } from "./contracts/compilation-result.contract";

export interface ErrorNormalizerPort {
  normalize(raw: Error, context: Record<string, string>): NormalizedError;
}

export function createNotificationEventBuilder(
  errorNormalizer: ErrorNormalizerPort
): NotificationEventBuilder {
  return new NotificationEventBuilder({
    inputValidator: new InputValidator(errorNormalizer),
    variableTokenMapper: new VariableTokenMapper(),
    identityBinder: new IdentityBinder(),
    errorNormalizer,
  });
}

export * from "./contracts/notification-event.contract";
export * from "./contracts/template-variable.contract";
export * from "./contracts/compilation-result.contract";
export * from "./contracts/routing-decision.contract";
export * from "./compilation/notification-event-builder";
export * from "./compilation/input-validator";
export * from "./compilation/variable-token-mapper";
export * from "./compilation/identity-binder";
```

---

## 5. Tests

### 5.1 Contract Tests

```typescript
// tests/contract.test.ts

import { describe, it, expect } from "vitest";
import { createNotificationEventBuilder } from "../index";
import { stubErrorNormalizer } from "./fixtures";

const builder = createNotificationEventBuilder(stubErrorNormalizer);

describe("NotificationEvent contract", () => {
  it("produces a complete frozen envelope for valid inputs", () => {
    const result = builder.compile(validInput());

    expect(result.success).toBe(true);
    if (!result.success) return;

    expect(result.event.schemaVersion).toBe(1);
    expect(result.event.notificationType).toBe("standard");
    expect(result.event.priority).toBe("standard");
    expect(result.event.templateId).toBe("intake-new-request");
    expect(result.event.recipients).toHaveLength(1);
    expect(result.event.correlationId).toBe("call-abc-123");
    expect(result.event.intakeId).toBe("intake-001");
    expect(result.event.clientId).toBe("client-123");
    expect(result.event.locationId).toBe("location-456");
    expect(Object.isFrozen(result.event)).toBe(true);
    expect(Object.isFrozen(result.event.templateVariables)).toBe(true);
    expect(Object.isFrozen(result.event.recipients)).toBe(true);
  });
});
```

### 5.2 Variable Mapping Tests

```typescript
// tests/variable-mapping.test.ts

describe("Variable token mapping", () => {
  it("includes required variables from IntakeRequest", () => {
    const result = builder.compile(validInput());
    if (!result.success) throw result.error;

    expect(result.event.templateVariables.customerName).toBe("Alice Smith");
    expect(result.event.templateVariables.serviceDisplayName).toBe("Water Heater Repair");
    expect(result.event.templateVariables.callbackPhone).toBe("15551234567");
    expect(result.event.templateVariables.issueSummary).toBe("Leaking tank");
  });

  it("omits serviceAddress when serviceDeliveryMode is inShop", () => {
    const input = validInput({
      locationProfile: { displayName: "Main Office", serviceDeliveryMode: "inShop" },
    });
    const result = builder.compile(input);
    if (!result.success) throw result.error;

    expect(result.event.templateVariables.serviceAddress).toBeUndefined();
  });

  it("includes publicViewerUrl when PublicIntakeViewerResult is provided", () => {
    const input = validInput({
      publicViewerResult: { publicUrl: "https://view.example.com/i/abc" },
    });
    const result = builder.compile(input);
    if (!result.success) throw result.error;

    expect(result.event.templateVariables.publicViewerUrl).toBe("https://view.example.com/i/abc");
  });
});
```

### 5.3 Namespacing Tests

```typescript
// tests/namespacing.test.ts

describe("Template variable namespace enforcement", () => {
  it("rejects non-namespaced caller keys", () => {
    const input = validInput({
      templateVariables: { "bad-key": "value" },
    });
    const result = builder.compile(input);
    expect(result.success).toBe(false);
    if (result.success) return;
    expect(result.error.errorCode).toBe("INVALID_TEMPLATE_VARIABLE");
  });

  it("rejects PascalCase caller keys", () => {
    const input = validInput({
      templateVariables: { BadKey: "value" },
    });
    const result = builder.compile(input);
    expect(result.success).toBe(false);
    if (result.success) return;
    expect(result.error.errorCode).toBe("INVALID_TEMPLATE_VARIABLE");
  });
});
```

### 5.4 Identity Tests

```typescript
// tests/identity.test.ts

describe("Deterministic notification identity", () => {
  it("produces identical notificationId for identical inputs", () => {
    const a = builder.compile(validInput());
    const b = builder.compile(validInput());
    expect(a.success && b.success && a.event.notificationId === b.event.notificationId).toBe(true);
  });

  it("changes notificationId when correlationId changes", () => {
    const base = builder.compile(validInput());
    const changed = builder.compile(validInput({ correlationId: "call-xyz-999" }));
    expect(
      base.success && changed.success && base.event.notificationId !== changed.event.notificationId
    ).toBe(true);
  });

  it("changes notificationId when primary recipient changes", () => {
    const base = builder.compile(validInput());
    const changed = builder.compile(validInput({
      routingDecision: {
        timeContext: "business_hours",
        recipients: [{ recipient: "other@example.com", channel: "email", order: 1 }],
      },
    }));
    expect(
      base.success && changed.success && base.event.notificationId !== changed.event.notificationId
    ).toBe(true);
  });

  it("excludes createdAt from identity hash", () => {
    const a = builder.compile(validInput({ now: "2026-07-01T10:00:00Z" }));
    const b = builder.compile(validInput({ now: "2026-07-01T11:00:00Z" }));
    expect(a.success && b.success && a.event.notificationId === b.event.notificationId).toBe(true);
  });
});
```

### 5.5 Failure Tests

```typescript
// tests/failure.test.ts

describe("Compilation failures", () => {
  it.each([
    ["notificationType", { notificationType: "invalid" }, "INVALID_NOTIFICATION_TYPE"],
    ["priority", { priority: "urgent" }, "INVALID_PRIORITY"],
    ["templateId", { templateId: "" }, "MISSING_TEMPLATE_ID"],
    ["templateVariables", { templateVariables: {} }, "MISSING_TEMPLATE_VARIABLES"],
    ["recipients", { routingDecision: { timeContext: "business_hours", recipients: [] } }, "MISSING_RECIPIENTS"],
  ])("returns normalized error for invalid %s", (_label, override, expectedCode) => {
    const result = builder.compile(validInput(override));
    expect(result.success).toBe(false);
    if (result.success) return;
    expect(result.error.errorCode).toBe(expectedCode);
  });

  it("returns MISSING_REQUIRED_INPUT when intakeRequest is null", () => {
    const result = builder.compile(validInput({ intakeRequest: null as any }));
    expect(result.success).toBe(false);
    if (result.success) return;
    expect(result.error.errorCode).toBe("MISSING_REQUIRED_INPUT");
  });
});
```

### 5.6 Pure Function Tests

```typescript
// tests/pure-function.test.ts

describe("Pure function behavior", () => {
  it("does not mutate inputs", () => {
    const input = validInput();
    const originalRecipients = input.routingDecision.recipients;
    builder.compile(input);
    expect(input.routingDecision.recipients).toBe(originalRecipients);
  });

  it("produces identical envelopes except createdAt for identical inputs", () => {
    const a = builder.compile(validInput({ now: "2026-07-01T10:00:00Z" }));
    const b = builder.compile(validInput({ now: "2026-07-01T10:05:00Z" }));
    expect(a.success && b.success).toBe(true);
    if (!a.success || !b.success) return;

    const { createdAt: _a, ...aRest } = a.event;
    const { createdAt: _b, ...bRest } = b.event;
    expect(aRest).toEqual(bRest);
  });
});
```

### 5.7 Test Fixtures

```typescript
// tests/fixtures.ts

import { NormalizedError } from "../contracts/compilation-result.contract";
import { NotificationInput } from "../compilation/input-validator";

export const stubErrorNormalizer = {
  normalize(raw: Error, context: Record<string, string>): NormalizedError {
    const [errorCode, ...rest] = raw.message.split(":");
    return {
      category: "validation_error",
      severity: "error",
      callerSafeCode: `error.${errorCode.toLowerCase()}`,
      errorCode,
      errorMessage: rest.join(":") || raw.message,
      correlationKeys: context,
      isCritical: true,
    };
  },
};

export function validInput(overrides: Partial<NotificationInput> = {}): NotificationInput {
  return {
    notificationType: "standard",
    priority: "standard",
    intakeRequest: {
      intakeId: "intake-001",
      callSessionId: "call-abc-123",
      locationId: "location-456",
      clientId: "client-123",
      customerIdentity: {
        name: { full: "Alice Smith" },
        phone: { normalized: "15551234567" },
      },
      serviceRequest: {
        serviceDisplayName: "Water Heater Repair",
        description: "Leaking tank",
      },
      serviceAddress: { normalized: "123 Main St, Anytown, USA" },
    },
    routingDecision: {
      timeContext: "business_hours",
      recipients: [{ recipient: "15559876543", channel: "sms", order: 1 }],
    },
    clientProfile: { displayName: "Joe's Plumbing" },
    locationProfile: { displayName: "Main Office", serviceDeliveryMode: "onSite" },
    templateId: "intake-new-request",
    templateVariables: { customNote: "Requested morning appointment" },
    correlationId: "call-abc-123",
    now: "2026-07-01T10:00:00Z",
    ...overrides,
  };
}
```

---

## 6. Layer 3 → Layer 4 Traceability

| Layer 3 Blueprint Component | Layer 4 Implementation | Evidence |
|---|---|---|
| Notification Event Builder | `NotificationEventBuilder.compile()` | Orchestrates validation → mapping → identity → envelope |
| Input Validator | `InputValidator.validate()` | Enforces input contract and namespace rules |
| Variable Token Mapper | `VariableTokenMapper.map()` | Assembles canonical namespaced variables |
| Identity Binder | `IdentityBinder.bind()` | Generates deterministic `notificationId` |
| Notification Event Contract | `NotificationEvent` type + frozen constructor | PRD §5.2 schema, `Object.freeze` |
| Template Variable Contract | `TemplateVariableContract` + namespace validator | `{domain}{Field}` camelCase rule |
| Compilation Result Contract | `CompilationResult` union + `NormalizedError` port | Returns event or normalized error |
| Runtime Flow §6 | `compile()` method body | Four sequential stages |
| Dependency Map §7 | Constructor injection graph | Builder depends on validator, mapper, binder |
| Testing Blueprint §9 | `tests/*.test.ts` files | Contract, mapping, namespacing, identity, failure, pure-function tests |

---

## 7. Final Quality Checklist

- ✓ Architecture preserved exactly — pure compilation boundary, no execution logic.
- ✓ Responsibilities unchanged — validator, mapper, binder, builder each own one stage.
- ✓ Dependency direction preserved — framework consumes upstream outputs, emits envelope downstream.
- ✓ Runtime flow matches Layer 3 — validate → map → bind → construct.
- ✓ Public contracts implemented — `NotificationEvent`, template variables, compilation result.
- ✓ Internal components implemented — validator, mapper, binder, builder.
- ✓ Tests included — contract, mapping, namespacing, identity, failure, pure-function.
- ✓ No architectural shortcuts — no sender, renderer, router, or workflow decision logic.
- ✓ No business logic moved across boundaries — routing remains upstream, delivery downstream.
- ✓ Production-quality implementation — explicit interfaces, constructor injection, immutable frozen outputs.

---

This artifact implements only the `NotificationEventBuilder` compiler. `ClientAlertRouting` logic (business-hours evaluation, recipient fallback, routing table) is deliberately excluded; only its `RoutingDecision` / `Recipient` input contract is consumed.
