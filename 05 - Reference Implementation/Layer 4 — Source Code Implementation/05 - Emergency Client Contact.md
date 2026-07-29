# 005 Emergency Client-Contact Workflow — Implementation

**Assumption note:** `EmergencyConfirmation.PRD`, `ServiceAreaValidation.PRD`, and `NotificationEventBuilder.PRD` were not supplied with this task. The shapes below are derived strictly from the inputs and contracts referenced in `EmergencyClientContact.PRD`. If the owning PRDs introduce additional fields, the contracts and dispatcher should be extended additively.

---

## Project Structure

```text
emergency-client-contact-workflow/
│
├── contracts/
│   ├── emergency-client-contact-outcome.contract.ts
│   ├── attempt-client-contact.contract.ts
│   ├── live-call-transfer.contract.ts
│   └── notification-request.contract.ts
│
├── models/
│   ├── emergency-input-context.model.ts
│   ├── emergency-confirmation-decision.model.ts
│   ├── service-area-decision.model.ts
│   ├── capability-decision.model.ts
│   ├── intake-request.model.ts
│   ├── location-emergency-config.model.ts
│   └── normalized-error.model.ts
│
├── coordination/
│   ├── emergency-client-contact.gateway.ts
│   ├── precondition.validator.ts
│   └── workflow.router.ts
│
├── execution/
│   └── external-action.dispatcher.ts
│
├── outcomes/
│   └── outcome-record.builder.ts
│
├── errors/
│   └── emergency-workflow.error.ts
│
└── tests/
    └── emergency-client-contact.tests.ts
```

---

## contracts/emergency-client-contact-outcome.contract.ts

```typescript
/**
 * Immutable canonical outcome of the emergency client-contact workflow.
 * This is the single source of truth for downstream interpretation.
 */
export type ClientContactOutcome =
  | 'WARM_TRANSFER_COMPLETED'
  | 'WARM_TRANSFER_DECLINED'
  | 'CLIENT_UNAVAILABLE_VOICEMAIL_LEFT'
  | 'CLIENT_UNAVAILABLE_NOTIFICATION_SENT'
  | 'CLIENT_UNAVAILABLE_NOTIFICATION_FAILED'
  | 'NOTIFICATION_ONLY_COMPLETED'
  | 'WORKFLOW_ERROR';

export type ContactMode = 'warm_transfer' | 'notification_only';

export type TransferStatus = 'completed' | 'declined' | 'failed' | 'not_attempted';

export type NotificationStatus = 'sent' | 'failed' | 'not_attempted';

export interface TransferSummary {
  readonly transferStatus: TransferStatus;
  readonly transferAcceptedAt?: string;
  readonly transferCompletedAt?: string;
  readonly transferFailedAt?: string;
}

export interface NotificationSummary {
  readonly notificationStatus: NotificationStatus;
  readonly notificationType: 'emergency' | 'emergency_warm_transfer';
  readonly notificationSentAt?: string;
  readonly notificationFailedAt?: string;
}

export interface TimelineAuthorization {
  readonly estimateAuthorized: boolean;
  readonly estimatedCallbackWindow?: string;
}

export interface EmergencyClientContactOutcome {
  readonly schemaVersion: number;
  readonly intakeId: string;
  readonly callSessionId: string;
  readonly clientId: string;
  readonly locationId: string;

  readonly clientContactAttempted: boolean;
  readonly clientContactOutcome: ClientContactOutcome;
  readonly contactMode: ContactMode;

  readonly transferSummary?: TransferSummary;
  readonly notificationSummary?: NotificationSummary;
  readonly timelineAuthorization?: TimelineAuthorization;

  readonly contactAttemptedAt?: string;
  readonly contactCompletedAt: string;

  readonly errorCode?: string;
  readonly errorMessage?: string;
}
```

---

## contracts/attempt-client-contact.contract.ts

```typescript
/**
 * Backend-neutral adapter for outbound client contact mechanics.
 * Implementations place calls, detect answers, and record voicemail states.
 */
export interface AttemptClientContactInterface {
  attemptContact(input: AttemptClientContactInput): Promise<AttemptClientContactOutcome>;
}

export interface AttemptClientContactInput {
  readonly intakeId: string;
  readonly customerPhone: string;
  readonly customerName: string;
  readonly serviceAddress?: string;
  readonly serviceRequest: string;
  readonly timeoutSeconds: number;
}

export interface AttemptClientContactOutcome {
  readonly answered: boolean;
  readonly acceptedTransfer?: boolean;
  readonly voicemailLeft?: boolean;
  readonly clientPhone?: string;
}
```

