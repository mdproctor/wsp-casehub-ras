# HANDOFF — casehub-ras

**Date:** 2026-06-26
**Branch closed:** issue-3-java-switch-ganglion → merged to main (1b20f36)
**Issue:** casehubio/casehub-ras#3 (closed)

## What was done

Epic 3: two built-in Ganglion implementations, both pure Java with zero external dependencies.

- **JavaSwitchGanglion** (api/) — abstract base class for synchronous detection. Developer
  subclasses and overrides `evaluate()`. Helper methods auto-embed ganglionId. Stateless.
- **NaiveBayesGanglion** (runtime/) — concrete Bayesian classifier configured via
  `NaiveBayesConfig`. Log-space arithmetic, `compact()` for Threshold ChainMode correctness,
  optional ANTI threshold for counter-evidence.
- **DetectionResult NaN guard** — pre-existing IEEE 754 gap: `Double.isNaN()` added to
  confidence validation. Same guard applied to all NaiveBayes config numeric values.
- **AbstractGanglionContractTest** — abstract base in api/src/test/ with test-jar for
  cross-module visibility. Contract tests added for all three ganglion implementations.
- **Rename** — `BayesFeatureExtractor` → `NaiveBayesFeatureExtractor`,
  `BayesSignalMapping` → `NaiveBayesSignalMapping` for future namespace clarity.

## Key decisions

- JavaSwitchGanglion in api/ (not runtime/) — consumers depend only on casehub-ras-api
- No type parameter `<T>` on JavaSwitchGanglion — multi-event-type ganglia force Object casts
- compact() uses processing order (last in list), not eventTime — out-of-order delivery
  means the last-processed detection has the most complete posterior
- NaiveBayes + Threshold works for persistent situations (compact fixes double-counting);
  windowed situations documented as needing Or/Count instead

## Known issues

- `SituationExpiryJobTest.removesExpiredSituations` fails — pre-existing, unrelated to Epic 3
- No ARC42STORIES.MD exists yet for this repo

## What's next

- No immediate follow-on. Deferred items: persistent BayesState, YAML-based NaiveBayesConfig,
  temporal decay on posteriors, full Bayesian network ganglion (drools-beliefs extraction).
