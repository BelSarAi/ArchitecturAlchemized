Updated todo list

# 002 Configuration Management Layer — Implementation

**Assumption note:** `ClientProfile.PRD` and `LocationProfile.PRD` were not supplied with this task. The canonical shapes below are derived strictly from the fixtures and validation rules in `ConfigurationLoader.PRD`. If the owning PRDs introduce additional fields, the validators and compilers should be extended additively.

---

## Project Structure

```text
configuration-management-layer/
├── contracts/
│   ├── schema-version.contract.ts
│   ├── config-store-adapter.contract.ts
│   └── configuration-load-result.contract.ts
├── models/
│   ├── load-request.model.ts
│   ├── client-profile.model.ts
│   ├── location-profile.model.ts
│   ├── service-area-ruleset.model.ts
│   └── index.ts
├── errors/
│   └── configuration-load.error.ts
├── compilation/
│   └── context-compilation.engine.ts
├── validation/
│   └── structural-verification.gate.ts
├── caching/
│   └── multi-tenant-cache.registry.ts
├── adapters/
│   └── in-memory-config-store.adapter.ts
├── loader/
│   └── configuration-loader.gateway.ts
└── tests/
    └── configuration-loader.tests.ts
```

---

## contracts/schema-version.contract.ts

```typescript
/**
 * Canonical schema version identifiers enforced by the Configuration Management Layer.
 * New schema versions are additive; unknown versions fail closed.
 */
export const SchemaVersion = {
  CLIENT_PROFILE: 'ClientProfile.v1',
  LOCATION_PROFILE: 'LocationProfile.v1',
  SERVICE_AREA_RULES: 'ServiceAreaRules.v1',
} as const;

export type SchemaVersion =
  | typeof SchemaVersion.CLIENT_PROFILE
  | typeof SchemaVersion.LOCATION_PROFILE
  | typeof SchemaVersion.SERVICE_AREA_RULES;
```

---

## contracts/config-store-adapter.contract.ts

```typescript
/**
 * Backend-neutral storage adapter contract.
 * Implementations own only physical retrieval and transport.
 * Validation, defaulting, and interpretation are strictly forbidden here.
 */
export interface ConfigStoreAdapterInterface {
  getClientProfile(clientId: string): Promise<unknown | null>;
  getLocationProfile(locationId: string): Promise<unknown | null>;
  getServiceAreaRuleset(rulesetId: string): Promise<unknown | null>;
}
```

---

## contracts/configuration-load-result.contract.ts

```typescript
import type { ClientProfile, LocationProfile, ServiceAreaRuleset } from '../models';

export type ConfigLoadErrorType =
  | 'validation_error'
  | 'not_found'
  | 'unavailable'
  | 'permission_denied';

export type ConfigurationLoadResult =
  | {
      ok: true;
      clientProfile: ClientProfile;
      locationProfile: LocationProfile;
      serviceAreaRuleset?: ServiceAreaRuleset;
      metadata: {
        clientProfileSource?: string;
        locationProfileSource?: string;
        serviceAreaRulesetSource?: string;
        loadedAtUtc: string;
        cacheHit: boolean;
        versionTag?: string;
      };
    }
  | {
      ok: false;
      errorType: ConfigLoadErrorType;
      errorCode: string;
      message: string;
      details?: unknown;
    };
```

---

## models/load-request.model.ts

```typescript
export interface LoadRequest {
  clientId: string;
  locationId: string;
  correlationKey?: string;
}
```

---

## models/client-profile.model.ts

```typescript
import type { PublicContact, ServiceCatalogRef } from './shared.model';

export interface ClientDefaults {
  masterServiceCatalogRef?: ServiceCatalogRef;
  masterServiceAreaRuleIds?: string[];
  timezone?: string;
  policyProfileRef?: { type: 'useClientDefault' };
}

export interface ClientProfile {
  schemaVersion: 'ClientProfile.v1';
  clientId: string;
  displayName: string;
  operationalStatus: 'active' | 'paused';
  defaults: ClientDefaults;
  publicContact?: PublicContact;
}
```

---

## models/location-profile.model.ts

```typescript
import type {
  BusinessHours,
  ClientAlerting,
  CustomerSms,
  PolicyProfileRef,
  PublicContact,
  ServiceAreaRef,
  ServiceCatalogRef,
} from './shared.model';

export interface LocationProfile {
  schemaVersion: 'LocationProfile.v1';
  locationId: string;
  clientId: string;
  displayName: string;
  timezone: string;
  operatingMode: string;
  operationalStatus: 'active' | 'paused';
  serviceDeliveryMode: 'inShop' | 'onSite' | 'both';
  businessHours?: BusinessHours;
  serviceCatalogRef: ServiceCatalogRef;
  serviceAreaRef: ServiceAreaRef;
  clientAlerting?: ClientAlerting;
  customerSms?: CustomerSms;
  policyProfileRef?: PolicyProfileRef;
  publicContact?: PublicContact;
  locales?: string[];
}
```

---

## models/service-area-ruleset.model.ts

```typescript
export interface GeoCoordinates {
  lat: number;
  lng: number;
}

export interface ServiceAreaRule {
  ruleId: string;
  ruleType: 'zipCode' | 'mileRadius' | string;
  displayName: string;
  description: string;
  zipCodes?: string[];
  centerAddress?: string;
  centerCoordinates?: GeoCoordinates;
  radiusMiles?: number;
  order: number;
  enabled: boolean;
}

export interface ServiceAreaRuleset {
  rulesetId: string;
  schemaVersion?: 'ServiceAreaRules.v1';
  rules: ServiceAreaRule[];
}
```

---

## models/shared.model.ts

```typescript
export type ServiceCatalogRef =
  | { type: 'byTag'; includeTags: string[] }
  | { type: 'inline'; items: string[] }
  | { type: 'allFromClientDefault' };

export interface ServiceAreaRef {
  ruleIds: string[];
}

export type PolicyProfileRef = { type: 'useClientDefault' };

export interface BusinessHours {
  regular: RegularHours[];
}

export interface RegularHours {
  dayOfWeek: string;
  openWindows: TimeWindow[];
}

export interface TimeWindow {
  startLocalTime: string;
  endLocalTime: string;
}

export interface ClientAlerting {
  recipients: AlertRecipient[];
  routingPlan: RoutingPlan;
}

export interface AlertRecipient {
  recipientId: string;
  displayName: string;
  smsE164?: string;
  voiceE164?: string;
  email?: string;
  enabled: boolean;
}

export interface RoutingPlan {
  emergency: RoutingStep;
  standard: RoutingStep;
}

export interface RoutingStep {
  type: string;
  primaryRecipientId: string;
  secondaryRecipientId?: string;
}

export interface CustomerSms {
  appointmentSms?: SmsToggle;
  intakeAcknowledgementSms?: SmsToggle;
}

export interface SmsToggle {
  enabled: boolean;
}

export interface PublicContact {
  phoneE164?: string;
  address?: string;
  website?: string;
}
```