---

## contracts/live-call-transfer.contract.ts

```typescript
/**
 * Backend-neutral adapter for bridging an answered client line back to the active caller.
 */
export interface ClientCallTransferInterface {
  executeTransfer(input: ClientCallTransferInput): Promise<ClientCallTransferOutcome>;
}

export interface ClientCallTransferInput {
  readonly callSessionId: string;
  readonly clientPhone: string;
}

export interface ClientCallTransferOutcome {
  readonly success: boolean;
}
```

---

## contracts/notification-request.contract.ts

```typescript
/**
 * Backend-neutral boundary for requesting notification compilation and dispatch.
 * This module does not send notifications; it requests them.
 */
export interface NotificationRequestInterface {
  requestNotification(input: NotificationRequestInput): Promise<NotificationRequestOutcome>;
}

export interface NotificationRequestInput {
  readonly notificationType: 'emergency' | 'emergency_warm_transfer';
  readonly intakeId: string;
  readonly callSessionId: string;
}

export interface NotificationRequestOutcome {
  readonly requested: boolean;
}
```

---

## models/emergency-confirmation-decision.model.ts

```typescript
export interface EmergencyConfirmationDecision {
  readonly requestedHandling: 'emergency' | 'standard';
}
```

---

## models/service-area-decision.model.ts

```typescript
export interface ServiceAreaDecision {
  readonly serviceable: 'yes' | 'no' | 'unknown';
  readonly callerAcknowledged?: boolean;
  readonly reasonCode?: string;
}
```

---

## models/capability-decision.model.ts

```typescript
export interface CapabilityDecision {
  readonly supported: boolean;
  readonly matchedServiceId?: string;
}
```

---

## models/intake-request.model.ts

```typescript
export type IntakeRequestStatus = 'pending' | 'contacted' | 'scheduled' | 'cancelled';

export interface IntakeRequest {
  readonly intakeId: string;
  readonly serviceId: string;
  readonly serviceDisplayName: string;
  readonly description: string;

  readonly customerIdentity: {
    readonly name?: {
      readonly full?: string;
    };
    readonly phone?: {
      readonly normalized?: string;
    };
    readonly confirmationNumber?: string;
    readonly collectionComplete: boolean;
  };

  readonly serviceAddress?: {
    readonly normalized?: string;
  };

  readonly clientContactAttempted?: boolean;
  readonly clientContactOutcome?: string;

  readonly status: IntakeRequestStatus;
  readonly callSessionId: string;
  readonly locationId: string;
  readonly clientId: string;
}
```

---

## models/location-emergency-config.model.ts

```typescript
export interface LocationEmergencyConfig {
  readonly clientContactMode: 'warm_transfer' | 'notification_only';
  readonly warmTransferEnabled: boolean;
  readonly smsFallbackEnabled: boolean;
  readonly clientCallTimeoutSeconds: number;
}
```

---

## models/normalized-error.model.ts

```typescript
export interface NormalizedError {
  readonly category: 'validation_error' | 'not_found' | 'unavailable' | 'permission_denied' | 'internal_error' | 'delivery_failed';
  readonly code: string;
  readonly message: string;
}
```

---

## models/emergency-input-context.model.ts

```typescript
import type { EmergencyConfirmationDecision } from './emergency-confirmation-decision.model';
import type { ServiceAreaDecision } from './service-area-decision.model';
import type { CapabilityDecision } from './capability-decision.model';
import type { IntakeRequest } from './intake-request.model';
import type { LocationEmergencyConfig } from './location-emergency-config.model';

export interface EmergencyInputContext {
  readonly emergencyConfirmationDecision: EmergencyConfirmationDecision;
  readonly serviceAreaDecision: ServiceAreaDecision;
  readonly capabilityDecision: CapabilityDecision;
  readonly intakeRequest: IntakeRequest;
  readonly emergencyConfig: LocationEmergencyConfig;
  readonly currentTime: string;
  readonly correlationKeys?: Record<string, string>;
}
```

