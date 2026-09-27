# DOGFOOD HACKATHON

# Tier 2 — Judging System

## Agent Implementation Specification

---

# 0. MISSION AND IMPLEMENTATION SCOPE

You are implementing **Tier 2 — Judging** as an extension of the existing DOGFOOD HACKATHON application.

Tier 2 is **not an isolated application** and this repository is **not assumed to be empty**.

Before writing code, you must first understand what Tier 1 already implements, verify that understanding against the actual repository, and then implement only the additional functionality required for Tier 2.

Do **not** recreate existing functionality merely because you would design it differently.

Do **not** replace the existing architecture unless there is a concrete Tier 2 requirement that makes a change necessary.

Do **not** create duplicate versions of concepts that already exist in Tier 1.

The goal is:

> **Existing Tier 1 system + Tier 2 judging capabilities = one coherent application.**

Tier 2 must integrate with the existing:

* authentication
* authorization
* users
* roles
* events
* tracks
* teams
* submissions
* database
* API
* frontend
* workflows
* migrations
* fixtures
* Docker/self-hosting setup
* tests

The implementation must satisfy:

1. Existing repository behavior
2. DOGFOOD platform requirements
3. Existing data and architecture
4. `fixtures.json`
5. `JUDGING.md`
6. Tier 2 judging requirements
7. Acceptance tests
8. Security and auditability requirements

---

## 0.1 BEFORE CODING — UNDERSTAND THE EXISTING SYSTEM

Before creating or modifying any code, perform a complete inspection of the existing implementation.

The repository already contains Tier 1 functionality.

Your first responsibility is therefore **integration discovery**, not implementation.

You must determine:

* what already exists
* what is already implemented
* what data models already exist
* what APIs already exist
* what authentication system already exists
* what authorization/role system already exists
* what event/track/submission structures already exist
* what workflow/state machines already exist
* what frontend structure already exists
* what database and migration system already exists
* what testing infrastructure already exists
* what fixtures and seed data already exist
* what Docker/self-hosting setup already exists

Do not assume that the architecture described in this document is identical to the repository.

This document describes what Tier 2 needs.

The repository determines what already exists and therefore what must be extended rather than recreated.

---

## 0.2 EXISTING IMPLEMENTATION DOCUMENT

There is an existing implementation/inventory document describing the system already implemented in the repository.

Read that document first and use it as an architectural map.

Then verify its claims against the actual repository.

The document is useful for understanding:

* existing application structure
* implemented features
* existing entities
* relationships
* authentication
* authorization
* existing roles
* database architecture
* API structure
* frontend structure
* existing workflows
* migrations
* tests
* fixtures
* Docker configuration
* existing Tier 1 behavior

However, **do not blindly trust documentation over code**.

After reading it, inspect the actual implementation.

If documentation and implementation disagree:

1. inspect the actual code/database/migrations
2. determine the currently authoritative behavior
3. preserve working Tier 1 behavior unless Tier 2 explicitly requires a change
4. do not silently rewrite unrelated systems
5. document any necessary compatibility change

---

## 0.3 VERIFY THE ACTUAL REPOSITORY

Before implementation, inspect at minimum:

* `README`
* architecture documentation
* data-model documentation
* Docker configuration
* `docker-compose`
* backend source
* frontend source
* database schema
* migrations
* ORM models
* API routes/controllers
* authentication middleware
* authorization/role checks
* existing workflows/state machines
* tests
* seed files
* fixtures
* configuration files

Identify existing equivalents for every major Tier 2 concept.

For example:

| Tier 2 Concept | First Question                                           |
| -------------- | -------------------------------------------------------- |
| User           | Does Tier 1 already have a User entity?                  |
| Judge          | Can an existing User/Role represent a judge?             |
| Event          | Does Tier 1 already have Event?                          |
| Track          | Does Tier 1 already have Track?                          |
| Submission     | Does Tier 1 already have Submission?                     |
| Role           | Does Tier 1 already have role/permission infrastructure? |
| Authentication | Does Tier 1 already authenticate users?                  |
| State          | Does Tier 1 already have a workflow/state model?         |
| Database       | What DB/ORM/migration system is already being used?      |

If an equivalent already exists:

> **Extend it. Do not create a parallel entity.**

---

## 0.4 DOGFOOD PLATFORM RULES

The implementation must also comply with the DOGFOOD platform requirements.

These are platform-level requirements and must be treated separately from the internal application architecture.

The system must support the DOGFOOD requirements around:

* self-hosting
* local execution
* Docker deployment
* offline operation
* data ownership
* judging integrity
* participant isolation
* organizer control
* reproducibility
* auditability
* privacy
* security
* deterministic behavior
* workflow integrity

Do not introduce an architecture that violates these requirements simply because it is convenient.

---

## 0.5 SELF-HOSTED AND OFFLINE REQUIREMENTS

The application must remain self-hostable.

The intended deployment must support:

```bash
docker compose up
```

The system must not require:

* hosted database infrastructure
* cloud authentication
* SaaS judging services
* external scoring services
* mandatory third-party APIs
* remote computation
* network access for core judging logic

After dependencies are installed, the application must be capable of operating without network access.

Judging calculations must execute locally.

The authoritative judging state must remain inside the application/database.

---

## 0.6 FIXTURES ARE CONTRACTUAL INPUT

Inspect the existing `fixtures.json`.

Treat it as reproducible contractual test input rather than disposable demo data.

Inspect:

* users
* roles
* organizers
* judges
* events
* tracks
* teams
* submissions
* IDs
* relationships
* statuses
* scenarios represented by the fixtures

Do not casually change existing fixture IDs or relationships.

If Tier 2 requires additional fixture data:

* extend the existing fixtures
* preserve existing IDs
* preserve existing relationships
* preserve Tier 1 compatibility
* add only the minimum required judging scenarios

The resulting fixtures must remain deterministic and reproducible.

### Fixture scores are historical input

The `scores` data already present in `fixtures.json` represents **historical judging input**. It must not be treated as the assignment/review state for a newly configured Tier 2 judging stage.

In particular, the implementation must not:

* use fixture scores to satisfy the new stage's required review count `R`
* infer assignments from existing fixture scores
* treat existing fixture scores as proof that the new assignment engine produced a valid assignment
* silently merge historical fixture scores into a new stage's calibration or ranking inputs

The new judging workflow must create and persist its own:

```text
stage
assignment version
assignments
reviews
raw scores / comparisons
calibration or normalization result
ranking result
```

If historical fixture scores are displayed or retained, they remain historical seed data unless an explicit, separately defined import operation deliberately converts them into reviews belonging to a particular judging stage/version. Such an import must preserve provenance and must not bypass assignment, authorization, versioning, or validation rules.

The presence of uneven, incomplete, duplicate, or low-variance fixture score data is therefore a test condition for data handling and historical evidence, not a shortcut around the new judging engine.

---

## 0.7 `JUDGING.md` IS THE SOURCE OF TRUTH FOR JUDGING LOGIC

Read `JUDGING.md` completely before implementing judging algorithms.

`JUDGING.md` is authoritative for judging mathematics and judging behavior.

This includes:

* assignment
* exact review counts
* duplicate prevention
* workload balancing
* deliberate overlap
* connectivity
* assignment repair
* scoring
* calibration
* normalization
* ranking
* dropout behavior
* track isolation
* overall judging
* finalization
* auditability

Do not simplify the judging system to:

```text
average(scores)
```

or any other shortcut that contradicts `JUDGING.md`.

The implementation must preserve the mathematical behavior specified there.

---

## 0.8 SOURCE-OF-TRUTH PRIORITY

Use the following priority when implementing Tier 2:

1. Existing repository contracts and working architecture
2. `JUDGING.md` for judging mathematics and judging behavior
3. DOGFOOD platform requirements
4. Existing Tier 1 behavior
5. This Tier 2 implementation plan
6. Engineering judgment

These sources have different responsibilities.

### Existing repository

Determines:

* how the application is structured
* existing entities
* existing authentication
* existing authorization
* existing database
* existing APIs
* existing frontend
* existing workflows

### `JUDGING.md`

Determines:

* judging mathematics
* assignment behavior
* overlap
* workload
* connectivity
* scoring
* calibration
* normalization
* ranking
* dropout
* judging isolation
* finalization behavior

### DOGFOOD specification

Determines:

* platform-level constraints
* self-hosting
* offline requirements
* deployment requirements
* data ownership
* judging integrity

### Tier 2 implementation plan

Determines:

* what additional functionality must be added
* how the existing system should be extended
* required integration points
* required acceptance behavior

---

## 0.9 DO NOT REIMPLEMENT EXISTING SYSTEMS

Before creating any new:

* entity
* table
* service
* API
* permission system
* authentication system
* workflow
* state machine

first determine whether an equivalent already exists.

Prefer:

```text
existing system
      ↓
extend
      ↓
Tier 2 capability
```

over:

```text
existing system       new parallel system
      ↓                      ↓
     T1                     T2
```

Examples of prohibited duplication:

```text
User             + JudgeUser
Role             + JudgeAuth
Submission       + JudgingSubmission
Track            + JudgingTrack
Event            + JudgingEvent
```

unless inspection proves that no equivalent exists and the separation is genuinely required.

---

# 1. PRODUCT GOAL

Tier 2 must implement the complete judging pipeline:

```text
Judge Management
        ↓
Judging Stage Configuration
        ↓
Organizer-selected Judging Method
        ↓
Judge Assignment / Pairwise Setup
        ↓
Independent Reviews / Comparisons
        ↓
Method-specific Scoring / Estimation
        ↓
Method-specific Calibration / Normalization when applicable
        ↓
Final Ranking
        ↓
Organizer Progress
        ↓
CSV Export
        ↓
Immutable Finalization
```

The system must support an **organizer-configured judging pipeline**. The organizer chooses how many judging stages the event will have and, for each stage, which judging method that stage uses.

A stage has two distinct concepts that must not be conflated:

* **scope** — which submissions/judges belong to the judging universe (global, track, or explicitly configured finalist/overall scope)
* **judging method** — how that stage evaluates/ranks submissions (for example, standard rubric scoring or the optional Pairwise/Bradley–Terry method)

The number of stages is not hard-coded. A configuration may contain one stage, multiple sequential stages, or a track/finalist pipeline when explicitly configured by the organizer.

For every stage, the organizer must explicitly configure at minimum:

* stage name/order
* submission scope
* judge scope/panel
* judging method
* required review/comparison configuration appropriate to that method
* rubric configuration when using rubric scoring
* assignment configuration
* calibration/normalization configuration when applicable
* ranking configuration
* whether the stage produces a finalist set for a later stage

The implementation must persist the stage configuration and use it as the authoritative source for the judging pipeline. Do not infer the number of stages or judging method from fixture data, route names, hard-coded stage numbers, or frontend state.

The standard rubric method is the normal Tier 2 judging path. **Pairwise Mode is an optional organizer-selected judging method**, primarily useful for shortlist/finalist ranking. Pairwise Mode is not required for every event or stage. When selected, the stage must execute the Pairwise/Bradley–Terry workflow defined by `JUDGING.md`; when not selected, the stage uses the configured rubric-based workflow.

---

## 1.1 ORGANIZER-CONFIGURED STAGE PIPELINE

The organizer decides how many judging stages exist. There is no assumption that every event has exactly one stage or exactly two stages.

A multi-stage event is represented explicitly, for example:

```text
Stage 1
  ↓
Stage 1 result / finalist set
  ↓
Stage 2
  ↓
Stage 2 result / finalist set
  ↓
...
```

Each stage independently declares its judging method. For example:

```text
Stage 1 → RUBRIC
Stage 2 → PAIRWISE
```

or:

```text
Stage 1 → RUBRIC
Stage 2 → RUBRIC
```

The stage order and any dependency on a previous finalist set must be persisted and validated server-side. A later stage must consume only the explicitly configured input universe from the preceding stage.

### Standard rubric stage

A rubric stage uses:

```text
submissions
    ↓
eligible judges
    ↓
assignments
    ↓
criterion reviews
    ↓
raw weighted scores
    ↓
cross-judge normalization when applicable
    ↓
ranking
```

### Optional Pairwise stage

A Pairwise stage uses the organizer-configured candidate universe and pairwise comparisons rather than criterion-weighted reviews. Its ranking/calculation must follow the Pairwise/Bradley–Terry specification in `JUDGING.md`.

Pairwise Mode is an optional method, not a mandatory requirement for every Tier 2 event.

---

## 1.2 TYPE 1 — NO TRACKS

There is one global judging universe for each configured stage.

```text
All submissions
      ↓
Eligible judges
      ↓
One judging stage
      ↓
Assignments
      ↓
Reviews
      ↓
Calibration
      ↓
Global ranking
```

There is, for each global stage:

* one submission pool
* one eligible judge pool
* one judging scope
* one ranking universe

Multiple global stages may exist when configured by the organizer.

---

## 1.2 TYPE 2 — TRACK PANELS

Each track is an independent judging universe.

```text
Track A submissions → Track A judges → Track A ranking

Track B submissions → Track B judges → Track B ranking

Track C submissions → Track C judges → Track C ranking
```

There must be:

* no cross-track calibration
* no cross-track normalization
* no cross-track ranking
* no accidental cross-track score comparison

Each track has its own judging stage/panel configuration.

An optional separate **overall/finalist judging stage** may exist if configured by the organizer. More generally, any number of later stages may be configured, provided their stage dependencies and input universes are explicit.

---

## 1.3 ONE REUSABLE JUDGING ENGINE

Do not implement completely separate judging engines for:

```text
global judging
track judging
overall judging
```

Instead, implement one reusable, configuration-driven judging engine.

The judging engine should receive a judging-stage configuration defining:

* stage order/dependencies
* submission scope
* judge scope
* panel
* judging method
* rubric when the selected method requires one
* required review/comparison count appropriate to the selected method
* assignment configuration
* calibration configuration when applicable
* ranking configuration
* finalist/output configuration

The same engine can then operate on:

```text
global stage
track stage
overall stage
```

---

## 1.4 TIER 2 EXTENDS TIER 1

Preserve existing Tier 1 functionality.

Do not unnecessarily rewrite:

* authentication
* sessions
* roles
* events
* tracks
* teams
* submissions
* deadlines
* gallery
* existing participant workflows

unless Tier 2 integration genuinely requires a change.

Any database change must use proper migrations/versioning.

Do not delete existing Tier 1 data.

Do not break existing Tier 1 routes or workflows without a concrete requirement.

---

# 2. SECURITY AND AUTHORIZATION

All judging authorization must be enforced server-side.

Never trust client-provided identifiers as proof of authorization.

The backend must derive or verify:

* authenticated actor
* actor role
* event
* judging stage
* track/panel
* assignment
* submission
* review ownership

Client-provided values such as:

```text
judge_id
review_id
assignment_id
submission_id
stage_id
track_id
```

must never be sufficient to authorize an operation.

---

## 2.1 JUDGE ISOLATION

Judge A must not be able to access:

* Judge B's reviews
* Judge B's scores
* Judge B's assignments
* private judge state
* aggregate judging results before allowed publication
* normalization internals
* organizer-only audit information

Judge A must not access submissions belonging to another track unless the current stage explicitly authorizes that access.

Frontend hiding is not authorization.

Every protected operation must be enforced by the backend.

---

## 2.2 CONTEXTUAL JUDGE AUTHORIZATION

Judge authorization is contextual.

A valid authorization decision may depend on:

```text
authenticated user
+
event
+
judging stage
+
assignment
+
submission
+
panel/track
```

A judge may belong to multiple:

* stages
* panels
* tracks

Therefore authorization must verify the specific current context.

---

# 3. EXISTING ROLE MODEL

Inspect the existing Tier 1 role/permission system before implementing judging authorization.

Extend the existing role model rather than creating a second authentication or permission system.

The expected conceptual access model is:

| Role            | Own judging data | Peer judging data | Aggregate | Audit |
| --------------- | ---------------: | ----------------: | --------: | ----: |
| Visitor         |               No |                No |        No |    No |
| Participant     |               No |                No |        No |    No |
| Judge           |              Yes |                No |        No |    No |
| Organizer/Admin |              Yes |               Yes |       Yes |   Yes |

The exact implementation must follow the repository's existing role architecture.

---

# 4. DATA MODEL

Do **not** blindly create every entity listed below.

First inspect:

* existing database models
* ORM models
* migrations
* fixtures
* relationships
* API contracts

For every concept below, determine whether an equivalent already exists.

Then:

* reuse it if possible
* extend it if necessary
* create a new entity only if genuinely required

Preserve:

* existing IDs
* existing relationships
* existing data
* existing behavior

The target is:

```text
Existing Tier 1 domain
        +
Required judging domain
        =
One coherent data model
```

Do not create parallel:

```text
User
Event
Submission
Track
Role
```

entities merely for judging.

---

## 4.1 REQUIRED JUDGING CONCEPTS

The judging domain must support these concepts:

* Judges
* Judging Stages
* Judge Panels
* Judge Assignments
* Rubrics
* Reviews
* Calibration / Normalization

These are conceptual requirements. Their physical implementation must fit the existing architecture.

---

## 4.2 JUDGES

A judge represents an existing application user authorized to participate in judging.