---

## models/index.ts

```typescript
export * from './load-request.model';
export * from './shared.model';
export * from './client-profile.model';
export * from './location-profile.model';
export * from './service-area-ruleset.model';
```

---

## errors/configuration-load.error.ts

```typescript
import type { ConfigLoadErrorType } from '../contracts/configuration-load-result.contract';

export class ConfigurationLoadError extends Error {
  constructor(
    public readonly errorType: ConfigLoadErrorType,
    public readonly errorCode: string,
    message: string,
    public readonly details?: unknown,
  ) {
    super(message);
    this.name = 'ConfigurationLoadError';
  }
}
```

---

## compilation/context-compilation.engine.ts

```typescript
import {
  type ClientDefaults,
  type ClientProfile,
  type GeoCoordinates,
  type LocationProfile,
  type ServiceAreaRule,
  type ServiceAreaRuleset,
} from '../models';
import { SchemaVersion } from '../contracts/schema-version.contract';
import { ConfigurationLoadError } from '../errors/configuration-load.error';

/**
 * Pure, side-effect-free compilation of raw records into canonical shapes.
 * This engine applies only deterministic defaulting; it performs no validation
 * and never queries storage.
 */
export class ContextCompilationEngine {
  compileClientProfile(raw: unknown): ClientProfile {
    if (!isObject(raw)) {
      throw new ConfigurationLoadError(
        'validation_error',
        'invalid_type',
        'ClientProfile raw record must be an object.',
      );
    }

    const clientId = String(raw.clientId ?? '');
    const displayName = String(raw.displayName ?? '');
    const operationalStatus = raw.operationalStatus === 'paused' ? 'paused' : 'active';

    const defaults = isObject(raw.defaults)
      ? this.compileClientDefaults(raw.defaults)
      : {};

    return Object.freeze({
      schemaVersion: SchemaVersion.CLIENT_PROFILE,
      clientId,
      displayName,
      operationalStatus,
      defaults,
      ...(raw.publicContact !== undefined && {
        publicContact: raw.publicContact as ClientProfile['publicContact'],
      }),
    });
  }

  private compileClientDefaults(raw: unknown): ClientDefaults {
    if (!isObject(raw)) return {};
    return Object.freeze({
      ...(raw.masterServiceCatalogRef !== undefined && {
        masterServiceCatalogRef: raw.masterServiceCatalogRef as ClientDefaults['masterServiceCatalogRef'],
      }),
      ...(raw.masterServiceAreaRuleIds !== undefined && {
        masterServiceAreaRuleIds: Array.isArray(raw.masterServiceAreaRuleIds)
          ? raw.masterServiceAreaRuleIds.map(String)
          : [],
      }),
      ...(raw.timezone !== undefined && { timezone: String(raw.timezone) }),
      ...(raw.policyProfileRef !== undefined && {
        policyProfileRef: raw.policyProfileRef as ClientDefaults['policyProfileRef'],
      }),
    });
  }

  compileLocationProfile(raw: unknown): LocationProfile {
    if (!isObject(raw)) {
      throw new ConfigurationLoadError(
        'validation_error',
        'invalid_type',
        'LocationProfile raw record must be an object.',
      );
    }

    const operationalStatus = raw.operationalStatus === 'paused' ? 'paused' : 'active';

    const policyProfileRef: LocationProfile['policyProfileRef'] =
      raw.policyProfileRef === undefined
        ? { type: 'useClientDefault' }
        : (raw.policyProfileRef as LocationProfile['policyProfileRef']);

    return Object.freeze({
      schemaVersion: SchemaVersion.LOCATION_PROFILE,
      clientId: String(raw.clientId ?? ''),
      locationId: String(raw.locationId ?? ''),
      displayName: String(raw.displayName ?? ''),
      timezone: String(raw.timezone ?? ''),
      operatingMode: String(raw.operatingMode ?? ''),
      operationalStatus,
      serviceDeliveryMode: String(raw.serviceDeliveryMode ?? '') as LocationProfile['serviceDeliveryMode'],
      serviceCatalogRef: raw.serviceCatalogRef as LocationProfile['serviceCatalogRef'],
      serviceAreaRef: this.compileServiceAreaRef(raw.serviceAreaRef),
      ...(raw.businessHours !== undefined && {
        businessHours: raw.businessHours as LocationProfile['businessHours'],
      }),
      ...(raw.clientAlerting !== undefined && {
        clientAlerting: raw.clientAlerting as LocationProfile['clientAlerting'],
      }),
      ...(raw.customerSms !== undefined && {
        customerSms: raw.customerSms as LocationProfile['customerSms'],
      }),
      ...(policyProfileRef && { policyProfileRef }),
      ...(raw.publicContact !== undefined && {
        publicContact: raw.publicContact as LocationProfile['publicContact'],
      }),
      ...(raw.locales !== undefined && {
        locales: Array.isArray(raw.locales) ? raw.locales.map(String) : [],
      }),
    });
  }

  private compileServiceAreaRef(raw: unknown): LocationProfile['serviceAreaRef'] {
    if (!isObject(raw)) {
      return { ruleIds: [] };
    }
    const ruleIds = Array.isArray(raw.ruleIds)
      ? raw.ruleIds.map(String)
      : [];
    return Object.freeze({ ruleIds });
  }

  compileServiceAreaRuleset(raw: unknown): ServiceAreaRuleset {
    if (!isObject(raw)) {
      throw new ConfigurationLoadError(
        'validation_error',
        'invalid_type',
        'ServiceAreaRuleset raw record must be an object.',
      );
    }

    const rules = Array.isArray(raw.rules)
      ? raw.rules.map((r) => this.compileServiceAreaRule(r))
      : [];

    return Object.freeze({
      rulesetId: String(raw.rulesetId ?? ''),
      ...(raw.schemaVersion !== undefined && {
        schemaVersion: String(raw.schemaVersion) as ServiceAreaRuleset['schemaVersion'],
      }),
      rules,
    });
  }

  private compileServiceAreaRule(raw: unknown): ServiceAreaRule {
    if (!isObject(raw)) {
      throw new ConfigurationLoadError(
        'validation_error',
        'invalid_type',
        'ServiceAreaRule must be an object.',
      );
    }

    const enabled = typeof raw.enabled === 'boolean' ? raw.enabled : true;

    return Object.freeze({
      ruleId: String(raw.ruleId ?? ''),
      ruleType: String(raw.ruleType ?? ''),
      displayName: String(raw.displayName ?? ''),
      description: String(raw.description ?? ''),
      ...(raw.zipCodes !== undefined && {
        zipCodes: Array.isArray(raw.zipCodes) ? raw.zipCodes.map(String) : [],
      }),
      ...(raw.centerAddress !== undefined && { centerAddress: String(raw.centerAddress) }),
      ...(raw.centerCoordinates !== undefined && {
        centerCoordinates: this.compileGeoCoordinates(raw.centerCoordinates),
      }),
      ...(raw.radiusMiles !== undefined && { radiusMiles: Number(raw.radiusMiles) }),
      order: Number(raw.order ?? 0),
      enabled,
    });
  }

  private compileGeoCoordinates(raw: unknown): GeoCoordinates {
    if (!isObject(raw)) {
      return Object.freeze({ lat: NaN, lng: NaN });
    }
    return Object.freeze({
      lat: Number(raw.lat ?? NaN),
      lng: Number(raw.lng ?? NaN),
    });
  }
}

function isObject(value: unknown): value is Record<string, unknown> {
  return value !== null && typeof value === 'object' && !Array.isArray(value);
}
```