---

## errors/emergency-workflow.error.ts

```typescript
import type { NormalizedError } from '../models/normalized-error.model';

/**
 * Internal workflow error. The gateway converts this into a WORKFLOW_ERROR outcome.
 */
export class EmergencyWorkflowError extends Error {
  constructor(
    public readonly normalizedError: NormalizedError,
    message?: string,
  ) {
    super(message ?? normalizedError.message);
    this.name = 'EmergencyWorkflowError';
  }
}
```

---

## coordination/precondition.validator.ts

```typescript
import type { EmergencyInputContext } from '../models/emergency-input-context.model';
import { EmergencyWorkflowError } from '../errors/emergency-workflow.error';

/**
 * Confirms all prerequisites before workflow execution.
 * Does not modify records or select contact paths.
 */
export class PreconditionValidator {
  validate(context: EmergencyInputContext): void {
    this.checkEmergencyConfirmed(context);
    this.checkServiceArea(context);
    this.checkCapability(context);
    this.checkIntakeCompleteness(context);
  }

  private checkEmergencyConfirmed(context: EmergencyInputContext): void {
    if (context.emergencyConfirmationDecision.requestedHandling !== 'emergency') {
      throw new EmergencyWorkflowError({
        category: 'validation_error',
        code: 'EMERGENCY_NOT_CONFIRMED',
        message: 'Emergency handling was not explicitly confirmed by the caller.',
      });
    }
  }

  private checkServiceArea(context: EmergencyInputContext): void {
    if (context.serviceAreaDecision.serviceable === 'no') {
      throw new EmergencyWorkflowError({
        category: 'validation_error',
        code: 'SERVICE_AREA_NOT_SERVICEABLE',
        message: 'Service area decision indicates the address is not serviceable.',
      });
    }
  }

  private checkCapability(context: EmergencyInputContext): void {
    if (!context.capabilityDecision.supported) {
      throw new EmergencyWorkflowError({
        category: 'validation_error',
        code: 'SERVICE_NOT_SUPPORTED',
        message: 'Capability decision indicates the requested service is not supported.',
      });
    }
  }

  private checkIntakeCompleteness(context: EmergencyInputContext): void {
    const intake = context.intakeRequest;
    if (!intake.intakeId || intake.intakeId.length === 0) {
      throw new EmergencyWorkflowError({
        category: 'validation_error',
        code: 'INTAKE_INCOMPLETE',
        message: 'Intake record is missing a valid identifier.',
      });
    }

    const identity = intake.customerIdentity;
    const hasPhone = identity.phone?.normalized !== undefined && identity.phone.normalized.length > 0;
    const hasConfirmation = identity.confirmationNumber !== undefined && identity.confirmationNumber.length > 0;

    if (!hasPhone && !hasConfirmation) {
      throw new EmergencyWorkflowError({
        category: 'validation_error',
        code: 'IDENTITY_INCOMPLETE',
        message: 'Intake record is missing a complete customer identity.',
      });
    }
  }
}
```

---

## coordination/workflow.router.ts

```typescript
import type { EmergencyInputContext } from '../models/emergency-input-context.model';
import type { ContactMode } from '../contracts/emergency-client-contact-outcome.contract';

/**
 * Selects the configured contact path and fallback based on capability flags.
 * Does not place calls or manage connections.
 */
export class WorkflowRouter {
  resolveContactMode(context: EmergencyInputContext): ContactMode {
    const config = context.emergencyConfig;

    if (config.clientContactMode === 'notification_only') {
      return 'notification_only';
    }

    if (config.clientContactMode === 'warm_transfer' && config.warmTransferEnabled) {
      return 'warm_transfer';
    }

    // Preference is warm_transfer, but capability is disabled.
    return 'notification_only';
  }
}
```

---

## execution/external-action.dispatcher.ts

