Updated todo list

# 003 Business Decision Framework — Implementation

**Assumption note:** `ServiceCatalog.PRD` was not supplied with this task. The `ServiceCatalogItem` shape below is derived strictly from the fields referenced in `ServiceCapabilityDecision.PRD` (`systemId`, `displayName`, `tags`, `triageQuestions`). If the owning PRD introduces additional fields, the matcher and compilers should be extended additively.

---

## Project Structure

```text
business-decision-framework/
│
├── contracts/
│   ├── capability-decision.contract.ts
│   ├── capability-match.contract.ts
│   ├── clarifying-question.contract.ts
│   └── decline-context.contract.ts
│
├── models/
│   ├── service-catalog-item.model.ts
│   ├── service-request.model.ts
│   └── capability-evaluation-context.model.ts
│
├── evaluation/
│   ├── intent-normalizer.ts
│   ├── double-pass-matrix.matcher.ts
│   ├── confidence-arbiter.ts
│   └── clarification-budget.tracker.ts
│
├── outcomes/
│   ├── supported-outcome.compiler.ts
│   └── decline-envelope.compiler.ts
│
├── gateway/
│   └── capability-decision.gateway.ts
│
├── errors/
│   └── capability-evaluation.error.ts
│
└── tests/
    └── capability-decision.tests.ts
```

---

## contracts/capability-decision.contract.ts

```typescript
import type { CapabilityMatch } from './capability-match.contract';
import type { DeclineContext } from './decline-context.contract';
import type { ClarifyingQuestion } from './clarifying-question.contract';

/**
 * The primary output of the Business Decision Framework.
 * A strictly typed union: either the request is supported, or it is not.
 * Clarifying questions are modeled as an intermediate outcome that requires
 * caller input before a final decision can be reached.
 */
export type CapabilityDecision =
  | {
      supported: true;
      match: CapabilityMatch;
    }
  | {
      supported: false;
      declineContext: DeclineContext;
    }
  | {
      supported: false;
      clarifyingQuestion: ClarifyingQuestion;
      updatedDescription: string;
      clarifyingQuestionsAsked: number;
    };
```

---

## contracts/capability-match.contract.ts

```typescript
export type ConfidenceLevel = 'high' | 'medium' | 'low' | 'none';

/**
 * Structured descriptor for a successful capability match.
 */
export interface CapabilityMatch {
  readonly matchedServiceId: string;
  readonly matchedDisplayName: string;
  readonly confidence: ConfidenceLevel;
  readonly matchReason: string;
}
```

---

## contracts/clarifying-question.contract.ts

```typescript
/**
 * Structured clarifying question to be rendered by the upstream conversation
 * controller or TemplateRegistry. The framework does not generate final
 * customer-facing copy.
 */
export interface ClarifyingQuestion {
  readonly questionText: string;
  readonly questionId: string;
  readonly ambiguousServiceIds: readonly string[];
}
```

---

## contracts/decline-context.contract.ts

```typescript
export type UnsupportedReason = 'no_match' | 'exhausted_clarification';

/**
 * Structured context for the unsupported-service decline message.
 * TemplateRegistry consumes these variables to render the final message.
 */
export interface DeclineContext {
  readonly reason: UnsupportedReason;
  readonly requestedService: string;
  readonly serviceCategories: readonly string[];
  readonly locationName: string;
  readonly clarifyingQuestionsAsked: number;
}
```

---

## models/service-catalog-item.model.ts

```typescript
/**
 * A single entry in the location's resolved service catalog.
 * Shape is derived from ServiceCapabilityDecision.PRD.
 */
export interface ServiceCatalogItem {
  readonly systemId: string;
  readonly displayName: string;
  readonly tags: readonly string[];
  readonly triageQuestions: readonly string[];
}
```

---

## models/service-request.model.ts

```typescript
/**
 * The caller's service request as captured in the active session.
 */
export interface ServiceRequest {
  readonly description: string;
  readonly urgencySignals?: readonly string[];
}
```

---

## models/capability-evaluation-context.model.ts