---

## validation/structural-verification.gate.ts

```typescript
import {
  type BusinessHours,
  type ClientAlerting,
  type ClientProfile,
  type CustomerSms,
  type LocationProfile,
  type PublicContact,
  type RoutingPlan,
  type ServiceAreaRule,
  type ServiceAreaRuleset,
} from '../models';
import { SchemaVersion } from '../contracts/schema-version.contract';
import { ConfigurationLoadError } from '../errors/configuration-load.error';

/**
 * Strict schema validation gate.
 * - Rejects unknown fields.
 * - Enforces required fields, enums, and cross-object inheritance contracts.
 * - Never repairs records; it only accepts or rejects.
 */
export class StructuralVerificationGate {
  validateClientProfile(profile: ClientProfile): void {
    this.assertObject(profile, 'clientProfile');
    this.assertNoExtraKeys(profile, CLIENT_PROFILE_KEYS, 'clientProfile');

    this.assertExactString(profile.schemaVersion, SchemaVersion.CLIENT_PROFILE, 'clientProfile.schemaVersion');
    this.assertNonEmptyString(profile.clientId, 'clientProfile.clientId');
    this.assertNonEmptyString(profile.displayName, 'clientProfile.displayName');
    this.assertEnum(profile.operationalStatus, ['active', 'paused'], 'clientProfile.operationalStatus');

    this.validateClientDefaults(profile.defaults, 'clientProfile.defaults');

    if (profile.publicContact !== undefined) {
      this.validatePublicContact(profile.publicContact, 'clientProfile.publicContact');
    }
  }

  validateLocationProfile(profile: LocationProfile, client: ClientProfile): void {
    this.assertObject(profile, 'locationProfile');
    this.assertNoExtraKeys(profile, LOCATION_PROFILE_KEYS, 'locationProfile');

    this.assertExactString(profile.schemaVersion, SchemaVersion.LOCATION_PROFILE, 'locationProfile.schemaVersion');
    this.assertNonEmptyString(profile.locationId, 'locationProfile.locationId');
    this.assertNonEmptyString(profile.clientId, 'locationProfile.clientId');
    this.assertCondition(profile.clientId === client.clientId, 'validation_error', 'location_profile_client_mismatch');
    this.assertNonEmptyString(profile.displayName, 'locationProfile.displayName');
    this.assertValidTimezone(profile.timezone, 'locationProfile.timezone');
    this.assertNonEmptyString(profile.operatingMode, 'locationProfile.operatingMode');
    this.assertEnum(profile.operationalStatus, ['active', 'paused'], 'locationProfile.operationalStatus');

    this.assertEnum(
      profile.serviceDeliveryMode,
      ['inShop', 'onSite', 'both'],
      'locationProfile.serviceDeliveryMode',
      'missing_required_field',
    );

    this.validateServiceCatalogRef(
      profile.serviceCatalogRef,
      client.defaults,
      'locationProfile.serviceCatalogRef',
    );

    this.validateServiceAreaRef(
      profile.serviceAreaRef,
      client.defaults,
      'locationProfile.serviceAreaRef',
    );

    if (profile.businessHours !== undefined) {
      this.validateBusinessHours(profile.businessHours, 'locationProfile.businessHours');
    }

    if (profile.clientAlerting !== undefined) {
      this.validateClientAlerting(profile.clientAlerting, 'locationProfile.clientAlerting');
    }

    if (profile.customerSms !== undefined) {
      this.validateCustomerSms(profile.customerSms, 'locationProfile.customerSms');
    }

    if (profile.policyProfileRef !== undefined) {
      this.assertNoExtraKeys(profile.policyProfileRef, ['type'], 'locationProfile.policyProfileRef');
      this.assertExactString(profile.policyProfileRef.type, 'useClientDefault', 'locationProfile.policyProfileRef.type');
    }

    if (profile.publicContact !== undefined) {
      this.validatePublicContact(profile.publicContact, 'locationProfile.publicContact');
    }

    if (profile.locales !== undefined) {
      this.validateLocaleList(profile.locales, 'locationProfile.locales');
    }
  }

  validateServiceAreaRuleset(ruleset: ServiceAreaRuleset): void {
    this.assertObject(ruleset, 'serviceAreaRuleset');
    this.assertNoExtraKeys(ruleset, SERVICE_AREA_RULESET_KEYS, 'serviceAreaRuleset');

    this.assertNonEmptyString(ruleset.rulesetId, 'serviceAreaRuleset.rulesetId');

    if (ruleset.schemaVersion !== undefined) {
      this.assertExactString(ruleset.schemaVersion, SchemaVersion.SERVICE_AREA_RULES, 'serviceAreaRuleset.schemaVersion');
    }

    this.assertCondition(Array.isArray(ruleset.rules), 'validation_error', 'invalid_type', 'serviceAreaRuleset.rules must be an array');

    ruleset.rules.forEach((rule, index) => {
      this.validateServiceAreaRule(rule, `serviceAreaRuleset.rules[${index}]`);
    });
  }

  private validateClientDefaults(defaults: ClientProfile['defaults'], path: string): void {
    this.assertObject(defaults, path);
    this.assertNoExtraKeys(defaults, CLIENT_DEFAULTS_KEYS, path);

    if (defaults.masterServiceCatalogRef !== undefined) {
      this.validateServiceCatalogRef(defaults.masterServiceCatalogRef, undefined, `${path}.masterServiceCatalogRef`);
    }

    if (defaults.masterServiceAreaRuleIds !== undefined) {
      this.assertStringArray(defaults.masterServiceAreaRuleIds, `${path}.masterServiceAreaRuleIds`);
    }

    if (defaults.timezone !== undefined) {
      this.assertValidTimezone(defaults.timezone, `${path}.timezone`);
    }

    if (defaults.policyProfileRef !== undefined) {
      this.assertNoExtraKeys(defaults.policyProfileRef, ['type'], `${path}.policyProfileRef`);
      this.assertExactString(defaults.policyProfileRef.type, 'useClientDefault', `${path}.policyProfileRef.type`);
    }
  }

  private validateServiceCatalogRef(
    ref: { type: string } & Record<string, unknown>,
    clientDefaults: ClientProfile['defaults'] | undefined,
    path: string,
  ): void {
    this.assertObject(ref, path);
    this.assertEnum(ref.type, ['byTag', 'inline', 'allFromClientDefault'], `${path}.type`);

    if (ref.type === 'byTag') {
      this.assertNoExtraKeys(ref, ['type', 'includeTags'], path);
      this.assertStringArray(ref.includeTags, `${path}.includeTags`);
    } else if (ref.type === 'inline') {
      this.assertNoExtraKeys(ref, ['type', 'items'], path);
      this.assertStringArray(ref.items, `${path}.items`);
    } else if (ref.type === 'allFromClientDefault') {
      this.assertNoExtraKeys(ref, ['type'], path);
      const master = clientDefaults?.masterServiceCatalogRef;
      this.assertCondition(
        master !== undefined,
        'validation_error',
        'missing_client_default_service_catalog',
        `${path} requires client.defaults.masterServiceCatalogRef`,
      );
    }
  }

  private validateServiceAreaRef(
    ref: LocationProfile['serviceAreaRef'],
    clientDefaults: ClientProfile['defaults'],
    path: string,
  ): void {
    this.assertObject(ref, path);
    this.assertNoExtraKeys(ref, ['ruleIds'], path);
    this.assertStringArray(ref.ruleIds, `${path}.ruleIds`);

    if (ref.ruleIds.length === 0) {
      const fallback = clientDefaults.masterServiceAreaRuleIds;
      this.assertCondition(
        Array.isArray(fallback) && fallback.length > 0,
        'validation_error',
        'missing_client_default_service_area',
        `${path}.ruleIds is empty and client.defaults.masterServiceAreaRuleIds is missing or empty`,
      );
    }
  }

  private validateBusinessHours(hours: BusinessHours, path: string): void {
    this.assertObject(hours, path);
    this.assertNoExtraKeys(hours, ['regular'], path);
    this.assertCondition(Array.isArray(hours.regular), 'validation_error', 'invalid_type', `${path}.regular must be an array`);

    hours.regular.forEach((day, index) => {
      const dayPath = `${path}.regular[${index}]`;
      this.assertObject(day, dayPath);
      this.assertNoExtraKeys(day, ['dayOfWeek', 'openWindows'], dayPath);
      this.assertNonEmptyString(day.dayOfWeek, `${dayPath}.dayOfWeek`);
      this.assertCondition(Array.isArray(day.openWindows), 'validation_error', 'invalid_type', `${dayPath}.openWindows must be an array`);
      day.openWindows.forEach((window, wIndex) => {
        const windowPath = `${dayPath}.openWindows[${wIndex}]`;
        this.assertObject(window, windowPath);
        this.assertNoExtraKeys(window, ['startLocalTime', 'endLocalTime'], windowPath);
        this.assertNonEmptyString(window.startLocalTime, `${windowPath}.startLocalTime`);
        this.assertNonEmptyString(window.endLocalTime, `${windowPath}.endLocalTime`);
      });
    });
  }

  private validateClientAlerting(alerting: ClientAlerting, path: string): void {
    this.assertObject(alerting, path);
    this.assertNoExtraKeys(alerting, ['recipients', 'routingPlan'], path);
    this.assertCondition(Array.isArray(alerting.recipients), 'validation_error', 'invalid_type', `${path}.recipients must be an array`);

    alerting.recipients.forEach((recipient, index) => {
      const rPath = `${path}.recipients[${index}]`;
      this.assertObject(recipient, rPath);
      this.assertNoExtraKeys(recipient, ['recipientId', 'displayName', 'smsE164', 'voiceE164', 'email', 'enabled'], rPath);
      this.assertNonEmptyString(recipient.recipientId, `${rPath}.recipientId`);
      this.assertNonEmptyString(recipient.displayName, `${rPath}.displayName`);
      if (recipient.smsE164 !== undefined) this.assertE164(recipient.smsE164, `${rPath}.smsE164`);
      if (recipient.voiceE164 !== undefined) this.assertE164(recipient.voiceE164, `${rPath}.voiceE164`);
      if (recipient.email !== undefined) this.assertNonEmptyString(recipient.email, `${rPath}.email`);
      this.assertCondition(typeof recipient.enabled === 'boolean', 'validation_error', 'invalid_type', `${rPath}.enabled must be boolean`);
    });

    this.validateRoutingPlan(alerting.routingPlan, `${path}.routingPlan`);
  }

  private validateRoutingPlan(plan: RoutingPlan, path: string): void {
    this.assertObject(plan, path);
    this.assertNoExtraKeys(plan, ['emergency', 'standard'], path);
    this.validateRoutingStep(plan.emergency, `${path}.emergency`);
    this.validateRoutingStep(plan.standard, `${path}.standard`);
  }

  private validateRoutingStep(step: RoutingPlan['emergency'], path: string): void {
    this.assertObject(step, path);
    this.assertNoExtraKeys(step, ['type', 'primaryRecipientId', 'secondaryRecipientId'], path);
    this.assertNonEmptyString(step.type, `${path}.type`);
    this.assertNonEmptyString(step.primaryRecipientId, `${path}.primaryRecipientId`);
    if (step.secondaryRecipientId !== undefined) {
      this.assertNonEmptyString(step.secondaryRecipientId, `${path}.secondaryRecipientId`);
    }
  }

  private validateCustomerSms(sms: CustomerSms, path: string): void {
    this.assertObject(sms, path);
    this.assertNoExtraKeys(sms, ['appointmentSms', 'intakeAcknowledgementSms'], path);
    if (sms.appointmentSms !== undefined) this.validateSmsToggle(sms.appointmentSms, `${path}.appointmentSms`);
    if (sms.intakeAcknowledgementSms !== undefined) this.validateSmsToggle(sms.intakeAcknowledgementSms, `${path}.intakeAcknowledgementSms`);
  }

  private validateSmsToggle(toggle: { enabled: boolean }, path: string): void {
    this.assertObject(toggle, path);
    this.assertNoExtraKeys(toggle, ['enabled'], path);
    this.assertCondition(typeof toggle.enabled === 'boolean', 'validation_error', 'invalid_type', `${path}.enabled must be boolean`);
  }

  private validatePublicContact(contact: PublicContact, path: string): void {
    this.assertObject(contact, path);
    this.assertNoExtraKeys(contact, ['phoneE164', 'address', 'website'], path);
    if (contact.phoneE164 !== undefined) this.assertE164(contact.phoneE164, `${path}.phoneE164`);
    if (contact.address !== undefined) this.assertNonEmptyString(contact.address, `${path}.address`);
    if (contact.website !== undefined) this.assertNonEmptyString(contact.website, `${path}.website`);
  }

  private validateLocaleList(locales: string[], path: string): void {
    this.assertCondition(Array.isArray(locales), 'validation_error', 'invalid_type', `${path} must be an array`);
    locales.forEach((locale, index) => {
      this.assertNonEmptyString(locale, `${path}[${index}]`);
    });
    const unique = new Set(locales);
    this.assertCondition(
      unique.size === locales.length,
      'validation_error',
      'invalid_locale_list',
      `${path} contains duplicate entries`,
    );
  }

  private validateServiceAreaRule(rule: ServiceAreaRule, path: string): void {
    this.assertObject(rule, path);
    this.assertNoExtraKeys(rule, SERVICE_AREA_RULE_KEYS, path);

    this.assertNonEmptyString(rule.ruleId, `${path}.ruleId`);
    this.assertNonEmptyString(rule.ruleType, `${path}.ruleType`);
    this.assertNonEmptyString(rule.displayName, `${path}.displayName`);
    this.assertCondition(typeof rule.enabled === 'boolean', 'validation_error', 'invalid_type', `${path}.enabled must be boolean`);

    if (rule.ruleType === 'zipCode') {
      this.assertCondition(
        Array.isArray(rule.zipCodes) && rule.zipCodes.length > 0,
        'validation_error',
        'invalid_selector_shape',
        `${path}.zipCodes is required for zipCode rules`,
      );
    }

    if (rule.ruleType === 'mileRadius') {
      this.assertCondition(
        rule.centerCoordinates !== undefined,
        'validation_error',
        'missing_center_coordinates',
        `${path}.centerCoordinates is required for mileRadius rules`,
      );

      const { lat, lng } = rule.centerCoordinates;
      this.assertCondition(
        typeof lat === 'number' && lat >= -90 && lat <= 90,
        'validation_error',
        'invalid_center_coordinates',
        `${path}.centerCoordinates.lat out of range`,
      );
      this.assertCondition(
        typeof lng === 'number' && lng >= -180 && lng <= 180,
        'validation_error',
        'invalid_center_coordinates',
        `${path}.centerCoordinates.lng out of range`,
      );
    }
  }

  private assertObject(value: unknown, path: string): void {
    this.assertCondition(
      value !== null && typeof value === 'object' && !Array.isArray(value),
      'validation_error',
      'invalid_type',
      `${path} must be an object`,
    );
  }

  private assertNoExtraKeys(value: object, allowed: string[], path: string): void {
    const actual = Object.keys(value);
    const extra = actual.filter((k) => !allowed.includes(k));
    this.assertCondition(
      extra.length === 0,
      'validation_error',
      'unexpected_field_present',
      `${path} contains unexpected fields: ${extra.join(', ')}`,
    );
  }

  private assertNonEmptyString(value: unknown, path: string): void {
    this.assertCondition(
      typeof value === 'string' && value.length > 0,
      'validation_error',
      'missing_required_field',
      `${path} is required and must be a non-empty string`,
    );
  }

  private assertExactString(value: unknown, expected: string, path: string): void {
    this.assertCondition(
      value === expected,
      'validation_error',
      'invalid_schema_version',
      `${path} must be exactly '${expected}'`,
    );
  }

  private assertEnum(
    value: unknown,
    allowed: string[],
    path: string,
    code: string = 'invalid_enum_value',
  ): void {
    this.assertCondition(
      typeof value === 'string' && allowed.includes(value),
      'validation_error',
      code,
      `${path} must be one of [${allowed.join(', ')}]`,
    );
  }

  private assertStringArray(value: unknown, path: string): void {
    this.assertCondition(
      Array.isArray(value) && value.every((v) => typeof v === 'string'),
      'validation_error',
      'invalid_type',
      `${path} must be an array of strings`,
    );
  }

  private assertE164(value: string, path: string): void {
    const e164Pattern = /^\+[1-9]\d{1,14}$/;
    this.assertCondition(
      e164Pattern.test(value),
      'validation_error',
      'invalid_e164_phone',
      `${path} must be a valid E.164 phone number`,
    );
  }

  private assertValidTimezone(value: string, path: string): void {
    try {
      Intl.DateTimeFormat(undefined, { timeZone: value });
    } catch {
      this.assertCondition(false, 'validation_error', 'invalid_timezone', `${path} must be a valid IANA timezone`);
    }
  }

  private assertCondition(
    condition: boolean,
    errorType: 'validation_error' | 'not_found' | 'unavailable' | 'permission_denied',
    errorCode: string,
    message: string,
  ): void {
    if (!condition) {
      throw new ConfigurationLoadError(errorType, errorCode, message);
    }
  }
}

const CLIENT_PROFILE_KEYS = [
  'schemaVersion',
  'clientId',
  'displayName',
  'operationalStatus',
  'defaults',
  'publicContact',
];

const CLIENT_DEFAULTS_KEYS = [
  'masterServiceCatalogRef',
  'masterServiceAreaRuleIds',
  'timezone',
  'policyProfileRef',
];

const LOCATION_PROFILE_KEYS = [
  'schemaVersion',
  'locationId',
  'clientId',
  'displayName',
  'timezone',
  'operatingMode',
  'operationalStatus',
  'serviceDeliveryMode',
  'businessHours',
  'serviceCatalogRef',
  'serviceAreaRef',
  'clientAlerting',
  'customerSms',
  'policyProfileRef',
  'publicContact',
  'locales',
];

const SERVICE_AREA_RULESET_KEYS = ['rulesetId', 'schemaVersion', 'rules'];

const SERVICE_AREA_RULE_KEYS = [
  'ruleId',
  'ruleType',
  'displayName',
  'description',
  'zipCodes',
  'centerAddress',
  'centerCoordinates',
  'radiusMiles',
  'order',
  'enabled',
];
```

