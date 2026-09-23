# DOGFOOD HACKATHON

# TIER 2 — JUDGING

## Coding Agent Implementation Plan

You are the senior implementation agent responsible for implementing **Tier 2 — Judging** of the DOGFOOD hackathon platform.

Your job is to build a **real, production-shaped judging system**, not a mock/demo.

The implementation must integrate cleanly with the existing Tier 1 system and must satisfy:

1. The DOGFOOD hackathon specification at `https://dogfoodhack.com/`
2. The existing application architecture
3. The existing Tier 1 functionality
4. The project's `JUDGING.md`
5. The project's acceptance/test requirements

---

# 0. SOURCE OF TRUTH — READ BEFORE CODING

Before changing code, inspect the repository thoroughly.

Read:

```text
README.md
ARCHITECTURE.md
DATA-MODEL.md
JUDGING.md
docker-compose.yml
existing source code
existing database schema/migrations
existing tests
existing seed/fixture system
```

If available, also inspect the DOGFOOD fixture/acceptance configuration.

## Priority of sources

Use this hierarchy:

```text
1. Existing repository architecture/contracts
2. JUDGING.md for judging mathematics and judging behavior
3. DOGFOOD website/specification for Tier requirements
4. Existing Tier 1 behavior
5. Normal engineering judgment
```

Do not silently invent a different judging algorithm.

Do not replace the mathematics in `JUDGING.md` with a simpler implementation such as:

```text
average(scores)
```

unless the source specification explicitly permits it.

The current `JUDGING.md` is the source of truth for:

* assignment
* overlap
* workload balancing
* connectivity
* repair
* scoring
* normalization
* ranking
* dropout handling
* track isolation
* overall judging
* finalization
* auditability

---

# 1. PRODUCT GOAL

Tier 2 adds the complete judging system.

The system must support:

```text
Judge Management
    ↓
Judge Assignment
    ↓
Rubric Configuration
    ↓
Independent Judge Reviews
    ↓
Raw Weighted Scores
    ↓
Cross-Judge Normalization
    ↓
Final Ranking
    ↓
Organizer Progress
    ↓
CSV Export
    ↓
Immutable Finalization
```

The platform must support both judging scopes defined by `JUDGING.md`:

```text
TYPE 1 — NO TRACKS
    One global judge pool
    One global judging scope

TYPE 2 — TRACK PANELS
    One independent panel per track
    Independent track judging
    Optional separate overall judging stage
```

Do not create two unrelated judging implementations.

Build one reusable judging engine whose scope/configuration determines whether it operates globally or inside a track panel.

---

# 2. CRITICAL DOGFOOD CONSTRAINTS

The implementation must preserve the DOGFOOD platform requirements.

## 2.1 Self-hosted

The finished application must continue to run locally.

The expected command is:

```bash
docker compose up
```

It must produce a working portal on localhost.

Do not introduce:

* hosted database dependencies
* hosted authentication
* cloud-only services
* required external APIs
* required SaaS judging services
* required external scoring services

The application must remain usable with the network unavailable after dependencies have been installed.

---

# 2.2 Do not break Tier 1

Tier 2 is an extension of the existing platform.

Do not rewrite or destabilize:

* authentication
* sessions
* roles
* event creation
* tracks
* team formation
* submissions
* deadline enforcement
* public gallery

unless a change is strictly necessary for Tier 2 integration.

If a schema/API needs to change, migrate it properly.

Never destroy existing data merely to simplify implementation.

---

# 2.3 Backend authorization is mandatory

This is one of the highest-priority requirements.

Frontend hiding is NOT authorization.

The following must be impossible through direct API requests:

```text
Judge A → Judge B's scores
Judge A → Judge B's reviews
Judge A → another judge's assignments
Track A Judge → Track B submissions
Track A Judge → Track B scores
Judge → aggregate judging results
Judge → normalization internals
Judge → organizer-only audit information
```

Authorization must be enforced server-side at the service/API/data-access boundary.

Assume an attacker can manually construct every HTTP request.

If changing an ID in the URL or request body exposes another judge's data, the implementation is incorrect.

---

# 3. ROLE MODEL