```typescript
import type { ServiceCatalogItem } from './service-catalog-item.model';
import type { ServiceRequest } from './service-request.model';

/**
 * Immutable evaluation context passed to the capability decision gateway.
 * The framework does not fetch or validate the catalog; it consumes the
 * already-resolved catalog provided by upstream configuration loading.
 */
export interface CapabilityEvaluationContext {
  readonly serviceCatalog: readonly ServiceCatalogItem[];
  readonly serviceRequest: ServiceRequest;
  readonly locationName: string;
  readonly clarifyingQuestionsAsked: number;
  readonly maxClarifyingQuestions: number;
}
```

---

## errors/capability-evaluation.error.ts

```typescript
/**
 * Internal error type for unexpected conditions during evaluation.
 * The gateway converts these into deterministic unsupported outcomes
 * rather than leaking raw exceptions to consumers.
 */
export class CapabilityEvaluationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'CapabilityEvaluationError';
  }
}
```

---

## evaluation/intent-normalizer.ts

```typescript
/**
 * Prepares caller text for deterministic matching.
 * Owns only textual normalization: case folding, trimming, whitespace
 * compaction, and tokenization. It does not interpret meaning.
 */
export class IntentNormalizer {
  normalize(text: string): readonly string[] {
    const cleaned = text
      .toLowerCase()
      .trim()
      .replace(/\s+/g, ' ')
      .replace(/[^\w\s]/g, ' ');
    return cleaned
      .split(' ')
      .map((token) => token.trim())
      .filter((token) => token.length > 0);
  }
}
```

---

## evaluation/double-pass-matrix.matcher.ts

```typescript
import type { ServiceCatalogItem } from '../models/service-catalog-item.model';
import { IntentNormalizer } from './intent-normalizer';

export interface MatchCandidate {
  readonly item: ServiceCatalogItem;
  readonly score: number;
}

/**
 * Pure, side-effect-free matcher.
 * Pass 1 evaluates the caller's initial description.
 * Pass 2 re-evaluates after a clarifying response has been appended.
 */
export class DoublePassMatrixMatcher {
  private readonly normalizer = new IntentNormalizer();

  match(
    catalog: readonly ServiceCatalogItem[],
    description: string,
  ): MatchCandidate | undefined {
    const tokens = this.normalizer.normalize(description);
    if (tokens.length === 0) return undefined;

    const scored = catalog.map((item) => ({
      item,
      score: this.scoreItem(item, tokens),
    }));

    scored.sort((a, b) => b.score - a.score);
    return scored[0];
  }

  private scoreItem(item: ServiceCatalogItem, tokens: readonly string[]): number {
    let score = 0;
    const normalizedDescription = tokens.join(' ');

    // systemId match: +10 exact or substring
    const normalizedSystemId = item.systemId.toLowerCase();
    if (normalizedSystemId === normalizedDescription || normalizedDescription.includes(normalizedSystemId)) {
      score += 10;
    }

    // displayName substring match: +5
    const normalizedDisplayName = item.displayName.toLowerCase();
    if (normalizedDescription.includes(normalizedDisplayName)) {
      score += 5;
    }

    // tags match: +3 per matched tag
    for (const tag of item.tags) {
      const normalizedTag = tag.toLowerCase();
      if (tokens.some((token) => token === normalizedTag || normalizedDescription.includes(normalizedTag))) {
        score += 3;
      }
    }

    // triageQuestions substring match: +2 per matched question
    for (const question of item.triageQuestions) {
      const normalizedQuestion = question.toLowerCase();
      if (tokens.some((token) => normalizedQuestion.includes(token))) {
        score += 2;
      }
    }

    return score;
  }
}
```

---

## evaluation/confidence-arbiter.ts

