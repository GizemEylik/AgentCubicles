# Clean Start Handoff

Preserves the three material engineering lessons identified by the final R&D
retention check on the transition repository, for use when designing the
independent AgentCubicles codebase. Each item is recorded at the
behavioral/requirement level — not as implementation to carry forward.

## 1. Provider-scoped event routing

- Event-to-agent/session/project matching must remain scoped to the provider
  that produced the event.
- Provider identity must be validated before processing.
- Coincidentally identical session IDs or project paths across providers must
  not permit cross-provider attribution.
- Evidence/reference: `c997c2f`.
- Destination in new codebase: `ARCHITECTURE.md`.

## 2. Test user-profile isolation

- Automated tests that may interact with home-relative/provider/product state
  must not operate against the developer's or CI runner's real user profile.
- Tests must use isolated temporary state and positively verify that
  isolation is effective, rather than merely assuming an environment override
  worked.
- Preserve the requirement, not the old helper implementation.
- Evidence/reference: `8aa2adf`.
- Destination in new codebase: `TEST_STRATEGY.md`.

## 3. Environment-variable casing semantics

- Environment isolation/manipulation must account for platform differences:
  Windows environment-variable names are case-insensitive, while POSIX can
  contain multiple differently-cased variables simultaneously.
- Logic that captures, filters, neutralizes, or restores environment
  variables must handle all relevant variants according to platform
  semantics.
- Do not prescribe the old snapshot/helper implementation.
- Evidence/reference: `944253f`.
- Destination in new codebase: `TEST_STRATEGY.md`, with an `ARCHITECTURE.md`
  cross-reference if production environment isolation uses this mechanism.

## Evidence Provenance

- The evidence commit hashes cited above (`c997c2f`, `8aa2adf`, `944253f`)
  refer to commits in the **AgentCubicles transition repository**.
- Current transition repository: `GizemEylik/AgentCubicles`.
- Relevant transition branch at handoff: `feat/multi-provider-codex`.
- The transition repository should be preserved / kept archive-accessible so
  these evidence commits remain resolvable.
- These references are **provenance/evidence only** — they are **not**
  implementation inputs for the new clean codebase.

## Clean Start Inputs

- `PROJECT_VISION.md`
- `PRODUCT_REQUIREMENTS.md`
- `PROVENANCE_DECISIONS.md`
- `CODEX_INTEGRATION_REQUIREMENTS.md`
- `CLEAN_START_HANDOFF.md`

These documents are **inputs to independent design, not permission to copy
transition-repository implementation**. The new architecture must be derived
from product requirements and the behavioral/security constraints recorded
across these documents. The transition repository's implementation structure
must not be treated as the default architecture.