---

## caching/multi-tenant-cache.registry.ts

```typescript
/**
 * In-memory cache registry with TTL.
 * Holds validated canonical contexts only; it never mutates them or
 * bypasses validation for new entries.
 */
interface CacheEntry<T> {
  value: T;
  expiresAt: number;
}

export class MultiTenantCacheRegistry {
  private readonly store = new Map<string, CacheEntry<unknown>>();

  constructor(private readonly ttlMs: number) {}

  get<T>(key: string): T | undefined {
    const entry = this.store.get(key);
    if (entry === undefined) return undefined;
    if (Date.now() >= entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value as T;
  }

  set<T>(key: string, value: T): void {
    this.store.set(key, {
      value,
      expiresAt: Date.now() + this.ttlMs,
    });
  }

  clear(): void {
    this.store.clear();
  }

  size(): number {
    return this.store.size;
  }
}
```

---

## adapters/in-memory-config-store.adapter.ts

```typescript
import type { ConfigStoreAdapterInterface } from '../contracts/config-store-adapter.contract';

/**
 * Development/test adapter backed by an in-memory fixture map.
 * Owns transport only; no validation or interpretation.
 */
export class InMemoryConfigStoreAdapter implements ConfigStoreAdapterInterface {
  constructor(
    private readonly fixtures: {
      clients?: Record<string, unknown>;
      locations?: Record<string, unknown>;
      rulesets?: Record<string, unknown>;
    } = {},
  ) {}

  async getClientProfile(clientId: string): Promise<unknown | null> {
    return this.fixtures.clients?.[clientId] ?? null;
  }

  async getLocationProfile(locationId: string): Promise<unknown | null> {
    return this.fixtures.locations?.[locationId] ?? null;
  }

  async getServiceAreaRuleset(rulesetId: string): Promise<unknown | null> {
    return this.fixtures.rulesets?.[rulesetId] ?? null;
  }
}
```