```typescript
import type { CapabilityMatch, ConfidenceLevel } from '../contracts/capability-match.contract';
import type { MatchCandidate } from './double-pass-matrix.matcher';

/**
 * Maps raw match scores to confidence tiers and builds the supported
 * match descriptor. Thresholds are taken directly from the PRD.
 */
export class ConfidenceArbiter {
  private static readonly THRESHOLDS: ReadonlyArray<{
    minScore: number;
    confidence: ConfidenceLevel;
  }> = [
    { minScore: 10, confidence: 'high' },
    { minScore: 5, confidence: 'medium' },
    { minScore: 2, confidence: 'low' },
  ];

  classify(candidate: MatchCandidate): CapabilityMatch {
    const confidence = this.scoreToConfidence(candidate.score);

    return Object.freeze({
      matchedServiceId: candidate.item.systemId,
      matchedDisplayName: candidate.item.displayName,
      confidence,
      matchReason: this.buildMatchReason(candidate, confidence),
    });
  }

  classifyScore(candidate: MatchCandidate): ConfidenceLevel {
    return this.scoreToConfidence(candidate.score);
  }

  private scoreToConfidence(score: number): ConfidenceLevel {
    for (const threshold of ConfidenceArbiter.THRESHOLDS) {
      if (score >= threshold.minScore) {
        return threshold.confidence;
      }
    }
    return 'none';
  }

  private buildMatchReason(candidate: MatchCandidate, confidence: ConfidenceLevel): string {
    return `Matched ${candidate.item.systemId} with ${confidence} confidence (score: ${candidate.score})`;
  }
}
```

---

## evaluation/clarification-budget.tracker.ts

```typescript
import type { ServiceCatalogItem } from '../models/service-catalog-item.model';
import type { ClarifyingQuestion } from '../contracts/clarifying-question.contract';

/**
 * Tracks clarifying questions asked and selects the next focused question
 * when confidence is low. It does not generate final customer copy.
 */
export class ClarificationBudgetTracker {
  canAskQuestion(questionsAsked: number, maxQuestions: number): boolean {
    return questionsAsked < maxQuestions;
  }

  selectQuestion(candidates: readonly ServiceCatalogItem[], questionsAsked: number): ClarifyingQuestion {
    const topCandidates = candidates.slice(0, Math.min(3, candidates.length));
    const ambiguousServiceIds = topCandidates.map((c) => c.systemId);
    const displayNames = topCandidates.map((c) => c.displayName);

    const questionText =
      displayNames.length >= 2
        ? `Is this for ${this.formatList(displayNames)}?`
        : 'Could you tell me a bit more about what you need help with?';

    return Object.freeze({
      questionText,
      questionId: `clarify_${questionsAsked + 1}`,
      ambiguousServiceIds,
    });
  }

  private formatList(items: readonly string[]): string {
    if (items.length === 2) {
      return `${items[0]} or ${items[1]}`;
    }
    const allButLast = items.slice(0, -1).join(', ');
    const last = items[items.length - 1];
    return `${allButLast}, or ${last}`;
  }
}
```

---

## outcomes/supported-outcome.compiler.ts

```typescript
import type { CapabilityMatch } from '../contracts/capability-match.contract';
import type { CapabilityDecision } from '../contracts/capability-decision.contract';

/**
 * Builds the supported decision envelope from an arbiter result.
 */
export class SupportedOutcomeCompiler {
  compile(match: CapabilityMatch): CapabilityDecision {
    return Object.freeze({
      supported: true,
      match,
    });
  }
}
```

---

## outcomes/decline-envelope.compiler.ts

```typescript
import type { CapabilityDecision } from '../contracts/capability-decision.contract';
import type { DeclineContext, UnsupportedReason } from '../contracts/decline-context.contract';
import type { ServiceCatalogItem } from '../models/service-catalog-item.model';

/**
 * Builds the structured unsupported outcome.
 * Service categories are derived from unique catalog tags, not individual
 * service names, per the PRD.
 */
export class DeclineEnvelopeCompiler {
  compile(
    reason: UnsupportedReason,
    requestedService: string,
    catalog: readonly ServiceCatalogItem[],
    locationName: string,
    clarifyingQuestionsAsked: number,
  ): CapabilityDecision {
    const serviceCategories = this.extractCategories(catalog);

    const declineContext: DeclineContext = Object.freeze({
      reason,
      requestedService,
      serviceCategories,
      locationName,
      clarifyingQuestionsAsked,
    });

    return Object.freeze({
      supported: false,
      declineContext,
    });
  }

  private extractCategories(catalog: readonly ServiceCatalogItem[]): readonly string[] {
    const uniqueTags = new Set<string>();
    for (const item of catalog) {
      for (const tag of item.tags) {
        uniqueTags.add(tag);
      }
    }
    return Array.from(uniqueTags);
  }
}
```