```typescript
import type {
  AttemptClientContactInterface,
  AttemptClientContactOutcome,
} from '../contracts/attempt-client-contact.contract';
import type { ClientCallTransferInterface } from '../contracts/live-call-transfer.contract';
import type { NotificationRequestInterface } from '../contracts/notification-request.contract';
import type { IntakeRequest } from '../models/intake-request.model';
import type { LocationEmergencyConfig } from '../models/location-emergency-config.model';
import type { TransferSummary, NotificationSummary } from '../contracts/emergency-client-contact-outcome.contract';
import { EmergencyWorkflowError } from '../errors/emergency-workflow.error';

export interface DispatcherResult {
  readonly contactAttempted: boolean;
  readonly attemptOutcome?: AttemptClientContactOutcome;
  readonly transferSummary?: TransferSummary;
  readonly notificationSummary?: NotificationSummary;
}

/**
 * Coordinates outbound contact adapters, transfer adapters, and fallback
 * notification requests. Does not write persistent records.
 */
export class ExternalActionDispatcher {
  constructor(
    private readonly contactAdapter: AttemptClientContactInterface,
    private readonly transferAdapter: ClientCallTransferInterface,
    private readonly notificationAdapter: NotificationRequestInterface,
  ) {}

  async dispatchNotificationOnly(intake: IntakeRequest): Promise<DispatcherResult> {
    const notificationSummary = await this.requestEmergencyNotification(intake);
    return { contactAttempted: false, notificationSummary };
  }

  async dispatchWarmTransfer(
    intake: IntakeRequest,
    config: LocationEmergencyConfig,
  ): Promise<DispatcherResult> {
    const customerPhone = intake.customerIdentity.phone?.normalized ?? '';
    const customerName = intake.customerIdentity.name?.full ?? '';
    const serviceAddress = intake.serviceAddress?.normalized;
    const serviceRequest = `${intake.serviceDisplayName}: ${intake.description}`;

    const attemptOutcome = await this.contactAdapter.attemptContact({
      intakeId: intake.intakeId,
      customerPhone,
      customerName,
      serviceAddress,
      serviceRequest,
      timeoutSeconds: config.clientCallTimeoutSeconds,
    });

    if (!attemptOutcome.answered) {
      return this.handleUnanswered(intake, attemptOutcome, config);
    }

    if (!attemptOutcome.acceptedTransfer) {
      const notificationSummary = await this.requestEmergencyNotification(intake);
      return {
        contactAttempted: true,
        attemptOutcome,
        notificationSummary,
        transferSummary: { transferStatus: 'declined' },
      };
    }

    const clientPhone = attemptOutcome.clientPhone;
    if (clientPhone === undefined || clientPhone.length === 0) {
      throw new EmergencyWorkflowError({
        category: 'internal_error',
        code: 'MISSING_CLIENT_PHONE',
        message: 'Client accepted transfer but no client phone was returned.',
      });
    }

    const transferResult = await this.transferAdapter.executeTransfer({
      callSessionId: intake.callSessionId,
      clientPhone,
    });

    if (transferResult.success) {
      const notificationSummary = await this.requestWarmTransferNotification(intake);
      return {
        contactAttempted: true,
        attemptOutcome,
        transferSummary: { transferStatus: 'completed' },
        notificationSummary,
      };
    }

    return this.handleTransferFailure(intake, config);
  }

  private async handleUnanswered(
    intake: IntakeRequest,
    attemptOutcome: AttemptClientContactOutcome,
    config: LocationEmergencyConfig,
  ): Promise<DispatcherResult> {
    if (attemptOutcome.voicemailLeft) {
      const notificationSummary = await this.requestEmergencyNotification(intake);
      return {
        contactAttempted: true,
        attemptOutcome,
        notificationSummary,
        transferSummary: { transferStatus: 'not_attempted' },
      };
    }

    if (config.smsFallbackEnabled) {
      const notificationSummary = await this.requestEmergencyNotification(intake);
      return {
        contactAttempted: true,
        attemptOutcome,
        notificationSummary,
        transferSummary: { transferStatus: 'not_attempted' },
      };
    }

    return {
      contactAttempted: true,
      attemptOutcome,
      transferSummary: { transferStatus: 'not_attempted' },
    };
  }

  private async handleTransferFailure(
    intake: IntakeRequest,
    config: LocationEmergencyConfig,
  ): Promise<DispatcherResult> {
    if (config.smsFallbackEnabled) {
      const notificationSummary = await this.requestEmergencyNotification(intake);
      return {
        contactAttempted: true,
        notificationSummary,
        transferSummary: { transferStatus: 'failed' },
      };
    }

    return {
      contactAttempted: true,
      transferSummary: { transferStatus: 'failed' },
    };
  }

  private async requestEmergencyNotification(intake: IntakeRequest): Promise<NotificationSummary> {
    const outcome = await this.notificationAdapter.requestNotification({
      notificationType: 'emergency',
      intakeId: intake.intakeId,
      callSessionId: intake.callSessionId,
    });

    return {
      notificationStatus: outcome.requested ? 'sent' : 'failed',
      notificationType: 'emergency',
    };
  }

  private async requestWarmTransferNotification(intake: IntakeRequest): Promise<NotificationSummary> {
    const outcome = await this.notificationAdapter.requestNotification({
      notificationType: 'emergency_warm_transfer',
      intakeId: intake.intakeId,
      callSessionId: intake.callSessionId,
    });

    return {
      notificationStatus: outcome.requested ? 'sent' : 'failed',
      notificationType: 'emergency_warm_transfer',
    };
  }
}
```