If Tier 1 already has a User entity, associate judging capabilities with that existing user.

Do not create a second authentication identity for judges.

Conceptually, a judge requires:

* user association
* judging status
* event/stage eligibility where required
* panel membership where applicable

Expected judging statuses:

```text
INVITED
ACTIVE
SUSPENDED
REMOVED
```

If the existing application already has an equivalent lifecycle, integrate with it rather than creating a conflicting lifecycle.

Rules:

* `INVITED` judges are not yet eligible to receive assignments unless explicitly activated.
* `ACTIVE` judges may receive assignments if otherwise eligible.
* `SUSPENDED` judges cannot receive new assignments.
* `REMOVED` judges cannot receive new assignments.
* Existing completed reviews must remain auditable after suspension/removal.

---

## 4.3 JUDGING STAGES

A judging stage is a reusable, independently configurable judging context.

Conceptually it contains:

```text
id
event / competition
name
order / sequence
scope
track (nullable)
input_stage / finalist_source (nullable)
judging_method
status
required_review_count / comparison_config
rubric_version (nullable)
assignment_version
calibration_config
ranking_config
created_at
updated_at
```

The exact physical schema must follow the existing repository.

A judging stage defines:

* stage order and dependencies
* submission/input scope
* judge scope
* panel
* judging method
* rubric when applicable
* review/comparison configuration
* assignment snapshot/version
* calibration configuration when applicable
* ranking configuration
* finalist/output configuration

A stage scope may represent:

* global judging
* track judging
* an explicitly configured finalist/overall universe

The judging method is configured independently of scope. Do not use `type` ambiguously to represent both concepts.

Avoid scattered logic such as:

```text
if track != null
```

throughout the application.

The stage configuration should define the judging universe.

### Type 1

A global stage has:

```text
track = null
```

and uses the global submission/judge universe.

### Type 2

Each track can have its own judging stage.

Each such stage has an independent:

* submission universe
* judge universe
* assignment
* review set
* calibration
* ranking

A later/finalist stage may use a separate explicitly configured universe. An overall stage is one possible configuration, not a hard-coded stage number.

---

# 5. JUDGING STAGE STATE MACHINE

Implement the judging stage as an explicit server-side state machine.

The state machine controls which judging operations are allowed at each point in the judging lifecycle.

Do not represent stage lifecycle only through scattered boolean flags such as:

```text
is_open
is_closed
is_finalized
has_assignments
has_calibration
```

Use one authoritative stage state.

The lifecycle is method-aware. A rubric stage may require calibration/normalization before finalization; a Pairwise stage follows its configured Pairwise/Bradley–Terry calculation lifecycle instead. The implementation must not force a rubric-specific calibration step onto a Pairwise stage when `JUDGING.md` does not require one.

If the existing project already has a state-machine/workflow convention, integrate with that convention rather than creating a competing state-machine framework.

The conceptual lifecycle is:

```text
DRAFT
  ↓
CONFIGURED
  ↓
ASSIGNING
  ↓
OPEN
  ↓
CLOSED
  ↓
CALIBRATING
  ↓
CALIBRATED
  ↓
FINALIZED
```

The exact stored enum/string names may follow the existing repository conventions, but the semantics below must remain intact.

---

## 5.1 STATE DEFINITIONS

### DRAFT

The judging stage is being created or edited.

Allowed operations include:

* configure judging scope
* select eligible judges
* configure panel
* configure the method-specific review/comparison requirement (`R` for the standard review model)
* configure the rubric when the selected method is `RUBRIC`
* configure Pairwise/Bradley–Terry settings when the selected method is `PAIRWISE`
* configure assignment/comparison settings
* configure calibration settings when required by the selected method

The stage is not yet ready for judging.

Not allowed:

* judge review submission
* calibration
* finalization
* treating assignments as an active judging round

---

### CONFIGURED

The judging configuration has been completed and passed configuration validation.

At minimum, the system must have a valid:

* judging scope
* submission pool
* eligible judge pool
* method-specific review/comparison configuration
* rubric version when the selected method is `RUBRIC`
* panel configuration where applicable
* assignment configuration

Configuration validation must verify that the stage is internally coherent before assignment generation begins.

The validator must also verify that the organizer-selected `judging_method` is compatible with the rest of the stage configuration. For example:

* `RUBRIC` requires a valid frozen rubric configuration before judging opens.
* `PAIRWISE` requires the Pairwise/Bradley–Terry configuration defined by `JUDGING.md` and must not require rubric criteria that the method does not use.
* method-specific review/comparison counts and ranking inputs must be validated before the stage can open.

A stage must not silently fall back from one judging method to another because configuration is incomplete or infeasible.

Examples of configuration failures include:

```text
no eligible judges
no submissions
invalid R
invalid rubric
invalid judge/panel scope
track/panel mismatch
insufficient judges for required R
```

The exact feasibility rules must follow `JUDGING.md`.

A stage must not enter `CONFIGURED` if its configuration is already known to be infeasible.

---

### ASSIGNING

The system is constructing or validating the assignment snapshot.

This state represents an assignment-generation operation.

During this state:

* the assignment engine may construct assignments
* assignment validation may run
* deterministic repair may run
* connectivity may be checked
* workload quotas may be checked

The assignment operation must produce a specific assignment version.

The stage must not become `OPEN` until the resulting assignment snapshot passes all required validation.

If assignment generation fails:

```text
ASSIGNING
    ↓
failure
```

must not produce an `OPEN` stage.

The system must expose a clear failure such as:

```text
FAILED_ASSIGNMENT
```

or the repository's equivalent failure representation.

Do not silently:

* reduce `R`
* remove judges
* ignore duplicate assignments
* weaken track isolation
* accept disconnected calibration graphs
* accept invalid workload distribution

---

### OPEN

The judging stage is actively accepting judge reviews.

Only assignments belonging to the active assignment version may be used for judging.

During `OPEN`:

* judges may view their authorized assignments
* judges may submit reviews for their own assignments
* duplicate reviews must be rejected
* rubric configuration is frozen
* the active assignment version is fixed
* judge authorization must be checked server-side

A review is valid only when all required relationships are valid:

```text
authenticated judge
        ↓
authorized for stage
        ↓
owns assignment
        ↓
assignment belongs to current stage/version
        ↓
submission belongs to assignment
        ↓
review uses valid rubric version
```

A client-provided `judge_id` is never sufficient proof of ownership.

---

### CLOSED

The judging stage no longer accepts normal review submissions.

When entering `CLOSED`:

* new reviews/comparisons are rejected
* the active judging period ends
* the completed review/comparison dataset becomes the input to the selected method
* assignment configuration cannot be silently changed
* method-specific configuration remains frozen

Completed valid reviews remain preserved.

Any organizer correction must follow the explicit review correction/versioning rules rather than silently modifying historical judging evidence.

The system must determine whether the stage satisfies the method-specific completion conditions before calculation/calibration can begin.

For a rubric stage, this includes the required review conditions and calibration prerequisites. For a Pairwise stage, this includes the required comparison conditions and the Pairwise/Bradley–Terry prerequisites defined by `JUDGING.md`.

A missing review or comparison must **not** automatically become a score of zero or an arbitrary pairwise outcome.

---

### CALIBRATING / CALCULATING

The system is calculating the method-specific result for the closed judging stage.

For a rubric stage, this includes raw weighted scoring and, when configured/required, cross-judge calibration/normalization. Calibration must operate on the valid review dataset belonging to the appropriate assignment/rubric versions.

For a Pairwise stage, this includes the Pairwise/Bradley–Terry calculation defined by `JUDGING.md`. Do not run rubric WLS calibration merely because the stage uses the generic lifecycle state.

Before calculation begins, validate the requirements appropriate to the selected method:

* stage is `CLOSED`
* required reviews/comparisons are satisfied or an explicitly authorized reduced-review exception exists
* raw reviews/comparisons are valid
* rubric versions are consistent when the method is `RUBRIC`
* assignment/comparison relationships are valid
* required overlap/calibration information exists when the selected method requires it
* the mathematical requirements for the selected calculation are satisfied
* the selected calculation procedure is applicable

The calculation/calibration procedure must follow `JUDGING.md`.

Do not replace the specified calibration model with a simple average or another shortcut.

If calibration fails:

```text
CALIBRATING
    ↓
failure
```

the stage must **not** become `CALIBRATED`.

The failure must be explicit and auditable.

Examples:

```text
DISCONNECTED_CALIBRATION_GRAPH
INVALID_REVIEW_DATA
INSUFFICIENT_REVIEWS
CALIBRATION_FAILURE
```

Use repository-appropriate error/status representations.

---

### CALIBRATED

Calibration completed successfully.

The stage now has a valid calibration/normalization result.

The system must preserve:

* raw review scores
* calibration version
* judge offsets
* normalized scores
* relevant diagnostics
* input/version references

Calibration must never overwrite raw judging evidence.

If calibration is rerun before finalization, create a new calibration version rather than silently replacing the previous calibration record.

---

### FINALIZED

The judging result is immutable.

Finalization represents the authoritative published judging result for that stage.

Before entering `FINALIZED`, verify that:

* the stage is successfully calibrated
* required judging data is valid
* ranking inputs are valid
* required versions are known
* the final result can be reproduced from the recorded snapshot

The finalized snapshot should identify, as applicable:

```text
stage
assignment_version
rubric_version
calibration_version
result/ranking version
input snapshot/hash
finalization timestamp
finalized by
```

After finalization, the system must not silently modify:

* assignments
* reviews
* rubric
* calibration
* normalized scores
* rankings
* final results

Any post-finalization correction must use an explicit versioning/reopening workflow if the product supports one.

Do not mutate the finalized result in place.

---

## 5.2 VALID TRANSITIONS

Only the following normal transitions are allowed:

```text
DRAFT
  ↓
CONFIGURED

CONFIGURED
  ↓
ASSIGNING

ASSIGNING
  ↓
OPEN

OPEN
  ↓
CLOSED

CLOSED
  ↓
CALIBRATING

CALIBRATING
  ↓
CALIBRATED

CALIBRATED
  ↓
FINALIZED
```

A transition must verify the current state before changing it.

For example:

```text
current_state == DRAFT
```

must be required before:

```text
DRAFT → CONFIGURED
```

Do not allow clients to directly set arbitrary state values.

---

## 5.3 INVALID TRANSITIONS

Reject transitions that skip required lifecycle stages.

Examples:

```text
DRAFT → OPEN
DRAFT → CLOSED
DRAFT → FINALIZED

CONFIGURED → OPEN
CONFIGURED → FINALIZED

ASSIGNING → FINALIZED

OPEN → FINALIZED

OPEN → CALIBRATED

CLOSED → OPEN

CLOSED → FINALIZED

CALIBRATING → FINALIZED

FINALIZED → OPEN
FINALIZED → CLOSED
FINALIZED → CALIBRATED
```

unless an explicit administrative/versioning workflow is intentionally implemented and documented.

Do not silently repair an invalid transition by changing state to whatever state makes the operation possible.

---

## 5.4 OPERATIONAL RULES BY STATE

The state machine must enforce the following minimum rules.

| Operation                | DRAFT | CONFIGURED | ASSIGNING | OPEN | CLOSED | CALIBRATING | CALIBRATED | FINALIZED |
| ------------------------ | ----: | ---------: | --------: | ---: | -----: | ----------: | ---------: | --------: |
| Edit stage configuration |   Yes |    Limited |        No |   No |     No |          No |         No |        No |
| Edit rubric              |   Yes |       Yes* |        No |   No |     No |          No |         No |        No |
| Generate assignment      |    No |        Yes |       Yes | No** |     No |          No |         No |        No |
| Submit review            |    No |         No |        No |  Yes |     No |          No |         No |        No |
| Run calibration          |    No |         No |        No |   No |    Yes |         Yes |         No |        No |
| Finalize                 |    No |         No |        No |   No |     No |          No |        Yes |        No |
| Mutate finalized result  |    No |         No |        No |   No |     No |          No |         No |        No |

`*` Rubric changes are allowed only before judging has begun and must result in a valid rubric version.

`**` Assignment changes during `OPEN` are prohibited unless an explicit reassignment/versioning workflow is invoked. Never silently mutate the active assignment snapshot.

---

## 5.5 RUBRIC IMMUTABILITY

Once the stage reaches:

```text
OPEN
```

the rubric version used by the judging stage is frozen.

Judges must not see one rubric version while the backend evaluates their submissions using another.

If the organizer needs to change the rubric after judging has begun:

1. do not mutate the active rubric version
2. create a new rubric version if the workflow permits it
3. determine whether the judging stage must be reopened/reassigned
4. preserve all historical reviews against their original rubric version
5. do not silently reinterpret existing scores

The exact correction/versioning behavior must remain consistent with the audit/versioning rules later in this document.

---

## 5.6 ASSIGNMENT IMMUTABILITY DURING ACTIVE JUDGING

Once the stage is `OPEN`, the active assignment version is fixed.

Do not modify assignment records in place.

If a judge dropout or organizer-approved reassignment occurs:

```text
existing assignment version
        ↓
preserved historically
        ↓
new assignment version
        ↓
replacement assignments
```

Reviews already completed under the previous assignment version remain associated with that version.

The replacement process must preserve:

* duplicate prevention
* exact review requirements where applicable
* judge eligibility
* track isolation
* workload constraints
* connectivity requirements
* auditability

---

## 5.7 REVIEW AUTHORIZATION

The backend must enforce:

```text
stage.state == OPEN
```

before accepting a normal review submission.

Additionally verify:

```text
authenticated user
    ==
authorized judge

judge
    owns assignment

assignment
    belongs to current stage

submission
    belongs to assignment

review
    does not already exist for this assignment/version
```

A judge must not be able to bypass the state machine by directly calling an API endpoint.

Frontend restrictions are insufficient.

---

## 5.8 CALIBRATION PRECONDITIONS

Calibration cannot begin merely because the stage is `CLOSED`.

Before entering `CALIBRATING`, the system must validate the actual judging data.

At minimum:

```text
stage is CLOSED
+
required reviews are available
+
reviews are valid
+
assignment relationships are valid
+
rubric version is consistent
+
calibration graph is valid
+
calibration is mathematically feasible
```

If `R = 1`, apply the special single-review behavior defined by `JUDGING.md`; do not attempt multi-judge overlap calibration.

For multi-judge calibration, a disconnected overlap graph must fail closed.

Do not invent missing calibration information.

---

## 5.9 FINALIZATION PRECONDITIONS

A stage may enter `FINALIZED` only if:

```text
stage == CALIBRATED
```

and the method-specific calculated result is valid.

For rubric stages, `CALIBRATED` means the required scoring/calibration/normalization result has been successfully produced. For Pairwise stages, `CALIBRATED` means the required Pairwise/Bradley–Terry calculation has successfully completed and produced a valid ranking result.

The system must not finalize:

* failed calculation/calibration
* incomplete calculation/calibration
* disconnected calibration where the selected method requires a connected calibration graph
* invalid ranking data
* inconsistent rubric versions where rubric scoring is used
* inconsistent assignment/comparison versions

Finalization must validate the final result again immediately before committing the finalized snapshot.

---

## 5.10 AUTHORIZATION FOR STATE TRANSITIONS

State transitions are organizer/admin operations unless the existing application explicitly defines another authorized role.

Judges must not be able to:

* configure stages
* open stages
* close stages
* regenerate assignments
* run calibration
* finalize results

merely because they are judges.

Every transition must verify:

```text
authenticated actor
+
required role/permission
+
target event
+
target stage
+
current stage state
+
valid transition
```

Do not trust:

```text
user_id
role
stage_id
```

provided by the client as proof of authorization.

Derive the authenticated actor from the server-side authentication context.

---

## 5.11 ATOMIC STATE TRANSITIONS

State transitions must be atomic.

The implementation must prevent race conditions such as:

```text
Judge submits review
        +
Organizer closes stage
```

or:

```text
Calibration starts
        +
Review is submitted
```

or:

```text
Finalization starts
        +
Review correction is submitted
```

The system must ensure that an operation observes one authoritative stage state.

For example, a review submission must not succeed against a stage that has already been atomically closed.

Use the repository's existing transaction/locking/concurrency mechanisms where available.

Do not create a second concurrency framework solely for judging.

---

## 5.12 STATE TRANSITION AUDIT

Important state changes must be auditable.

Record, directly or through the existing audit system:

```text
stage
previous_state
new_state
actor
timestamp
```

Where relevant, also record:

```text
assignment_version
rubric_version
calibration_version
```

Do not rely solely on application logs for authoritative state history.

If the existing application already has an audit/event mechanism, extend it.

---

## 5.13 DETERMINISM

State transitions themselves must not depend on:

* request arrival order
* frontend behavior
* database insertion order
* random runtime decisions

Assignment generation, calibration, and finalization must reference explicit versions/configuration so that the resulting state can be reproduced.

The state machine should therefore make the judging lifecycle explicit:

```text
configuration
    ↓
assignment version
    ↓
review dataset
    ↓
calibration version
    ↓
final result
    ↓
immutable finalized snapshot
```

---

## 5.14 REQUIRED TESTS

Implement tests for:

### Valid transitions

