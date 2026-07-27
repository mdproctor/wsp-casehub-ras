# HANDOFF — casehub-ras

**Date:** 2026-07-27
**Issues:** #42 (closed)

## What was done

Closed #42 on branch `issue-42-situation-templates` (eb70de4). Added YAML template
support to `YamlSituationDefinitionProvider`: `templates:` section with `${param}`
placeholders, consumer `fromTemplate:` + `parameters:` instantiation, typed parameter
substitution, deep merge overrides, identity field injection, error wrapping, two-file
loading with 4 built-in templates (`streak-breach`, `threshold-crossing`,
`count-accumulation`, `rate-breach`), and optional ganglion bundling. Design review
($13.28, 3 rounds, 17 issues). Zero API changes — templates are internal to the
YAML provider.

## Key decisions

- Templates are a YAML parse concern — registry sees fully resolved registrations
- Whole-value `${param}` preserves types (int/double/list); substring interpolation produces strings
- Identity fields (situationId, eventTypes) implicitly available as template parameters
- Consumer templates override built-in with same ID (last-parsed wins)
- Ganglion bundling resolved per-instantiation, not per-template-definition

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #40 | RAS feedback loop | L | High | Case outcomes into detection tuning; refs parent#365 |
| #41 | Meta-situations | L | High | Situations observing other situations |
| #43 | Passive observation mode | M | Med | Richer query API |
| #44 | Situation replay | M | Med | Validate definitions against historical events |
| #45 | Adaptive thresholds | L | High | Self-tuning; depends on #40 |
| #29 | DroolsSessionStore journal-based reliability | L | High | Replaces experimental H2MVStore |
| #30 | DroolsSessionStore clustered session sharing | L | High | Needs networked backend |
| #5 | Platform stream infrastructure | XL | High | Epic, lives in casehub-platform |