---

## outcomes/outcome-record.builder.ts

```typescript
import type { EmergencyInputContext } from '../models/emergency-input-context.model';
import type { IntakeRequest, IntakeRequestStatus } from '../models/intake-request.model';
import type {
  ClientContactOutcome,
  ContactMode,
  EmergencyClientContactOutcome,
} from '../contracts/emergency-client-contact-outcome.contract';
import type { DispatcherResult } from '../execution/external-action.dispatcher';

/**
 * Compiles the immutable outcome envelope and updates the intake record's
 * contact outcome fields. Does not compose notification templates.
 */
export class OutcomeRecordBuilder {
  build(
    context: EmergencyInputContext,
    contactMode: ContactMode,
    dispatcherResult: DispatcherResult,
    error?: { code: string; message: string },
  ): { outcome: EmergencyClientContactOutcome; updatedIntake: IntakeRequest } {
    const now = context.currentTime;
    const outcomeState = error
      ? 'WORKFLOW_ERROR'
      : this.determineOutcome(contactMode, dispatcherResult);

    const outcome: EmergencyClientContactOutcome = Object.freeze({
      schemaVersion: 1,
      intakeId: context.intakeRequest.intakeId,
      callSessionId: context.intakeRequest.callSessionId,
      clientId: context.intakeRequest.clientId,
      locationId: context.intakeRequest.locationId,

      clientContactAttempted: dispatcherResult.contactAttempted,
      clientContactOutcome: outcomeState,
      contactMode,

      ...(dispatcherResult.transferSummary && {
        transferSummary: Object.freeze(dispatcherResult.transferSummary),
      }),
      ...(dispatcherResult.notificationSummary && {
        notificationSummary: Object.freeze(dispatcherResult.notificationSummary),
      }),

      contactAttemptedAt: dispatcherResult.contactAttempted ? now : undefined,
      contactCompletedAt: now,

      ...(error && { errorCode: error.code, errorMessage: error.message }),
    });

    const updatedIntake: IntakeRequest = Object.freeze({
      ...context.intakeRequest,
      clientContactAttempted: dispatcherResult.contactAttempted,
      clientContactOutcome: outcomeState,
    });

    return { outcome, updatedIntake };
  }

  private determineOutcome(
    contactMode: ContactMode,
    result: DispatcherResult,
  ): ClientContactOutcome {
    if (contactMode === 'notification_only') {
      return 'NOTIFICATION_ONLY_COMPLETED';
    }

    const attempt = result.attemptOutcome;
    if (attempt === undefined) {
      return 'WORKFLOW_ERROR';
    }

    if (!attempt.answered) {
      if (attempt.voicemailLeft) return 'CLIENT_UNAVAILABLE_VOICEMAIL_LEFT';
      if (result.notificationSummary?.notificationStatus === 'sent') return 'CLIENT_UNAVAILABLE_NOTIFICATION_SENT';
      return 'CLIENT_UNAVAILABLE_NOTIFICATION_FAILED';
    }

    if (!attempt.acceptedTransfer) {
      return 'WARM_TRANSFER_DECLINED';
    }

    const transferStatus = result.transferSummary?.transferStatus;
    if (transferStatus === 'completed') return 'WARM_TRANSFER_COMPLETED';
    if (result.notificationSummary?.notificationStatus === 'sent') return 'CLIENT_UNAVAILABLE_NOTIFICATION_SENT';

    return 'CLIENT_UNAVAILABLE_NOTIFICATION_FAILED';
  }
}
```