Preserve the existing roles and integrate judging permissions.

At minimum:

```text
VISITOR
PARTICIPANT
JUDGE
ORGANIZER
ADMIN
```

Expected judging access:

| Actor       |  Own Scores | Peer Scores | Other Track | Aggregate | Audit |
| ----------- | ----------: | ----------: | ----------: | --------: | ----: |
| Visitor     |          No |          No |          No |        No |    No |
| Participant |          No |          No |          No |        No |    No |
| Judge       |         Yes |          No |          No |        No |    No |
| Organizer   | Yes/allowed |         Yes |         Yes |       Yes |   Yes |
| Admin       |         Yes |         Yes |         Yes |       Yes |   Yes |

The exact permission mechanism should follow the existing architecture.

Do not create a parallel authorization system.

---

# 4. DATA MODEL

Extend the existing schema cleanly.

Do not duplicate concepts that already exist.

Use existing:

* users
* events
* submissions
* tracks
* roles

where appropriate.

Add the judging domain.

---

## 4.1 Judges

Create/use a judge entity associated with the existing user.

Conceptually:

```text
Judge
------
id
user_id
status
invited_at
accepted_at
removed_at
```

Judge status should support at least:

```text
INVITED
ACTIVE
SUSPENDED
REMOVED
```

---

# 4.2 Judging stages

Create a reusable concept for a judging stage.

Example:

```text
JudgingStage
------------
id
competition/event_id
name
type/scope
track_id nullable
status
review_count
rubric_version_id
assignment_version
created_at
opened_at
closed_at
finalized_at
```

A stage may represent:

```text
Track A Judging
Track B Judging
Track C Judging
Overall Judging
```

This is important because Type 2B has a separate overall judging stage.

Do NOT hard-code:

```text
if track then ...
```

throughout the codebase.

Use a proper stage abstraction.

---

# 4.3 Judge panels

For track judging:

```text
JudgePanel
----------
id
stage_id
track_id
name
```

and:

```text
PanelJudge
----------
panel_id
judge_id
```

Panel membership is an organizer decision.

The assignment algorithm must NOT invent which judge belongs to which track.

---

# 4.4 Assignments

Create/use:

```text
JudgeAssignment
---------------
id
stage_id
submission_id
judge_id
assignment_version
status
assigned_at
started_at
completed_at
```

Statuses:

```text
ASSIGNED
IN_PROGRESS
COMPLETED
CANCELLED
```

Assignment and review are separate concepts.

A judge may be assigned a submission but have no review yet.

---

# 4.5 Rubrics

Create versioned rubrics.

```text
Rubric
------
id
event_id
name
version
status
created_at
```

Criteria:

```text
RubricCriterion
---------------
id
rubric_id
name
description
max_score
weight
display_order
```

Validate:

```text
max_score > 0
weight > 0
Σ weight = 1
```

Once judging starts, do not mutate the active rubric in place.

A changed rubric must create a new version.

---

# 4.6 Reviews

Conceptually:

```text
Review
------
id
assignment_id
stage_id
submission_id
judge_id
status
raw_score
version
submitted_at
created_at
updated_at
```

Criterion-level scores:

```text
ReviewScore
-----------
review_id
criterion_id
score
```

A review belongs to exactly one assignment.

A judge cannot create a review unless they are authorized for that assignment.

---

# 4.7 Calibration

Persist normalization results.

Create concepts equivalent to:

```text
CalibrationRun
--------------
id
stage_id
algorithm_version
status
created_at
finalized_at
```

```text
JudgeCalibrationOffset
----------------------
calibration_run_id
judge_id
offset
```

```text
CalibrationPairStat
-------------------
calibration_run_id
judge_i
judge_j
overlap_count
mean_difference
residual
```

```text
NormalizedReviewScore
---------------------
calibration_run_id
review_id
raw_score
judge_offset
normalized_score
```

Do not overwrite raw scores.

Raw judging evidence and calibrated values must remain separately auditable.

---

# 5. JUDGING STAGE STATE MACHINE

Implement explicit state transitions.

Suggested:

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

Do not allow invalid transitions.

Examples:

* cannot review a stage that is not OPEN
* cannot modify assignments after finalization
* cannot modify rubric after judging has begun
* cannot normalize before required reviews exist
* cannot finalize a failed calibration
* cannot modify finalized results

If the existing project has a different state-machine convention, integrate with it rather than creating a competing convention.

---

# 6. ASSIGNMENT ENGINE

Build assignment as a backend/domain service.

Do not implement assignment logic in the frontend.

Suggested structure:

```text
judging/
    assignment/
        constructor
        validator
        overlap
        connectivity
        repair
        dropout
```

The engine must support:

```text
manual assignment
batch assignment
algorithmic assignment
```

All three must use the same validation rules.

---

# 7. ASSIGNMENT INPUTS

The algorithm receives:

```text
stage
submission pool
eligible judge pool
R
judge capacities
panel scope
assignment configuration
deterministic seed/version
```

For Type 1:

```text
submissions = global judging pool
judges = global eligible judge pool
```

For Type 2:

```text
submissions = submissions belonging to track
judges = judges belonging to that track's panel
```

For an overall stage:

```text
submissions = organizer-selected finalists
judges = overall panel
```

---

# 8. EXACT REVIEW COUNT

If:

```text
R = 3
```

then every submission must receive exactly:

```text
3 distinct judges
```

unless the organizer explicitly authorizes a documented exception.

Never silently create:

```text
R = 3
submission gets 2
```

and pretend judging is complete.

Assignment validation must enforce exact review count.

---

# 9. NO DUPLICATE JUDGES

A judge cannot review the same submission twice.

The assignment engine must enforce:

```text
unique(submission_id, judge_id)
```

at both:

1. algorithm level
2. database constraint level where practical

Never rely only on application logic.

---

# 10. EXACT WORKLOAD BALANCING

Let:

```text
total_slots = submissions × R
J = number of active eligible judges
```

Then:

```text
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

when all judges are eligible.

Do not use arbitrary workload tolerances when exact balancing is possible.

---

# 11. DELIBERATE OVERLAP

Overlap is not an accidental side effect.

For judges `i` and `j`:

```text
Oij =
number of submissions reviewed by both judges
```

For `K` submissions and `R` reviews per submission:

```text
Σ Oij = K × C(R,2)
```

This invariant must be tested.

The ideal average pair overlap is:

```text
P =
K × C(R,2)
----------------
C(J,2)
```

The objective is:

```text
PairLoss =
Σ(i<j) (Oij - P)²
```

The goal is broad, balanced overlap.

Do NOT maximize overlap between one favorite pair of judges.

---

# 12. SCALABLE ASSIGNMENT ALGORITHM

Do not enumerate every possible:

```text
C(J,R)
```

judge combination.

Use the greedy construction defined in `JUDGING.md`.

For each submission:

1. Build feasible judges.
2. Select a judge using the marginal objective.
3. Add that judge.
4. Recalculate.
5. Repeat until R judges are selected.
6. Validate.
7. Repair if required.

For candidate judge `j`:

```text
ΔPairLoss(j) =
Σk [
    (Ojk + 1 - P)²
    -
    (Ojk - P)²
]
```

Selection tuple:

```text
(
    connectivity_penalty,
    ΔPairLoss,
    quota_penalty,
    canonical_judge_id
)
```

Use stable canonical IDs.

Do not use:

```text
database insertion order
random runtime order
completion time
```

as a hidden tie-break.

The constructor is a heuristic, not a claim of global combinatorial optimality.

The output must still pass hard validation.

---

# 13. CONNECTIVITY

Create the judge overlap graph.

```text
node = judge
edge(i,j) = judges i and j share ≥1 submission
```

For a single normalization model, the graph must be connected.

Why:

If:

```text
Judge A ─ Judge B

