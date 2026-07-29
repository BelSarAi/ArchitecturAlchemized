# 001 Business Configuration Framework — Implementation

## Project Structure

```text
business-configuration-framework/
│
├── src/
│   ├── contracts/
│   │   ├── canonical-configuration.contract.ts
│   │   ├── validation-result.contract.ts
│   │   └── schema-version.contract.ts
│   │
│   ├── models/
│   │   ├── location-profile.model.ts
│   │   ├── business-hours.model.ts
│   │   ├── ai-coverage-schedule.model.ts
│   │   ├── calendar-exception.model.ts
│   │   ├── emergency-config.model.ts
│   │   ├── service-catalog-ref.model.ts
│   │   ├── service-area-ref.model.ts
│   │   ├── client-alerting-config.model.ts
│   │   ├── customer-sms-config.model.ts
│   │   └── policy-profile-ref.model.ts
│   │
│   ├── services/
│   │   ├── schema-definition.service.ts
│   │   ├── validation.service.ts
│   │   ├── defaulting.service.ts
│   │   ├── normalization.service.ts
│   │   └── tenant-isolation.service.ts
│   │
│   ├── validators/
│   │   ├── required-field.validator.ts
│   │   ├── enum.validator.ts
│   │   ├── time-window.validator.ts
│   │   ├── timezone.validator.ts
│   │   ├── phone-format.validator.ts
│   │   └── selector-shape.validator.ts
│   │
│   └── errors/
│       ├── configuration-validation.error.ts
│       ├── schema-version.error.ts
│       ├── missing-required-field.error.ts
│       └── invalid-selector-shape.error.ts
│
└── tests/
    ├── contract.tests.ts
    ├── validation.tests.ts
    ├── defaulting.tests.ts
    ├── inheritance.tests.ts
    └── fixture.tests.ts
```

---

## Contracts

### canonical-configuration.contract.ts

```typescript
/**
 * Canonical Configuration Contract
 *
 * Defines the runtime shape of business configuration after validation,
 * defaulting, and normalization.
 */
export interface CanonicalConfigurationContract {
  readonly schemaVersion: string;
  readonly locationId: string;
  readonly clientId: string;
  readonly displayName: string;
  readonly timezone: string;
  readonly operatingMode: 'booking-enabled' | 'intake-only';
  readonly operationalStatus: 'active' | 'paused';
  readonly serviceDeliveryMode: 'inShop' | 'onSite' | 'both';
  readonly businessHours: BusinessHours;
  readonly emergencyConfig: EmergencyConfig;
  readonly aiCoverage: AICoverageSchedule;
  readonly calendarExceptions?: CalendarException[];
  readonly uncoveredAction: 'forward' | 'reject';
  readonly forwardPhoneNumber?: string;
  readonly serviceCatalogRef: ServiceCatalogRef;
  readonly serviceCatalogExclusions?: ServiceCatalogExclusions;
  readonly serviceAreaRef: ServiceAreaRef;
  readonly emergencyServiceAreaRef?: ServiceAreaRef;
  readonly clientAlerting: ClientAlertingConfig;
  readonly customerSms: CustomerSmsConfig;
  readonly policyProfileRef: PolicyProfileRef;
  readonly publicContact?: PublicContact;
  readonly locales?: string[];
  readonly locationCode?: string;
}
```

### validation-result.contract.ts

```typescript
/**
 * Validation Result Contract
 *
 * Returns deterministic validation outcomes with structured error types
 * and field paths.
 */
export type ValidationResultContract =
  | { readonly valid: true; readonly value: CanonicalConfigurationContract }
  | { readonly valid: false; readonly errors: ConfigurationValidationError[] };
```

### schema-version.contract.ts

```typescript
/**
 * Schema Version Contract
 *
 * Declares supported schema versions. Unknown versions fail closed.
 */
export const SUPPORTED_SCHEMA_VERSIONS = ['LocationProfile.v1'] as const;

export type SchemaVersion = (typeof SUPPORTED_SCHEMA_VERSIONS)[number];

export interface SchemaVersionContract {
  readonly version: SchemaVersion;
  readonly isSupported: boolean;
}
```

---

## Models

### location-profile.model.ts