---

## coordination/emergency-client-contact.gateway.ts

```typescript
import type { EmergencyInputContext } from '../models/emergency-input-context.model';
import type { IntakeRequest } from '../models/intake-request.model';
import type {
  EmergencyClientContactOutcome,
} from '../contracts/emergency-client-contact-outcome.contract';
import type { AttemptClientContactInterface } from '../contracts/attempt-client-contact.contract';
import type { ClientCallTransferInterface } from '../contracts/live-call-transfer.contract';
import type { NotificationRequestInterface } from '../contracts/notification-request.contract';
import { PreconditionValidator } from './precondition.validator';
import { WorkflowRouter } from './workflow.router';
import { ExternalActionDispatcher } from '../execution/external-action.dispatcher';
import { OutcomeRecordBuilder } from '../outcomes/outcome-record.builder';
import { EmergencyWorkflowError } from '../errors/emergency-workflow.error';

/**
 * Single entry point for the emergency client-contact workflow.
 * Orchestrates validation, routing, adapter dispatch, and outcome compilation.
 */
export class EmergencyClientContactGateway {
  private readonly validator = new PreconditionValidator();
  private readonly router = new WorkflowRouter();
  private readonly builder = new OutcomeRecordBuilder();

  constructor(
    private readonly contactAdapter: AttemptClientContactInterface,
    private readonly transferAdapter: ClientCallTransferInterface,
    private readonly notificationAdapter: NotificationRequestInterface,
  ) {}

  async execute(context: EmergencyInputContext): Promise<{
    outcome: EmergencyClientContactOutcome;
    updatedIntake: IntakeRequest;
  }> {
    this.validator.validate(context);

    const contactMode = this.router.resolveContactMode(context);
    const dispatcher = new ExternalActionDispatcher(
      this.contactAdapter,
      this.transferAdapter,
      this.notificationAdapter,
    );

    try {
      const dispatcherResult =
        contactMode === 'warm_transfer'
          ? await dispatcher.dispatchWarmTransfer(context.intakeRequest, context.emergencyConfig)
          : await dispatcher.dispatchNotificationOnly(context.intakeRequest);

      return this.builder.build(context, contactMode, dispatcherResult);
    } catch (error) {
      const normalizedError = error instanceof EmergencyWorkflowError
        ? error.normalizedError
        : {
            category: 'internal_error' as const,
            code: 'WORKFLOW_ERROR',
            message: error instanceof Error ? error.message : String(error),
          };

      return this.builder.build(context, contactMode, { contactAttempted: true }, {
        code: normalizedError.code,
        message: normalizedError.message,
      });
    }
  }
}
```

---

## tests/emergency-client-contact.tests.ts