Judge C ─ Judge D
```

with no connection between the components, the relative offsets between the two components are not identifiable.

Therefore:

```text
DISCONNECTED_CALIBRATION_GRAPH
```

must be treated as a real failure condition.

Do not fabricate offsets.

Do not silently normalize disconnected components against arbitrary constants.

---

# 14. ASSIGNMENT REPAIR

After initial construction, validate:

```text
exact review count
judge distinctness
judge quotas
pair overlap
graph connectivity
eligibility
determinism
```

If invalid, perform deterministic local swaps.

Candidate ordering:

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

Choose the lexicographically smallest valid improving swap.

Use a safety ceiling to prevent infinite repair loops.

If repair fails:

```text
FAILED_ASSIGNMENT
```

Fail closed.

Do not silently reduce R.

Do not silently remove judges.

Do not silently weaken isolation.

---

# 15. MANUAL ASSIGNMENT

Organizer must be able to manually assign:

```text
Submission A → Judge 1
Submission A → Judge 3
Submission A → Judge 7
```

But manual assignments MUST still pass validation.

Manual mode does not bypass:

```text
eligibility
duplicate review constraints
track isolation
R requirements
capacity constraints
connectivity requirements where applicable
```

If a manual assignment violates a rule, return a useful validation error explaining exactly which rule failed.

---

# 16. BATCH ASSIGNMENT

Organizer must be able to select a set of submissions and generate assignments.

Example:

```text
40 submissions
10 judges
R = 3
```

The assignment engine should produce one coherent assignment snapshot.

Do not generate unrelated assignments one submission at a time without considering global overlap/workload.

---

# 17. ASSIGNMENT VERSIONING

Every generated assignment must have a version.

Example:

```text
assignment_version = 7
```

Store:

```text
algorithm version
configuration
seed
judge pool
submission pool
R
result
created timestamp
```

Once judging starts, assignment history must remain auditable.

If reassignment is required, create a new assignment version.

Do not silently mutate historical assignment records.

---

# 18. JUDGE DROPOUT

If a judge is removed:

1. Keep completed valid reviews.
2. Remove judge from future assignments.
3. Find outstanding requirements.
4. Determine active judges.
5. Reassign remaining work.
6. Preserve duplicate-review constraints.
7. Rebalance remaining workload.
8. Revalidate overlap.
9. Revalidate connectivity.
10. Freeze replacement assignment snapshot.

For remaining slots:

```text
q_min = floor(remaining_slots / active_judges)

q_max = ceil(remaining_slots / active_judges)
```

Do not rewrite historical workload.

If replacement is impossible:

```text
FAILED_REPLACEMENT_CAPACITY
```

The organizer must explicitly authorize a reduced-review exception.

---

# 19. JUDGE INVITATION

Implement organizer workflow:

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

The implementation should use the existing authentication/session architecture.

Do not introduce a third-party auth dependency.

---

# 20. RUBRIC CONFIGURATION

Organizer can:

```text
create rubric
add criterion
edit criterion before judging
set max score
set weight
reorder criteria
save rubric version
```

Validation:

```text
all weights > 0
sum(weights) = 1
all max scores > 0
```

Judging must reference a frozen rubric version.

---

# 21. RAW RUBRIC SCORE

For criterion `c`:

```text
q(s,j,c) =
score(s,j,c) / max_c
```

Raw weighted review score:

```text
r(s,j) =
100 × Σ_c weight_c × q(s,j,c)
```

Therefore:

```text
0 <= r(s,j) <= 100
```

Implement this calculation in one backend/domain function.

Do not duplicate the formula independently in frontend and backend.

The frontend may display calculated values, but the backend is authoritative.

---

# 22. REVIEW WORKFLOW

Judge must be able to:

```text
view assigned submission
view rubric
enter criterion scores
write feedback
save draft
resume draft
submit final review
```

A judge may only submit a review for a submission assigned to them.

After final submission, the review should be immutable except through the explicit correction/invalidation process.

---

# 23. JUDGE UX

The judge should see:

```text
My Judging