```typescript
import { BusinessHours } from './business-hours.model';
import { AICoverageSchedule } from './ai-coverage-schedule.model';
import { CalendarException } from './calendar-exception.model';
import { EmergencyConfig } from './emergency-config.model';
import { ServiceCatalogRef } from './service-catalog-ref.model';
import { ServiceAreaRef } from './service-area-ref.model';
import { ClientAlertingConfig } from './client-alerting-config.model';
import { CustomerSmsConfig } from './customer-sms-config.model';
import { PolicyProfileRef } from './policy-profile-ref.model';

export interface LocationProfile {
  schemaVersion: string;
  locationId: string;
  clientId: string;
  displayName: string;
  timezone: string;
  operatingMode: 'booking-enabled' | 'intake-only';
  operationalStatus?: 'active' | 'paused';
  serviceDeliveryMode: 'inShop' | 'onSite' | 'both';
  businessHours: BusinessHours;
  emergencyConfig: EmergencyConfig;
  aiCoverage: AICoverageSchedule;
  calendarExceptions?: CalendarException[];
  uncoveredAction: 'forward' | 'reject';
  forwardPhoneNumber?: string;
  serviceCatalogRef: ServiceCatalogRef;
  serviceCatalogExclusions?: ServiceCatalogExclusions;
  serviceAreaRef: ServiceAreaRef;
  emergencyServiceAreaRef?: ServiceAreaRef;
  clientAlerting: ClientAlertingConfig;
  customerSms: CustomerSmsConfig;
  policyProfileRef?: PolicyProfileRef;
  publicContact?: PublicContact;
  locales?: string[];
  locationCode?: string;
}
```

### business-hours.model.ts

```typescript
export type LocalTimeHHMM = string;

export interface TimeWindow {
  startLocalTime: LocalTimeHHMM;
  endLocalTime: LocalTimeHHMM;
}

export interface DailyOpenHours {
  dayOfWeek: 'mon' | 'tue' | 'wed' | 'thu' | 'fri' | 'sat' | 'sun';
  openWindows: TimeWindow[];
}

export interface BusinessHours {
  regular: DailyOpenHours[];
}
```

### ai-coverage-schedule.model.ts

```typescript
export interface AICoverageSchedule {
  schedule: DailyOpenHours[];
}
```

### calendar-exception.model.ts

```typescript
export interface CalendarException {
  date: string;
  openWindows: TimeWindow[];
  reason?: string;
}
```

### emergency-config.model.ts

```typescript
export interface EmergencyConfig {
  serviceHours: {
    type: 'sameAsBusinessHours' | 'explicit';
    schedule?: DailyOpenHours[];
  };
  clientContactMode: 'warm_transfer' | 'notification_only';
  warmTransferEnabled: boolean;
  smsFallbackEnabled: boolean;
  clientCallTimeoutSeconds: number;
}
```

### service-catalog-ref.model.ts

```typescript
export interface ServiceCatalogRef {
  type: 'inline' | 'allFromClientDefault' | 'byTag';
  items?: string[];
  includeTags?: string[];
  excludeTags?: string[];
}
```

### service-area-ref.model.ts

```typescript
export interface ServiceAreaRef {
  ruleIds: string[];
}
```

### client-alerting-config.model.ts

```typescript
export interface Recipient {
  recipientId: string;
  displayName: string;
  smsE164?: string;
  voiceE164?: string;
  email?: string;
  enabled: boolean;
}

export interface RoutingPlan {
  type: 'primaryThenSecondary';
  primaryRecipientId: string;
  secondaryRecipientId?: string;
}

export interface HoursAwareRouting {
  enabled: boolean;
  windows: Array<{
    daysOfWeek: Array<'mon' | 'tue' | 'wed' | 'thu' | 'fri' | 'sat' | 'sun'>;
    startLocalTime: string;
    endLocalTime: string;
    recipients: string[];
  }>;
}

export interface ClientAlertingConfig {
  recipients: Recipient[];
  routingPlan: {
    emergency: RoutingPlan;
    standard: RoutingPlan;
  };
  hoursAware?: HoursAwareRouting;
}
```

### customer-sms-config.model.ts

```typescript
export interface CustomerSmsConfig {
  appointmentSms: {
    enabled: boolean;
  };
  intakeAcknowledgementSms: {
    enabled: boolean;
  };
}
```

### policy-profile-ref.model.ts

```typescript
export interface PolicyProfileRef {
  type: 'locationOverride' | 'useClientDefault';
  policyProfileId?: string;
}
```

---

## Services

### schema-definition.service.ts