```text
DRAFT → CONFIGURED
CONFIGURED → ASSIGNING
ASSIGNING → OPEN
OPEN → CLOSED
CLOSED → CALIBRATING
CALIBRATING → CALIBRATED
CALIBRATED → FINALIZED
```

### Invalid transitions

Attempt invalid transitions and verify that:

* the request is rejected
* the stage remains in its original state
* no partial state change occurs

### Review lifecycle

Verify:

* review rejected before `OPEN`
* review accepted during `OPEN`
* review rejected after `CLOSED`
* review cannot bypass stage authorization
* duplicate review remains rejected

### Rubric lifecycle

Verify:

* rubric editable before judging begins
* rubric frozen once judging opens
* historical rubric version remains associated with reviews
* finalized rubric cannot be silently modified

### Assignment lifecycle

Verify:

* assignment generation occurs before `OPEN`
* active assignments cannot silently change during judging
* reassignment creates a new version
* historical assignments remain auditable

### Calibration lifecycle

Verify:

* method-specific calculation/calibration cannot run before `CLOSED`
* calculation/calibration cannot run with invalid/incomplete required data
* disconnected calibration fails where the selected method requires connectivity
* `R = 1` follows the special pass-through behavior for the standard review model
* failed calculation/calibration cannot be finalized

### Finalization

Verify:

* only stages with a valid `CALIBRATED` method-specific result can finalize
* finalized results are immutable
* unauthorized users cannot finalize
* concurrent mutation cannot invalidate a finalized snapshot

### Authorization

Verify:

* participant cannot transition a judging stage
* judge cannot perform organizer-only transitions
* organizer cannot modify another event's judging stage without authorization
* client-supplied IDs cannot bypass authorization

# 6. ASSIGNMENT ENGINE
Build assignment as a **backend/domain service**.

Assignment logic must never live in the frontend.

The frontend may:

* request assignment generation
* provide organizer configuration
* display assignments
* submit manual assignments
* display validation errors

But the backend/domain layer must be the authoritative source for:

* eligibility
* assignment construction
* review count
* duplicate prevention
* workload quotas
* overlap
* connectivity
* repair
* determinism
* assignment versioning

The client must never be able to bypass assignment validation by directly creating assignment records.

---

## 6.1 ASSIGNMENT ENGINE RESPONSIBILITIES

The assignment engine is responsible for producing a valid assignment snapshot for a judging stage.

Conceptually:

```text id="z0g5zv"
Stage Configuration
        ↓
Assignment Inputs
        ↓
Feasibility Validation
        ↓
Assignment Construction
        ↓
Assignment Validation
        ↓
Connectivity Validation
        ↓
Repair if necessary
        ↓
Final Validation
        ↓
Assignment Snapshot
```

The engine must not persist a partially constructed assignment as the authoritative assignment version.

Only a successfully validated assignment may become active.

---

## 6.2 ASSIGNMENT MODES

The system must support three assignment modes:

```text id="8r1o3d"
MANUAL
BATCH
ALGORITHMIC
```

All three modes must use the same authoritative validation rules.

The modes differ only in **how candidate assignments are created**.

They must not have different definitions of:

* eligible judge
* valid assignment
* duplicate
* review count
* track isolation
* workload
* connectivity

---

## 6.3 MANUAL MODE

Manual mode allows an organizer to explicitly select judge/submission pairs.

Example:

```text id="ojz7cw"
Submission A → Judge 1
Submission A → Judge 3
Submission A → Judge 7
```

The backend must validate the complete resulting assignment snapshot.

Manual assignment must not bypass:

```text id="z2l0b7"
judge eligibility
stage scope
panel membership
track isolation
duplicate prevention
required review count
capacity
assignment versioning
connectivity where required
```

A manual assignment that violates a hard constraint must be rejected.

Do not silently remove the invalid assignment and continue.

Return a structured validation error explaining the violated rule.

---

## 6.4 BATCH MODE

Batch mode allows an organizer to generate assignments for a selected group of submissions.

Example:

```text id="3h9x9c"
selected submissions
        ↓
same judging stage
        ↓
assignment engine
        ↓
one coherent assignment result
```

Batch generation must consider the assignment state of the complete batch.

Do not independently assign every submission without considering:

* current judge workload
* pair overlap
* connectivity
* remaining judge capacity
* stage-level quotas

The result must still be validated as one assignment snapshot.

---

## 6.5 ALGORITHMIC MODE

Algorithmic mode automatically constructs assignments according to the rules in `JUDGING.md`.

The algorithm must:

1. validate feasibility
2. calculate required workload
3. calculate target pair overlap
4. construct assignments deterministically
5. validate every hard constraint
6. validate overlap/connectivity
7. perform deterministic repair when necessary
8. perform final validation
9. produce an assignment version

The algorithm is a heuristic constructor.

It is **not** required to prove globally optimal combinatorial assignment.

However:

> The final assignment must satisfy all hard constraints.

---

## 6.6 ASSIGNMENT SERVICE BOUNDARY

Keep assignment logic isolated behind a domain/service boundary.

Conceptually:

```text id="jzv4q7"
JudgingService
      ↓
AssignmentService
      ├── FeasibilityValidator
      ├── Constructor
      ├── OverlapCalculator
      ├── ConnectivityValidator
      ├── AssignmentValidator
      └── RepairEngine
```

The exact module names may follow the existing repository architecture.

Do not create this exact folder structure if the repository already has an established service/domain structure.

The important requirement is separation of responsibilities, not literal filenames.

---

## 6.7 ASSIGNMENT CONSTRUCTION CONTRACT

The assignment constructor receives a complete immutable input snapshot.

Conceptually:

```text id="a2qg7e"
AssignmentInput
    stage
    submissions
    eligible judges
    R
    capacities
    panel scope
    configuration
    deterministic seed/version
```

It returns either:

```text id="3n0px1"
AssignmentSuccess
    assignments
    assignment metadata
```

or:

```text id="g7n9e0"
AssignmentFailure
    failure code
    explanation
    diagnostics
```

It must never return a partially valid assignment and label it successful.

---

## 6.8 HARD CONSTRAINTS VS OPTIMIZATION OBJECTIVES

The engine must distinguish **hard constraints** from **optimization objectives**.

Hard constraints must never be sacrificed to improve an optimization metric.

Hard constraints include, where applicable:

```text id="q9v9di"
judge eligibility
submission eligibility
track/panel isolation
exact review count
distinct judges per submission
judge capacity/quota
assignment scope
valid stage
valid judge membership
```

Optimization objectives include:

```text id="yep9n4"
balanced pair overlap
good connectivity
balanced workload when multiple valid choices exist
deterministic canonical selection
```

A better PairLoss score does not justify violating a hard constraint.

---

## 6.9 FEASIBILITY MUST BE CHECKED BEFORE CONSTRUCTION

Do not start constructing an assignment if the input is mathematically impossible.

The engine must perform the feasibility checks defined in `JUDGING.md`.

At minimum, reason about:

```text id="o7o6fk"
N = number of submissions
J = number of eligible judges
R = required reviews per submission
```

Total assignment slots:

```text
A = N × R
```

If:

```text
R > J
```

the assignment is impossible because a submission cannot receive `R` distinct judges.

Return the appropriate feasibility failure.

If:

```text
N × R < J
```

the requested exact workload distribution cannot give every eligible judge at least one assignment.

Apply the exact feasibility semantics from `JUDGING.md`.

For multi-judge calibration, also check the necessary connectivity condition defined there:

```text
N × C(R,2) >= J - 1
```

This is a necessary condition, not sufficient proof of connectivity.

The actual constructed assignment must still build and validate the overlap graph.

---

## 6.10 ASSIGNMENT OUTPUT

A successful assignment result must identify, at minimum:

```text id="mck7gt"
stage
assignment_version
submission
judge
assignment status
creation metadata
```

The exact persistence model must follow Section 4 and the existing repository.

The resulting assignment version must be reproducible from its recorded configuration.

---

## 6.11 NO PARTIAL ACTIVE ASSIGNMENTS

The engine may construct assignments in memory or in a temporary transaction.

But an incomplete construction must never become the active assignment version.

The system must avoid states such as:

```text id="7o1g6q"
10 submissions
R = 3

9 submissions → valid
1 submission → only 2 judges

stage marked ready anyway
```

The stage must remain unready until the assignment passes complete validation.

---

## 6.12 DETERMINISM

For identical inputs, the assignment engine must produce the same result.

Identical inputs include:

```text id="nqkdrq"
same stage configuration
same submission set
same eligible judge set
same R
same capacities
same panel scope
same algorithm version
same deterministic seed
```

Do not allow output to depend on:

* database insertion order
* hash-map iteration order
* request arrival order
* worker completion order
* random runtime state
* timestamps used as hidden tie-breakers

Use stable canonical IDs wherever a deterministic tie-break is required.

---

## 6.13 ASSIGNMENT VALIDATION

Every assignment mode must run through the same final validator.

The validator must check at minimum:

```text id="zcn2yx"
all submissions are eligible
all judges are eligible
every assignment belongs to the stage
every submission has required review count
no submission has duplicate judge
judge workloads satisfy quota rules
panel/track isolation is preserved
overlap invariants hold
connectivity holds where required
assignment version is coherent
```

The validator must return structured failure information.

Do not reduce validation to a single boolean if the existing architecture can support useful diagnostics.

---

## 6.14 FAILURE IS EXPLICIT

Assignment failure must be explicit.

Examples include:

```text id="8z2kck"
FEASIBILITY_TOO_FEW_JUDGES
FEASIBILITY_TOO_MANY_JUDGES
INVALID_JUDGE_POOL
INVALID_SUBMISSION_POOL
DUPLICATE_ASSIGNMENT
INVALID_REVIEW_COUNT
CAPACITY_EXCEEDED
DISCONNECTED_CALIBRATION_GRAPH
FAILED_ASSIGNMENT
```

Use the repository's established error representation where available.

Do not silently:

* lower `R`
* drop submissions
* drop judges
* assign ineligible judges
* create duplicates
* weaken track isolation
* accept disconnected calibration graphs

unless an explicitly documented organizer exception exists.

---

# 7. ASSIGNMENT INPUTS

The assignment engine receives a complete, validated judging-stage input set.

Conceptually:

```text id="5qzj7w"
stage
submission pool
eligible judge pool
R
judge capacities
panel scope
assignment configuration
deterministic seed/version
```

These inputs must be resolved by the backend.

Do not trust the frontend to define the authoritative judge or submission universe.

---

## 7.1 STAGE

The stage determines the judging universe.

The engine must know:

```text id="k5qqt6"
stage
event
scope/type
track if applicable
panel if applicable
required review count R
rubric/version context
assignment configuration
algorithm version
```

The assignment engine must reject an invalid or inactive stage configuration.

---

## 7.2 SUBMISSION POOL

The submission pool is the set of submissions eligible for the current judging stage.

The backend must derive this pool from the stage configuration and existing application data.

For Type 1:

```text id="q7p5f5"
submissions
=
global judging pool
```

For Type 2 track judging:

```text id="7qg4f0"
submissions
=
submissions belonging to the stage's track
```

For an overall stage:

```text id="qzj8t7"
submissions
=
organizer-selected finalists
```

The assignment engine must not assign a judge to a submission outside the stage's submission scope.

---

## 7.3 ELIGIBLE JUDGE POOL

The judge pool is the set of judges allowed to receive assignments for the stage.

Eligibility must be derived server-side.

A judge may be excluded because of:

* inactive/suspended/removed status
* not belonging to the required panel
* track mismatch
* event mismatch
* explicit organizer exclusion
* capacity restrictions
* other repository-defined eligibility rules

Do not allow the frontend to bypass these restrictions by supplying a judge ID manually.

---

## 7.4 TYPE 1 — NO TRACKS

For a global judging stage:

```text id="kpg8p0"
submissions
=
global eligible submission pool

judges
=
global eligible judge pool
```

All assignments operate inside this one judging universe.

The resulting overlap graph and calibration model apply to this global judge pool.

---

## 7.5 TYPE 2 — TRACK PANELS

For a track judging stage:

```text id="c4oy9u"
submissions
=
submissions belonging to the track

judges
=
eligible judges belonging to the track's panel
```

The assignment engine must enforce track isolation.

A judge from Panel A must not be assigned to Track B merely because that judge has available capacity.

Likewise, a Track B submission must not enter Track A's assignment pool.

Each track's assignment must be independently validated.

Do not combine multiple track assignment pools into one global assignment problem.

---

## 7.6 OVERALL STAGE

If an overall judging stage exists:

```text id="q6v4fi"
submissions
=
organizer-selected overall judging pool

judges
=
eligible overall panel
```

The overall stage is its own judging universe.

Its assignments, reviews, calibration, and ranking must be associated with the overall stage rather than silently reusing track assignments.

---

## 7.7 REQUIRED REVIEW COUNT `R`

`R` is the number of distinct judges required to review each submission in the stage.

Example:

```text id="g8x4uh"
R = 3
```

means:

```text
each submission
    ↓
exactly 3 distinct judges
```

The engine must not interpret `R` as:

* maximum reviews
* target reviews
* average reviews
* minimum reviews

unless an explicitly documented exception is active.

Normal assignment semantics are exact.

---

## 7.8 JUDGE CAPACITIES

The engine may receive judge capacity information.

Capacity must be interpreted consistently with the exact workload requirements in `JUDGING.md`.

When all eligible judges are equally available, the required quotas are:

```text id="1f40ul"
total_slots = N × R

baseQuota = floor(total_slots / J)

extra = total_slots mod J
```

Exactly `extra` judges receive:

```text
baseQuota + 1
```

and every other judge receives:

```text
baseQuota
```

Therefore:

```text
max(load) - min(load) <= 1
```

when all eligible judges participate under equal-capacity conditions.

If explicit capacity constraints make exact equalization impossible, the engine must detect and report the feasibility condition rather than silently violating the configured capacity.

---

## 7.9 PANEL SCOPE

Panel scope determines which judges are eligible for the stage.

For a panel-based stage:

```text id="7y8m40"
stage
  ↓
panel
  ↓
panel judges
  ↓
eligible judge pool
```

The assignment algorithm does not invent panel membership.

Panel membership is an organizer/configuration decision.

The assignment engine only assigns among judges who are already eligible for the stage.

---

## 7.10 ASSIGNMENT CONFIGURATION

The assignment configuration may contain values such as:

```text id="c3kqfi"
review count R
assignment mode
algorithm version
deterministic seed
capacity configuration
repair configuration
```

The exact configuration fields must follow the repository and `JUDGING.md`.

Configuration affecting assignment must be recorded with the resulting assignment version.

This allows historical assignments to be reproduced and audited.

---

## 7.11 DETERMINISTIC SEED / VERSION

If the algorithm uses a deterministic seed, the seed must be an explicit assignment input.

Do not use implicit randomness such as:

```text id="7g0n5c"
current timestamp
process ID
database insertion order
runtime random state
```

The assignment snapshot must record enough information to reproduce the result.

At minimum, associate the result with:

```text id="4v3qax"
assignment algorithm version
deterministic seed/configuration
judge pool
submission pool
R
stage
```

---

## 7.12 INPUT SNAPSHOT

Before constructing assignments, resolve the authoritative input set.

Conceptually:

```text id="t8yq0h"
Stage
   ↓
Resolve submissions
   ↓
Resolve eligible judges
   ↓
Resolve R
   ↓
Resolve capacities
   ↓
Resolve panel/track scope
   ↓
Resolve algorithm configuration
   ↓
Validate feasibility
   ↓
Construct assignment
```

Do not allow the underlying pools to change halfway through one assignment-generation operation.

Use the repository's transaction/versioning mechanisms to ensure the assignment is generated against a coherent snapshot.

---

## 7.13 INPUT VALIDATION

Before construction, validate:

### Stage

```text
stage exists
stage belongs to correct event
stage configuration is valid
stage is in a state that permits assignment generation
```

### Submissions

```text
submission exists
submission belongs to stage scope
submission is eligible
submission is not duplicated
```

### Judges

```text
judge exists
judge is active/eligible
judge belongs to correct event/panel/track where required
judge is not excluded
```

### Review count

```text
R > 0
R <= number of eligible judges
```

### Configuration

```text
assignment configuration is valid
algorithm version is known
deterministic inputs are available
```

If validation fails, do not begin assignment construction.

---

## 7.14 IMMUTABLE INPUTS DURING ONE RUN

Once assignment construction begins, the logical input snapshot must remain fixed.

Do not allow:

```text id="g6v3mt"
assignment starts
      ↓
judge pool changes
      ↓
submission pool changes
      ↓
assignment continues using mixed data
```

The assignment run must either:

1. operate on a consistent snapshot, or
2. detect the underlying version change and fail/retry deterministically.

Never silently produce an assignment based on a partially changed input set.

# 8. EXACT REVIEW COUNT

For a normal judging stage, every eligible submission must receive exactly `R` distinct judge assignments.

If:

```text id="e7y4qk"
R = 3
```

then the required assignment state is:

```text id="v4c9kp"
Submission A
    ├── Judge 1
    ├── Judge 4
    └── Judge 7
```

Therefore:

```text id="9a6wq3"
review_count(Submission A) = 3
```

The three judges must be distinct.

---

## 8.1 EXACT MEANS EXACT

`R` is the required number of independent judges per submission.

It must not be interpreted as:

* a maximum
* a target
* an average
* an approximate value

For:

```text id="r5w0fh"
R = 3
```

these are invalid normal assignments:

```text id="e7q9bm"
Submission A → 2 judges
Submission B → 4 judges
```

The system must not compensate for one submission having fewer reviews by giving another submission extra reviews.

The normal invariant is:

```text id="q7l1mz"
∀ submission s:

number_of_distinct_assigned_judges(s) = R
```

---

## 8.2 ASSIGNMENT VALIDATION

The assignment validator must check exact review count for **every submission** before an assignment version can become active.

Conceptually:

```text id="w3k2d8"
for every submission s:
    assigned_judges(s) == R
```

If any submission violates this:

```text id="j8x3p1"
assignment_status = INVALID
```

The stage must not transition to the active judging state.

Do not mark the assignment as successful merely because most submissions satisfy `R`.

---

## 8.3 UNDER-ASSIGNED SUBMISSION

Example:

```text id="0v9m8c"
R = 3

Submission A
    → Judge 1
    → Judge 4
```

This is incomplete.

The system must report that:

```text id="9j6q1r"
required = 3
actual = 2
missing = 1
```

It must not:

* mark the submission complete
* calculate its final score as if it had 3 judges
* silently lower `R`
* silently assign a random replacement
* hide the discrepancy from the organizer

The assignment engine should attempt deterministic repair/reassignment where the current workflow permits it.

If repair is impossible, the assignment must fail explicitly.

---

## 8.4 OVER-ASSIGNED SUBMISSION

Example:

```text id="7p5w2a"
R = 3

Submission A
    → Judge 1
    → Judge 4
    → Judge 7
    → Judge 9
```

This is also invalid for the normal assignment model.

Do not silently choose three of the four and discard the fourth.

The assignment must be rejected or explicitly repaired through the assignment/versioning workflow.

Historical assignments must not be silently deleted.

---

## 8.5 ORGANIZER EXCEPTION

An organizer may explicitly authorize a documented reduced-review exception if the normal requirement cannot be satisfied, for example after judge dropout.

Such an exception must **not** be treated as an ordinary `R`-complete submission.

The system must record at minimum:

```text id="4k8z3v"
submission
stage
required_R
actual_review_count
exception_reason
authorized_by
authorized_at
```

The exact persistence mechanism should use the existing audit/versioning architecture.

A reduced-review exception must remain visible to downstream judging logic.

Do not silently convert:

```text id="j3m0tw"
R = 3
actual = 2
```

into:

```text id="e1v9cd"
R = 2
```

for mathematical convenience.

---

## 8.6 DOWNSTREAM EFFECT OF REDUCED REVIEW

The standard judging formulas assume the required valid review set.

Therefore, if a reduced-review exception is permitted, downstream calibration/ranking logic must explicitly recognize that exception.

Do not treat a missing review as a score of zero.

Do not invent a replacement score.

Do not duplicate another judge's score.

The exact reduced-review behavior must follow `JUDGING.md`.

If `JUDGING.md` does not define a particular reduced-review case, do not invent a mathematical rule silently; surface the case as an explicit organizer/system exception.

---

## 8.7 REVIEW COUNT VS ASSIGNMENT COUNT

The system must distinguish:

```text id="o7r0r4"
assigned judges
```

from:

```text id="9b5m3k"
completed valid reviews
```

For example:

```text id="x7h1qf"
R = 3

Assignments:
    Judge 1
    Judge 4
    Judge 7

Completed reviews:
    Judge 1
    Judge 4
```

The submission has:

```text
3 assigned judges
2 completed reviews
```

It must **not** be treated as having completed three reviews.

Assignment completeness and judging completeness are separate concepts.

---

## 8.8 VALID REVIEW COUNT

A review counts toward the required review count only if it is valid according to the judging rules.

A review that is:

* invalid
* rejected
* cancelled
* unauthorized
* associated with the wrong stage
* associated with the wrong assignment/version

must not silently count toward `R`.

The authoritative count must be derived server-side.

---

## 8.9 TESTS

Test at minimum:

### Exact assignment

```text
R = 3
N submissions
```

Verify every submission has exactly 3 distinct assigned judges.

### Under-assignment

```text
R = 3
submission has 2 assignments
```

Verify the assignment is invalid/incomplete.

### Over-assignment

```text
R = 3
submission has 4 assignments
```

Verify the assignment is invalid unless an explicitly defined workflow handles it.

### Completed review count

Verify:

```text
3 assignments
2 completed reviews
```

does not count as 3 completed reviews.

### Exception

Verify that an organizer-authorized reduced-review exception is:

* explicitly recorded
* auditable
* not silently converted into a lower `R`

---

# 9. NO DUPLICATE JUDGES

A judge must not be assigned to review the same submission more than once within the same judging stage/assignment universe.

The core invariant is:

```text id="4n8v1x"
unique(stage_id, submission_id, judge_id)
```

The exact database key may follow the existing schema/versioning model, but the semantic rule must remain:

```text id="3x1z8q"
one judge
+
one submission
+
one judging stage
=
at most one active assignment
```

---

## 9.1 ASSIGNMENT-LEVEL DUPLICATE PREVENTION

The assignment constructor must reject duplicate judge/submission pairs.

Invalid:

```text id="8s4v7c"
Submission A
    → Judge 1
    → Judge 1
    → Judge 4
```

Even if the total assignment count happens to equal `R`, this is invalid because Judge 1 does not provide two independent reviews.

For:

```text id="l5t2q8"
R = 3
```

the valid structure is:

```text id="m1c9xa"
Submission A
    → Judge 1
    → Judge 4
    → Judge 7
```

not:

```text id="d2r7vk"
Submission A
    → Judge 1
    → Judge 1
    → Judge 4
```

---

## 9.2 DATABASE ENFORCEMENT

Where the existing database architecture permits it, enforce duplicate prevention with a database-level uniqueness constraint.

Conceptually:

```text id="v9s3q0"
UNIQUE(stage_id, submission_id, judge_id)
```

If assignment versioning requires historical assignment rows to coexist, the physical constraint must be designed around the repository's versioning model while preserving the semantic rule that the same judge cannot have two active assignments for the same submission in the same judging universe.

Do not rely exclusively on frontend validation.

Do not rely exclusively on application-level checks.

The database should provide the final integrity boundary where practical.

---

## 9.3 ASSIGNMENT VS REVIEW DUPLICATES

Duplicate assignment prevention and duplicate review prevention are related but separate.

The system must prevent both:

### Duplicate assignment

```text id="q8w1kc"
Judge 1
    assigned twice
    to Submission A
```

and:

### Duplicate review

```text id="c2y9n7"
Judge 1
    submits Review A
    submits another Review A
    for the same assignment
```

The review layer must therefore have its own uniqueness/integrity protection.

A review must belong to exactly one assignment.

---

## 9.4 SAME JUDGE ACROSS DIFFERENT SUBMISSIONS

The same judge may review many different submissions.

This is valid:

```text id="0z5h4v"
Judge 1 → Submission A
Judge 1 → Submission B
Judge 1 → Submission C
```

The restriction is specifically:

```text id="k1q6wp"
same judge
+
same submission
+
same judging stage
```

not:

```text id="t7v4ne"
same judge
+
multiple submissions
```

---

## 9.5 SAME JUDGE ACROSS DIFFERENT STAGES

A judge may review the same submission in different judging stages if the product configuration explicitly permits it.

For example:

```text id="5n3y8x"
Stage 1 → Judge 1 → Submission A

Stage 2 → Judge 1 → Submission A
```

is not automatically a duplicate because the judging universes are different.

Authorization and stage configuration must determine whether the second assignment is allowed.

Do not use a global:

```text
UNIQUE(submission_id, judge_id)
```

constraint if that would incorrectly prevent legitimate independent judging stages.

---

## 9.6 TRACK ISOLATION

For track-based judging, duplicate validation must operate inside the stage/track judging universe.

A judge cannot bypass track isolation by creating a second assignment record manually.

For example, if Judge 1 is eligible only for Track A:

```text id="5z0p9x"
Track A / Submission A → Judge 1
```

may be valid, while:

```text id="w8r2v6"
Track B / Submission B → Judge 1
```

must be rejected unless Judge 1 is explicitly eligible for Track B.

Track membership must be determined server-side.

---

## 9.7 CONCURRENCY

Duplicate prevention must remain correct under concurrent requests.

Example:

```text id="7j4r2m"
Request A:
Judge 1 → Submission A

Request B:
Judge 1 → Submission A
```

Both requests may arrive at nearly the same time.

The system must not allow both to become active assignments.

Use the existing transaction/database concurrency mechanisms.

Application-level:

```text id="w4x6s2"
check → then insert
```

is insufficient by itself if two requests can pass the check concurrently.

---

## 9.8 REASSIGNMENT AND VERSIONING

If an assignment is replaced because of:

* judge dropout
* organizer correction
* assignment repair
* explicit reassignment

historical assignment records must remain auditable.

A new assignment version may contain:

```text id="2m8r4c"
Submission A → Judge 7
```

after the previous assignment was:

```text
Submission A → Judge 1
```

provided the old assignment is no longer active and the new assignment satisfies all assignment rules.

Do not create two simultaneously active assignments for the same:

```text
stage + submission + judge
```

pair.

---

## 9.9 VALIDATION ORDER

Duplicate validation should occur:

1. during candidate construction
2. during assignment validation
3. at persistence/database level where practical

The system should detect duplicates as early as possible, but the final integrity boundary must not depend only on the constructor.

---

## 9.10 TESTS

Test at minimum:

### Direct duplicate

```text id="9m6q4t"
Submission A → Judge 1
Submission A → Judge 1
```

Must be rejected.

### Duplicate under R

```text id="x4z7kp"
R = 3

Judge 1
Judge 1
Judge 4
```

Must be rejected even though there are three assignment rows.

### Concurrent duplicate

Two simultaneous requests creating:

```text id="p5k1y9"
stage + submission + judge
```

must result in at most one active assignment.

### Different submissions

```text id="m7c3x8"
Judge 1 → Submission A
Judge 1 → Submission B
```

must be allowed when otherwise eligible.

### Different stages

The same judge/submission pair may be allowed in different stages when stage configuration permits it.

### Track isolation

A judge must not be able to create a duplicate/cross-track assignment through direct API manipulation.

# 10. EXACT WORKLOAD BALANCING

Workload balancing is a **hard assignment requirement**, not merely an optimization preference.

Let:

```text
N = number of submissions in the assignment scope
R = required number of distinct judges per submission
J = number of active eligible judges

total_slots = N × R
```

The assignment engine must distribute exactly `total_slots` assignment slots across the `J` eligible judges.

Calculate:

```text
baseQuota = floor(total_slots / J)

extra = total_slots mod J
```

Exactly `extra` judges receive:

```text
baseQuota + 1
```

assignments.

Every other eligible judge receives:

```text
baseQuota
```

assignments.

Therefore:

```text
max(load) - min(load) <= 1
```

when all eligible judges participate in the assignment pool.

## 10.1 Quota Invariants

For every successful assignment:

```text
Σ load(j) = total_slots
```

and for every eligible judge:

```text
load(j) ∈ {baseQuota, baseQuota + 1}
```

and:

```text
number_of_judges_with_load(baseQuota + 1) = extra
```

The validator must reject an assignment if these conditions do not hold.

Do not replace exact balancing with arbitrary tolerances such as:

```text
±5%
±10%
"approximately balanced"
```

when exact balancing is mathematically possible.

## 10.2 Interaction With Overlap

Workload balancing and overlap balancing are separate objectives.

A judge may not be given extra assignments merely because doing so improves overlap if that causes the exact workload invariant to fail.

The assignment engine must therefore treat:

1. eligibility,
2. exact review count,
3. distinctness,
4. exact workload quotas,
5. panel/track isolation,
6. connectivity

as hard constraints.

Overlap quality is optimized only within the space of assignments satisfying those hard constraints.

## 10.3 Failure Behavior

If the requested configuration cannot satisfy exact workload balancing together with the other hard constraints, the assignment must not be silently published.

The backend must return a deterministic failure state explaining which constraint could not be satisfied.

Do not silently:

* reduce `R`;
* remove eligible judges;
* add ineligible judges;
* change the judging scope;
* weaken track isolation;
* publish an under-balanced assignment.

Any organizer override must be explicit, authorized, versioned, and audited.

# 11. DELIBERATE OVERLAP

Overlap is an intentional property of the assignment design.

It creates the shared observations required for cross-judge calibration.

For judges `i` and `j`, define:

```text
Oij =
number of submissions reviewed by both judges
```

For `K` submissions and `R` reviews per submission:

```text
Σ Oij = K × C(R,2)
```

where the sum is over all unordered judge pairs.

This invariant must be validated after assignment construction.

## 11.1 Why The Invariant Holds

Each submission assigned to `R` distinct judges creates exactly:

```text
C(R,2)
```

judge pairs that overlap on that submission.

Across `K` submissions:

```text
K × C(R,2)
```

pair-overlap contributions are created.

The assignment implementation must count overlap from the actual assignment relation and verify this equality.

## 11.2 Ideal Pair Overlap

The ideal average pair overlap is:

```text
P =
K × C(R,2)
----------------
C(J,2)
```

where:

```text
K = number of submissions
R = reviews per submission
J = number of eligible judges
```

`P` is the target average overlap across judge pairs. It does not require every pair to have exactly `P` overlaps, since `P` may not be an integer and the assignment constraints may make perfect equality impossible.

## 11.3 PairLoss

Define:

```text
PairLoss =
Σ(i<j) (Oij - P)²
```

The assignment constructor should minimize `PairLoss` subject to the hard assignment constraints.

The objective is therefore to produce **broad, balanced overlap** across the judge population.

Do not optimize for one preferred judge pair.

Do not maximize the overlap of a single pair while leaving other judges disconnected or weakly connected.

A lower `PairLoss` is useful only when the candidate assignment remains valid under all hard constraints.

## 11.4 R = 1 Special Case

When:

```text
R = 1
```

there is no pairwise overlap:

```text
C(1,2) = 0
```

Therefore:

```text
Σ Oij = 0
```

and there is no cross-judge calibration information.

The implementation must not attempt to construct a cross-judge overlap graph or invoke WLS normalization as though overlap existed.

The `R = 1` path must be handled explicitly.

# 12. SCALABLE ASSIGNMENT ALGORITHM

The assignment engine must not enumerate every possible:

```text
C(J,R)
```

judge combination for every submission.

Such exhaustive enumeration does not scale and is unnecessary for the required assignment behavior.

The implementation must use the greedy construction defined in `JUDGING.md`, followed by deterministic validation and repair.

## 12.1 Construction Process

For each submission:

1. Build the set of currently feasible judges.
2. Remove judges that would violate hard constraints.
3. Evaluate the remaining candidates.
4. Select the candidate with the lexicographically smallest selection tuple.
5. Add that judge to the submission's assignment.
6. Update workload and overlap state.
7. Recalculate candidate effects.
8. Repeat until exactly `R` distinct judges have been selected.
9. Continue until every submission has exactly `R` assignments.
10. Run complete assignment validation.
11. Run deterministic repair if the constructed assignment requires repair.
12. Publish the assignment only if final validation succeeds.

The constructor must never publish a partially validated assignment as active.

## 12.2 Marginal PairLoss

For candidate judge `j`, define:

```text
ΔPairLoss(j) =

Σk [
    (Ojk + 1 - P)²
    -
    (Ojk - P)²
]
```

where `k` ranges over the other judges whose current overlap with `j` would be affected.

This represents the marginal change in `PairLoss` from adding judge `j` to the current submission.

A lower value is preferred, subject to hard constraints.

## 12.3 Selection Tuple

Candidate judges must be compared using:

```text
(
    connectivity_penalty,
    ΔPairLoss,
    quota_penalty,
    canonical_judge_id
)
```

The tuple is evaluated lexicographically.

This provides deterministic tie-breaking while prioritizing the assignment properties required by the judging model.

The exact meaning of each component must remain consistent throughout the implementation:

```text
connectivity_penalty
    → effect on maintaining/building the required overlap connectivity

ΔPairLoss
    → marginal effect on global pair-overlap balance

quota_penalty
    → effect on satisfying exact workload quotas

canonical_judge_id
    → deterministic final tie-break
```

## 12.4 Stable Determinism

The implementation must use stable canonical judge identifiers.

It must not use any of the following as an implicit tie-break:

```text
database insertion order
database row order without explicit ordering
random runtime order
thread completion order
request completion time
timestamp of object creation
```

Identical logical inputs and identical configuration must produce the same assignment output.

This is required for reproducible testing, debugging, auditing, and finalization.

## 12.5 Heuristic Boundary

The greedy constructor is a **heuristic construction algorithm**.

It is not required to prove global combinatorial optimality of:

```text
PairLoss
```

or of the complete assignment problem.

The correctness requirement is instead:

```text
construct
    ↓
validate
    ↓
repair if required
    ↓
validate again
    ↓
publish only if valid
```