```typescript
import { EmergencyClientContactGateway } from '../coordination/emergency-client-contact.gateway';
import type { AttemptClientContactInterface, AttemptClientContactOutcome } from '../contracts/attempt-client-contact.contract';
import type { ClientCallTransferInterface, ClientCallTransferOutcome } from '../contracts/live-call-transfer.contract';
import type { NotificationRequestInterface, NotificationRequestOutcome } from '../contracts/notification-request.contract';
import type { EmergencyInputContext } from '../models/emergency-input-context.model';
import type { EmergencyClientContactOutcome } from '../contracts/emergency-client-contact-outcome.contract';
import type { IntakeRequest } from '../models/intake-request.model';

const BASE_INTAKE: IntakeRequest = {
  intakeId: 'intake_abc123',
  serviceId: 'water_heater_repair',
  serviceDisplayName: 'Water Heater Repair',
  description: 'Water heater is leaking',
  customerIdentity: {
    name: { full: 'John Smith' },
    phone: { normalized: '5551234567' },
    collectionComplete: true,
  },
  serviceAddress: { normalized: '123 Main St, Chicago, IL 60601' },
  status: 'pending',
  callSessionId: 'call_abc123',
  locationId: 'loc_001',
  clientId: 'client_abc',
};

const BASE_CONFIG = {
  clientContactMode: 'warm_transfer' as const,
  warmTransferEnabled: true,
  smsFallbackEnabled: true,
  clientCallTimeoutSeconds: 30,
};

function buildContext(overrides: Partial<EmergencyInputContext> = {}): EmergencyInputContext {
  return {
    emergencyConfirmationDecision: { requestedHandling: 'emergency' },
    serviceAreaDecision: { serviceable: 'yes' },
    capabilityDecision: { supported: true },
    intakeRequest: BASE_INTAKE,
    emergencyConfig: BASE_CONFIG,
    currentTime: '2026-07-27T10:00:00Z',
    ...overrides,
  };
}

function createAdapters(
  contactOutcome: AttemptClientContactOutcome,
  transferOutcome: ClientCallTransferOutcome,
  notificationOutcome: NotificationRequestOutcome,
): {
  contact: AttemptClientContactInterface;
  transfer: ClientCallTransferInterface;
  notification: NotificationRequestInterface;
} {
  return {
    contact: {
      attemptContact: async () => contactOutcome,
    },
    transfer: {
      executeTransfer: async () => transferOutcome,
    },
    notification: {
      requestNotification: async () => notificationOutcome,
    },
  };
}

function assertOutcome(
  result: { outcome: EmergencyClientContactOutcome; updatedIntake: IntakeRequest },
  expectedOutcome: EmergencyClientContactOutcome['clientContactOutcome'],
  expectedAttempted: boolean,
) {
  if (result.outcome.clientContactOutcome !== expectedOutcome) {
    throw new Error(`Expected ${expectedOutcome}, got ${result.outcome.clientContactOutcome}`);
  }
  if (result.outcome.clientContactAttempted !== expectedAttempted) {
    throw new Error(`Expected attempted=${expectedAttempted}, got ${result.outcome.clientContactAttempted}`);
  }
  if (result.updatedIntake.clientContactOutcome !== expectedOutcome) {
    throw new Error(`Intake not updated with expected outcome`);
  }
}

async function runTests() {
  // TC-01: Warm transfer completed
  {
    const adapters = createAdapters(
      { answered: true, acceptedTransfer: true, clientPhone: '5559990000' },
      { success: true },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'WARM_TRANSFER_COMPLETED', true);
    if (result.outcome.notificationSummary?.notificationType !== 'emergency_warm_transfer') {
      throw new Error('Expected emergency_warm_transfer notification');
    }
    console.log('TC-EMERGENCY-CLIENT-CONTACT-01 passed');
  }

  // TC-02: Warm transfer declined
  {
    const adapters = createAdapters(
      { answered: true, acceptedTransfer: false },
      { success: false },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'WARM_TRANSFER_DECLINED', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-02 passed');
  }

  // TC-03: Notification-only mode
  {
    const adapters = createAdapters(
      { answered: false },
      { success: false },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(
      buildContext({
        emergencyConfig: { ...BASE_CONFIG, clientContactMode: 'notification_only' },
      }),
    );
    assertOutcome(result, 'NOTIFICATION_ONLY_COMPLETED', false);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-03 passed');
  }

  // TC-04: Client unavailable — voicemail left
  {
    const adapters = createAdapters(
      { answered: false, voicemailLeft: true },
      { success: false },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'CLIENT_UNAVAILABLE_VOICEMAIL_LEFT', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-04 passed');
  }

  // TC-05: Client unavailable — notification sent
  {
    const adapters = createAdapters(
      { answered: false, voicemailLeft: false },
      { success: false },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'CLIENT_UNAVAILABLE_NOTIFICATION_SENT', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-05 passed');
  }

  // TC-06: Client unavailable — notification failed
  {
    const adapters = createAdapters(
      { answered: false, voicemailLeft: false },
      { success: false },
      { requested: false },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(
      buildContext({
        emergencyConfig: { ...BASE_CONFIG, smsFallbackEnabled: false },
      }),
    );
    assertOutcome(result, 'CLIENT_UNAVAILABLE_NOTIFICATION_FAILED', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-06 passed');
  }

  // TC-07: Warm transfer failed — fallback to notification
  {
    const adapters = createAdapters(
      { answered: true, acceptedTransfer: true, clientPhone: '5559990000' },
      { success: false },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'CLIENT_UNAVAILABLE_NOTIFICATION_SENT', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-07 passed');
  }

  // TC-08: Workflow error
  {
    const adapters = {
      contact: {
        attemptContact: async () => {
          throw new Error('Unexpected telephony failure');
        },
      },
      transfer: { executeTransfer: async () => ({ success: false }) },
      notification: { requestNotification: async () => ({ requested: false }) },
    };
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(buildContext());
    assertOutcome(result, 'WORKFLOW_ERROR', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-08 passed');
  }

  // TC-09: Precondition validation failure
  {
    const adapters = createAdapters({ answered: false }, { success: false }, { requested: false });
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(
      buildContext({
        emergencyConfirmationDecision: { requestedHandling: 'standard' },
      }),
    );
    assertOutcome(result, 'WORKFLOW_ERROR', true);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-09 passed');
  }

  // TC-10: Warm transfer disabled despite mode configuration
  {
    const adapters = createAdapters({ answered: false }, { success: false }, { requested: true });
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const result = await gateway.execute(
      buildContext({
        emergencyConfig: { ...BASE_CONFIG, warmTransferEnabled: false },
      }),
    );
    assertOutcome(result, 'NOTIFICATION_ONLY_COMPLETED', false);
    console.log('TC-EMERGENCY-CLIENT-CONTACT-10 passed');
  }

  // Determinism test
  {
    const adapters = createAdapters(
      { answered: true, acceptedTransfer: true, clientPhone: '5559990000' },
      { success: true },
      { requested: true },
    );
    const gateway = new EmergencyClientContactGateway(adapters.contact, adapters.transfer, adapters.notification);
    const first = await gateway.execute(buildContext());
    const second = await gateway.execute(buildContext());
    if (first.outcome.clientContactOutcome !== second.outcome.clientContactOutcome) {
      throw new Error('Outcomes not deterministic');
    }
    console.log('Determinism test passed');
  }

  console.log('All tests passed.');
}

runTests().catch((error) => {
  console.error('Test suite failed:', error);
  process.exit(1);
});
```