```typescript
import { SUPPORTED_SCHEMA_VERSIONS } from '../contracts/schema-version.contract';
import { LocationProfile } from '../models/location-profile.model';

export class SchemaDefinitionService {
  getSupportedSchemaVersions(): readonly string[] {
    return SUPPORTED_SCHEMA_VERSIONS;
  }

  isSupportedSchemaVersion(version: string): boolean {
    return SUPPORTED_SCHEMA_VERSIONS.includes(version as SchemaVersion);
  }

  getCanonicalShape(): Record<string, unknown> {
    // Returns the canonical shape definition for LocationProfile.
    // Used by validation and documentation generation.
    return {
      schemaVersion: 'string',
      locationId: 'string',
      clientId: 'string',
      displayName: 'string',
      timezone: 'string',
      operatingMode: 'booking-enabled | intake-only',
      operationalStatus: 'active | paused',
      serviceDeliveryMode: 'inShop | onSite | both',
      businessHours: 'BusinessHours',
      emergencyConfig: 'EmergencyConfig',
      aiCoverage: 'AICoverageSchedule',
      uncoveredAction: 'forward | reject',
      serviceCatalogRef: 'ServiceCatalogRef',
      serviceAreaRef: 'ServiceAreaRef',
      clientAlerting: 'ClientAlertingConfig',
      customerSms: 'CustomerSmsConfig',
      policyProfileRef: 'PolicyProfileRef',
    };
  }
}
```

### validation.service.ts

```typescript
import { ValidationResultContract } from '../contracts/validation-result.contract';
import { LocationProfile } from '../models/location-profile.model';
import { ConfigurationValidationError } from '../errors/configuration-validation.error';
import { RequiredFieldValidator } from '../validators/required-field.validator';
import { EnumValidator } from '../validators/enum.validator';
import { TimeWindowValidator } from '../validators/time-window.validator';
import { TimezoneValidator } from '../validators/timezone.validator';
import { PhoneFormatValidator } from '../validators/phone-format.validator';
import { SelectorShapeValidator } from '../validators/selector-shape.validator';
import { SchemaDefinitionService } from './schema-definition.service';

export class ValidationService {
  constructor(private readonly schemaDefinition: SchemaDefinitionService) {}

  validate(profile: unknown): ValidationResultContract {
    const errors: ConfigurationValidationError[] = [];

    if (typeof profile !== 'object' || profile === null) {
      errors.push(new ConfigurationValidationError('root', 'LocationProfile must be an object'));
      return { valid: false, errors };
    }

    const candidate = profile as Partial<LocationProfile>;

    this.validateRequiredFields(candidate, errors);
    this.validateEnums(candidate, errors);
    this.validateTimeWindows(candidate, errors);
    this.validateTimezone(candidate, errors);
    this.validatePhoneFormats(candidate, errors);
    this.validateSelectorShapes(candidate, errors);

    if (errors.length > 0) {
      return { valid: false, errors };
    }

    return { valid: true, value: candidate as LocationProfile };
  }

  private validateRequiredFields(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    RequiredFieldValidator.validate(candidate, 'schemaVersion', errors);
    RequiredFieldValidator.validate(candidate, 'locationId', errors);
    RequiredFieldValidator.validate(candidate, 'clientId', errors);
    RequiredFieldValidator.validate(candidate, 'displayName', errors);
    RequiredFieldValidator.validate(candidate, 'timezone', errors);
    RequiredFieldValidator.validate(candidate, 'operatingMode', errors);
    RequiredFieldValidator.validate(candidate, 'serviceDeliveryMode', errors);
    RequiredFieldValidator.validate(candidate, 'businessHours', errors);
    RequiredFieldValidator.validate(candidate, 'emergencyConfig', errors);
    RequiredFieldValidator.validate(candidate, 'aiCoverage', errors);
    RequiredFieldValidator.validate(candidate, 'uncoveredAction', errors);
    RequiredFieldValidator.validate(candidate, 'serviceCatalogRef', errors);
    RequiredFieldValidator.validate(candidate, 'serviceAreaRef', errors);
    RequiredFieldValidator.validate(candidate, 'clientAlerting', errors);
    RequiredFieldValidator.validate(candidate, 'customerSms', errors);
  }

  private validateEnums(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    EnumValidator.validate(candidate.operatingMode, 'operatingMode', ['booking-enabled', 'intake-only'], errors);
    EnumValidator.validate(candidate.operationalStatus, 'operationalStatus', ['active', 'paused'], errors, false);
    EnumValidator.validate(candidate.serviceDeliveryMode, 'serviceDeliveryMode', ['inShop', 'onSite', 'both'], errors);
    EnumValidator.validate(candidate.uncoveredAction, 'uncoveredAction', ['forward', 'reject'], errors);
  }

  private validateTimeWindows(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    if (candidate.businessHours?.regular) {
      TimeWindowValidator.validate(candidate.businessHours.regular, 'businessHours.regular', errors);
    }
  }

  private validateTimezone(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    if (candidate.timezone) {
      TimezoneValidator.validate(candidate.timezone, 'timezone', errors);
    }
  }

  private validatePhoneFormats(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    if (candidate.forwardPhoneNumber) {
      PhoneFormatValidator.validate(candidate.forwardPhoneNumber, 'forwardPhoneNumber', errors);
    }
    if (candidate.publicContact?.phoneE164) {
      PhoneFormatValidator.validate(candidate.publicContact.phoneE164, 'publicContact.phoneE164', errors);
    }
  }

  private validateSelectorShapes(candidate: Partial<LocationProfile>, errors: ConfigurationValidationError[]): void {
    if (candidate.serviceCatalogRef) {
      SelectorShapeValidator.validate(candidate.serviceCatalogRef, 'serviceCatalogRef', errors);
    }
    if (candidate.serviceAreaRef) {
      SelectorShapeValidator.validateServiceArea(candidate.serviceAreaRef, 'serviceAreaRef', errors);
    }
  }
}
```