A mathematically attractive heuristic result is still invalid if it violates a hard constraint.

Conversely, the implementation must not claim that the greedy result is globally optimal unless such a proof/algorithm is actually implemented.

## 12.6 Failure Behavior

If the constructor cannot produce a valid assignment:

```text
FAILED_ASSIGNMENT
```

must be returned.

The system must not silently publish the best partial result.

It must not silently lower `R`, remove judges, duplicate judges, or weaken isolation to force success.

# 13. CONNECTIVITY

When cross-judge normalization is required, construct the **judge overlap graph**.

```text
node = judge

edge(i,j) =
    judges i and j share at least one valid assigned submission
```

The graph is based on actual assignments, not merely on configured eligibility.

## 13.1 Connectivity Requirement

For a single global normalization model, the judge overlap graph must be connected.

Formally, for every pair of judges:

```text
there exists a path from judge i to judge j
```

through judge-overlap edges.

This is required because calibration estimates relative judge offsets.

For example, if the graph is:

```text
Judge A ─ Judge B


Judge C ─ Judge D
```

with no overlap path between the two components, the relative offset between:

```text
{A,B}
```

and:

```text
{C,D}
```

cannot be identified from shared submissions.

The calibration system therefore cannot determine one common zero-sum set of judge offsets from the observed overlap data alone.

## 13.2 Required Failure

If a stage requiring one normalization model produces a disconnected graph, calibration must fail with:

```text
DISCONNECTED_CALIBRATION_GRAPH
```

This is a real domain failure and must be surfaced explicitly.

It must not be converted into a successful calibration by silently making assumptions.

## 13.3 No Fabricated Offsets

The implementation must never:

* assign arbitrary offsets to disconnected components;
* normalize each component independently and pretend the results are globally comparable;
* anchor an unconnected component to zero without a valid shared observation;
* fabricate overlap records;
* invent review scores to connect components.

The system must fail closed.

## 13.4 Connectivity Validation

After assignment construction, the backend must:

1. Build the overlap graph from actual assignments.
2. Identify all active judges in the judging scope.
3. Traverse the graph.
4. Determine the number of connected components.
5. Require exactly one component when a single normalization model is required.
6. Persist the validation result with the assignment version.

A disconnected result must prevent progression into calibration.

## 13.5 R = 1

When:

```text
R = 1
```

no submission is shared by two judges.

Therefore the overlap graph contains no edges between judges.

This is expected and does **not** represent a broken assignment for the direct-score path.

However, the system must not run cross-judge normalization in this configuration.

For `R >= 2`, connectivity is a required condition whenever the stage uses one shared calibration model.

## 13.6 Track Isolation

Connectivity is evaluated within the relevant judging universe.

For a track-isolated stage:

```text
Track A judges
    ↕
Track A submissions
```

must be calibrated independently from:

```text
Track B judges
    ↕
Track B submissions
```

A judge overlap edge across tracks must not be used to create a cross-track calibration graph unless the stage configuration explicitly defines a common judging universe.

Track isolation therefore applies to:

* assignment construction;
* overlap calculation;
* connectivity validation;
* calibration;
* normalized scores;
* ranking.

The assignment engine must never create cross-track calibration merely because the same person is eligible to judge multiple tracks.

# 14. ASSIGNMENT REPAIR

Initial construction is not sufficient by itself.

Every constructed assignment must pass a complete validation phase before it can become an active assignment snapshot.

Validate at minimum:

```text
exact review count
judge distinctness
judge eligibility
judge workload quotas
pair-overlap invariant
pair-overlap balance
graph connectivity where applicable
panel/track isolation
determinism
```

The validator must evaluate the **actual constructed assignment**, not merely the inputs used to construct it.

## 14.1 Repair Trigger

If the initial assignment violates a repairable constraint, the backend may attempt deterministic local repair.

Repair must not be used to conceal a fundamentally infeasible configuration.

Hard impossibilities should fail immediately with the appropriate feasibility error.

## 14.2 Local Swap

A repair operation may replace:

```text
submission S → removed judge J1
```

with:

```text
submission S → added judge J2
```

provided the resulting assignment remains valid with respect to all hard constraints.

The candidate swap must be evaluated against the complete current assignment state.

Do not evaluate a swap only against the affected submission while ignoring its effect on:

* removed judge workload;
* added judge workload;
* other pair overlaps;
* graph connectivity;
* judge eligibility;
* track/panel isolation;
* duplicate assignments;
* exact `R`.

## 14.3 Candidate Ordering

Candidate repair operations must be ordered lexicographically using:

```text
(
    hard_constraint_status,
    connectivity_penalty,
    ΔPairLoss,
    affected_submission_count,
    removed_judge_id,
    added_judge_id,
    submission_id
)
```

The implementation must choose the **lexicographically smallest valid improving swap**.

The ordering must be deterministic.

Canonical judge identifiers and canonical submission identifiers must be used for the final deterministic components.

## 14.4 Improving Swap

A repair swap must actually improve the relevant invalid condition or objective.

Do not perform arbitrary swaps merely because they produce another valid assignment.

The repair engine should prefer a swap that:

1. preserves all hard constraints;
2. repairs the current violation;
3. improves connectivity when connectivity is deficient;
4. reduces `PairLoss` where overlap balance is the relevant issue;
5. improves workload compliance where applicable;
6. changes the smallest necessary assignment scope.

## 14.5 Repair Safety Ceiling

Repair must have a deterministic safety ceiling.

The implementation must prevent infinite loops caused by repeatedly reversing equivalent swaps.

The ceiling may be based on a configured maximum number of repair iterations or candidate evaluations, but it must be deterministic and recorded.

The repair engine must also avoid revisiting the same assignment state indefinitely.

## 14.6 Final Validation

After every successful repair phase:

```text
construct
    ↓
repair
    ↓
validate
```

must result in a fully valid assignment.

If validation still fails:

```text
FAILED_ASSIGNMENT
```

must be returned.

No partially repaired assignment may become active.

## 14.7 Fail Closed

If repair cannot produce a valid assignment, the system must fail closed.

It must not silently:

```text
reduce R
remove judges
duplicate judges
ignore capacity violations
weaken track isolation
skip connectivity requirements
publish an invalid assignment
```

Any organizer-authorized exception must be explicit, versioned, and auditable.

# 15. MANUAL ASSIGNMENT

The organizer must be able to create assignments manually.

For example:

```text
Submission A → Judge 1
Submission A → Judge 3
Submission A → Judge 7
```

Manual mode is an alternative **assignment construction method**.

It is not an alternative validation model.

## 15.1 Server-Side Validation

Every manually entered assignment must pass the same authoritative backend validation used for generated assignments.

The backend must validate at minimum:

```text
judge eligibility
submission eligibility
stage/panel scope
track isolation
judge distinctness
exact R requirement
capacity constraints
assignment version
connectivity where applicable
```

The client must never be trusted to enforce these rules.

## 15.2 Useful Validation Errors

If a manual assignment is rejected, return a structured error identifying the violated rule.

Examples:

```text
JUDGE_NOT_ELIGIBLE
DUPLICATE_JUDGE_ASSIGNMENT
REVIEW_COUNT_INVALID
JUDGE_CAPACITY_EXCEEDED
TRACK_ISOLATION_VIOLATION
DISCONNECTED_CALIBRATION_GRAPH
SUBMISSION_NOT_IN_STAGE
JUDGE_NOT_IN_PANEL
```

The exact error should identify enough context for the organizer to correct the assignment without exposing private judging information to unauthorized actors.

## 15.3 Manual Assignment Completion

A manual assignment must not become active merely because individual rows were successfully entered.

The complete assignment snapshot must pass final validation before judging can open.

Therefore:

```text
manual edits
    ↓
draft assignment
    ↓
complete validation
    ↓
assignment snapshot
    ↓
OPEN
```

This prevents an organizer from accidentally opening a partially configured judging stage.

# 16. BATCH ASSIGNMENT

The organizer must be able to select a set of submissions and generate assignments for that set.

For example:

```text
40 submissions
10 judges
R = 3
```

The backend must treat the batch as **one coherent assignment problem**.

It must not independently optimize each submission without considering the global effects on:

```text
judge workload
pair overlap
connectivity
eligibility
track/panel isolation
exact review count
```

## 16.1 Batch Inputs

A batch assignment request must resolve authoritative server-side inputs:

```text
judging stage
submission set
eligible judge set
R
judge capacities
panel/track scope
assignment configuration
assignment version
deterministic seed/configuration where applicable
```

The backend must reject stale or inconsistent input snapshots rather than combining data from different assignment states.

## 16.2 Global Construction

The assignment engine must calculate workload and overlap across the entire batch.

For:

```text
N = 40
R = 3
J = 10
```

the total number of assignment slots is:

```text
40 × 3 = 120
```

The workload and overlap objectives must therefore be evaluated over the complete 120-slot assignment rather than restarting their state independently for each submission.

## 16.3 Atomic Publication

The batch must be published as one coherent assignment snapshot.

Do not expose a partially generated batch as the active judging assignment.

If construction or validation fails, the active assignment must remain unchanged.

This ensures that a judging stage cannot begin with only part of its intended assignment.

# 17. ASSIGNMENT VERSIONING

Every generated or manually finalized assignment must have an explicit version.

Example:

```text
assignment_version = 7
```

The version identifies the exact assignment state used by the judging process.

## 17.1 Required Metadata

Store at minimum:

```text
assignment version
algorithm version
assignment configuration
deterministic seed/configuration
judge pool
submission pool
R
panel/track scope
assignment result
validation result
created timestamp
creator/actor
```

Where appropriate, also persist the deterministic input snapshot or hashes representing the resolved input sets.

## 17.2 Immutable Published Versions

Once an assignment version is published for judging, its historical assignment records must not be silently mutated.

If a change is required:

```text
assignment version 7
        ↓
new assignment construction
        ↓
assignment version 8
```

The new version must be separately validated and auditable.

Do not update version 7 in place and pretend that version 8 was always the active assignment.

## 17.3 Review Reference

A review must identify the assignment version under which it was created.

Conceptually:

```text
Review
  ↓
Assignment Version
  ↓
Judge + Submission
```

This prevents a historical review from becoming ambiguous after reassignment.

A review created under assignment version 7 must not silently become a review under assignment version 8.

## 17.4 Reassignment

Reassignment must preserve historical state.

When a judge drops out or an organizer changes assignments:

1. Preserve the old assignment version.
2. Determine outstanding work.
3. Construct the replacement assignment.
4. Validate it.
5. Create a new assignment version.
6. Activate the new version according to stage rules.
7. Preserve completed valid reviews from the prior version.

Historical assignment state must remain auditable.

# 18. JUDGE DROPOUT

Judge dropout is a controlled reassignment event.

If an active judge becomes unavailable:

1. Keep completed valid reviews from that judge.
2. Remove the judge from future assignment eligibility.
3. Identify outstanding assignments belonging to that judge.
4. Determine the remaining review requirements.
5. Determine the new active eligible judge pool.
6. Construct replacement assignments.
7. Preserve judge distinctness.
8. Rebalance remaining workload.
9. Recalculate overlap.
10. Revalidate connectivity where applicable.
11. Validate the replacement assignment.
12. Freeze the replacement assignment snapshot.
13. Activate the replacement version according to stage state.

Completed valid reviews must not be rewritten merely because the judge dropped out.

## 18.1 Remaining Review Slots

Let:

```text
remaining_slots =
number of outstanding review assignments that must still be completed
```

and:

```text
active_judges =
number of judges eligible to perform replacement work
```

Then:

```text
q_min = floor(remaining_slots / active_judges)

q_max = ceil(remaining_slots / active_judges)
```

The replacement assignment should distribute the remaining work within these exact bounds when all active judges are eligible for the replacement pool.

The implementation must distinguish:

```text
historical workload
```

from:

```text
replacement workload
```

Do not rewrite historical workload merely to make current workload totals appear balanced.

## 18.2 Replacement Constraints

Replacement assignments must still satisfy:

```text
judge eligibility
judge distinctness
submission eligibility
panel/track isolation
remaining review requirement
capacity constraints
overlap requirements where applicable
connectivity where applicable
```

A replacement judge must not be assigned merely because they have capacity.

The complete assignment must remain mathematically valid.

## 18.3 Failed Replacement

If the remaining work cannot be assigned validly:

```text
FAILED_REPLACEMENT_CAPACITY
```

must be returned.

Do not silently:

```text
reduce R
assign an ineligible judge
assign the same judge twice
break track isolation
invent a review
rewrite completed reviews
```

The organizer must explicitly authorize any reduced-review exception.

Such an exception must be recorded and audited rather than being represented as a normal `R`-complete judging stage.

# 19. JUDGE INVITATION

Judge participation must follow the existing authentication and account architecture.

The organizer workflow is:

```text
Organizer
    ↓
Invite Judge
    ↓
Invitation
    ↓
Judge accepts
    ↓
Judge becomes ACTIVE
```

## 19.1 Invitation State

An invitation must not automatically make a person an active judge.

The judge becomes eligible for assignment only after the required acceptance/activation step succeeds.

Conceptually:

```text
INVITED
   ↓
ACCEPTED
   ↓
ACTIVE
```

Only `ACTIVE` judges should enter the eligible judge pool unless the stage explicitly defines another eligibility state.

## 19.2 Existing Authentication

Use the existing authentication/session architecture from Tier 1.

Do not introduce a second authentication system for judges.

Do not create a parallel user table merely for judging if the existing system already represents users/accounts.

The judging domain should extend the existing role model.

## 19.3 Authorization

Invitation and activation operations must be server-authorized.

The backend must verify:

```text
actor identity
actor role
event ownership/scope
target user
judge state
```

A client must not be able to activate itself as a judge by changing a request parameter.

## 19.4 Auditability

Judge invitation state changes should be auditable.

At minimum, the system should be able to determine:

```text
who invited the judge
which event the invitation belonged to
when the invitation was created
whether it was accepted
when the judge became active
```

Do not expose invitation or account-management information to unauthorized participants or other judges.

# 20. RUBRIC CONFIGURATION

The organizer must be able to configure the rubric used by a judging stage **when that stage selects the rubric judging method**. A Pairwise stage does not require a rubric unless the explicit stage configuration says otherwise.

The supported workflow for a rubric stage is:

```text
create rubric
    ↓
add criterion
    ↓
set maximum score
    ↓
set weight
    ↓
reorder criteria
    ↓
validate
    ↓
save rubric version
    ↓
freeze for judging
```

## 20.1 Criterion

Each criterion must have, at minimum:

```text
criterion identity
name/label
maximum score
weight
display/order position
```

Criteria must belong to a specific rubric version.

## 20.2 Validation

Before a rubric can be used for judging:

```text
all weights > 0
sum(weights) = 1
all max scores > 0
```

The backend must perform this validation.

Do not rely on frontend validation for mathematical correctness.

If the weights do not sum to exactly `1` under the system's defined numeric representation/tolerance policy, the rubric must not become active.

## 20.3 Versioning

Rubrics are versioned configuration.

For example:

```text
Rubric v1
    ↓
frozen for Stage 1
```

If an organizer later wants to change the rubric:

```text
Rubric v1
    ↓
Rubric v2
```

The implementation must not silently alter v1 after reviews have begun.

## 20.4 Frozen Judging Rubric

Once judging starts, the stage must reference a frozen rubric version.

Every review must be associated with the exact rubric version used to produce its raw score.

Conceptually:

```text
Review
  ↓
Rubric Version
```

This ensures that historical scores remain interpretable even if an organizer creates a later rubric version.

## 20.5 No Mid-Stage Mutation

Do not allow an organizer to modify:

```text
criterion weight
maximum score
criterion identity
criterion ordering
```

in a way that changes the meaning of already-submitted reviews.

If a substantive change is required, create a new rubric version and apply it according to the stage/versioning rules.

Any downstream score, calibration, ranking, or finalization state affected by such a change must be treated as stale and recalculated from the appropriate versioned inputs.

# 21. RAW RUBRIC SCORE

The backend must calculate every submitted review's raw weighted score from the frozen rubric version associated with that review.

For criterion `c`:

```text id="p5n4cl"
q(s,j,c) =
    score(s,j,c) / max_c
```

where:

```text id="0v8wjy"
score(s,j,c) = judge j's score for criterion c
max_c        = maximum allowed score for criterion c
```

The raw weighted review score is:

```text id="m6k3zj"
r(s,j) =
    100 × Σ_c weight_c × q(s,j,c)
```

Therefore:

```text id="6g2k1f"
0 <= r(s,j) <= 100
```

provided the criterion scores satisfy the rubric's configured bounds and the rubric weights satisfy the rubric invariants.

## 21.1 Backend Authority

The calculation must exist in one authoritative backend/domain function.

Conceptually:

```text id="6qv8f3"
calculateRawReviewScore(review, rubricVersion)
```

All downstream scoring operations must consume this authoritative raw score.

Do not independently reimplement the formula in:

```text id="a2f9rm"
frontend
backend API layer
ranking code
export code
```

The frontend may display the calculated value, but the frontend must never be the source of truth.

## 21.2 Criterion Validation

Before calculating `r(s,j)`, the backend must verify that:

```text id="f4qz9m"
criterion belongs to the referenced rubric version
score >= 0
score <= max_c
criterion weight > 0
rubric version is valid/frozen
```

A review with an invalid criterion score must not enter calibration or ranking.

## 21.3 Raw Score Immutability

Once a review is finally submitted, its raw score is part of the historical judging record.

The system must not silently recalculate or overwrite it because:

```text id="u2lq7b"
the frontend changed
the current rubric changed
the judge's assignment changed
the judge later dropped out
the ranking was recalculated
```

If a correction is required, use the explicit correction/invalidation/versioning process.

Any correction that changes a raw score must invalidate affected downstream calibration and ranking state.

# 22. REVIEW WORKFLOW

A judge must be able to complete the following workflow:

```text id="1y2qg0"
view assigned submission
        ↓
view applicable rubric
        ↓
enter criterion scores
        ↓
write feedback
        ↓
save draft
        ↓
resume draft
        ↓
submit final review
```

## 22.1 Assignment Authorization

A judge may submit a review only when the backend verifies that:

```text id="6n3fqs"
authenticated actor = judge
judge is authorized for the event
judge is authorized for the stage
judge is assigned to the submission
assignment version is valid
submission belongs to the judging scope
rubric version is valid
```

The client-provided `judge_id`, `submission_id`, or `assignment_id` must never be trusted by itself.

The server must resolve and verify the relationship.

## 22.2 Draft Reviews

Draft reviews may be saved and resumed by the owning judge.

A judge must only be able to access their own draft reviews.

Draft state must not become visible to other judges.

Draft reviews must not participate in:

```text id="n7y9bz"
calibration
normalization
ranking
finalization
```

until they are validly submitted.

## 22.3 Final Submission

A final review must satisfy all required rubric validation before submission succeeds.

The backend must atomically:

1. validate the assignment;
2. validate rubric criteria;
3. validate score bounds;
4. calculate the raw score;
5. persist the final review;
6. persist the authoritative raw score;
7. mark the review submitted.

This prevents partially persisted final reviews.

## 22.4 Immutable Final Review

After final submission, the judge must not be able to edit the review.

Any legitimate correction must use the explicit organizer correction/invalidation workflow.

Corrections must preserve the historical version/audit trail and invalidate downstream derived results where required.

# 23. JUDGE UX

The judge-facing interface should clearly communicate only the judge's own workload and review state.

Example:

```text id="j6f8e3"
My Judging

15 assigned
11 completed
4 remaining
```

Submission list:

```text id="q0e7nm"
Project A    Complete
Project B    Complete
Project C    In Progress
Project D    Not Started
```

The counts must be calculated from authoritative assignment/review state.

In particular:

```text id="y5u1j7"
assigned = valid assignments belonging to the current judge
completed = valid final reviews belonging to the current judge
remaining = assigned - completed
```

## 23.1 Information Isolation

A judge must not receive:

```text id="x1g4fa"
other judges' scores
other judges' comments
other judges' drafts
live ranking
aggregate ranking
cross-judge calibration results
normalization offsets
another track's private projects
another track's judging context
```

This restriction applies to the backend API as well as the UI.

Do not implement security by merely hiding fields in the frontend.

An endpoint returning peer scores that the UI happens not to render is still an information leak.

## 23.2 Scope Enforcement

Every judge-facing data request must enforce:

```text id="k6x0fa"
actor
event
stage
panel/track
assignment ownership
submission scope
```

server-side.

Changing an ID in a request must not allow a judge to access another judge's review or another track's submission.

## 23.3 Judge Progress

Progress indicators must not expose aggregate judging information.

For example, showing:

```text
11 of my 15 completed
```

is permitted.

Showing:

```text
82% of all judges have completed judging
```

must be treated as organizer/aggregate information rather than judge-private progress.

# 24. CROSS-JUDGE NORMALIZATION

When `R >= 2` and the stage requires cross-judge calibration, do **not** independently z-score each judge.

Do **not** simply average raw judge scores.

The system must use the additive Weighted Least Squares model defined in `JUDGING.md`.

For judges `i` and `j`:

```text id="svz9u3"
ωij =
number of valid shared submissions
```

Only valid, authorized, finalized reviews from the same calibration universe may contribute to `ωij`.

Define the mean score difference:

```text id="1e2ps9"
d̄ij =

(1 / ωij)
Σ_s [r(s,i) - r(s,j)]
```

where the sum is over valid submissions reviewed by both judges.

Let `bj` represent judge `j`'s estimated systematic scoring offset.

Estimate the judge offsets by minimizing:

```text id="1g2hqb"
L_cal =

Σ(i<j)
ωij [
    (bi - bj) - d̄ij
]²
```

subject to:

```text id="x0v7nz"
Σ_j bj = 0
```

The zero-sum constraint fixes the otherwise arbitrary additive constant.

## 24.1 Calibration Universe

Calibration must only use reviews belonging to the same normalization universe.

For a global stage:

```text id="h8k5ap"
global eligible judges
        ↕
global valid overlap
```

For an isolated track:

```text id="x8q2mz"
Track A judges
        ↕
Track A valid overlap
```

must be calibrated independently from another track.

A shared person being eligible in multiple tracks does not by itself create a cross-track calibration relationship.

## 24.2 Valid Review Requirement

Only valid finalized reviews may contribute to:

```text id="1o8h3s"
r(s,j)
ωij
d̄ij
L_cal
```

Draft, invalidated, unauthorized, duplicate, or otherwise excluded reviews must not contribute.

## 24.3 R = 1

When:

```text id="c6zq1x"
R = 1
```

there is no cross-judge overlap.

Therefore cross-judge WLS calibration must not be invoked.

The direct raw score path must be used instead.

The implementation must explicitly prevent division by zero in:

```text id="h5n8qb"
d̄ij = ... / ωij
```

when no shared submissions exist.

# 25. WHY WLS USES OVERLAP COUNT

Under the baseline model used by the judging system:

```text id="x8f3rd"
Var(d̄ij) ∝ 1 / ωij
```

Therefore inverse-variance weighting is proportional to:

```text id="0u5y9j"
ωij
```

The overlap count is consequently used as the WLS weight:

```text id="4m9h3q"
weight(i,j) = ωij
```

This gives judge pairs with more shared valid submissions greater influence because their estimated mean difference is expected to contain less sampling noise under the baseline model.

## 25.1 Diagnostics and Assumptions

The implementation must record calibration diagnostics because real judging data may not perfectly satisfy the baseline assumptions.

The system must not represent the WLS model as proof that every judge has a single perfectly stable scoring bias.

The calibration result is a model-based estimate derived from the observed overlap evidence.

Diagnostics therefore remain part of the calibration record.

# 26. CALIBRATION DIAGNOSTICS

Before a calibration result can be accepted, the backend must evaluate and persist diagnostics including:

```text id="3b7j5p"
graph connectivity
overlap counts
judge offsets
residuals
score ranges
near-zero variance conditions
contradictory overlap evidence
unusually large offsets
```

## 26.1 Connectivity

The calibration graph must satisfy the connectivity requirement defined in Section 13.

If a single normalization model is required and the graph is disconnected:

```text id="6q0g4t"
DISCONNECTED_CALIBRATION_GRAPH
```

must prevent successful calibration.

## 26.2 Residuals

For each observed judge-pair relationship, the calibration system should be able to determine the residual between the observed mean difference and the fitted offset difference:

```text id="8d2nqk"
residual_ij =
(bi - bj) - d̄ij
```

Residual diagnostics help the organizer understand how well the additive-offset model explains the observed overlap evidence.

A large residual is a diagnostic signal, not automatic proof of misconduct.

## 26.3 Offset Diagnostics

Large or unusual offsets must be surfaced to organizers as review signals.

The system must not automatically label a judge:

```text id="f8x2vr"
malicious
biased
invalid
```

solely because of:

```text id="c7w4zm"
large offset
high disagreement
unusual score distribution
```

These observations may result from legitimate differences in interpretation, submission mix, rubric understanding, calibration quality, or other factors.

The system should expose the evidence and diagnostics to authorized organizers rather than making a disciplinary judgment automatically.

## 26.4 Near-Zero Variance

The implementation must detect numerical conditions in which the available score/overlap evidence provides little or no useful variation.

Such conditions must not cause silent numerical failure.

The calibration process should return an explicit diagnostic or failure state when the model cannot be solved reliably under the configured numerical rules.

## 26.5 Calibration Record

A successful calibration must be versioned and auditable.

Persist sufficient information to identify:

```text id="r3k0py"
assignment version
rubric version
calibration universe
valid review set/input snapshot
judge offsets
diagnostics
calibration version
creation timestamp
```

The exact input snapshot/hash should be retained according to the system's versioning model.

# 27. APPLYING NORMALIZATION

Once calibration succeeds, the normalized score for judge `j` is:

```text id="6qj9t8"
n(s,j) =
    r(s,j) - bj
```

Do **not** clamp individual normalized reviews.

A normalized review may temporarily fall outside `[0,100]` because the additive calibration offset is applied after the raw score.

The final project score is:

```text id="1j8m0e"
F_raw(s) =

(1/R) Σ_j n(s,j)
```

where the sum contains the required valid calibrated reviews for submission `s`.

Every valid judge contribution receives equal weight in the final average.

## 27.1 Required Review Count

Under normal operation:

```text id="0s5mve"
number_of_valid_calibrated_reviews(s) = R
```

The system must not silently replace a missing review with:

```text id="invalid"
0
```

or with another judge's score.

If a reduced-review exception is explicitly supported, its scoring semantics must be represented explicitly and must not be confused with an ordinary `R`-complete result.

## 27.2 Publication Clamp

The publication score is:

```text id="z7t3v1"
F(s) =
    min(100, max(0, F_raw(s)))
```

The clamp exists for publication/display purposes.

It must not modify the underlying normalized contributions or `F_raw`.

## 27.3 Ranking Uses F_raw

This is a critical invariant:

```text id="5r6c2p"
RANK USING F_raw
NOT publication score F
```

Example:

```text id="7j4m9w"
Project A: F_raw = 108
Project B: F_raw = 101
```

Both publication scores become:

```text id="0v3s7q"
100
```

but Project A still ranks above Project B because:

```text id="3h8k1m"
108 > 101
```

The implementation must therefore retain sufficient internal precision for `F_raw`.

## 27.4 No Premature Rounding

Do not:

```text id="8x5k0v"
round individual criterion values unnecessarily
round raw review scores before calibration
round normalized reviews before aggregation
round F_raw before ranking
```

Use the system's defined numeric precision consistently through the calculation pipeline.

Only format/round values when presenting them to users or exporting them for display.

# 28. TIE BREAKING

Ranking must use the full internal precision of `F_raw`.

For two submissions `A` and `B`:

```text id="r7k2cw"
if F_raw(A) != F_raw(B):
    higher F_raw ranks first
```

Only an exact equality under the system's defined numeric representation invokes the deterministic tie-break.

Displayed decimal rounding must never determine rank.

For example, if:

```text id="q6p3vm"
A = 97.12341
B = 97.12339
```

both display as `97.12`, but they are not tied internally.

## 28.1 Deterministic Tie-Break

If the full-precision ranking values are exactly equal, use a stable canonical submission identifier or deterministic hash.

The tie-break must be reproducible across:

```text id="x2m9vb"
API requests
page reloads
exports
recalculation
server restarts
finalization
```

## 28.2 Forbidden Tie-Breaks

Never use:

```text id="8f0s4z"
database insertion order
completion time
judge ID
request arrival order
runtime iteration order
```

as a hidden ranking tie-break.

These values are not stable ranking semantics.

## 28.3 Ranking Consistency

The same immutable inputs must produce the same ranking.

Conceptually:

```text id="r1w6kp"
assignment version
        +
rubric version
        +
valid reviews
        +
calibration version
        ↓
F_raw
        ↓
deterministic ranking
```

If any upstream versioned input changes, the derived ranking must be treated as stale and regenerated rather than silently mixing values from different judging states.

# 29. TYPE 2 — TRACK PANELS

Type 2 judging must reuse the same judging engine implemented for the global judging model.

Do not create a separate track-specific scoring, assignment, calibration, or ranking engine.

The track becomes the **judging universe/scope** supplied to the existing engine.

For each track:

```text
Track
  ↓
Track submissions
  ↓
Organizer-defined judge panel
  ↓
Assignment
  ↓
Independent reviews
  ↓
Track calibration/normalization
  ↓
Track ranking
```

The same backend services must be reused for:

```text
assignment
review validation
raw scoring
overlap calculation
connectivity validation
calibration
normalization
ranking
finalization
```

Only the judging scope changes.

## 29.1 Track Judging Universe

For a track `T`, the judging universe consists of:

```text
T
+
submissions belonging to T
+
judges assigned/eligible for T
+
T's judging stage
+
T's rubric version
+
T's assignment version
```

All judging operations must resolve their universe from authoritative backend state.

A client must not be able to change the track by supplying a different `track_id` in a request.

## 29.2 Organizer-Defined Panel

The organizer defines which judges belong to each track panel.

The assignment engine consumes that panel as its eligible judge pool.

It must not silently add judges from another track merely because they are globally active.

For example:

```text
Track A
    ↓
Panel A = {Judge 1, Judge 3, Judge 7}

Track B
    ↓
Panel B = {Judge 2, Judge 4, Judge 8}
```

The same judge may belong to multiple panels if the organizer explicitly configures that membership.

Panel membership does not itself create cross-track calibration.

## 29.3 Track Isolation

A Track A judge may access only the judging data authorized for Track A.

At minimum:

```text
Track A submissions
Track A rubric
Track A assignments
Track A own reviews
Track A judging progress
```

A Track A judge must not access:

```text
Track B submissions
Track B assignments
Track B reviews
Track B ballots
Track B scores
Track B rankings
Track B calibration data
Track B judge data
```

This must be enforced server-side.

Hiding Track B information in the frontend is not sufficient.

## 29.4 Authorization Scope

Every judge-facing request must validate the complete scope:

```text
authenticated actor
event
judging stage
track/panel
assignment ownership
submission membership
review ownership
```

For example, a request such as:

```text
GET /submissions/{submission_id}
```

must not return a submission merely because the caller is an authenticated judge.

The backend must verify that the requested submission belongs to the caller's authorized judging universe.

## 29.5 Track Ranking

Each track produces its own derived judging results:

```text
Track
  ↓
valid reviews
  ↓
track calibration
  ↓
track normalized scores
  ↓
track F_raw
  ↓
track ranking
```

The resulting ranking belongs to that track's judging universe.

It must not automatically become part of another track's ranking or calibration.

# 30. NO CROSS-TRACK NORMALIZATION

Cross-track normalization is prohibited for independent track panels.

Do **not** build one global calibration model from:

```text
Track A judges
+
Track B judges
```

when the tracks are independent judging universes.

For example:

```text
Track A judges
    ↕
Track A submissions

Track B judges
    ↕
Track B submissions
```

If there is no shared submission connecting the two panels, their relative scoring offsets are not identifiable from the observed judging data.

Therefore calibration must be scoped as:

```text
Track A
  ↓
Track A overlap graph
  ↓
Track A calibration
```

and independently:

```text
Track B
  ↓
Track B overlap graph
  ↓
Track B calibration
```

Not:

```text
Track A + Track B
        ↓
global calibration
```

## 30.1 Calibration Boundary

The calibration input set must be restricted to reviews within the same judging universe.

For Track A:

```text
valid reviews
    ∩
Track A submissions
    ∩
Track A judge panel
```

For Track B:

```text
valid reviews
    ∩
Track B submissions
    ∩
Track B judge panel
```

A judge appearing in both panels does not by itself merge the calibration universes.

## 30.2 Cross-Track Data Must Not Create Edges

The overlap graph for Track A must be constructed only from Track A submissions.

Likewise, Track B's graph must be constructed only from Track B submissions.

Do not create an overlap edge between:

```text
Judge A from Track A
```

and:

```text
Judge B from Track B
```

because they happen to judge the same person in another context.

The graph represents shared **judging submissions within the current judging universe**, not shared identities or global eligibility.

## 30.3 Failure and Isolation

If a track's own calibration graph is disconnected and that track requires one shared normalization model:

```text
DISCONNECTED_CALIBRATION_GRAPH
```

must apply to that track.

The implementation must not solve the problem by borrowing overlap information from another independent track.

That would violate the judging boundary.

## 30.4 Shared Judge Across Tracks

A judge may be assigned to multiple tracks if explicitly configured.

For example:

```text
Judge 1
 ├── Track A panel
 └── Track B panel
```

This does not imply:

```text
Track A calibration ↔ Track B calibration
```

The same judge can therefore have separate calibration offsets in separate judging universes.

The calibration identity is effectively:

```text
judging_universe + judge
```

rather than merely:

```text
judge
```

This distinction must be reflected in the data model and authorization logic.

# 31. TYPE 2A — TRACK WINNERS ONLY

When the organizer configures Type 2A, each track is independently judged and produces its own winner or winners.

The pipeline is:

```text
Track A
  ↓
Track A judging
  ↓
Track A calibration/normalization
  ↓
Track A ranking
  ↓
Track A winner(s)

Track B
  ↓
Track B judging
  ↓
Track B calibration/normalization
  ↓
Track B ranking
  ↓
Track B winner(s)
```