---

## loader/configuration-loader.gateway.ts

```typescript
import type { ConfigStoreAdapterInterface } from '../contracts/config-store-adapter.contract';
import type { ConfigurationLoadResult } from '../contracts/configuration-load-result.contract';
import { ConfigurationLoadError } from '../errors/configuration-load.error';
import type { LoadRequest } from '../models';
import type { ClientProfile, LocationProfile, ServiceAreaRuleset } from '../models';
import { ContextCompilationEngine } from '../compilation/context-compilation.engine';
import { StructuralVerificationGate } from '../validation/structural-verification.gate';
import { MultiTenantCacheRegistry } from '../caching/multi-tenant-cache.registry';

/**
 * Configuration Loader Gateway.
 * Single entry point for loading, canonicalizing, validating, and caching
 * multi-tenant configuration before any downstream module consumes it.
 */
export class ConfigurationLoaderGateway {
  private readonly compiler = new ContextCompilationEngine();
  private readonly verifier = new StructuralVerificationGate();

  constructor(
    private readonly store: ConfigStoreAdapterInterface,
    private readonly cache: MultiTenantCacheRegistry,
  ) {}

  async load(request: LoadRequest): Promise<ConfigurationLoadResult> {
    const { clientId, locationId, correlationKey } = request;

    if (!clientId || typeof clientId !== 'string') {
      return this.fail('validation_error', 'missing_required_field', 'clientId is required', correlationKey);
    }
    if (!locationId || typeof locationId !== 'string') {
      return this.fail('validation_error', 'missing_required_field', 'locationId is required', correlationKey);
    }

    const trimmedClientId = clientId.trim();
    const trimmedLocationId = locationId.trim();

    if (trimmedClientId.length === 0) {
      return this.fail('validation_error', 'missing_required_field', 'clientId cannot be empty', correlationKey);
    }
    if (trimmedLocationId.length === 0) {
      return this.fail('validation_error', 'missing_required_field', 'locationId cannot be empty', correlationKey);
    }

    const rulesetId = this.resolvePrimaryRulesetId(trimmedClientId, trimmedLocationId);
    const cacheKey = `${trimmedClientId}::${trimmedLocationId}::${rulesetId ?? 'no-ruleset'}`;

    const cached = this.cache.get<ConfigurationLoadResult>(cacheKey);
    if (cached !== undefined) {
      return this.withCacheHit(cached);
    }

    try {
      const [rawClient, rawLocation] = await Promise.all([
        this.store.getClientProfile(trimmedClientId),
        this.store.getLocationProfile(trimmedLocationId),
      ]);

      if (rawClient === null) {
        return this.fail('not_found', 'config_record_not_found', `ClientProfile not found: ${trimmedClientId}`, correlationKey);
      }
      if (rawLocation === null) {
        return this.fail('not_found', 'config_record_not_found', `LocationProfile not found: ${trimmedLocationId}`, correlationKey);
      }

      const clientProfile = this.compiler.compileClientProfile(rawClient);
      const locationProfile = this.compiler.compileLocationProfile(rawLocation);

      const effectiveRulesetId = this.resolveRulesetId(locationProfile, clientProfile);
      let serviceAreaRuleset: ServiceAreaRuleset | undefined;

      if (effectiveRulesetId !== undefined) {
        const rawRuleset = await this.store.getServiceAreaRuleset(effectiveRulesetId);
        if (rawRuleset === null) {
          return this.fail(
            'not_found',
            'config_record_not_found',
            `ServiceAreaRuleset not found: ${effectiveRulesetId}`,
            correlationKey,
          );
        }
        serviceAreaRuleset = this.compiler.compileServiceAreaRuleset(rawRuleset);
      }

      this.verifier.validateClientProfile(clientProfile);
      this.verifier.validateLocationProfile(locationProfile, clientProfile);
      if (serviceAreaRuleset !== undefined) {
        this.verifier.validateServiceAreaRuleset(serviceAreaRuleset);
      }

      const result: ConfigurationLoadResult = {
        ok: true,
        clientProfile,
        locationProfile,
        ...(serviceAreaRuleset && { serviceAreaRuleset }),
        metadata: {
          loadedAtUtc: new Date().toISOString(),
          cacheHit: false,
        },
      };

      this.cache.set(cacheKey, result);
      return result;
    } catch (error) {
      if (error instanceof ConfigurationLoadError) {
        return this.fail(error.errorType, error.errorCode, error.message, correlationKey, error.details);
      }
      return this.fail('unavailable', 'config_store_unavailable', normalizeError(error), correlationKey);
    }
  }

  private resolvePrimaryRulesetId(clientId: string, locationId: string): string | undefined {
    // Cache key needs a stable identifier before loading; the actual ruleset
    // is resolved after client/location are canonicalized. This preliminary
    // value is only used for cache lookup; a mismatch would result in a cache miss.
    return undefined;
  }

  private resolveRulesetId(
    location: LocationProfile,
    client: ClientProfile,
  ): string | undefined {
    if (location.serviceAreaRef.ruleIds.length > 0) {
      return location.serviceAreaRef.ruleIds[0];
    }
    const fallback = client.defaults.masterServiceAreaRuleIds;
    if (fallback !== undefined && fallback.length > 0) {
      return fallback[0];
    }
    return undefined;
  }

  private withCacheHit(result: ConfigurationLoadResult): ConfigurationLoadResult {
    if (!result.ok) return result;
    return {
      ...result,
      metadata: {
        ...result.metadata,
        cacheHit: true,
      },
    };
  }

  private fail(
    errorType: 'validation_error' | 'not_found' | 'unavailable' | 'permission_denied',
    errorCode: string,
    message: string,
    correlationKey?: string,
    details?: unknown,
  ): ConfigurationLoadResult {
    return {
      ok: false,
      errorType,
      errorCode,
      message,
      details: {
        ...(correlationKey && { correlationKey }),
        ...(details ?? {}),
      },
    };
  }
}

function normalizeError(error: unknown): string {
  if (error instanceof Error) return error.message;
  return String(error);
}
```