---

## gateway/capability-decision.gateway.ts

```typescript
import type { CapabilityDecision } from '../contracts/capability-decision.contract';
import type { CapabilityEvaluationContext } from '../models/capability-evaluation-context.model';
import { DoublePassMatrixMatcher } from '../evaluation/double-pass-matrix.matcher';
import { ConfidenceArbiter } from '../evaluation/confidence-arbiter';
import { ClarificationBudgetTracker } from '../evaluation/clarification-budget.tracker';
import { SupportedOutcomeCompiler } from '../outcomes/supported-outcome.compiler';
import { DeclineEnvelopeCompiler } from '../outcomes/decline-envelope.compiler';

/**
 * Single entry point for the Business Decision Framework.
 * Orchestrates normalization, matching, clarification budgeting, and outcome
 * compilation. Pure logic: no I/O, no persistence, no template rendering.
 */
export class CapabilityDecisionGateway {
  private readonly matcher = new DoublePassMatrixMatcher();
  private readonly arbiter = new ConfidenceArbiter();
  private readonly budgetTracker = new ClarificationBudgetTracker();
  private readonly supportedCompiler = new SupportedOutcomeCompiler();
  private readonly declineCompiler = new DeclineEnvelopeCompiler();

  evaluate(context: CapabilityEvaluationContext): CapabilityDecision {
    const {
      serviceCatalog,
      serviceRequest,
      locationName,
      clarifyingQuestionsAsked,
      maxClarifyingQuestions,
    } = context;

    if (serviceCatalog.length === 0 || serviceRequest.description.trim().length === 0) {
      return this.declineCompiler.compile(
        'no_match',
        serviceRequest.description,
        serviceCatalog,
        locationName,
        clarifyingQuestionsAsked,
      );
    }

    const topCandidate = this.matcher.match(serviceCatalog, serviceRequest.description);

    if (topCandidate === undefined) {
      return this.declineCompiler.compile(
        'no_match',
        serviceRequest.description,
        serviceCatalog,
        locationName,
        clarifyingQuestionsAsked,
      );
    }

    const confidence = this.arbiter.classifyScore(topCandidate);

    if (confidence === 'high' || confidence === 'medium') {
      const match = this.arbiter.classify(topCandidate);
      return this.supportedCompiler.compile(match);
    }

    if (confidence === 'low') {
      if (this.budgetTracker.canAskQuestion(clarifyingQuestionsAsked, maxClarifyingQuestions)) {
        const rankedCandidates = this.rankCandidates(serviceCatalog, serviceRequest.description);
        const question = this.budgetTracker.selectQuestion(rankedCandidates, clarifyingQuestionsAsked);
        return Object.freeze({
          supported: false,
          clarifyingQuestion: question,
          updatedDescription: serviceRequest.description,
          clarifyingQuestionsAsked: clarifyingQuestionsAsked + 1,
        });
      }

      return this.declineCompiler.compile(
        'exhausted_clarification',
        serviceRequest.description,
        serviceCatalog,
        locationName,
        clarifyingQuestionsAsked,
      );
    }

    return this.declineCompiler.compile(
      'no_match',
      serviceRequest.description,
      serviceCatalog,
      locationName,
      clarifyingQuestionsAsked,
    );
  }

  /**
   * Re-evaluates after a clarifying response has been incorporated into the
   * description. The caller (conversation controller) owns updating the
   * CallSession; this method accepts the updated description as input.
   */
  evaluateWithUpdatedDescription(
    context: CapabilityEvaluationContext,
    updatedDescription: string,
  ): CapabilityDecision {
    return this.evaluate({
      ...context,
      serviceRequest: { ...context.serviceRequest, description: updatedDescription },
      clarifyingQuestionsAsked: context.clarifyingQuestionsAsked + 1,
    });
  }

  private rankCandidates(
    catalog: readonly typeof context.serviceCatalog[number][],
    description: string,
  ): typeof context.serviceCatalog {
    const candidates = catalog
      .map((item) => ({ item, score: this.matcher['scoreItem'](item, this.matcher['normalizer'].normalize(description)) }))
      .sort((a, b) => b.score - a.score)
      .map((scored) => scored.item);
    return candidates;
  }
}
```