The same process applies independently to every configured track.

## 31.1 No Cross-Track Final Ranking

There is no automatic global ranking across tracks.

Do not:

```text
take Track A F_raw
+
take Track B F_raw
↓
sort all projects globally
```

unless the organizer explicitly creates a separate overall judging stage as described in Section 32.

Track scores and rankings are scoped to their own judging universe.

## 31.2 Track Finalization

Each track may be finalized independently according to the stage/finalization rules.

The finalized result must retain its:

```text
track
assignment version
rubric version
calibration version
ranking inputs
finalization snapshot
```

This ensures that a finalized Track A result is not silently changed because Track B is recalculated.

# 32. TYPE 2B — TRACK + OVERALL

If the organizer enables an overall stage, it must be a **separate judging stage**.

The pipeline is:

```text
Track judging
  ↓
Track rankings/finalists
  ↓
Organizer-defined finalist set
  ↓
Separate overall judging stage
  ↓
Overall panel
  ↓
Overall assignment
  ↓
Overall reviews
  ↓
Overall calibration/normalization
  ↓
Overall ranking
```

The overall stage is a new judging universe.

## 32.1 Finalist Selection

Track judging produces the information used to determine which submissions enter the overall stage.

The finalist set must be explicitly defined and persisted.

For example:

```text
Track A
  ↓
Track A ranking
  ↓
Finalists A

Track B
  ↓
Track B ranking
  ↓
Finalists B

Finalist A ∪ Finalist B
  ↓
Overall stage
```

The exact finalist-selection rule must be organizer/configuration-defined.

Do not silently infer a finalist set from arbitrary score thresholds.

## 32.2 Independent Overall Assignment

The overall stage receives its own:

```text
submission pool
judge panel
R
rubric
assignment version
```

The overall assignment must be generated and validated using the same assignment engine.

It must not reuse the track assignment merely because the same submissions or judges are involved.

## 32.3 Independent Overall Rubric

The overall stage may use its own rubric version.

Do not assume that:

```text
Track rubric = Overall rubric
```

unless the organizer explicitly configures the same rubric/version for both stages.

Every overall review must reference the exact rubric version used by the overall stage.

## 32.4 Independent Overall Calibration

The overall stage gets its own overlap graph and calibration.

For example:

```text
Track calibration
       ↓
Track F_raw

Overall calibration
       ↓
Overall F_raw
```

The overall calibration must be computed from valid overall-stage reviews.

Track-stage reviews must not automatically enter the overall WLS model.

## 32.5 No Silent Score Combination

Do not calculate an overall result as:

```text
track score + overall score
```

or:

```text
weighted(track score, overall score)
```

unless the product specification explicitly defines such a mathematical model.

The baseline Type 2B design is:

```text
track judging
    ↓
select finalists
    ↓
overall judging
    ↓
overall ranking
```

The overall ranking is therefore produced by the overall stage's own judging inputs.

## 32.6 Stage Isolation

Track and overall stages must remain distinguishable in the data model and audit trail.

A review must identify its judging stage.

An assignment must identify its judging stage/version.

A rubric must identify its stage/version.

A calibration result must identify its stage/universe/version.

This prevents a Track A review from accidentally being interpreted as an Overall review.

## 32.7 Overall Authorization

An overall-stage judge may access only the submissions and judging information authorized for the overall stage.

A judge who participated in Track A does not automatically receive access to Track A's historical private data while judging the overall stage.

Likewise, participating in the overall stage does not grant access to another judge's overall reviews.

All stage and track authorization checks remain server-side.

# 33. PROGRESS DASHBOARD

The organizer dashboard must provide live judging progress for the selected judging stage.

Progress must be derived from the authoritative assignment and review state.

At minimum, provide:

```text id="z6x0dq"
Total judges
Started judges
Not started judges
Completed judges
Total assignments
Completed reviews
In-progress reviews
Remaining reviews
```

Example:

```text id="q1h5yb"
JUDGING PROGRESS

Judges

12 total
9 started
3 not started

Reviews

180 assigned
121 completed
14 in progress
45 remaining
```

## 33.1 Judge-Level Progress

Provide a judge-level progress table:

```text id="k8m1pa"
Judge
Assigned
Completed
Remaining
Status
```

Supported statuses:

```text id="6g7v2n"
NOT_STARTED
IN_PROGRESS
COMPLETE
REMOVED
```

The status must be derived from the judge's current assignment/review state.

## 33.2 Progress Definitions

For an active judge:

```text id="6p8j5w"
Assigned =
number of active assignments belonging to the judge
```

```text id="4z9m2a"
Completed =
number of valid final reviews submitted for those assignments
```

```text id="j0q6vs"
Remaining =
Assigned - Completed
```

An assignment with a saved draft but no final submitted review contributes to:

```text id="7w2k1m"
In Progress
```

rather than `Completed`.

A judge with no completed or in-progress review activity is:

```text id="9v6r3t"
NOT_STARTED
```

A judge with at least one started review but outstanding work is:

```text id="0q4k7p"
IN_PROGRESS
```

A judge with all required active assignments completed is:

```text id="f3n8cx"
COMPLETE
```

A removed judge is:

```text id="d5m1zs"
REMOVED
```

and must remain distinguishable from a judge who has simply not started.

## 33.3 Stage Scope

Progress must be scoped to the selected judging stage.

For Type 2:

```text id="0a8g5m"
Track A progress
```

must not silently include:

```text id="k3p7ds"
Track B assignments
Overall-stage assignments
```

Likewise, overall-stage progress must be calculated from the overall stage's own assignments and reviews.

## 33.4 Derived State

Do not maintain fragile counters as the only source of truth.

For example, do not rely exclusively on:

```text id="v8h2qt"
completed_reviews_count
remaining_reviews_count
```

stored as mutable counters.

The authoritative state is:

```text id="0d5g6k"
assignments
+
review records
+
review status
+
assignment/stage/version scope
```

Dashboard counters may be cached or materialized for performance, but they must be reproducible from the underlying state.

If a cached counter disagrees with authoritative state, authoritative state wins.

## 33.5 Authorization

Progress dashboards are organizer/admin functionality.

A judge must not receive organizer-wide progress merely by calling the underlying progress API.

Judge-facing progress remains limited to their own judging scope as defined in Section 23.

# 34. EXPORT

CSV export must be available at every applicable judging stage.

Exports must be generated from authoritative versioned judging data.

At minimum, implement the following export types.

## 34.1 Assignment Export

The assignment export must contain:

```text id="p7m2vn"
stage
track
submission_id
judge_id
assignment_status
assigned_at
completed_at
```

The export must reflect the assignment version/scope selected by the authorized organizer.

## 34.2 Raw Review Export

The raw review export must contain:

```text id="a9c4kw"
stage
submission_id
judge_id
criterion
criterion_score
raw_review_score
submitted_at
```

Each row must correspond to the authoritative submitted review and rubric criterion.

Draft reviews must not be exported as final raw reviews.

## 34.3 Normalization Export

The normalization export must contain:

```text id="m1x7zr"
stage
submission_id
judge_id
raw_score
judge_offset
normalized_score
```

Where applicable, the export should identify the calibration version used to produce the normalized score.

## 34.4 Final Results Export

The final results export must contain:

```text id="h8q5yc"
rank
submission_id
final_raw_score
publication_score
```

The `rank` must be generated using the same deterministic ranking logic used by the application.

Do not independently sort or rank CSV rows using rounded publication scores.

## 34.5 Diagnostics

Organizer/admin exports may additionally include authorized calibration diagnostics such as:

```text id="2m6pqs"
judge offsets
pair overlap counts
residuals
connectivity status
calibration version
```

These fields must not be exposed through judge-facing exports.

## 34.6 Data Isolation

Export authorization must be enforced server-side.

A judge must never be able to request:

```text id="r9c2fd"
another judge's raw reviews
another judge's comments
aggregate calibration data
another track's assignments
another track's rankings
organizer-only diagnostics
```

by changing an export parameter.

The backend must resolve the export scope from the authenticated actor and requested stage/version.

## 34.7 Deterministic CSV

CSV output must be valid and deterministic.

For identical:

```text id="f7k2vb"
data
stage
version
export configuration
```

the logical row ordering must be stable.

Do not rely on unspecified database row order.

Use explicit deterministic ordering, such as canonical identifiers and the defined ranking order.

CSV fields containing commas, quotes, or newlines must be correctly escaped according to CSV rules.

The export must represent the exact selected judging state rather than a mixture of current and historical versions.

# 35. FINALIZATION

Finalization creates the immutable published result for a judging stage.

A stage must not be finalized until all required validation succeeds.

Before finalization, validate:

```text id="c3m7qa"
assignment / comparison configuration
reviews / comparisons
method-specific configuration
calibration when applicable
ranking
exceptions
version consistency
```

For stages using cross-judge normalization, calibration must be successfully completed.

For Pairwise stages, the Pairwise/Bradley–Terry calculation must be successfully completed.

For `R = 1`, the direct-score path must be validated instead of requiring a nonexistent cross-judge calibration.

## 35.1 Version Consistency

The final result must be internally coherent.

The finalization process must ensure that:

```text id="w5k9pd"
assignment version
        ↓
valid review set
        ↓
rubric version
        ↓
calibration version
        ↓
normalized scores
        ↓
F_raw
        ↓
publication scores
        ↓
ranking
```

all refer to the intended judging state.

Do not finalize a ranking that combines reviews from one assignment version with calibration derived from another incompatible version.

## 35.2 Finalization Snapshot

The immutable snapshot must persist at minimum:

```text id="n4x8sb"
competition configuration
algorithm versions
assignment version
deterministic seed/configuration
judge pool/input scope
submission pool/input scope
rubric version
valid review IDs
raw scores
calibration version
calibration offsets
normalized scores
final raw scores
publication scores
ranking/tie-break results
exceptions
corrections/invalidation history
finalization timestamp
finalizing actor
```

Where appropriate, persist deterministic hashes of the relevant inputs/results so the final snapshot can be independently identified.

## 35.3 Immutable State

After successful finalization:

```text id="t8v2lm"
results immutable
assignment immutable
calibration immutable
ranking immutable
rubric reference immutable
finalization snapshot immutable
```

The normal organizer UI must not provide an in-place edit path for finalized results.

## 35.4 Corrections After Finalization

If a correction is required after finalization, do not silently mutate the historical snapshot.

Instead:

```text id="k6y1rz"
historical finalization
        ↓
correction/invalidation event
        ↓
new version
        ↓
recalculation
        ↓
new finalization
```

The historical result must remain auditable.

A correction that changes a raw review, calibration input, assignment, rubric, or ranking must invalidate the downstream derived state affected by that change.

## 35.5 Finalization Concurrency

Finalization must be protected against concurrent state changes.

The system must not allow:

```text id="v1q7na"
review submission
```

or another state-changing operation to race with finalization and produce an internally inconsistent snapshot.

The finalization transaction/process must establish a coherent input boundary before writing the immutable snapshot.

# 36. AUDIT TRAIL

The judging system must maintain an append-only audit trail for important state changes and judging operations.

At minimum, record:

```text id="q8f4cz"
judge invited
judge accepted
judge removed
panel changed
rubric created
rubric version changed
assignment generated
assignment manually changed
assignment repaired
review started
review submitted
review corrected
review invalidated
calibration executed
calibration failed
calibration finalized
result finalized
export generated
```

## 36.1 Audit Event Contents

Each audit event should identify, where applicable:

```text id="m7z2kp"
event type
event timestamp
actor
event/competition
judging stage
track/panel
affected entity
entity/version identifier
previous state/reference
new state/reference
reason or metadata
```

Do not store only a human-readable message when structured identifiers are available.

The audit record should allow an organizer to reconstruct what happened and which versioned object was affected.

## 36.2 Append-Only

Audit records must not be silently edited or deleted as part of normal application operation.

If an audit correction is ever required, create a new audit event describing the correction rather than overwriting the historical event.

## 36.3 Assignment Audit

Assignment events must preserve the distinction between:

```text id="z3k6qa"
generated assignment
manual modification
repair
replacement after dropout
new assignment version
```

This makes it possible to determine how the active assignment was produced.

## 36.4 Review Audit

Review auditing must distinguish:

```text id="p6m1xv"
draft started
draft saved
final submitted
corrected
invalidated
```

A correction or invalidation must retain the historical relationship to the original review/version.

## 36.5 Calibration Audit

Calibration events should identify:

```text id="b4w8ns"
assignment version
rubric version
calibration version
calibration status
input/review scope
```

A failed calibration must also be recorded.

For example:

```text id="j2r7mc"
calibration failed
reason = DISCONNECTED_CALIBRATION_GRAPH
```

This prevents a failed calibration from appearing as though calibration simply never ran.

## 36.6 Finalization Audit

Finalization must create an audit event referencing the immutable finalization snapshot.

The event should identify:

```text id="u9c5kd"
finalization version/snapshot
stage
track where applicable
assignment version
rubric version
calibration version where applicable
finalizing actor
timestamp
```

## 36.7 Export Audit

Every organizer/admin export should generate an audit event containing enough information to identify:

```text id="s4n8vb"
who generated the export
which stage
which track/scope
which version
export type
when it was generated
```

Do not record the exported private judging data itself in the audit log.

## 36.8 Audit Access

Audit information is organizer/admin-only by default.

Judges must not be able to read the audit log unless explicitly authorized by the existing permission model.

Participants must never receive internal judging audit information.

Authorization must be enforced server-side; hiding an audit page in the frontend is not sufficient.

# 37. API DESIGN

Follow the existing project's API conventions, routing structure, naming, authentication middleware, validation patterns, and error-response format.

The judging system must expose equivalent functionality for:

```text
Judge management
Judge invitations / activation
Judging stages
Judge panels / track membership
Assignments
Assignment generation
Assignment validation
Rubric configuration and versions
Review creation / draft / submission
Judging progress
Calibration / normalization
Results
Exports
Finalization
Audit history
```

Conceptually, the API may contain functionality equivalent to:

```text
/judges
/judges/:id

/judging/stages
/judging/stages/:id
/judging/stages/:id/panel
/judging/stages/:id/assignments
/judging/stages/:id/assignments/generate
/judging/stages/:id/assignments/validate
/judging/stages/:id/rubric
/judging/assignments/:id
/judging/assignments/:id/review
/judging/stages/:id/progress
/judging/stages/:id/calibration
/judging/stages/:id/results
/judging/stages/:id/export
/judging/stages/:id/finalize
```

These paths are conceptual, not mandatory.

If the existing application uses a different API architecture, integrate with that architecture instead of introducing a parallel routing system.

Every judging endpoint must:

1. authenticate the current actor;
2. determine the actor's role;
3. verify the relevant event/stage;
4. verify track/panel scope where applicable;
5. verify assignment ownership where applicable;
6. verify the requested resource belongs to the permitted judging universe;
7. apply the required role permission;
8. return only data the actor is authorized to see.

Do not create a second authentication or authorization architecture for Tier 2.

---

# 38. FRONTEND REQUIREMENTS

Build the minimum complete interfaces required to operate Tier 2.

## Organizer

The organizer interface must provide the necessary workflows for:

```text
Judges
Judge invitations
Judge activation/status
Judge panels
Judging stages
Rubric configuration
Rubric versions
Assignments
Manual assignment
Batch assignment
Algorithmic assignment
Assignment validation
Assignment repair
Judging progress
Calibration
Calibration diagnostics
Results
Exports
Finalization
Audit history
```

The organizer must be able to understand the current state of the judging stage without inspecting the database directly.

Progress must be derived from actual assignment/review state rather than fragile frontend counters.

## Judge

The judge interface must provide:

```text
My assignments
Submission review
Rubric
Draft review
Submit review
My progress
My submitted scores
```

The judge must never receive peer-review data merely because it is hidden by the frontend.

Do not send unauthorized data to the browser and rely on JavaScript to hide it.

Do not expose:

```text
peer scores
peer reviews
other judges' assignments
unauthorized track submissions
aggregate results
normalization diagnostics
organizer-only audit information
```

unless the existing role model explicitly permits that actor to access the information.

---

# 39. AUTHORIZATION AND SECURITY TESTING

Authorization is a backend requirement, not a frontend feature.

Treat every client-provided ID as untrusted.

Never perform:

```text
findReview(reviewId)
```

and return the record merely because it exists.

Every request must establish:

```text
current actor
→ role
→ event
→ stage
→ panel/track scope
→ assignment ownership
→ resource ownership
→ permission
```

as applicable to that operation.

## Required authorization tests

Explicitly test direct API attempts including:

```text
Judge A requests Judge B's review
Judge A requests Judge B's score
Judge A changes review_id
Judge A changes assignment_id
Judge A requests an unassigned submission
Judge A submits a review for another judge
Judge A requests a Track B submission
Judge A requests a Track B score
Judge A requests Track B ranking/results
Judge A requests aggregate results
Judge A requests audit information
Judge A attempts to modify an assignment
Judge A attempts to modify a rubric
Judge A attempts to run calibration
Judge A attempts to access another judging stage
```