---

## tests/configuration-loader.tests.ts

```typescript
import { ConfigurationLoaderGateway } from '../loader/configuration-loader.gateway';
import { InMemoryConfigStoreAdapter } from '../adapters/in-memory-config-store.adapter';
import { MultiTenantCacheRegistry } from '../caching/multi-tenant-cache.registry';
import type { ConfigurationLoadResult } from '../contracts/configuration-load-result.contract';
import type { LoadRequest } from '../models';

const FIXTURE_A = {
  client: {
    schemaVersion: 'ClientProfile.v1',
    clientId: 'client_abc',
    displayName: 'Acme Plumbing',
    operationalStatus: 'active',
    defaults: {
      masterServiceCatalogRef: { type: 'byTag', includeTags: ['plumbing'] },
      masterServiceAreaRuleIds: ['area_zip_60601'],
    },
  },
  location: {
    schemaVersion: 'LocationProfile.v1',
    locationId: 'loc_001',
    clientId: 'client_abc',
    displayName: 'Downtown Branch',
    timezone: 'America/Chicago',
    operatingMode: 'booking-enabled',
    operationalStatus: 'active',
    serviceDeliveryMode: 'both',
    businessHours: {
      regular: [
        { dayOfWeek: 'mon', openWindows: [{ startLocalTime: '09:00', endLocalTime: '17:00' }] },
        { dayOfWeek: 'tue', openWindows: [{ startLocalTime: '09:00', endLocalTime: '17:00' }] },
        { dayOfWeek: 'wed', openWindows: [] },
        { dayOfWeek: 'thu', openWindows: [] },
        { dayOfWeek: 'fri', openWindows: [] },
        { dayOfWeek: 'sat', openWindows: [] },
        { dayOfWeek: 'sun', openWindows: [] },
      ],
    },
    serviceCatalogRef: { type: 'inline', items: ['svc_plumbing'] },
    serviceAreaRef: { ruleIds: ['area_zip_60601'] },
    clientAlerting: {
      recipients: [
        {
          recipientId: 'tech_1',
          displayName: 'On-call Tech',
          smsE164: '+15551234567',
          voiceE164: '+15557654321',
          enabled: true,
        },
      ],
      routingPlan: {
        emergency: { type: 'primaryThenSecondary', primaryRecipientId: 'tech_1' },
        standard: { type: 'primaryThenSecondary', primaryRecipientId: 'tech_1' },
      },
    },
    customerSms: {
      appointmentSms: { enabled: true },
      intakeAcknowledgementSms: { enabled: false },
    },
    policyProfileRef: { type: 'useClientDefault' },
  },
  serviceAreaRuleset: {
    rulesetId: 'area_zip_60601',
    schemaVersion: 'ServiceAreaRules.v1',
    rules: [
      {
        ruleId: 'chicago_core',
        ruleType: 'zipCode',
        displayName: 'Chicago Core ZIP Codes',
        zipCodes: ['60601', '60602', '60603'],
        order: 1,
        enabled: true,
      },
    ],
  },
};

function createLoader(fixtures: Record<string, unknown>) {
  const store = new InMemoryConfigStoreAdapter({
    clients: { client_abc: fixtures.client },
    locations: { loc_001: fixtures.location },
    rulesets: { area_zip_60601: fixtures.serviceAreaRuleset, area_radius_25mi: fixtures.serviceAreaRuleset },
  });
  const cache = new MultiTenantCacheRegistry(60_000);
  return new ConfigurationLoaderGateway(store, cache);
}

function assertOk(result: ConfigurationLoadResult) {
  if (!result.ok) {
    throw new Error(`Expected ok result but got error: ${result.errorCode} - ${result.message}`);
  }
}

function assertError(
  result: ConfigurationLoadResult,
  expectedType: string,
  expectedCode: string,
) {
  if (result.ok) {
    throw new Error(`Expected error ${expectedCode} but got ok result`);
  }
  if (result.errorType !== expectedType || result.errorCode !== expectedCode) {
    throw new Error(`Expected ${expectedType}/${expectedCode}, got ${result.errorType}/${result.errorCode}`);
  }
}

async function runTests() {
  // TC-CONFIG-01: Valid load returns canonical objects
  {
    const loader = createLoader(FIXTURE_A);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertOk(result);
    console.log('TC-CONFIG-01 passed');
  }

  // TC-CONFIG-02: Default operationalStatus applied
  {
    const fixtures = structuredClone(FIXTURE_A) as any;
    delete fixtures.client.operationalStatus;
    delete fixtures.location.operationalStatus;
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertOk(result);
    if (result.clientProfile.operationalStatus !== 'active') throw new Error('client operationalStatus not defaulted');
    if (result.locationProfile.operationalStatus !== 'active') throw new Error('location operationalStatus not defaulted');
    console.log('TC-CONFIG-02 passed');
  }

  // TC-CONFIG-03: Reject missing LocationProfile.serviceDeliveryMode
  {
    const fixtures = structuredClone(FIXTURE_A) as any;
    delete fixtures.location.serviceDeliveryMode;
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'validation_error', 'missing_required_field');
    console.log('TC-CONFIG-03 passed');
  }

  // TC-CONFIG-04: Not found returns fail-closed
  {
    const loader = createLoader(FIXTURE_A);
    const result = await loader.load({ clientId: 'client_missing', locationId: 'loc_001' });
    assertError(result, 'not_found', 'config_record_not_found');
    console.log('TC-CONFIG-04 passed');
  }

  // TC-CONFIG-05: Adapter unavailable returns unavailable
  {
    const store = new InMemoryConfigStoreAdapter();
    const cache = new MultiTenantCacheRegistry(60_000);
    const loader = new ConfigurationLoaderGateway(
      {
        getClientProfile: () => Promise.reject(new Error('timeout')),
        getLocationProfile: () => Promise.reject(new Error('timeout')),
        getServiceAreaRuleset: () => Promise.reject(new Error('timeout')),
      },
      cache,
    );
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'unavailable', 'config_store_unavailable');
    console.log('TC-CONFIG-05 passed');
  }

  // TC-CONFIG-06: Service catalog inheritance requirement enforced
  {
    const fixtures = structuredClone(FIXTURE_A) as any;
    fixtures.location.serviceCatalogRef = { type: 'allFromClientDefault' };
    delete fixtures.client.defaults.masterServiceCatalogRef;
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'validation_error', 'missing_client_default_service_catalog');
    console.log('TC-CONFIG-06 passed');
  }

  // TC-CONFIG-07: Service area fallback requirement enforced
  {
    const fixtures = structuredClone(FIXTURE_A) as any;
    fixtures.location.serviceAreaRef = { ruleIds: [] };
    delete fixtures.client.defaults.masterServiceAreaRuleIds;
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'validation_error', 'missing_client_default_service_area');
    console.log('TC-CONFIG-07 passed');
  }

  // TC-CONFIG-08: Mile radius with valid centerCoordinates passes
  {
    const fixtures = {
      client: {
        schemaVersion: 'ClientProfile.v1',
        clientId: 'client_abc',
        displayName: 'Acme Plumbing',
        operationalStatus: 'active',
        defaults: { masterServiceAreaRuleIds: ['area_radius_25mi'] },
      },
      location: {
        schemaVersion: 'LocationProfile.v1',
        locationId: 'loc_001',
        clientId: 'client_abc',
        displayName: 'Downtown Branch',
        timezone: 'America/Chicago',
        operatingMode: 'booking-enabled',
        operationalStatus: 'active',
        serviceDeliveryMode: 'both',
        serviceAreaRef: { ruleIds: ['area_radius_25mi'] },
      },
      serviceAreaRuleset: {
        rulesetId: 'area_radius_25mi',
        schemaVersion: 'ServiceAreaRules.v1',
        rules: [
          {
            ruleId: '25_mile_radius',
            ruleType: 'mileRadius',
            displayName: '25-Mile Radius from Downtown',
            centerAddress: '123 Main St, Chicago, IL 60601',
            centerCoordinates: { lat: 41.8781, lng: -87.6298 },
            radiusMiles: 25,
            order: 1,
            enabled: true,
          },
        ],
      },
    };
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertOk(result);
    console.log('TC-CONFIG-08 passed');
  }

  // TC-CONFIG-09: Mile radius missing centerCoordinates fails
  {
    const fixtures = structuredClone((await import('node:fs')).existsSync ? {} : {}) as any;
    // Inline to avoid fs dependency
    const base = {
      client: {
        schemaVersion: 'ClientProfile.v1',
        clientId: 'client_abc',
        displayName: 'Acme Plumbing',
        operationalStatus: 'active',
        defaults: { masterServiceAreaRuleIds: ['area_radius_25mi'] },
      },
      location: {
        schemaVersion: 'LocationProfile.v1',
        locationId: 'loc_001',
        clientId: 'client_abc',
        displayName: 'Downtown Branch',
        timezone: 'America/Chicago',
        operatingMode: 'booking-enabled',
        operationalStatus: 'active',
        serviceDeliveryMode: 'both',
        serviceAreaRef: { ruleIds: ['area_radius_25mi'] },
      },
      serviceAreaRuleset: {
        rulesetId: 'area_radius_25mi',
        rules: [
          {
            ruleId: '25_mile_radius',
            ruleType: 'mileRadius',
            displayName: '25-Mile Radius from Downtown',
            centerAddress: '123 Main St, Chicago, IL 60601',
            radiusMiles: 25,
            order: 1,
            enabled: true,
          },
        ],
      },
    };
    const loader = createLoader(base);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'validation_error', 'missing_center_coordinates');
    console.log('TC-CONFIG-09 passed');
  }

  // TC-CONFIG-10: Invalid centerCoordinates (out of range) fails
  {
    const fixtures = {
      client: {
        schemaVersion: 'ClientProfile.v1',
        clientId: 'client_abc',
        displayName: 'Acme Plumbing',
        operationalStatus: 'active',
        defaults: { masterServiceAreaRuleIds: ['area_radius_25mi'] },
      },
      location: {
        schemaVersion: 'LocationProfile.v1',
        locationId: 'loc_001',
        clientId: 'client_abc',
        displayName: 'Downtown Branch',
        timezone: 'America/Chicago',
        operatingMode: 'booking-enabled',
        operationalStatus: 'active',
        serviceDeliveryMode: 'both',
        serviceAreaRef: { ruleIds: ['area_radius_25mi'] },
      },
      serviceAreaRuleset: {
        rulesetId: 'area_radius_25mi',
        rules: [
          {
            ruleId: '25_mile_radius',
            ruleType: 'mileRadius',
            displayName: '25-Mile Radius from Downtown',
            centerAddress: '123 Main St, Chicago, IL 60601',
            centerCoordinates: { lat: 100, lng: -200 },
            radiusMiles: 25,
            order: 1,
            enabled: true,
          },
        ],
      },
    };
    const loader = createLoader(fixtures);
    const result = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertError(result, 'validation_error', 'invalid_center_coordinates');
    console.log('TC-CONFIG-10 passed');
  }

  // Cache determinism: identical inputs produce cache hit on second call
  {
    const loader = createLoader(FIXTURE_A);
    const first = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    const second = await loader.load({ clientId: 'client_abc', locationId: 'loc_001' });
    assertOk(first);
    assertOk(second);
    if (!second.metadata.cacheHit) throw new Error('Expected cache hit on second load');
    console.log('Cache determinism passed');
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

| Layer 3 Component | Implementation Unit |
|---|---|
| Configuration Load Result Contract | `contracts/configuration-load-result.contract.ts` |
| Configuration Store Adapter Contract | `contracts/config-store-adapter.contract.ts` |
| Schema Version Contract | `contracts/schema-version.contract.ts` |
| Configuration Loader Gateway | `loader/configuration-loader.gateway.ts` |
| Structural Verification Gate | `validation/structural-verification.gate.ts` |
| Context Compilation Engine | `compilation/context-compilation.engine.ts` |
| Multi-Tenant Cache Registry | `caching/multi-tenant-cache.registry.ts` |
| Storage Adapter implementations | `adapters/in-memory-config-store.adapter.ts` |
| Contract / Boundary / Failure / Adapter Isolation Tests | `tests/configuration-loader.tests.ts` |

## Design Decisions Captured

- **Defaulting before validation**: The Layer 3 construction checklist explicitly requires defaulting before validation; the runtime flow diagram is interpreted conceptually. Without this ordering, defaultable-but-missing fields such as `operationalStatus` would fail.
- **Ruleset resolution**: The location’s `serviceAreaRef.ruleIds` array is treated as an ordered list of ruleset references. The MVP loads the first referenced ruleset; if the location list is empty, the client default fallback is used.
- **No timezone inheritance**: `LocationProfile.timezone` is validated independently; `ClientProfile.defaults.timezone` is never used as a fallback.
- **Immutable outputs**: Canonical objects are frozen before handoff to prevent downstream mutation.
- **Backend-neutral storage**: Domain code depends only on `ConfigStoreAdapterInterface`; no vendor identifiers appear in contracts or models.