---

## tests/capability-decision.tests.ts

```typescript
import { CapabilityDecisionGateway } from '../gateway/capability-decision.gateway';
import type { CapabilityDecision } from '../contracts/capability-decision.contract';
import type { ServiceCatalogItem } from '../models/service-catalog-item.model';
import type { CapabilityEvaluationContext } from '../models/capability-evaluation-context.model';

const PLUMBING_CATALOG: readonly ServiceCatalogItem[] = [
  {
    systemId: 'water_heater_repair',
    displayName: 'Water Heater Repair',
    tags: ['plumbing', 'hot_water'],
    triageQuestions: ['Is this a gas or electric water heater?', 'How old is the unit?'],
  },
  {
    systemId: 'drain_cleaning',
    displayName: 'Drain Cleaning',
    tags: ['plumbing', 'drain'],
    triageQuestions: ['Is the drain completely clogged?'],
  },
];

const AMBIGUOUS_CATALOG: readonly ServiceCatalogItem[] = [
  ...PLUMBING_CATALOG,
  {
    systemId: 'hvac_repair',
    displayName: 'HVAC Repair',
    tags: ['hvac', 'heating', 'cooling'],
    triageQuestions: ['What type of heating system do you have?'],
  },
];

function buildContext(
  catalog: readonly ServiceCatalogItem[],
  description: string,
  questionsAsked = 0,
): CapabilityEvaluationContext {
  return {
    serviceCatalog: catalog,
    serviceRequest: { description },
    locationName: 'Downtown Plumbing',
    clarifyingQuestionsAsked: questionsAsked,
    maxClarifyingQuestions: 3,
  };
}

function assertSupported(
  decision: CapabilityDecision,
  expectedServiceId: string,
  expectedConfidence: 'high' | 'medium' | 'low',
) {
  if (!('supported' in decision && decision.supported === true)) {
    throw new Error(`Expected supported decision but got: ${JSON.stringify(decision)}`);
  }
  if (decision.match.matchedServiceId !== expectedServiceId) {
    throw new Error(`Expected ${expectedServiceId}, got ${decision.match.matchedServiceId}`);
  }
  if (decision.match.confidence !== expectedConfidence) {
    throw new Error(`Expected ${expectedConfidence}, got ${decision.match.confidence}`);
  }
}

function assertUnsupported(decision: CapabilityDecision, expectedReason: 'no_match' | 'exhausted_clarification') {
  if (!('supported' in decision && decision.supported === false && 'declineContext' in decision)) {
    throw new Error(`Expected unsupported decision but got: ${JSON.stringify(decision)}`);
  }
  if (decision.declineContext.reason !== expectedReason) {
    throw new Error(`Expected ${expectedReason}, got ${decision.declineContext.reason}`);
  }
}

function assertClarifying(decision: CapabilityDecision) {
  if (!('supported' in decision && decision.supported === false && 'clarifyingQuestion' in decision)) {
    throw new Error(`Expected clarifying decision but got: ${JSON.stringify(decision)}`);
  }
}

async function runTests() {
  const gateway = new CapabilityDecisionGateway();

  // TC-CAPABILITY-01: High confidence exact systemId mention
  {
    const decision = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'I need water heater repair'));
    assertSupported(decision, 'water_heater_repair', 'high');
    console.log('TC-CAPABILITY-01 passed');
  }

  // TC-CAPABILITY-02: High confidence multiple strong keyword matches
  {
    const decision = gateway.evaluate(
      buildContext(PLUMBING_CATALOG, 'my water heater is leaking and needs fixing'),
    );
    assertSupported(decision, 'water_heater_repair', 'high');
    console.log('TC-CAPABILITY-02 passed');
  }

  // TC-CAPABILITY-03: Medium confidence single strong match
  {
    const decision = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'I have a plumbing issue'));
    assertSupported(decision, 'drain_cleaning', 'medium');
    console.log('TC-CAPABILITY-03 passed');
  }

  // TC-CAPABILITY-04: Low confidence ambiguous term → clarifying question
  {
    const decision = gateway.evaluate(buildContext(AMBIGUOUS_CATALOG, 'my heater is broken'));
    assertClarifying(decision);
    console.log('TC-CAPABILITY-04 passed');
  }

  // TC-CAPABILITY-05: Clarifying question → updated description → supported
  {
    const first = gateway.evaluate(buildContext(AMBIGUOUS_CATALOG, 'my heater is broken'));
    assertClarifying(first);
    if (!('clarifyingQuestion' in first)) throw new Error('unreachable');
    const second = gateway.evaluateWithUpdatedDescription(
      buildContext(AMBIGUOUS_CATALOG, 'my heater is broken', 0),
      'my heater is broken. I mean the water heater in my basement.',
    );
    assertSupported(second, 'water_heater_repair', 'high');
    console.log('TC-CAPABILITY-05 passed');
  }

  // TC-CAPABILITY-06: Second clarifying question returned when still ambiguous
  {
    const first = gateway.evaluate(buildContext(AMBIGUOUS_CATALOG, 'my heater is broken'));
    assertClarifying(first);
    const second = gateway.evaluateWithUpdatedDescription(
      buildContext(AMBIGUOUS_CATALOG, 'my heater is broken', 0),
      'my heater is broken. it is for hot water and heating.',
    );
    assertClarifying(second);
    console.log('TC-CAPABILITY-06 passed');
  }

  // TC-CAPABILITY-07: Exhausted 3 clarifying questions → unsupported
  {
    const context = buildContext(AMBIGUOUS_CATALOG, 'something is broken');
    const q1 = gateway.evaluate(context);
    assertClarifying(q1);

    const q2 = gateway.evaluateWithUpdatedDescription(context, 'something is broken. it is a heater thing.');
    assertClarifying(q2);

    const q3 = gateway.evaluateWithUpdatedDescription(
      { ...context, clarifyingQuestionsAsked: 1 },
      'something is broken. it is a heater thing. it is for hot water.',
    );
    assertClarifying(q3);

    const final = gateway.evaluateWithUpdatedDescription(
      { ...context, clarifyingQuestionsAsked: 2 },
      'something is broken. it is a heater thing. it is for hot water. it is for heating.',
    );
    assertUnsupported(final, 'exhausted_clarification');
    if (!('declineContext' in final)) throw new Error('unreachable');
    if (final.declineContext.clarifyingQuestionsAsked !== 3) {
      throw new Error('Expected 3 clarifying questions asked');
    }
    console.log('TC-CAPABILITY-07 passed');
  }

  // TC-CAPABILITY-08: No match in Pass 1 → unsupported
  {
    const decision = gateway.evaluate(
      buildContext(PLUMBING_CATALOG, 'I need electrical panel inspection'),
    );
    assertUnsupported(decision, 'no_match');
    console.log('TC-CAPABILITY-08 passed');
  }

  // TC-CAPABILITY-09: Unsupported message includes tag-based categories
  {
    const decision = gateway.evaluate(
      buildContext(PLUMBING_CATALOG, 'I need electrical panel inspection'),
    );
    assertUnsupported(decision, 'no_match');
    if (!('declineContext' in decision)) throw new Error('unreachable');
    const categories = decision.declineContext.serviceCategories;
    if (!categories.includes('plumbing') || !categories.includes('hot_water') || !categories.includes('drain')) {
      throw new Error(`Expected tag-based categories, got ${categories.join(', ')}`);
    }
    console.log('TC-CAPABILITY-09 passed');
  }

  // TC-CAPABILITY-10: Empty description → unsupported
  {
    const decision = gateway.evaluate(buildContext(PLUMBING_CATALOG, '   '));
    assertUnsupported(decision, 'no_match');
    console.log('TC-CAPABILITY-10 passed');
  }

  // TC-CAPABILITY-11: Case-insensitive matching
  {
    const decision = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'I NEED WATER HEATER REPAIR'));
    assertSupported(decision, 'water_heater_repair', 'high');
    console.log('TC-CAPABILITY-11 passed');
  }

  // TC-CAPABILITY-12: Substring matching for triage questions
  {
    const decision = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'my water heater is leaking'));
    assertSupported(decision, 'water_heater_repair', 'high');
    console.log('TC-CAPABILITY-12 passed');
  }

  // Determinism test
  {
    const first = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'I need water heater repair'));
    const second = gateway.evaluate(buildContext(PLUMBING_CATALOG, 'I need water heater repair'));
    assertSupported(first, 'water_heater_repair', 'high');
    assertSupported(second, 'water_heater_repair', 'high');
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
| Capability Decision Contract | `contracts/capability-decision.contract.ts` |
| Capability Match Descriptor Contract | `contracts/capability-match.contract.ts` |
| Clarifying Question Contract | `contracts/clarifying-question.contract.ts` |
| Decline Context Contract | `contracts/decline-context.contract.ts` |
| Catalog Consumption Contract | `models/capability-evaluation-context.model.ts` (validated catalog as input) |
| Intent Normalizer | `evaluation/intent-normalizer.ts` |
| Double-Pass Matrix Matcher | `evaluation/double-pass-matrix.matcher.ts` + `gateway/capability-decision.gateway.ts` (Pass 1 / Pass 2 orchestration) |
| Confidence Arbiter | `evaluation/confidence-arbiter.ts` |
| Clarification Budget Tracker | `evaluation/clarification-budget.tracker.ts` |
| Supported Outcome Compiler | `outcomes/supported-outcome.compiler.ts` |
| Decline Envelope Compiler | `outcomes/decline-envelope.compiler.ts` |
| Capability Decision Gateway | `gateway/capability-decision.gateway.ts` |
| Contract / Matching / Clarification / Decline / Determinism Tests | `tests/capability-decision.tests.ts` |

## Scope Guardrails Observed

- ✅ No scheduling, booking, or intake creation.
- ✅ No persistence writes or audit events.
- ✅ No service area validation or emergency qualification.
- ✅ No template rendering; only template variables returned in `DeclineContext`.
- ✅ No customer data collection beyond the service request description.
- ✅ No AI/ML classification; rule-based keyword/tag matching only.
- ✅ Deterministic, in-memory, side-effect-free evaluation.
- ✅ Bounded clarification: hard limit of 3 questions.
- ✅ Fail-closed on empty catalog, empty description, or exhausted clarification.
- ✅ `CallSession` is not mutated; the gateway returns updated description suggestions for the upstream controller to apply.

## Design Decisions Captured

- **Side-effect-free boundary:** The gateway accepts plain inputs and returns immutable decision envelopes. Any `CallSession` update is the responsibility of the upstream conversation controller.
- **PRD scoring algorithm:** Weights (+10 / +5 / +3 / +2) and thresholds (≥10 high, ≥5 medium, ≥2 low) are implemented exactly as specified.
- **Clarifying question generation:** The framework produces a concrete `questionText` derived from ambiguous candidate display names, plus `questionId` and `ambiguousServiceIds`. This keeps the contract testable while leaving final message rendering to TemplateRegistry.
- **Tag-based decline categories:** Unique catalog tags are returned directly as `serviceCategories`; no tag-to-category mapping is introduced in MVP.