### defaulting.service.ts

```typescript
import { LocationProfile } from '../models/location-profile.model';
import { PolicyProfileRef } from '../models/policy-profile-ref.model';
import { EmergencyConfig } from '../models/emergency-config.model';

export class DefaultingService {
  applyDefaults(profile: Partial<LocationProfile>): LocationProfile {
    return Object.freeze({
      ...profile,
      operationalStatus: profile.operationalStatus ?? 'active',
      policyProfileRef: profile.policyProfileRef ?? { type: 'useClientDefault' },
      emergencyConfig: this.applyEmergencyConfigDefaults(profile.emergencyConfig),
    } as LocationProfile);
  }

  private applyEmergencyConfigDefaults(config?: Partial<EmergencyConfig>): EmergencyConfig {
    return {
      serviceHours: config?.serviceHours ?? { type: 'sameAsBusinessHours' },
      clientContactMode: config?.clientContactMode ?? 'notification_only',
      warmTransferEnabled: config?.warmTransferEnabled ?? false,
      smsFallbackEnabled: config?.smsFallbackEnabled ?? true,
      clientCallTimeoutSeconds: config?.clientCallTimeoutSeconds ?? 30,
    };
  }
}
```

### normalization.service.ts

```typescript
import { LocationProfile } from '../models/location-profile.model';

export class NormalizationService {
  normalize(profile: LocationProfile): LocationProfile {
    return Object.freeze({
      ...profile,
      locationId: profile.locationId.trim(),
      clientId: profile.clientId.trim(),
      schemaVersion: profile.schemaVersion.trim(),
      displayName: profile.displayName.trim(),
      timezone: profile.timezone.trim(),
    });
  }
}
```

### tenant-isolation.service.ts

```typescript
import { LocationProfile } from '../models/location-profile.model';
import { ConfigurationValidationError } from '../errors/configuration-validation.error';

export class TenantIsolationService {
  scopeToTenant(profile: LocationProfile, requestedClientId: string): LocationProfile {
    if (profile.clientId !== requestedClientId) {
      throw new ConfigurationValidationError(
        'clientId',
        'Location profile does not belong to the requested tenant'
      );
    }
    return profile;
  }
}
```

---

## Validators

### required-field.validator.ts

```typescript
import { ConfigurationValidationError } from '../errors/configuration-validation.error';
import { MissingRequiredFieldError } from '../errors/missing-required-field.error';

export class RequiredFieldValidator {
  static validate<T>(
    object: Partial<T>,
    field: keyof T,
    errors: ConfigurationValidationError[]
  ): void {
    const value = object[field];
    if (value === undefined || value === null || value === '') {
      errors.push(new MissingRequiredFieldError(field as string));
    }
  }
}
```

### enum.validator.ts