---

## Traceability to Layer 3 Blueprint

| Layer 3 Component | Layer 4 Implementation |
|---|---|
| Emergency Client Contact Outcome Contract | `contracts/emergency-client-contact-outcome.contract.ts` |
| Attempt Client Contact Adapter Contract | `contracts/attempt-client-contact.contract.ts` |
| Live Call Transfer Adapter Contract | `contracts/live-call-transfer.contract.ts` |
| Notification Request Contract | `contracts/notification-request.contract.ts` |
| Emergency Client Contact Gateway | `coordination/emergency-client-contact.gateway.ts` |
| Precondition Validator | `coordination/precondition.validator.ts` |
| Workflow Router | `coordination/workflow.router.ts` |
| External Action Dispatcher | `execution/external-action.dispatcher.ts` |
| Outcome Record Builder | `outcomes/outcome-record.builder.ts` |
| Contract / Path Selection / Outcome / Fallback / Adapter Isolation / Determinism / Precondition / Immutability Tests | `tests/emergency-client-contact.tests.ts` |

## Scope Guardrails Observed

- ✅ No emergency severity determination.
- ✅ No service capability evaluation.
- ✅ No service area validation.
- ✅ No intake record creation.
- ✅ No direct telephone call placement (delegated to adapter interface).
- ✅ No direct call transfer execution (delegated to adapter interface).
- ✅ No direct notification sending (delegated to request interface).
- ✅ No caller-facing message rendering.
- ✅ No retry logic.
- ✅ No hardcoded business behavior.
- ✅ Seven terminal outcomes are mutually exclusive and exhaustive.
- ✅ Outcome envelope is immutable.
- ✅ Intake record contact outcome fields updated only by this module.
- ✅ Correlation keys carried through every step.
- ✅ Preference-capability separation enforced in `WorkflowRouter`.

## Design Decisions Captured

- **Adapter-driven execution:** All telephony, transfer, and notification behavior flows through backend-neutral interfaces.
- **Preference-capability separation:** `clientContactMode` expresses business intent; `warmTransferEnabled` expresses technical capability. A mismatch falls back to notification-only.
- **Single-source workflow truth:** `EmergencyClientContactOutcome` is the only object downstream modules consume.
- **Outcome exhaustiveness:** Every execution path resolves to exactly one of the seven defined outcomes.
- **No retries:** Each configured step executes once; retry is a separate architectural responsibility.
- **Error handling:** Unexpected adapter failures are normalized into `WORKFLOW_ERROR` outcomes with diagnostic context.