15 assigned
11 completed
4 remaining
```

Submission list:

```text
Project A     Complete
Project B     Complete
Project C     In Progress
Project D     Not Started
```

The judge must NOT see:

```text
other judges' scores
other judges' comments
live ranking
aggregate ranking
another track's projects
another track's judging context
```

Do not expose hidden information through API responses merely because the UI does not render it.

---

# 24. CROSS-JUDGE NORMALIZATION

Do NOT independently z-score each judge.

Do NOT simply average raw judge scores.

Use the additive WLS model from `JUDGING.md`.

For judges `i` and `j`:

```text
ωij =
number of valid shared submissions
```

Mean difference:

```text
d̄ij =
(1 / ωij)
Σ_s [r(s,i) - r(s,j)]
```

Let `bj` be judge `j`'s systematic scoring offset.

Estimate offsets by minimizing:

```text
L_cal =
Σ(i<j)
ωij [
    (bi - bj) - d̄ij
]²
```

subject to:

```text
Σ_j bj = 0
```

---

# 25. WHY WLS USES OVERLAP COUNT

Under the baseline model:

```text
Var(d̄ij) ∝ 1 / ωij
```

Therefore inverse-variance weighting is proportional to:

```text
ωij
```

So overlap count is the WLS weight.

The system must record diagnostics because real judging data may not perfectly satisfy the baseline assumptions.

---

# 26. CALIBRATION DIAGNOSTICS

Before accepting a calibration result, check:

```text
graph connectivity
overlap counts
judge offsets
residuals
score ranges
near-zero variance
contradictory overlap evidence
unusually large offsets
```

Diagnostics are not automatically proof of bad judging.

Do not label a judge malicious solely because they have:

```text
large offset
high disagreement
unusual score distribution
```

These are organizer-review signals.

---

# 27. APPLYING NORMALIZATION

For judge `j`:

```text
n(s,j) =
r(s,j) - bj
```

Do NOT clamp individual normalized reviews.

Then:

```text
F_raw(s) =
(1/R) Σ_j n(s,j)
```

Every valid judge contributes equally.

Publication score:

```text
F(s) =
min(100, max(0, F_raw(s)))
```

CRITICAL:

```text
RANK USING F_raw
NOT publication score
```

Example:

```text
Project A: F_raw = 108
Project B: F_raw = 101
```

Both may display as:

```text
100
```

but A ranks above B because ranking uses full-precision `F_raw`.

Never round before ranking.

Never clamp before ranking.

---

# 28. TIE BREAKING

Use full internal precision.

If:

```text
F_raw(A) != F_raw(B)
```

the higher value ranks first.

Only exact equality invokes the deterministic tie-break.

The tie-break must be based on a stable canonical identifier/hash.

Never use:

```text
database insertion order
completion time
judge ID
runtime ordering
```

Displayed decimal rounding must never determine ranking.

---

# 29. TYPE 2 — TRACK PANELS

Implement track judging by reusing the same judging engine.

For each track:

```text
Track
 ↓
Track submissions
 ↓
Organizer-defined panel
 ↓
Assignment
 ↓
Independent reviews
 ↓
Track normalization
 ↓
Track ranking
```

The track panel is isolated.

A Track A judge can only access:

```text
Track A submissions
Track A rubric
Track A assignments
Track A own reviews
```

They cannot access:

```text
Track B submissions
Track B ballots
Track B scores
Track B rankings
Track B judge data
```

---

# 30. NO CROSS-TRACK NORMALIZATION

This is mandatory.

Do NOT build one global normalization model across independent track panels.

Example:

```text
Track A judges
Track B judges
```

If there is no shared submission connecting the panels, their relative scoring offsets are not identifiable.

Therefore:

```text
Track A → Track A calibration
Track B → Track B calibration
```

not:

```text
Track A + Track B → global calibration
```

---

# 31. TYPE 2A — TRACK WINNERS ONLY

If configured:

```text
Track A
 ↓
Track judging
 ↓
Track normalization
 ↓
Track ranking
 ↓
Track winner(s)

Track B
 ↓
Track judging
 ↓
Track normalization
 ↓
Track ranking
 ↓
Track winner(s)
```

There is no cross-track final ranking.

---

# 32. TYPE 2B — TRACK + OVERALL

If the organizer enables an overall stage:

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
Overall normalization
 ↓
Overall ranking
```

Do not silently combine:

```text
track score
+
overall score
```

The overall stage is independent.

---

# 33. PROGRESS DASHBOARD

Organizer dashboard must provide live progress.

At minimum:

```text
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

```text
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

Judge-level table:

```text
Judge
Assigned
Completed
Remaining
Status
```

Statuses:

```text
NOT_STARTED
IN_PROGRESS
COMPLETE
REMOVED
```

Calculate progress from assignment/review state.

Do not maintain fragile counters as the only source of truth.

---

# 34. EXPORT

Implement CSV export at every applicable judging stage.

At minimum:

## Assignment export

```text
stage
track
submission_id
judge_id
assignment_status
assigned_at
completed_at
```

## Raw review export

```text
stage
submission_id
judge_id
criterion
criterion_score
raw_review_score
submitted_at
```

## Normalization export

```text
stage
submission_id
judge_id
raw_score
judge_offset
normalized_score
```

## Final results export

```text
rank
submission_id
final_raw_score
publication_score
```

Organizer/admin exports may include calibration diagnostics.

Judge exports must never expose another judge's data.

CSV output must be deterministic and valid.

---

# 35. FINALIZATION

Finalization must produce an immutable snapshot.

Before finalization:

```text
validate assignment
validate reviews
validate rubric
validate calibration
validate ranking
validate exceptions
```

Then persist:

```text
competition configuration
algorithm versions
assignment version
deterministic seed
valid review IDs
raw scores
calibration offsets
normalized scores
final raw scores
publication scores
exceptions
corrections
finalization timestamp
```

After finalization:

```text
results immutable
assignment immutable
calibration immutable
ranking immutable
```

Corrections after finalization must create a new version/audit event rather than silently editing the historical result.

---

# 36. AUDIT TRAIL

Record important judging events:

```text
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

Audit information is organizer/admin only.

Judges cannot read the audit log unless explicitly authorized by the existing permission model.

---

# 37. API DESIGN

Follow the project's existing API conventions.

Conceptually the API should support:

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

Do not blindly create these exact paths if the existing API architecture uses a different convention.

The important requirement is that equivalent functionality exists and follows existing project patterns.

---

# 38. AUTHORIZATION TESTING

Write explicit tests for direct API attacks.

Test:

```text
Judge A requests Judge B review
Judge A requests Judge B score
Judge A changes review ID
Judge A changes assignment ID
Judge A requests Track B submission
Judge A requests Track B score
Judge A posts review for unassigned submission
Judge A submits review as another judge
Judge A requests aggregate results
Judge A requests audit log
Judge A attempts to modify assignment
Judge A attempts to modify rubric
Judge A attempts to run normalization
```

All unauthorized cases must fail at the backend.

Do not accept:

```text
HTTP 200 + filtered frontend
```

as isolation.

---

# 39. MATHEMATICAL TESTING

Write unit/property tests for the judging mathematics.

At minimum:

```text
weighted rubric score
review score bounds
review count
judge distinctness
quota balancing
overlap invariant
PairLoss
ΔPairLoss
connectivity
repair
WLS offsets
zero-sum constraint
normalized score
F_raw
publication clamp
ranking
tie-break
```

---

# 40. ASSIGNMENT INVARIANT TEST

Given:

```text
K submissions
J judges
R reviews/submission
```

verify:

```text
total assignments = K × R
```

and:

```text
Σ Oij = K × C(R,2)
```

and:

```text
max(load) - min(load) <= 1
```

when exact balancing is feasible.

---

# 41. DETERMINISM TEST

Run the same assignment twice:

```text
assignment(input)
assignment(input)
```

Expected:

```text
result_1 == result_2
```

The same configuration, canonical IDs and deterministic seed/version must produce the same assignment.

Do not introduce hidden randomness.

---

# 42. NORMALIZATION TEST

Create synthetic judges with known offsets.

Example:

```text
Judge A = baseline
Judge B = baseline + 10
Judge C = baseline - 5
```

Create overlapping submissions.

Verify that WLS calibration estimates relative offsets correctly within expected numerical tolerance.

Then verify:

```text
raw score
→ offset
→ normalized score
→ F_raw
```

and ensure raw scores remain unchanged.

---

# 43. TRACK ISOLATION TEST

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

All must fail.

Then verify that:

```text
Track A normalization
```

does not use Track B judges.

---

# 44. FINALIZATION TEST

Verify that after:

```text
stage.finalize()
```

the following cannot be modified silently:

```text
assignment
review
rubric
calibration
ranking
final score
```

Any correction must create a new auditable version.

---

# 45. FRONTEND REQUIREMENTS

Build the minimum complete interfaces needed for Tier 2.

## Organizer

```text
Judges
Judge invitations
Judge panels
Rubric
Assignments
Assignment generation
Assignment validation
Judging progress
Calibration
Results
Exports
Finalization
Audit
```

## Judge

```text
My assignments
Submission review
Rubric
Draft review
Submit review
My progress
My submitted scores
```

Do not show peer scores.

Do not show unauthorized track data.

---

# 46. ERROR HANDLING

Use explicit domain errors.

Examples:

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

Do not swallow errors and produce plausible-looking results.

---

# 47. FAILURE-CLOSED PRINCIPLE

This is critical.

If the judging engine cannot satisfy a required invariant:

```text
DO NOT FABRICATE A RESULT.
```

Examples:

```text
cannot create valid assignment
→ fail assignment