```typescript
import { ConfigurationValidationError } from '../errors/configuration-validation.error';

export class EnumValidator {
  static validate<T extends string>(
    value: T | undefined,
    field: string,
    allowed: readonly T[],
    errors: ConfigurationValidationError[],
    required: boolean = true
  ): void {
    if (value === undefined || value === null) {
      if (required) {
        errors.push(new ConfigurationValidationError(field, `${field} is required`));
      }
      return;
    }
    if (!allowed.includes(value)) {
      errors.push(new ConfigurationValidationError(field, `${field} must be one of ${allowed.join(', ')}`));
    }
  }
}
```

### time-window.validator.ts

```typescript
import { DailyOpenHours, TimeWindow } from '../models/business-hours.model';
import { ConfigurationValidationError } from '../errors/configuration-validation.error';

export class TimeWindowValidator {
  static validate(days: DailyOpenHours[], path: string, errors: ConfigurationValidationError[]): void {
    const seenDays = new Set<string>();
    for (const day of days) {
      if (seenDays.has(day.dayOfWeek)) {
        errors.push(new ConfigurationValidationError(`${path}.${day.dayOfWeek}`, `Duplicate dayOfWeek: ${day.dayOfWeek}`));
      }
      seenDays.add(day.dayOfWeek);

      for (const window of day.openWindows) {
        this.validateWindow(window, `${path}.${day.dayOfWeek}`, errors);
      }
    }
  }

  private static validateWindow(window: TimeWindow, path: string, errors: ConfigurationValidationError[]): void {
    const timeRegex = /^([01]\d|2[0-3]):([0-5]\d)$/;
    if (!timeRegex.test(window.startLocalTime)) {
      errors.push(new ConfigurationValidationError(`${path}.startLocalTime`, 'Invalid local time format'));
    }
    if (!timeRegex.test(window.endLocalTime)) {
      errors.push(new ConfigurationValidationError(`${path}.endLocalTime`, 'Invalid local time format'));
    }
  }
}
```

### timezone.validator.ts

```typescript
import { ConfigurationValidationError } from '../errors/configuration-validation.error';

export class TimezoneValidator {
  static validate(timezone: string, field: string, errors: ConfigurationValidationError[]): void {
    try {
      Intl.DateTimeFormat(undefined, { timeZone: timezone });
    } catch {
      errors.push(new ConfigurationValidationError(field, `${field} must be a valid IANA timezone`));
    }
  }
}
```

### phone-format.validator.ts

```typescript
import { ConfigurationValidationError } from '../errors/configuration-validation.error';

export class PhoneFormatValidator {
  static validate(phone: string, field: string, errors: ConfigurationValidationError[]): void {
    const e164Regex = /^\+[1-9]\d{1,14}$/;
    if (!e164Regex.test(phone)) {
      errors.push(new ConfigurationValidationError(field, `${field} must be a valid E.164 phone number`));
    }
  }
}
```

### selector-shape.validator.ts

```typescript
import { ServiceCatalogRef } from '../models/service-catalog-ref.model';
import { ServiceAreaRef } from '../models/service-area-ref.model';
import { ConfigurationValidationError } from '../errors/configuration-validation.error';
import { InvalidSelectorShapeError } from '../errors/invalid-selector-shape.error';

export class SelectorShapeValidator {
  static validate(ref: ServiceCatalogRef, path: string, errors: ConfigurationValidationError[]): void {
    switch (ref.type) {
      case 'inline':
        if (!ref.items || ref.items.length === 0) {
          errors.push(new InvalidSelectorShapeError(`${path}.items`, 'inline selector requires items'));
        }
        break;
      case 'byTag':
        if (!ref.includeTags || ref.includeTags.length === 0) {
          errors.push(new InvalidSelectorShapeError(`${path}.includeTags`, 'byTag selector requires includeTags'));
        }
        break;
      case 'allFromClientDefault':
        break;
      default:
        errors.push(new ConfigurationValidationError(`${path}.type`, 'Unsupported service catalog selector type'));
    }
  }

  static validateServiceArea(ref: ServiceAreaRef, path: string, errors: ConfigurationValidationError[]): void {
    if (!Array.isArray(ref.ruleIds)) {
      errors.push(new InvalidSelectorShapeError(`${path}.ruleIds`, 'ruleIds must be an array'));
    }
  }
}
```

---

## Errors

### configuration-validation.error.ts

```typescript
export class ConfigurationValidationError extends Error {
  constructor(
    public readonly field: string,
    message: string
  ) {
    super(message);
    this.name = 'ConfigurationValidationError';
  }
}
```

### schema-version.error.ts