All unauthorized requests must fail at the backend.

A response such as:

```text
HTTP 200
+ filtered frontend fields
```

does not constitute isolation.

The tests must verify both:

1. the operation is denied; and
2. unauthorized data is not leaked in the response.

Organizer/admin operations must also be tested against the existing event/role model rather than assuming that possession of an ID grants access.

---

# 40. MATHEMATICAL AND ASSIGNMENT INVARIANT TESTING

Implement unit/property tests for the mathematical functions independently from HTTP and UI code.

At minimum test:

```text
weighted rubric score
review score bounds
required review count
judge distinctness
exact assignment count
quota balancing
overlap invariant
PairLoss
ΔPairLoss
connectivity
assignment validation
assignment repair
WLS offsets
zero-sum constraint
normalized score
F_raw
publication clamp
ranking
deterministic tie-break
```

For an assignment with:

```text
K submissions
J eligible judges
R required reviews/submission
```

verify:

```text
total assignments = K × R
```

and:

```text
Σ Oij = K × C(R,2)
```

where `Oij` is the number of submissions jointly reviewed by judges `i` and `j`.

When exact workload balancing is feasible for the generated assignment:

```text
max(load) - min(load) <= 1
```

Every submission must have exactly `R` distinct assigned judges.

The tests must also verify that assignment generation does not silently:

```text
reduce R
remove a judge
duplicate a judge
ignore an eligible submission
break track isolation
accept an invalid judge
```

All mathematical expectations must follow `JUDGING.md`. Do not create a competing mathematical definition inside the test suite.

---

# 41. DETERMINISM TESTING

Assignment generation must be deterministic.

Run the same assignment operation twice using identical:

```text
eligible judges
eligible submissions
stage configuration
track/panel configuration
R
algorithm version
assignment seed/configuration
canonical judge IDs
canonical submission IDs
```

Expected result:

```text
assignment_1 == assignment_2
```

The implementation must not depend on:

```text
database insertion order
request completion order
unordered collection iteration
process timing
randomness without a persisted deterministic seed
```

Canonical IDs must be used for deterministic tie-breaking.

If a deterministic seed is part of the assignment version, persist the seed/configuration required to reproduce the assignment.

---

# 42. NORMALIZATION AND CALIBRATION TESTING

Create synthetic judge data with known relative offsets.

Example:

```text
Judge A = baseline
Judge B = baseline + 10
Judge C = baseline - 5
```

Create overlapping submissions so the calibration graph contains the required connections.

Verify the WLS calibration defined in `JUDGING.md` estimates relative judge offsets within an appropriate numerical tolerance.

Then verify the complete transformation:

```text
raw rubric score
    ↓
raw review score
    ↓
calibration offset
    ↓
normalized review score
    ↓
F_raw
    ↓
publication clamp
```

Raw review scores must remain unchanged.

The test must verify the zero-sum calibration constraint:

```text
Σ bj = 0
```

where applicable.

The test must also verify that the normalization graph is built only from the relevant judging universe.

A disconnected graph must produce the documented calibration failure rather than arbitrary component offsets.

---

# 43. TRACK ISOLATION TESTING

Track judging must be tested as an independent judging universe.

Create:

```text
Track A
Track B

Judge A ∈ Track A
Judge B ∈ Track B
```

Attempt:

```text
Judge A → Track B submission
Judge A → Track B score
Judge A → Track B ranking
```

All unauthorized operations must fail.

Then verify:

```text
Track A calibration
```

does not use:

```text
Track B judges
Track B reviews
Track B overlap edges
Track B calibration offsets
```

If the same judge belongs to multiple tracks, membership in one track must not merge the calibration universes.

Each track must maintain its own:

```text
assignment universe
review set
overlap graph
calibration
normalized scores
ranking
```

An overall judging stage, when configured, must be tested separately according to its own stage/panel/assignment/rubric/calibration configuration.

---

# 44. FINALIZATION AND IMMUTABILITY TESTING

After:

```text
stage.finalize()
```

the finalized judging snapshot must be immutable.

Test that the finalized snapshot cannot be silently changed through:

```text
assignment modification
review modification
rubric modification
calibration modification
ranking modification
final-score modification
```

Finalization must capture the exact versions and inputs used to produce the result.

At minimum the final snapshot must identify:

```text
judging stage
assignment version
rubric version
valid review IDs
raw scores
calibration version
calibration offsets
normalized scores
F_raw
publication scores
ranking/tie-break information
exceptions
corrections/invalidation history
algorithm version
relevant configuration/seed
snapshot timestamp
finalizing actor
```

If a correction is permitted after finalization, it must not mutate the existing finalized snapshot.

Instead:

```text
correction
    ↓
new version/audit record
    ↓
invalidate affected downstream artifacts
    ↓
recalculate affected outputs
    ↓
new finalization
```

Finalization must be atomic with respect to concurrent review submission and other state-changing operations.

---

# 45. ERROR HANDLING AND FAILURE-CLOSED BEHAVIOR

Use explicit domain errors.

Examples include:

```text
INSUFFICIENT_JUDGES
INSUFFICIENT_PANEL_CAPACITY
DUPLICATE_ASSIGNMENT
INVALID_JUDGE
INVALID_TRACK_SCOPE
ASSIGNMENT_CONSTRAINT_FAILURE
FAILED_ASSIGNMENT_REPAIR
DISCONNECTED_CALIBRATION_GRAPH
INSUFFICIENT_OVERLAP
INVALID_RUBRIC
INVALID_RUBRIC_WEIGHTS
REVIEW_NOT_ASSIGNED
REVIEW_ALREADY_SUBMITTED
STAGE_NOT_OPEN
STAGE_ALREADY_FINALIZED
UNAUTHORIZED_JUDGING_ACCESS
FAILED_REPLACEMENT_CAPACITY
```

Use the project's existing error conventions where they exist.

Do not swallow domain errors and return plausible-looking results.

## Failure-closed rule

If a required invariant cannot be satisfied:

```text
DO NOT FABRICATE A RESULT.
```

Examples:

```text
cannot create valid assignment
→ fail assignment

assignment repair cannot satisfy required invariants
→ fail repair

calibration graph is disconnected
→ fail calibration

required valid reviews are missing
→ do not finalize

unauthorized request
→ deny

invalid rubric
→ reject configuration

replacement judge cannot satisfy required capacity
→ fail replacement
```

Never silently:

```text
reduce R
remove a judge
ignore an assignment
fabricate an offset
replace normalization with raw averaging
publish incomplete results
```

The system must prefer an explicit, diagnosable failure over a statistically invalid result that appears complete.

---

# 46. DATABASE INTEGRITY, TRANSACTIONS, AND CONCURRENCY

Where appropriate, enforce important invariants at the database layer.

For assignments, enforce uniqueness equivalent to:

```text
unique(stage_id, submission_id, judge_id)
```

Use foreign keys for relationships that must remain valid.

Use transactions for operations such as:

```text
review submission
assignment generation
assignment repair
calibration snapshot
finalization
```

## Concurrency

Handle simultaneous requests explicitly.

Test cases include:

```text
judge submits the same review twice
organizer finalizes while a review is being submitted
two organizers modify assignments simultaneously
two assignment-generation jobs run simultaneously
two replacement operations affect the same judge
```

Use the project's appropriate combination of:

```text
transactions
row/version locking
optimistic concurrency
idempotency
unique constraints
state checks
```

Two assignment-generation operations must not silently overwrite one another.

A review must not become valid twice because of concurrent requests.

Finalization must establish an atomic boundary after which the finalized snapshot cannot be silently changed.

---

# 47. PERFORMANCE AND SCALABILITY

Do not implement brute-force assignment by enumerating all judge combinations.

The assignment algorithm must scale reasonably for the official DOGFOOD fixture and larger inputs.

The fixture may contain approximately:

```text
40 projects
30 judges
8 tracks
```

and may intentionally contain difficult cases such as:

```text
incomplete batches
duplicate entries
low-variance reviewers
```

These are test conditions, not hard-coded system limits.

Prefer practical complexity around:

```text
O(submissions × judges × R)
```

or a similarly practical complexity justified by the implementation.

Do not enumerate:

```text
C(J,R)
```

judge combinations.

The implementation must remain configuration-driven with respect to:

```text
number of submissions
number of judges
number of tracks
R
panel membership
rubric criteria
```

Do not hard-code the fixture dimensions.

---

# 48. DOCUMENTATION REQUIREMENTS

Documentation must describe the actual implementation.

Update or create:

```text
README.md
ARCHITECTURE.md
DATA-MODEL.md
JUDGING.md
```

Do not casually overwrite an existing `JUDGING.md`.

If the existing `JUDGING.md` contains the agreed judging mathematics, preserve it and implement that specification.

## README.md

Explain:

```text
how to run the system locally
how to initialize the database
how to load fixture/seed data
how to operate the judging workflow
how to run tests
```

## ARCHITECTURE.md

Document the actual:

```text
frontend
backend
database
authentication
authorization
judging domain
assignment engine
scoring engine
normalization engine
export system
audit system
```

Explain the actual integration with Tier 1.

Do not generate generic architecture prose that does not describe the implementation.

## DATA-MODEL.md

Document:

```text
entities
relationships
important constraints
assignment data
review data
rubric versions
calibration data
finalization snapshots
audit records
import/export paths
```

The documentation must match the actual database schema.

## JUDGING.md

Document or preserve the authoritative:

```text
assignment algorithm
workload balancing
overlap
connectivity
repair
raw scoring
WLS normalization
track isolation
overall judging
failure behavior
finalization
auditability
```

Do not maintain a second conflicting mathematical specification elsewhere.

---

# 49. ACCEPTANCE TESTING

Before declaring Tier 2 complete, run the application from a clean environment.

Use the project's documented startup process, including:

```bash
docker compose up
```

when Docker Compose is the existing project startup mechanism.

Verify:

```text
application starts
database initializes
seed/fixture data loads
authentication works
Tier 1 still works

judge invitations work
judge activation works
stage count is organizer-configurable
stage order/dependencies are persisted
organizer can select the judging method per stage
rubric-stage configuration is validated when selected
Pairwise-stage configuration is validated when selected
manual assignment works
batch assignment works
algorithmic assignment works
assignment validation works
assignment repair works

rubric configuration works
rubric versioning works

judge review works
review drafts work for methods that support drafts
review submission works
Pairwise comparisons work when a stage selects Pairwise Mode
method-specific completion and calculation work

backend authorization works
track isolation works

organizer progress works
calibration works when required by the selected stage method
normalization works when required by the selected stage method
Pairwise/Bradley–Terry calculation works when selected
results work
CSV exports work

finalization works
audit records are created
```

Also run:

```text
official acceptance suite, if available
project test suite
mathematical/property tests
authorization tests
concurrency tests
determinism tests
```

Do not claim Tier 2 completion while core acceptance tests are failing.

---

# 50. IMPLEMENTATION ORDER

Follow this dependency order unless the existing architecture requires a small variation.

```text
PHASE 1
Inspect repository
    ↓
Inspect Tier 1 architecture
    ↓
Confirm existing T1 contracts
    ↓
Identify reusable entities
    ↓
Identify schema/API extension points

PHASE 2
Judging database model
    ↓
Migrations
    ↓
Seed/fixture support

PHASE 3
Judge management
    ↓
Judge invitation
    ↓
Judge activation/status
    ↓
Panel membership

PHASE 4
Judging stages
    ↓
Stage state machine

PHASE 5
Rubric management
    ↓
Rubric versioning
    ↓
Criterion/weight validation

PHASE 6
Assignment domain
    ↓
Manual assignment
    ↓
Batch assignment
    ↓
Algorithmic assignment
    ↓
Validation
    ↓
Repair
    ↓
Connectivity

PHASE 7
Judge review workflow
    ↓
Draft
    ↓
Submit
    ↓
Immutable valid review

PHASE 8
Backend authorization
    ↓
Ownership checks
    ↓
Track checks
    ↓
Organizer/admin permissions
    ↓
Security tests

PHASE 9
Raw scoring
    ↓
Weighted rubric calculation

PHASE 10
Normalization
    ↓
Overlap graph
    ↓
WLS calibration
    ↓
Diagnostics
    ↓
Normalized scores

PHASE 11
Ranking
    ↓
F_raw
    ↓
Publication clamp
    ↓
Deterministic tie-break

PHASE 12
Organizer dashboard
    ↓
Judge progress
    ↓
Assignment progress
    ↓
Calibration status

PHASE 13
CSV exports

PHASE 14
Track judging
    ↓
Track-specific panels
    ↓
Track-specific calibration

PHASE 15
Overall judging stage, if configured

PHASE 16
Dropout/reassignment

PHASE 17
Finalization
    ↓
Immutable snapshot
    ↓
Audit

PHASE 18
Full acceptance testing
    ↓
Documentation verification
    ↓
Final cleanup
```

After each major phase:

```text
run relevant tests
check migrations
check authorization
check mathematical invariants where applicable
check Tier 1 behavior
```

Do not knowingly continue while introducing regressions.

---

# 51. CODING STYLE AND ARCHITECTURE DISCIPLINE

Follow the existing project's conventions.

Prefer:

```text
small domain services
explicit validation
typed interfaces
transactions
pure mathematical functions
unit-testable algorithms
```

Keep mathematical calculations independent from HTTP/UI code.

Conceptually, functions should be independently testable, such as:

```text
calculateRawReviewScore()
calculatePairOverlap()
calculatePairLoss()
buildAssignment()
validateAssignment()
repairAssignment()
buildCalibrationGraph()
solveJudgeOffsets()
normalizeReviews()
calculateFinalScore()
rankSubmissions()
```

Use the project's existing naming and module conventions rather than blindly creating these exact functions.

Do not introduce unnecessary infrastructure.

The DOGFOOD platform must remain self-hostable and locally operable.

Do not add:

```text
Kubernetes
microservice fleet
Kafka
cloud queues
external analytics
managed databases
third-party auth
```

unless the existing project already requires them.

A clean modular monolith is acceptable.

The goal is:

```text
correctness
maintainability
self-hostability
adoptability
```

not infrastructure complexity.

---

# 52. DEFINITION OF DONE AND FINAL AGENT PROTOCOL

Tier 2 is complete only when the implementation satisfies the following checklist:

```text
[ ] Judges can be invited
[ ] Judges can become active
[ ] Judges can be assigned manually
[ ] Judges can be assigned in batches
[ ] Judges can be assigned algorithmically
[ ] Assignments are deterministic
[ ] Assignments balance workload when feasible
[ ] Assignments deliberately create overlap
[ ] Assignment graph satisfies required connectivity
[ ] Assignment repair works
[ ] Judge dropout/reassignment is handled
[ ] Weighted rubric works
[ ] Rubric is versioned
[ ] Judges can independently score
[ ] Judges can save drafts
[ ] Judges can submit reviews
[ ] Raw reviews remain preserved
[ ] Backend role isolation is enforced
[ ] Judge cannot access peer scores
[ ] Judge cannot access unauthorized tracks
[ ] Judge cannot bypass isolation through API
[ ] Organizer sees judging progress
[ ] Cross-judge normalization follows JUDGING.md
[ ] Calibration diagnostics are recorded
[ ] Track panels normalize independently
[ ] Overall stage is separate when configured
[ ] F_raw is calculated correctly
[ ] Publication clamp occurs only after F_raw
[ ] Ranking uses full-precision F_raw
[ ] Tie-break is deterministic
[ ] CSV export works
[ ] Finalization creates immutable snapshot
[ ] Audit trail exists
[ ] Tier 1 still works
[ ] Local startup works
[ ] Offline operation works
[ ] Tests pass
[ ] Acceptance suite passes
[ ] Documentation matches implementation
```

## Final instruction to the coding agent

Do not start by generating a large amount of code.

First:

```text
1. Inspect the repository.
2. Inspect the existing Tier 1 architecture.
3. Inspect JUDGING.md completely.
4. Inspect the supplied fixture/seed data.
5. Identify existing entities that can be reused.
6. Identify existing authentication and authorization mechanisms.
7. Identify schema/API conflicts.
8. Identify the smallest set of required Tier 2 extensions.
9. Produce a short implementation assessment.
10. Then implement incrementally.
```

Do not create duplicate:

```text
user systems
event systems
submission systems
authentication systems
authorization systems
track systems
```

when equivalent Tier 1 functionality already exists.

At the end, report exactly:

```text
IMPLEMENTED
----------------
what was actually built

TESTED
----------------
what was actually tested and passed

NOT IMPLEMENTED
----------------
anything intentionally left out

KNOWN LIMITATIONS
----------------
real limitations only

FILES CHANGED
----------------
important files

DATABASE CHANGES
----------------
migrations/schema changes

ACCEPTANCE STATUS
----------------
Tier 2 requirement → PASS/FAIL
```

Be honest.

Do not claim functionality merely because code exists.

Do not claim an acceptance test passed unless it was actually run and passed.

The acceptance criterion is:

> **The judging system actually works, survives direct API access attempts, implements the mathematics in `JUDGING.md`, runs locally from one command, preserves Tier 1 behavior, and can be operated by an organizer.**

A smaller correct implementation is preferable to a larger implementation with broken invariants.