calibration graph disconnected
→ fail calibration

insufficient reviews
→ do not finalize

unauthorized request
→ deny

invalid rubric
→ reject configuration
```

The system must prefer an explicit failure over an apparently complete but statistically invalid result.

---

# 48. PERFORMANCE

Do not implement brute-force assignment.

The algorithm must scale reasonably for the DOGFOOD fixture shape and beyond.

The official fixture is expected to include roughly:

```text
40 projects
30 judges
8 tracks
```

and deliberately contains difficult judging cases such as incomplete batches, duplicate entries and a low-variance reviewer.

The implementation must not assume clean/tidy input.

Design algorithms around:

```text
O(submissions × judges × R)
```

or similarly practical complexity where possible.

Do not enumerate all judge combinations.

---

# 49. DATABASE INTEGRITY

Where appropriate, enforce invariants in the database too.

Examples:

```text
unique(stage_id, submission_id, judge_id)
```

for assignments.

Use foreign keys.

Use transactions for:

```text
review submission
assignment generation
assignment repair
calibration snapshot
finalization
```

Do not rely entirely on frontend validation.

---

# 50. CONCURRENCY

Consider two requests arriving simultaneously.

Examples:

```text
judge submits review twice
organizer finalizes while review is submitted
two organizers modify assignments
two assignment-generation jobs run simultaneously
```

Use transactions/locking/idempotency where necessary.

A review must not accidentally be accepted twice.

Two assignment-generation operations must not silently overwrite one another.

Finalization must be atomic.

---

# 51. SECURITY PRINCIPLE

Treat all IDs received from the client as untrusted.

Never do:

```text
findReview(reviewId)
```

and return it merely because it exists.

Instead enforce ownership/scope:

```text
find review
verify current actor
verify stage
verify assignment
verify panel/track
verify permission
then return
```

This applies to every judging endpoint.

---

# 52. DO NOT DO THESE THINGS

Do NOT:

```text
average raw scores and call that normalization

z-score every judge independently

normalize tracks globally

let frontend controls enforce isolation

return all judging data and hide fields in JavaScript

use database insertion order for ranking

round scores before ranking

clamp normalized scores before calculating F_raw

silently reduce R

silently remove a judge from assignment

silently fabricate calibration offsets

overwrite raw reviews with normalized values

modify finalized results in place

hard-code one fixed rubric

hard-code exactly 3 judges

hard-code one number of submissions

hard-code one number of tracks

hard-code one judge pool

enumerate C(J,R) combinations

create separate Type 1 and Type 2 implementations with duplicated logic

introduce a cloud dependency