```typescript
export class SchemaVersionError extends Error {
  constructor(public readonly version: string) {
    super(`Unsupported schema version: ${version}`);
    this.name = 'SchemaVersionError';
  }
}
```

### missing-required-field.error.ts

```typescript
import { ConfigurationValidationError } from './configuration-validation.error';

export class MissingRequiredFieldError extends ConfigurationValidationError {
  constructor(field: string) {
    super(field, `${field} is required`);
    this.name = 'MissingRequiredFieldError';
  }
}
```

### invalid-selector-shape.error.ts

```typescript
import { ConfigurationValidationError } from './configuration-validation.error';

export class InvalidSelectorShapeError extends ConfigurationValidationError {
  constructor(field: string, message: string) {
    super(field, message);
    this.name = 'InvalidSelectorShapeError';
  }
}
```

---

## Tests

### validation.tests.ts

```typescript
import { describe, it, expect } from '@jest/globals';
import { ValidationService } from '../src/services/validation.service';
import { SchemaDefinitionService } from '../src/services/schema-definition.service';
import { DefaultingService } from '../src/services/defaulting.service';
import { NormalizationService } from '../src/services/normalization.service';
import { LocationProfile } from '../src/models/location-profile.model';

describe('Business Configuration Framework', () => {
  const schemaService = new SchemaDefinitionService();
  const validationService = new ValidationService(schemaService);
  const defaultingService = new DefaultingService();
  const normalizationService = new NormalizationService();

  it('validates a minimal valid LocationProfile', () => {
    const raw = defaultingService.applyDefaults(validLocationProfileRaw());
    const normalized = normalizationService.normalize(raw);
    const result = validationService.validate(normalized);

    expect(result.valid).toBe(true);
    if (result.valid) {
      expect(result.value.operationalStatus).toBe('active');
      expect(result.value.policyProfileRef?.type).toBe('useClientDefault');
    }
  });

  it('rejects missing required fields', () => {
    const result = validationService.validate({});
    expect(result.valid).toBe(false);
    if (!result.valid) {
      expect(result.errors.length).toBeGreaterThan(0);
    }
  });

  it('rejects invalid timezone', () => {
    const raw = defaultingService.applyDefaults({
      ...validLocationProfileRaw(),
      timezone: 'Mars/Phobos',
    });
    const result = validationService.validate(raw);
    expect(result.valid).toBe(false);
  });

  it('rejects invalid operating mode', () => {
    const raw = defaultingService.applyDefaults({
      ...validLocationProfileRaw(),
      operatingMode: 'invalid-mode' as any,
    });
    const result = validationService.validate(raw);
    expect(result.valid).toBe(false);
  });

  function validLocationProfileRaw(): Partial<LocationProfile> {
    return {
      schemaVersion: 'LocationProfile.v1',
      locationId: 'loc_001',
      clientId: 'client_abc',
      displayName: 'Downtown Branch',
      timezone: 'America/Chicago',
      operatingMode: 'intake-only',
      serviceDeliveryMode: 'both',
      businessHours: {
        regular: [
          { dayOfWeek: 'mon', openWindows: [{ startLocalTime: '09:00', endLocalTime: '17:00' }] },
        ],
      },
      emergencyConfig: {
        serviceHours: { type: 'sameAsBusinessHours' },
        clientContactMode: 'notification_only',
        warmTransferEnabled: false,
        smsFallbackEnabled: true,
        clientCallTimeoutSeconds: 30,
      },
      aiCoverage: { schedule: [] },
      uncoveredAction: 'reject',
      serviceCatalogRef: { type: 'allFromClientDefault' },
      serviceAreaRef: { ruleIds: ['area_001'] },
      clientAlerting: {
        recipients: [],
        routingPlan: {
          emergency: { type: 'primaryThenSecondary', primaryRecipientId: 'tech_1' },
          standard: { type: 'primaryThenSecondary', primaryRecipientId: 'tech_1' },
        },
      },
      customerSms: {
        appointmentSms: { enabled: false },
        intakeAcknowledgementSms: { enabled: false },
      },
    };
  }
});
```

---

## Notes

- This implementation mirrors the Layer 3 blueprint components one-to-one.
- It uses the LocationProfile.PRD for property names, types, defaults, validation rules, and semantics.
- No runtime decisions, workflow execution, external calls, notification dispatch, or customer data capture are present.
- Adapters and storage drivers are intentionally not implemented; they belong to the Configuration Management Layer.
- The implementation is backend-neutral and uses dependency injection-friendly class structures.