break existing Tier 1 functionality
```

---

# 53. DOCUMENTATION REQUIREMENTS

Update/create:

```text
README.md
ARCHITECTURE.md
DATA-MODEL.md
JUDGING.md
```

Do not overwrite an existing `JUDGING.md` casually.

If the current `JUDGING.md` already contains the agreed mathematical design, implement it and preserve it.

Documentation must explain:

```text
assignment algorithm
workload balancing
overlap
connectivity
repair
scoring
WLS normalization
track isolation
overall judging
failure behavior
finalization
auditability
```

The DOGFOOD judging criteria explicitly care about whether the normalization is documented and defensible.

---

# 54. ARCHITECTURE REQUIREMENTS

`ARCHITECTURE.md` should explain:

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

Explain why the judging system is structured this way.

Do not generate generic AI architecture prose.

Document the actual implementation.

---

# 55. DATA-MODEL REQUIREMENTS

`DATA-MODEL.md` must document:

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

The documentation must match the actual schema.

---

# 56. ACCEPTANCE TESTING

Before declaring Tier 2 complete:

Run:

```bash
docker compose up
```

from a clean environment.

Verify:

```text
application starts
database initializes
seed data loads
authentication works
Tier 1 still works
judge invitations work
judge assignments work
manual assignment works
batch assignment works
algorithmic assignment works
rubric works
judge review works
backend isolation works
progress works
normalization works
track isolation works
CSV exports work
finalization works
```

Run the official acceptance suite if available.

Run the project's own test suite.

Do not claim Tier 2 if core Tier 2 acceptance tests fail.

---

# 57. IMPLEMENTATION ORDER

Implement in this exact dependency order unless the existing architecture requires a small variation.

```text
PHASE 1
Inspect existing architecture
        ↓
Confirm existing T1 contracts
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
Criterion validation

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
Overall judging stage

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
Documentation
        ↓
Final cleanup
```

---

# 58. CODING STYLE

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

For example:

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

These should be testable independently.

---

# 59. DO NOT OVERENGINEER

Do not introduce unnecessary infrastructure.

The DOGFOOD requirement is a self-hostable platform that can run locally.

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

unless the existing project already requires something similar.

A clean modular monolith is completely acceptable.

The goal is correctness and adoptability.

---

# 60. DEFINITION OF DONE

Tier 2 is complete only when all of the following are true:

```text
[ ] Judges can be invited
[ ] Judges can become active
[ ] Judges can be assigned manually
[ ] Judges can be assigned in batches
[ ] Judges can be assigned algorithmically
[ ] Assignments are deterministic
[ ] Assignments balance workload
[ ] Assignments deliberately create overlap
[ ] Assignment graph satisfies connectivity where required
[ ] Assignment repair works
[ ] Judge dropout is handled
[ ] Organizer can configure weighted rubric
[ ] Rubric is versioned
[ ] Judges can independently score
[ ] Judges can save drafts
[ ] Judges can submit reviews
[ ] Raw reviews remain preserved
[ ] Backend role isolation is enforced
[ ] Judge cannot access peer scores
[ ] Judge cannot access unauthorized tracks
[ ] Judge cannot bypass isolation via API
[ ] Organizer sees live judging progress
[ ] Cross-judge normalization uses JUDGING.md WLS method
[ ] Calibration diagnostics are recorded
[ ] Track panels normalize independently
[ ] Overall stage is separate when configured
[ ] F_raw is calculated correctly
[ ] Publication clamp occurs only after F_raw
[ ] Ranking uses full precision F_raw
[ ] Tie-break is deterministic
[ ] CSV export works
[ ] Finalization creates immutable snapshot
[ ] Audit trail exists
[ ] Tier 1 still works
[ ] docker compose up works
[ ] application works offline
[ ] tests pass
[ ] acceptance suite passes
[ ] documentation matches implementation
```

---

# 61. FINAL INSTRUCTION TO THE CODING AGENT

Do not start by generating a large amount of code.

First:

```text
1. Inspect the repository.
2. Inspect the existing Tier 1 architecture.
3. Inspect JUDGING.md completely.
4. Identify existing entities that can be reused.
5. Identify schema/API conflicts.
6. Produce a short implementation assessment.
7. Then implement incrementally.
```

After each major phase:

```text
run tests
check migrations
check authorization
check existing Tier 1 behavior
```

Do not proceed while introducing known regressions.

At the end, provide:

```text
IMPLEMENTED
----------------
what was built

TESTED
----------------
what passed

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

The acceptance criterion is:

> **The judging system actually works, survives direct API access attempts, implements the mathematics in `JUDGING.md`, runs locally from one command, and can be operated by an organizer.**

A smaller correct implementation is preferable to a larger implementation with broken invariants.
