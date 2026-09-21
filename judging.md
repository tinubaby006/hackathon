# DOGFOOD HACKATHON

# JUDGING.md

## Type 1 — No Tracks — Complete Judging Architecture

> **Scope:** This document is the complete end-to-end judging specification for **Type 1 (No Tracks)**.
>
> It defines both judging phases:
>
> 1. **Phase 1 — Standard pairwise triage and exact-K shortlisting**
> 2. **Phase 2 — Rubric judging, overlap-based judge calibration, normalization, and final scoring**
>
> Phase 1 and Phase 2 solve different problems and must not be collapsed into one scoring model.
>
> **Phase 1 asks:** which submissions have enough evidence to deserve detailed judging?
>
> **Phase 2 asks:** given the shortlisted submissions, what are their final normalized rubric scores?
>
> The Phase 1 output is the Phase 2 input. No Phase 2 score exists for a submission that does not enter the Phase 2 shortlist.

---

# 0. End-to-End Type 1 Architecture

The complete Type 1 flow is:

```text
ALL VALID SUBMISSIONS
        |
        v
PHASE 1 — STANDARD MODE
        |
        +--> deterministic anonymization
        |
        +--> bootstrap pairwise evidence
        |
        +--> Bradley–Terry model
        |
        +--> protected adaptive investigation
        |
        +--> uncertainty / shortlist stability
        |
        v
EXACT K SHORTLIST
        |
        v
PHASE 2 — DETAILED JUDGING
        |
        +--> validate N, J, R, rubric
        |
        +--> exact assignment quotas
        |
        +--> deliberate judge overlap
        |
        +--> connected judge-overlap graph
        |
        +--> rubric reviews
        |
        +--> immutable raw scores
        |
        +--> overlap-based judge offset calibration
        |
        +--> calibrated / normalized scores
        |
        +--> final submission scores
        |
        +--> disagreement metrics
        |
        v
FINAL TYPE 1 RESULTS
```

## 0.1 Core separation

Phase 1 is **relative triage**. It uses pairwise choices and a Bradley–Terry model. It does not produce the final competition score.

Phase 2 is **detailed evaluation**. It uses the competition rubric and produces the final score after judge calibration and normalization.

The system must never treat a Phase 1 Bradley–Terry estimate as a Phase 2 rubric score.

## 0.2 Core invariants across both phases

The implementation must preserve these invariants:

- invalid submissions are excluded before judging;
- judges never receive unauthorized submissions or other judges' results;
- Phase 1 comparison graphs are valid and connected when a global model is used;
- Phase 1 produces exactly `K` shortlisted submissions;
- Phase 2 assigns exactly `R` distinct judges to every shortlisted submission;
- Phase 2 judge workloads are exactly balanced within one review;
- Phase 2 overlap is deliberate and measurable;
- the Phase 2 judge-overlap graph is connected when global calibration is required;
- raw Phase 2 scores are immutable evidence;
- calibration changes derived values, not raw evidence;
- final scores are deterministic and reconstructable;
- hard constraints fail closed rather than being silently relaxed.

---

# PART I — PHASE 1

## 1. Purpose

Phase 1 is a **rapid triage and shortlist-generation system**.

Its purpose is to reduce a large number of submissions to an exact organizer-defined shortlist for Phase 2 while minimizing the chance that a genuinely strong submission is incorrectly excluded.

Phase 1 is **not the final judging system**.

Phase 1 should answer:

> "Which submissions have enough evidence to deserve detailed Phase 2 judging?"

It should not attempt to replace the detailed rubric-based evaluation performed in Phase 2.

### Core objective

The primary optimization target is:

> **Minimize false exclusion of genuinely strong submissions.**

A submission that is temporarily ranked lower because of limited or noisy early evidence must not be permanently eliminated merely because of that temporary estimate.

---

# 2. Organizer Inputs

The organizer provides:

* `N` = number of valid submissions
* `J` = number of active Phase 1 judges
* `K` = exact number of submissions that advance to Phase 2

Example:

```text
N = 2500
J = 20
K = 20
```

The organizer should not manually configure:

* comparisons per submission
* comparisons per judge
* pair-selection rules
* judge workload
* adaptive thresholds

The system calculates these automatically.

There is only one judging mode:

> **STANDARD**

No separate "light", "detailed", or manually tuned judging mode is required.

---

# 3. What the Judge Sees

Each Phase 1 judging action presents exactly two anonymized submissions:

```text
Submission A

[submission content]
```

and

```text
Submission B

[submission content]
```

The judge answers:

> Which submission would you advance if you could advance only one?

The judge selects:

```text
A
```

or

```text
B
```

There is no:

* numerical score
* ranking list
* tie button
* skip button
* confidence score
* visibility of other judges' decisions

This keeps the judging interaction extremely simple.

---

# 4. Why Phase 1 Uses Pairwise Judging

A numerical score such as:

```text
A = 82
B = 79
```

requires judges to independently interpret what an absolute number means.

Pairwise judging asks a simpler question:

> "If I had to choose one, which would I advance?"

This is particularly useful when the submission pool is large and heterogeneous.

The system is interested primarily in **relative evidence** during Phase 1.

The pairwise result is therefore:

```text
A beats B
```

or:

```text
B beats A
```

That result becomes evidence for the statistical ranking model.

---

# 5. Important Interpretation of Forced Choice

A forced choice does **not** mean the system assumes the winner is dramatically better.

Suppose both submissions are excellent.

The judge still answers:

> "Which one would I advance if I could advance only one?"

This is intentional.

The system does not require judges to estimate an absolute quality gap.

However, forced-choice comparisons introduce noise when two submissions are very similar.

Therefore the backend must not treat one comparison as decisive.

Repeated and independent evidence, uncertainty tracking, and adaptive investigation exist specifically to protect against this problem.

---

# 6. The Fundamental Principle

The most important Phase 1 rule is:

> **A submission must never be excluded from further consideration solely because its current Bradley–Terry estimate is below the current shortlist boundary.**

The current estimate is evidence.

It is not a verdict.

This protects against situations such as:

* an excellent project losing several early comparisons by bad luck
* two excellent projects being nearly indistinguishable
* a strong project initially being matched against other strong projects
* a strong project receiving too little evidence
* a moderate project temporarily benefiting from weak opponents
* individual judges having different levels of noise

---

# 7. Phase 1 Pipeline

The complete pipeline is:

```text
SUBMISSIONS
    |
    v
VALIDATION
    |
    v
DETERMINISTIC ANONYMIZATION
    |
    v
MINIMUM EVIDENCE / BOOTSTRAP
    |
    v
INITIAL BRADLEY–TERRY MODEL
    |
    v
PROTECTED ADAPTIVE INVESTIGATION
    |
    v
UPDATED BRADLEY–TERRY MODEL
    |
    v
UNCERTAINTY / EVIDENCE CHECK
    |
    +---- not sufficiently resolved ----> more adaptive investigation
    |
    v
EXACT K SUBMISSIONS
    |
    v
PHASE 2
```

---

# 8. Submission Validation

Before judging begins, every submission must pass validation.

Required checks include:

* valid submission ID
* submission belongs to the competition
* submission is eligible for Phase 1
* submission is not duplicated
* submission content is available
* submission is not already disqualified
* submission is in the correct competition/version
* no judge identity information is exposed through the judging payload

Invalid submissions must be removed before assignment generation.

The system must not silently change `N`.

The validated number of submissions becomes the Phase 1 population:

```text
N = validated submission count
```

---

# 9. Deterministic Anonymization

Judges must not receive identifying information that could influence the comparison.

Each submission receives a deterministic anonymous presentation identifier.

Example:

```text
Submission A
Submission B
```

or:

```text
Project X7F2
Project M91A
```

The anonymous identity must not reveal:

* submission order
* team name
* organization
* participant identity
* geographic information unless intentionally part of judging
* previous ranking
* previous score
* judge decisions

The browser must receive only information required for judging.

Hidden PII must not be sent to the browser merely because the UI does not display it.

---

# 10. Minimum Evidence

Every submission must receive a meaningful minimum amount of Phase 1 evidence before the system relies heavily on its estimated position.

This minimum evidence is **not permanently hard-coded as an arbitrary number**.

The exact Standard-mode budget must be determined through simulation.

The reason is important:

A fixed rule such as:

```text
12 comparisons per submission
```

looks simple but is not automatically safe.

The required evidence depends on:

* `N`
* `J`
* `K`
* shortlist ratio `K/N`
* judge noise
* similarity between strong submissions
* distribution of submission quality
* available judging capacity

Therefore the production system must have a **budget engine** rather than a universally hard-coded comparison count.

---

# 11. Budget Engine

The system receives:

```text
N
J
K
```

and calculates:

```text
total comparison budget B
minimum evidence requirement
adaptive reserve
judge workload
```

The budget engine must be deterministic.

Conceptually:

```text
B = BudgetEngine(N, J, K, StandardConfiguration)
```

The production budget engine is a deterministic versioned function, not a
runtime hand-tuned value. Its configuration must explicitly contain:

```text
minimum_comparisons_per_submission
bootstrap_comparisons_per_submission
adaptive_reserve_fraction
minimum_total_budget
maximum_total_budget
judge_capacity_per_round
```

The engine computes:

```text
B_bootstrap = N * bootstrap_comparisons_per_submission

B_reserve = ceil(B_bootstrap * adaptive_reserve_fraction)

B = clamp(
        B_bootstrap + B_reserve,
        minimum_total_budget,
        maximum_total_budget
    )
```

The implementation must additionally increase `B` when required to satisfy hard
graph-connectivity and judge-capacity constraints, and must record the resulting
budget and every adjustment.

The exact configuration values are simulation-calibrated and versioned. They are
not silently tuned per competition.

The resulting budget must satisfy:

1. every submission receives minimum evidence
2. enough budget remains for adaptive investigation
3. judge workload is feasible
4. judge workload is balanced
5. the resulting comparison graph can remain connected
6. the budget has been validated through simulation

The exact production constants are configuration/version data, not unexplained magic numbers.

---

# 12. Why the Budget Is Simulation-Calibrated

The system is not optimizing for perfect ranking of all submissions.

It is optimizing for:

> **not accidentally throwing away genuinely strong submissions.**

Those are different objectives.

For example, if:

```text
N = 2500
K = 20
```

the system does not necessarily need to determine the exact order of submissions 500 through 2500.

It needs to confidently identify the approximately relevant region around:

```text
the Phase 2 boundary
```

while protecting submissions that could plausibly belong above that boundary.

Therefore the correct budget must be established empirically.

---

# 13. Required Simulation Before Production

The Phase 1 implementation is not considered complete until its budget and adaptive policy pass simulation.

The simulation must model synthetic competitions with known underlying strengths.

For example:

```text
true_strength[i]
```

is generated for every submission.

Synthetic judges then produce noisy pairwise decisions.

The simulator should support:

* low-noise judges
* high-noise judges
* heterogeneous judge noise
* strong projects close together
* moderate projects close together
* unlucky strong projects
* lucky moderate projects
* strong projects facing strong opponents
* strong projects facing weak opponents
* sparse evidence
* different `N`
* different `K/N` ratios
* different `J`

````

Required population sizes should include at least:

```text
N = 50
N = 100
N = 500
N = 2500
````

Larger stress populations may also be tested.

---

# 14. Primary Simulation Metric

The primary metric is:

```text
False Exclusion Rate =
genuinely strong submissions excluded from Phase 2
/
genuinely strong submissions that should have reached Phase 2
```

The system should also measure:

* shortlist recall
* shortlist precision
* boundary error
* number of comparisons
* comparisons per submission
* comparisons per judge
* maximum judge workload
* minimum judge workload
* workload imbalance
* number of repeated pairs
* uncertainty at the shortlist boundary
* number of submissions requiring additional investigation
* runtime
* deterministic reproducibility

The budget should be selected based on these measured outcomes rather than an arbitrary round number.

---

# 15. Judge Capacity

Judge workload must be calculated by the system.

If the total number of pairwise comparisons is:

```text
B
```

then the average judge workload is:

```text
B / J
```

The assignment system should distribute comparisons so that judge workloads differ by at most one whenever possible.

For example:

```text
B = 15,000
J = 20
```

gives:

```text
15,000 / 20 = 750
```

so the target is:

```text
750 comparisons per judge
```

If the division is not exact, judges receive either:

```text
floor(B/J)
```

or:

```text
ceil(B/J)
```

comparisons.

The system should never silently assign an unreasonable workload merely because the organizer entered too few judges.

If capacity becomes insufficient, the system should report that more judges are required.

---

# 16. Judge Assignment

Judge assignment must be:

* deterministic
* balanced
* auditable
* reproducible
* isolated

The assignment engine must know:

```text
N
J
B
judge IDs
submission IDs
assignment_version
algorithm_version
seed
```

A deterministic seed may be derived from:

```text
hash(competition_id + assignment_version + algorithm_version)
```

The same inputs must produce the same assignment.


## 16.1 Exact Phase 1 Comparison-to-Judge Assignment

A Phase 1 comparison is a logical task:

```text
comparison_id = canonical(min(submission_i, submission_j),
                          max(submission_i, submission_j),
                          comparison_version)
```

The comparison scheduler first creates the deterministic comparison list and then
assigns each comparison to one judge.

For comparison index `q` in deterministic comparison order:

```text
preferred_judge_index =
    (hash(seed || comparison_id) mod J)
```

The scheduler selects the first judge in cyclic order from that index who:

1. is active and eligible
2. has remaining Phase 1 capacity
3. has not already judged the same submission pair in the same comparison round
4. does not violate any competition-specific hard constraint

If no judge is feasible, the scheduler performs deterministic capacity repair by
recomputing the remaining feasible judge set for that comparison.

Judge loads must satisfy:

```text
max(load_j) - min(load_j) <= 1
```

whenever `B >= J` and all judges are eligible.

If `B < J`, only `B` judges receive work; unused eligible judges receive zero
Phase 1 tasks. The assignment record must explicitly state this rather than
pretending that all judges were engaged.

A Phase 1 comparison must never be duplicated solely to consume unused judge
capacity.

---

# 17. Bootstrap Stage

The first stage provides every submission with baseline evidence.

The purpose of bootstrap judging is **not** to immediately identify the final top K.

It exists to create enough initial evidence to estimate relative strength.

The bootstrap should:

* avoid self-comparisons
* avoid duplicate pairs
* provide meaningful coverage
* avoid concentrating all comparisons on apparently strong projects
* preserve graph connectivity
* be deterministic
* balance judge workload

---

# 18. Why Bootstrap Must Not Depend on Early Ranking

If the first few results determine who gets compared next, an early lucky or unlucky result can create a feedback loop.

Example:

```text
Excellent project
        |
loses two noisy comparisons
        |
temporarily ranked low
        |
receives fewer comparisons
        |
remains poorly estimated
        |
excluded
```

That is precisely the failure mode Phase 1 is designed to prevent.

Therefore initial comparisons must be generated independently of the initial ranking.

---

# 19. Comparison Graph

Represent submissions as nodes.

Represent a comparison between two submissions as an edge.

Example:

```text
A ----- B
|       |
C ----- D
```

The graph must be connected when the global Bradley–Terry model is used.

Why?

Suppose:

```text
A compares B
C compares D
```

but there are no comparisons connecting `{A,B}` to `{C,D}`.

The system cannot establish the relative scale between those two groups from the available evidence.

Therefore:

> **A disconnected comparison graph is not acceptable for a global Phase 1 model.**

---

# 20. Graph Invariant

At all times, the system tracks:

```text
number of submissions
number of unique pairs
degree of every submission
connected components
```

The final comparison graph must contain exactly one connected component.

If the graph becomes disconnected during an intermediate construction step, the system must repair it before finalization.

---

# 21. Duplicate Comparisons

Duplicate comparisons are not automatically forbidden.

A repeated comparison can be useful when additional evidence is genuinely valuable.

However:

> repeated comparisons must not be the default way of spending the budget.

The adaptive engine should normally prefer an informative new opponent.

A repeated comparison becomes reasonable when:

* two submissions are very close
* the shortlist boundary depends on their relative relationship
* previous evidence is noisy
* additional evidence can materially reduce uncertainty
* the pair has high decision relevance

---

# 22. Initial Bradley–Terry Model

After the bootstrap evidence, the system fits a Bradley–Terry model.

Each submission receives a latent parameter:

```text
theta_i
```

For submissions `i` and `j`:

```text
P(i beats j) =
exp(theta_i) /
(exp(theta_i) + exp(theta_j))
```

The model is fitted from observed pairwise outcomes.

To remove the arbitrary location of the parameters:

```text
sum(theta_i) = 0
```

is imposed.

## 22.1 Regularized BT fitting

Production fitting must use a finite, deterministic regularized objective:

```text
maximize:

log L(theta)
-
lambda * sum_i(theta_i^2)

subject to:

sum(theta_i) = 0
```

where:

```text
lambda > 0
```

is a versioned competition configuration parameter.

The regularization is required because early Phase 1 evidence can contain
submissions with only wins or only losses. Unregularized Bradley–Terry MLE can
then diverge toward `+infinity` or `-infinity`.

The production solver must:

1. use numerically stable logistic/log-sum-exp calculations
2. enforce the zero-sum constraint
3. use the configured positive `lambda`
4. stop only after the configured convergence tolerance is met or the configured iteration ceiling is reached
5. record solver version, `lambda`, tolerance, iteration count, and convergence status
6. fail closed if a finite solution cannot be produced

Regularization is a numerical stabilization device. It must not be interpreted as
evidence that a submission is intrinsically weak or strong.

# 23. What Bradley–Terry Means

A submission's:

```text
theta
```

is a relative-strength estimate.

It does NOT mean:

```text
theta = absolute quality
```

and it does NOT mean:

```text
theta = Phase 2 score
```

For example:

```text
A: theta = +1.2
B: theta = +0.8
C: theta = -0.5
```

means the model currently estimates:

```text
A > B > C
```

in relative pairwise strength.

It does not mean that A has received a final competition score.

---

# 24. Early Ranking Is Temporary

The initial Bradley–Terry ranking is explicitly treated as:

> **a working hypothesis, not an elimination list.**

The system must not say:

```text
rank > K => eliminated
```

after bootstrap.

Instead it enters the adaptive investigation stage.

---

# 25. Adaptive Investigation

The remaining comparison budget is allocated adaptively.

This is where the system spends comparisons where additional evidence is most useful.

The adaptive stage has two simultaneous responsibilities:

### Responsibility A — Boundary protection

Investigate submissions around the Phase 2 cutoff.

If:

```text
K = 20
```

then submissions near the estimated positions:

```text
18
19
20
21
22
...
```

may be highly decision-relevant.

### Responsibility B — Strong-project protection

Do not allow a genuinely strong project to disappear merely because of insufficient or unlucky early evidence.

A submission receives additional protection when it has evidence such as:

* strong observed wins
* strong wins against strong opponents
* insufficient total evidence
* high uncertainty
* current estimate near the shortlist region
* uncertainty interval that reaches the shortlist region
* evidence pattern inconsistent with its current low estimate

---

# 26. The Adaptive Rule

The adaptive engine evaluates candidate comparisons.

A candidate pair:

```text
(i, j)
```

is considered only if:

1. `i != j`
2. both submissions are valid
3. both have remaining evidence capacity
4. the pair does not violate hard assignment constraints
5. the comparison is permitted by the current judge workload
6. the comparison contributes to maintaining a valid connected graph

The candidate is then evaluated by its expected usefulness.

The conceptual priority is:

```text
HARD VALIDITY
    >
PROTECTION OF POSSIBLY STRONG SUBMISSIONS
    >
SHORTLIST-BOUNDARY RELEVANCE
    >
INFORMATION VALUE
    >
EVIDENCE BALANCING
    >
DETERMINISTIC TIE-BREAK
```

This hierarchy is intentional.

---

## 26.1 Exact Adaptive Priority Function

After hard-validity filtering, every candidate pair receives normalized component
scores in `[0,1]`.

Let:

```text
theta_K = current K-th highest BT estimate
se_i = standard error of theta_i
se_j = standard error of theta_j
z = 1.96
```

Define the conservative upper evidence bound:

```text
U_i = theta_i + z * se_i
```

and lower evidence bound:

```text
L_i = theta_i - z * se_i
```

### Boundary relevance

```text
B_i =
exp(
    - abs(theta_i - theta_K)
    /
    max(median_SE, epsilon)
)
```

where `median_SE` is the median current submission standard error and `epsilon`
is the configured numerical floor.

For a candidate pair:

```text
B_ij = max(B_i, B_j)
```

### Strong-project protection

A submission is potentially strong when its conservative upper bound reaches the
current shortlist region:

```text
U_i >= theta_K
```

Define:

```text
S_i =
1                         if U_i >= theta_K
0                         otherwise
```

and:

```text
S_ij = max(S_i, S_j)
```

To prevent a binary flag from dominating every comparison, the candidate's
evidence-deficit factor is also included:

```text
D_i =
1 /
sqrt(
    1 + comparisons_i
)
```

```text
S_ij = max(S_i * D_i, S_j * D_j)
```

### Information value

For:

```text
p_ij = P(i beats j)
```

define:

```text
I_ij = 4 * p_ij * (1 - p_ij)
```

so that:

```text
0 <= I_ij <= 1
```

### Evidence balancing

Let:

```text
m_i = comparisons_i
m_target = mean(comparisons across valid submissions)
```

Define:

```text
E_i =
max(
    0,
    (m_target - m_i) /
    max(m_target, 1)
)
```

and:

```text
E_ij = max(E_i, E_j)
```

### Final score

The candidate priority is:

```text
P_ij =
0.40 * S_ij
+
0.30 * B_ij
+
0.20 * I_ij
+
0.10 * E_ij
```

with:

```text
0 <= P_ij <= 1
```

The coefficients:

```text
(0.40, 0.30, 0.20, 0.10)
```

are part of `algorithm_version` and must be simulation-validated before
production. They are not organizer-tunable at runtime.

Candidates are ordered by descending `P_ij`, then by the deterministic
canonical pair key.

The hard-validity hierarchy remains absolute: a lower-priority candidate that
satisfies a required graph-repair condition can be selected over a higher-score
candidate that would violate that condition.


# 27. Why "Current Rank" Is Not the Priority

A naive adaptive system might say:

> "Compare the projects currently ranked 19–22."

That is unsafe.

Imagine:

```text
Project A = genuinely excellent
Project B = genuinely excellent
```

but A happens to lose several early noisy comparisons.

Its current rank may become:

```text
A = 140
```

The naive system may stop investigating A.

That creates exactly the false-exclusion problem we want to avoid.

Therefore current rank alone can never determine whether a submission receives more evidence.

---

# 28. Uncertainty

Every Bradley–Terry estimate must be interpreted together with its uncertainty.

A submission with:

```text
theta = 1.10
```

and very little evidence is not equivalent to:

```text
theta = 1.10
```

with extensive evidence.

The adaptive engine therefore considers both:

```text
estimated strength
```

and:

```text
uncertainty
```

---

# 29. Conservative Candidate Protection

The system should maintain a conservative evidence region around each submission's current estimate.

Conceptually:

```text
lower evidence bound
        |
        theta
        |
upper evidence bound
```

The exact statistical construction is part of the simulation-calibrated implementation.

The key rule is:

> If a submission's plausible upper strength reaches the shortlist region, it remains eligible for adaptive investigation.

This prevents an uncertain but potentially excellent submission from being eliminated prematurely.

---

# 30. Strong-vs-Strong Protection

Consider:

```text
A = excellent
B = excellent
```

If A loses to B, that does not provide strong evidence that A is mediocre.

The opponent's strength matters.

Bradley–Terry naturally accounts for opponent strength.

Therefore:

```text
A loses to a strong B
```

is treated differently from:

```text
A loses to a weak C
```

The system does not simply count wins and losses.

---

# 31. Weak-Opponent Protection

The reverse problem also matters.

Suppose:

```text
Moderate A
```

beats several weak submissions.

A naive win-rate system may rank A extremely highly.

Bradley–Terry reduces this problem by considering who was defeated.

Therefore the system should not use:

```text
win percentage
```

as the primary ranking mechanism.

---

# 32. Information Value

For two submissions with current predicted probability:

```text
p = P(i beats j)
```

a basic pairwise uncertainty measure is:

```text
U = p(1-p)
```

This is highest when:

```text
p ≈ 0.5
```

because the outcome is most uncertain.

A comparison where:

```text
p = 0.50
```

is usually more informative about relative ordering than:

```text
p = 0.99
```

However, information value alone is not sufficient.

The comparison must also matter to shortlist protection.

Therefore:

> information gain is subordinate to protecting potentially strong submissions and the shortlist boundary.

---

# 33. Adaptive Comparison Example

Suppose:

```text
K = 20
```

and current estimates suggest:

```text
Project A: position 17, low uncertainty
Project B: position 19, high uncertainty
Project C: position 21, high uncertainty
Project D: position 23, very high uncertainty
Project E: position 140, strong wins against strong opponents but insufficient evidence
```

A naive rank-only system focuses on:

```text
B, C, D
```

The protected system may also investigate:

```text
E
```

because its current position alone is not sufficient evidence for exclusion.

That is a deliberate safety mechanism.

---

# 34. Exploration vs Exploitation

Adaptive judging must balance:

### Exploration

Investigating submissions that have not received enough evidence.

and:

### Exploitation

Investigating comparisons that are likely to affect the final shortlist.

If the system only explores:

```text
random pairs
```

it wastes judging capacity.

If it only exploits:

```text
current top-ranked projects
```

it can lock in early mistakes.

Therefore Standard Phase 1 deliberately combines both.

---

# 35. Adaptive Budget Reserve

The total budget is divided conceptually into:

```text
minimum evidence budget
+
adaptive reserve
```

The exact proportion is determined by simulation.

The system must not consume the entire budget during bootstrap.

There must always be capacity for:

* boundary investigation
* strong-project protection
* uncertainty reduction
* graph repair
* evidence balancing

---

# 36. Stable Enough Decision

The adaptive process repeatedly performs:

```text
collect comparisons
        ↓
update Bradley–Terry
        ↓
update uncertainty
        ↓
evaluate shortlist stability
```

If the evidence indicates that the shortlist is sufficiently resolved, the system can finalize.

If not, it continues using the remaining adaptive budget.

---

# 37. Shortlist Stability

Let:

```text
theta_K = K-th highest final BT estimate
theta_K1 = (K+1)-th highest final BT estimate
```

and let:

```text
se_K
se_K1
```

be their standard errors.

Define the cutoff separation:

```text
Delta_K = theta_K - theta_K1
```

and conservative boundary uncertainty:

```text
U_K = 1.96 * sqrt(se_K^2 + se_K1^2)
```

The shortlist is considered **stable** only when:

```text
Delta_K > U_K
```

and all submissions whose 95% upper bound reaches `theta_K` have received at
least the configured minimum adaptive evidence.

Otherwise:

```text
SHORTLIST_BOUNDARY_UNCERTAIN
```

remains active and adaptive investigation continues while budget remains.

The threshold multiplier `1.96` and minimum adaptive evidence are versioned
algorithm parameters and must be simulation-validated.

---

# 38. Boundary Uncertainty

If the budget is exhausted while the boundary remains materially uncertain, the system must not pretend that the ranking is certain.

It should record:

```text
SHORTLIST_BOUNDARY_UNCERTAIN
```

and preserve the evidence and uncertainty information for organizers.

The system must still produce the exact required shortlist if the competition rules require exactly K advancement slots, but the audit record must clearly indicate that the boundary was uncertain.

---

# 39. Exact Shortlist Size

The organizer specifies:

```text
K
```

Therefore the final output must contain exactly:

```text
K submissions
```

unless the competition explicitly defines another failure policy.

For:

```text
K = 20
```

the system returns:

```text
20 submissions
```

not:

```text
19
```

and not:

```text
25
```

---

# 40. Final Shortlist Selection

After the evidence budget is exhausted or the stability criterion is met:

1. fit the final Bradley–Terry model
2. calculate final estimates
3. calculate uncertainty/evidence information
4. order submissions by the final model estimate
5. select exactly K submissions

Ties must be handled deterministically.

A tie-break must not use:

* random unseeded behavior
* database insertion order
* judge identity
* hidden organizer preferences

The deterministic tie-break must be versioned and auditable.

---

# 41. No Phase 1 Rubric Replacement

Phase 1 must not attempt to recreate the Phase 2 rubric.

Do not turn Phase 1 into:

```text
innovation = 7
execution = 8
impact = 9
```

That would create two overlapping scoring systems.

Phase 1 is intentionally lightweight.

Its job is:

```text
relative triage
```

Phase 2 is:

```text
detailed evaluation
```

---

# 42. No Judge Calibration in Phase 1

Phase 1 should not attempt to perform the full Phase 2 judge-calibration system.

Why?

Phase 1 is designed for fast triage.

Introducing detailed calibration here would add:

* complexity
* latency
* implementation risk
* additional assumptions

without being necessary for the core purpose.

Phase 1 instead relies on:

* multiple comparisons
* graph connectivity
* Bradley–Terry aggregation
* uncertainty
* adaptive evidence collection
* simulation validation

Phase 2 performs the detailed judge calibration.

---

# 43. Judge Isolation

A judge must not see:

* other judges' decisions
* current project ranking
* Bradley–Terry estimates
* shortlist boundary
* another judge's workload
* another judge's assignments
* aggregate win rates
* uncertainty
* which projects the system considers "important"

The judge sees only the comparison required for the current task.

This prevents feedback effects.

---

# 44. Judge Authentication

A Phase 1 judging request is valid only when:

```text
authenticated judge
+
active competition
+
active assignment
+
valid submission pair
```

The backend must verify all four.

The client cannot choose:

```text
judge_id
```

or:

```text
submission_id
```

and expect the server to trust it.

The server determines what the judge is allowed to judge.

---

# 45. Security Rules

A judge must not be able to:

* score an unassigned comparison
* modify another judge's decision
* submit a fake judge ID
* see another judge's results
* see rankings
* inspect hidden submissions
* alter comparison IDs
* replay a completed comparison improperly
* submit duplicate decisions
* access organizer-only metadata

All sensitive operations must be authorized server-side.

---

# 46. Comparison Submission

When a judge selects A or B, the backend validates:

```text
judge authenticated
competition active
comparison exists
comparison assigned to judge
comparison not already completed
both submissions valid
submission versions correct
```

Then the result is stored transactionally.

Example:

```text
comparison_id
competition_id
assignment_version
judge_id
submission_a
submission_b
winner
timestamp
algorithm_version
```

---

# 47. Judge Edits

Before the judging deadline, the system may allow an organizer-defined correction workflow.

However, ordinary judges should not be able to silently modify historical decisions.

If a correction is allowed:

```text
old decision retained
new decision recorded
actor recorded
timestamp recorded
reason recorded
```

The audit trail must remain reconstructable.

---

# 48. Comparison Identity

Every comparison receives a unique identifier.

For example:

```text
comparison_id = UUID
```

The system must distinguish:

```text
comparison assignment
```

from:

```text
comparison result
```

This makes the system robust against retries and network failures.

---

# 49. Idempotency

If the same request reaches the backend twice because of a network retry, the system must not accidentally create two judging decisions.

The API should use an idempotency mechanism.

Conceptually:

```text
same judge
+
same comparison
+
same judging action
```

must not create duplicate logical results.

---

# 50. Workload Monitoring

The organizer should be able to see aggregate judging progress.

For example:

```text
Total comparisons: 15,000
Completed: 9,200
Remaining: 5,800
```

and:

```text
Judge 1: 460 / 750
Judge 2: 461 / 750
...
```

However, judges themselves must not see other judges' workload.

---

# 51. Judge Addition

If the organizer determines that judging capacity is insufficient, the system should support adding judges.

The system then recalculates remaining workload.

It must not silently invalidate completed decisions.

Already completed evidence remains valid.

Only remaining assignments are redistributed according to the current assignment version.

Any reassignment creates a new version.

---

# 52. Versioning

The following must be versioned:

```text
competition_version
assignment_version
algorithm_version
budget_version
anonymization_version
model_version
```

If the adaptive algorithm changes, the algorithm version changes.

If the budget policy changes, the budget version changes.

This is necessary for reproducibility.

---

# 53. Determinism

Given identical:

```text
competition
submission set
judge set
N
J
K
configuration
algorithm_version
seed
```

the system must produce the same:

```text
comparison budget
initial assignments
bootstrap comparisons
adaptive tie-breaks
final ordering
```

unless external judging outcomes differ.

---

# 54. Deterministic Tie-Breaking

Whenever multiple candidates have exactly equal priority, use a deterministic key such as:

```text
hash(seed + submission_id)
```

or:

```text
hash(seed + canonical_pair_id)
```

The system must never depend on:

* unordered database results
* hash-map iteration order
* machine-dependent randomness
* unseeded random functions

---

# 55. Numerical Stability

The Bradley–Terry implementation must use numerically stable calculations.

Avoid directly evaluating:

```text
exp(theta)
```

when values can become very large.

Use a numerically stable logistic/log-sum-exp implementation.

The optimizer must have:

* convergence criteria
* maximum iterations
* failure detection
* deterministic initialization

If model fitting fails, the system must fail safely rather than silently producing a ranking.

---

# 56. Bradley–Terry Failure Handling

If the model cannot be fitted:

```text
MODEL_FIT_FAILURE
```

must be recorded.

Possible causes include:

* invalid comparison data
* disconnected graph
* numerical failure
* corrupted outcomes
* insufficient evidence

The system must not fabricate scores.

---

# 57. Comparison Graph Validation

Before finalization, validate:

```text
all submissions represented
all comparisons valid
no self-pairs
no invalid submissions
no unauthorized judge decisions
graph connected
```

The system must calculate connected components.

Required result:

```text
connected_components = 1
```

---

# 58. Workload Validation

Before judging starts:

```text
total assignments = B
```

and:

```text
sum(judge assignments) = B
```

Every judge must have a valid assignment count.

The difference between the highest and lowest workload should be at most one whenever mathematically possible.

---

# 59. Adaptive Assignment Validation

Every adaptive comparison must pass:

```text
submission A valid
submission B valid
A != B
judge capacity available
pair permitted
comparison not duplicated unless repetition justified
graph constraints satisfied
budget not exceeded
```

No adaptive action may bypass the hard constraints.

---

# 60. No Silent Budget Expansion

The system must never silently increase the judging budget because the adaptive algorithm is struggling.

If the budget is exhausted:

```text
budget_exhausted = true
```

and the system finalizes according to the defined policy.

If more judging is required, that must be an explicit system state or organizer action.

---

# 61. No Silent Budget Reduction

The system must also not quietly reduce evidence requirements simply because the judging process is inconvenient.

If the configured minimum evidence cannot be achieved:

```text
PHASE1_EVIDENCE_INFEASIBLE
```

must be raised.

The organizer can then add judges or modify the competition configuration before judging begins.

---

# 62. Failure-Closed Design

The system must prefer:

```text
explicit failure
```

over:

```text
silent degradation
```

Examples:

If:

```text
J = 0
```

fail.

If:

```text
K > N
```

fail.

If the comparison graph cannot be connected within the permitted budget:

```text
fail
```

If the required evidence cannot be delivered:

```text
fail
```

Do not silently continue with a fundamentally different judging process.

---

# 63. Configuration Validation

At competition setup:

```text
N > 0
J > 0
K > 0
K <= N
```

must hold.

The system must also verify that the available judges can handle the calculated workload.

If not:

```text
INSUFFICIENT_JUDGE_CAPACITY
```

---

# 64. Competition Lifecycle

Recommended lifecycle:

```text
DRAFT
   |
   v
VALIDATING
   |
   v
READY
   |
   v
JUDGING
   |
   v
ANALYZING
   |
   v
FINALIZED
```

Failure states:

```text
FAILED_VALIDATION
FAILED_ASSIGNMENT
FAILED_MODEL
FAILED_FINALIZATION
```

A failed run must never overwrite the last valid finalized result.

---

# 65. Audit Log

The system must record:

```text
actor
action
target
timestamp
reason
before
after
competition_version
algorithm_version
```

Important audit events include:

* judge added
* judge removed
* assignment generated
* assignment regenerated
* comparison submitted
* comparison corrected
* algorithm version changed
* budget generated
* model fitted
* model refitted
* shortlist finalized

---

# 66. Reconstructability

An organizer must be able to reconstruct:

```text
Submission
   ↓
Phase 1 comparison assignments
   ↓
Judge decisions
   ↓
Comparison graph
   ↓
Bradley–Terry model
   ↓
Uncertainty/evidence
   ↓
Adaptive decisions
   ↓
Final shortlist
```

This is a core correctness requirement.

---

# 67. Finalization Record

When Phase 1 is finalized, store:

```text
finalization_id
competition_id
N
J
K
budget_version
assignment_version
algorithm_version
model_version
seed
finalized_at
finalized_by
final shortlist
model estimates
uncertainty information
boundary status
```

A snapshot hash should also be stored.

---

# 68. Final Output

The Phase 1 output should contain:

```text
shortlist = exactly K submissions
```

plus internal audit information.

For organizers, useful information includes:

```text
submission_id
final_model_estimate
evidence_count
uncertainty
final_position
boundary_status
```

Judges do not need access to this information.

---

# 69. Important Separation of Views

### Judge view

Only:

```text
current comparison
submission A
submission B
A/B selection
```

### Organizer view

Can additionally access:

```text
progress
aggregate statistics
assignment health
model diagnostics
uncertainty
boundary status
audit information
```

### Engineering/admin view

Can additionally access:

```text
algorithm version
budget version
assignment version
seed
model diagnostics
failure codes
performance metrics
```

---

# 70. Performance

Phase 1 is designed for potentially thousands of submissions.

The adaptive engine must not repeatedly scan every possible pair.

For:

```text
N = 2500
```

there are:

```text
C(2500,2) = 3,123,750
```

possible unique pairs.

Scanning all pairs repeatedly would be unnecessarily expensive.

Therefore candidate generation must be restricted.

Candidate pools can be built from:

* local comparison neighborhoods
* shortlist-boundary neighborhoods
* high-uncertainty submissions
* insufficiently evidenced submissions
* strong-project protection candidates
* existing comparison graph neighborhoods
* high-information candidate pairs

---

# 71. Batch Model Updates

The system does not need to refit Bradley–Terry after every individual comparison.

Instead, comparisons can be collected in batches.

Conceptually:

```text
collect batch
    ↓
fit model
    ↓
update uncertainty
    ↓
select next batch
```

This significantly reduces computational overhead.

The batch size should be simulation- and performance-tested.

---

# 72. Expected Scale Example

For:

```text
N = 2500
J = 20
K = 20
```

the system may eventually produce a budget on the order of tens of thousands of comparisons, depending on the validated Standard-mode budget policy.

If:

```text
B = 15,000
```

then:

```text
average workload = 15,000 / 20
                 = 750 comparisons/judge
```

This example is illustrative only.

It is **not the frozen production budget**.

The production budget must come from the simulation-calibrated budget version.

---

# 73. Why We Do Not Freeze "12 Comparisons per Submission"

A fixed rule such as:

```text
12 comparisons/submission
```

is attractive because it is easy to explain.

But it has a serious weakness:

It treats:

```text
N = 100, K = 20
```

and:

```text
N = 2500, K = 20
```

as if the same evidence structure were sufficient per submission.

They are not necessarily the same statistical problem.

Likewise:

```text
K = 5
```

and:

```text
K = 500
```

create different shortlist-boundary structures.

Therefore the system must use:

```text
N + J + K
```

to determine the judging plan.

---

# 74. Why We Do Not Simply Use Random Pairing

Pure random pairing is simple, but it wastes information.

It can produce:

* insufficient evidence for boundary projects
* weak coverage around the cutoff
* unnecessary comparisons between obviously separated projects
* poor protection of uncertain strong submissions

Randomness may be useful inside controlled exploration, but it should not be the entire allocation strategy.

---

# 75. Why We Do Not Simply Compare Current Top K

This is also unsafe.

If the current ranking is wrong, the system reinforces the error.

Example:

```text
early lucky project
      ↓
moves into top K
      ↓
receives more attention
      ↓
keeps top position
```

while:

```text
early unlucky strong project
      ↓
falls down
      ↓
receives less attention
      ↓
never recovers
```

The protected adaptive system is specifically designed to avoid this feedback loop.

---

# 76. Why We Do Not Use Win Rate

Consider:

```text
Project A:
8 wins / 10 comparisons

Project B:
6 wins / 6 comparisons
```

Simple win rate gives:

```text
A = 80%
B = 100%
```

But B may have defeated very weak opponents.

A may have faced extremely strong opponents.

Win rate ignores opponent strength.

Bradley–Terry explicitly models relative strength based on the opponents faced.

---

# 77. Why We Do Not Remove "Outliers"

A judge's single decision is not automatically considered an outlier.

The system should not say:

```text
Judge X disagreed with everyone
→ delete Judge X
```

because legitimate disagreement exists.

Phase 1 is designed around noisy human judgments.

The appropriate response is:

```text
more evidence
+
statistical aggregation
```

rather than arbitrary deletion.

---

# 78. Why We Do Not Add a Tie Button

A tie/skip mechanism initially appears attractive for situations where both submissions are excellent.

But it introduces another state:

```text
A
B
Tie
Skip
```

which requires additional statistical handling.

For the Standard Phase 1 system, the simpler approach is:

> If the judge had to advance one, which would they choose?

The resulting noise is handled statistically.

This keeps the human task extremely simple.

---

# 79. Main Failure Scenario

The most important nightmare scenario is:

```text
Genuinely excellent project
        ↓
unlucky early comparisons
        ↓
temporarily low BT estimate
        ↓
adaptive system ignores it
        ↓
project never gets enough evidence
        ↓
false exclusion
```

The architecture directly addresses this through:

```text
minimum evidence
+
protected exploration
+
uncertainty
+
opponent-aware BT
+
boundary protection
+
adaptive reserve
+
simulation validation
```

This is the central defense of the system.

---

# 80. Opposite Failure Scenario

The opposite problem is:

```text
moderate project
        ↓
easy early opponents
        ↓
many wins
        ↓
high temporary estimate
        ↓
receives too much attention
```

This is mitigated by:

```text
opponent-aware Bradley–Terry modeling
+
adaptive evidence
+
comparison diversity
```

The system does not equate:

```text
number of wins
```

with:

```text
quality
```

---

# 81. Strong-vs-Strong Scenario

Suppose:

```text
A = 95th percentile
B = 96th percentile
```

and they repeatedly produce close decisions.

The system should not interpret:

```text
A loses to B
```

as evidence that A is mediocre.

Instead:

```text
A and B are both receiving evidence against strong competition.
```

If their relative ordering matters for the shortlist boundary, additional evidence can be allocated.

If both are comfortably within the advancement region, excessive additional comparisons are unnecessary.

---

# 82. Strong Project with Weak Evidence

Suppose a project has:

```text
few comparisons
strong wins
high uncertainty
```

Even if its current estimated position is not top K, the system should retain it as an adaptive candidate.

This is one of the most important differences between this architecture and a naive ranking system.

---

# 83. Boundary Scenario

Suppose:

```text
K = 20
```

and:

```text
#18  1.41
#19  1.38
#20  1.37
#21  1.36
#22  1.35
```

If uncertainty is large, the system should not pretend that:

```text
#20
```

is definitively better than:

```text
#21
```

The adaptive system should spend evidence where it can resolve that boundary.

---

# 84. If the Boundary Cannot Be Fully Resolved

The system may reach the maximum approved budget while:

```text
#20
```

and:

```text
#21
```

remain statistically close.

The correct behavior is not to manufacture certainty.

Instead:

```text
SHORTLIST_BOUNDARY_UNCERTAIN
```

is recorded.

A deterministic final selection is still made if the competition requires exactly K finalists.

The audit record makes the uncertainty visible.

---

# 85. Phase 1 Does Not Guarantee Perfect Selection

No noisy human judging system can guarantee that the mathematically strongest submissions will always be selected.

The system instead minimizes avoidable sources of error.

Its defenses are:

```text
multiple evidence
+
balanced judging
+
connected graph
+
opponent-aware modeling
+
uncertainty
+
adaptive investigation
+
strong-project protection
+
boundary protection
+
simulation
+
auditability
```

That is the correct engineering goal.

---

# 86. Test Plan

The implementation must pass at least:

### Test 1 — Configuration

Verify:

```text
N > 0
J > 0
K > 0
K <= N
```

### Test 2 — Determinism

Run the same configuration twice.

Expected:

```text
same budget
same assignments
same deterministic decisions
same hash
```

### Test 3 — Workload

Verify:

```text
max_load - min_load <= 1
```

where mathematically possible.

### Test 4 — Coverage

Verify every submission receives minimum evidence.

### Test 5 — No self-pairs

Expected:

```text
A != B
```

for every comparison.

### Test 6 — Graph connectivity

Expected:

```text
connected_components = 1
```

### Test 7 — Duplicate handling

Verify repeated pairs occur only when explicitly justified.

### Test 8 — Judge isolation

A judge must not access another judge's assignments or results.

### Test 9 — Unauthorized comparison

Attempt to submit a comparison not assigned to the judge.

Expected:

```text
403 / authorization failure
```

### Test 10 — Duplicate submission

Replay the same judging request.

Expected:

```text
idempotent behavior
```

### Test 11 — Strong unlucky project

Simulate an excellent project losing early comparisons.

Verify it receives additional protection rather than automatic elimination.

### Test 12 — Lucky moderate project

Simulate a moderate project winning weak opponents.

Verify the model does not treat raw win count as sufficient.

### Test 13 — Strong-vs-strong

Verify close high-quality projects receive appropriate additional evidence when necessary.

### Test 14 — Boundary uncertainty

Verify the adaptive system continues investigating an unstable cutoff.

### Test 15 — Budget exhaustion

Verify the system stops at the configured hard budget.

### Test 16 — Model failure

Inject invalid/disconnected data.

Expected:

```text
MODEL_FIT_FAILURE
```

rather than fabricated output.

### Test 17 — Audit reconstruction

Reconstruct the final shortlist from stored evidence.

Expected:

```text
identical result
```

### Test 18 — Simulation acceptance

Run the synthetic benchmark suite and verify the selected budget/policy satisfies the agreed false-exclusion target.

---

# 87. Simulation Acceptance Gate

The following must be true before the Standard budget is frozen:

```text
Budget version tested
Adaptive algorithm tested
N/J/K combinations tested
Judge-noise scenarios tested
False-exclusion metric measured
Boundary performance measured
Workload measured
Runtime measured
```

Only after this should the system freeze:

```text
STANDARD_BUDGET_VERSION
STANDARD_ADAPTIVE_VERSION
```

This is an intentional engineering gate.

---

# 88. Recommended Internal Data Model

A Phase 1 comparison record should contain approximately:

```text
comparison_id
competition_id
submission_a
submission_b
judge_id
winner
assignment_version
algorithm_version
created_at
submitted_at
status
```

A submission Phase 1 state should contain:

```text
submission_id
comparison_count
wins
losses
theta
uncertainty
adaptive_priority
final_position
boundary_status
```

A Phase 1 run should contain:

```text
run_id
competition_id
N
J
K
budget_version
assignment_version
algorithm_version
model_version
seed
total_budget
completed_comparisons
status
created_at
finalized_at
```

---

# 89. Phase 1 Statuses

Recommended statuses:

```text
DRAFT
VALIDATING
READY
JUDGING
ANALYZING
FINALIZED
FAILED
```

Additional failure reasons:

```text
INVALID_CONFIGURATION
INSUFFICIENT_JUDGE_CAPACITY
ASSIGNMENT_FAILURE
DISCONNECTED_GRAPH
MODEL_FIT_FAILURE
BUDGET_EXHAUSTED
EVIDENCE_INFEASIBLE
VALIDATION_FAILURE
EXECUTION_ERROR
TIMEOUT
```

---

# 90. Engineering Principle

The Phase 1 system should be thought of as:

> **an evidence-allocation system, not merely a ranking system.**

This distinction is fundamental.

A ranking system asks:

> "Who is currently first?"

The Phase 1 system asks:

> "Where should we spend the next piece of judging evidence so that we can safely decide who deserves Phase 2?"

That is why adaptive evidence allocation is necessary.

---

# 91. Final Architecture

The final Standard Phase 1 architecture is:

```text
                  ORGANIZER
                     |
                  N, J, K
                     |
                     v
              CONFIGURATION
                 VALIDATION
                     |
                     v
               BUDGET ENGINE
                     |
          +----------+----------+
          |                     |
          v                     v
   MINIMUM EVIDENCE       JUDGE CAPACITY
          |                     |
          +----------+----------+
                     |
                     v
             DETERMINISTIC
               BOOTSTRAP
                     |
                     v
            BRADLEY–TERRY MODEL
                     |
                     v
          UNCERTAINTY / EVIDENCE
                     |
                     v
        PROTECTED ADAPTIVE ENGINE
             /             \
            /               \
           v                 v
  STRONG-PROJECT         BOUNDARY
    PROTECTION          PROTECTION
           \               /
            \             /
             v           v
             INFORMATION
                VALUE
                  |
                  v
             MORE JUDGING
                  |
                  v
           UPDATED MODEL
                  |
                  v
        SHORTLIST STABILITY
             /         \
            NO          YES
            |            |
            v            v
       MORE EVIDENCE   FINALIZE
                         |
                         v
                    EXACT K
                    SUBMISSIONS
                         |
                         v
                      PHASE 2
```

---

# 92. Final Design Rules

The production implementation must preserve these rules:

1. Phase 1 is triage, not final scoring.
2. The organizer specifies N, J and K.
3. There is one Standard judging mode.
4. Judges perform simple binary A/B decisions.
5. No tie/skip mechanism is required.
6. Every submission receives meaningful minimum evidence.
7. The exact comparison budget is simulation-calibrated.
8. The budget depends on N, J and K.
9. Judge workload is automatically balanced.
10. Initial comparisons are not driven by early ranking.
11. The comparison graph must be connected.
12. Bradley–Terry is the primary relative-strength model.
13. Raw win rate is not the ranking mechanism.
14. Early BT ranking is not an elimination mechanism.
15. Current rank alone can never stop investigation.
16. Adaptive judging protects both the shortlist boundary and potentially strong submissions.
17. Uncertainty matters alongside estimated strength.
18. Repeated pairs are allowed when additional evidence is justified.
19. Information gain is useful but is not allowed to override strong-project protection.
20. Judges cannot see other judges' work or system rankings.
21. Assignment and results are deterministic and auditable.
22. No silent budget relaxation occurs.
23. No silent evidence reduction occurs.
24. Model failures fail closed.
25. The final shortlist contains exactly K submissions.
26. Boundary uncertainty is explicitly recorded.
27. All decisions are reconstructable from the audit trail.
28. The Standard budget and adaptive policy are frozen only after simulation validation.

---

# 93. Phase 1 → Phase 2 Boundary

Phase 1 produces:

```text
exactly K shortlisted submissions
```

Phase 2 then takes over.

Phase 1 does NOT produce:

```text
final competition scores
```

Phase 2 performs the detailed rubric-based evaluation, multiple independent judges, calibration, normalization, disagreement analysis, and final scoring.

Therefore the two systems have clearly separated responsibilities:

```text
PHASE 1
Fast
Pairwise
Relative
Adaptive
Evidence-protective
Shortlisting

        ↓

PHASE 2
Detailed
Rubric-based
Multi-judge
Calibrated
Normalized
Auditable
Final scoring
```

---

# 94. Final Defense of the Architecture

The strongest reason for this architecture is that it directly attacks the most dangerous failure mode:

> **A genuinely strong submission gets unlucky early and disappears before receiving enough evidence.**

A simple ranking system cannot reliably protect against that.

A simple random system cannot efficiently resolve the shortlist boundary.

A simple win-rate system ignores opponent strength.

A simple "top K after bootstrap" system locks in early noise.

A fixed comparison count ignores the fact that `N`, `J`, and `K` change the statistical problem.

The proposed system instead combines:

```text
baseline coverage
+
connected comparison graph
+
Bradley–Terry opponent-aware modeling
+
uncertainty
+
protected exploration
+
boundary-focused investigation
+
balanced judge workload
+
simulation-calibrated budget
+
deterministic execution
+
full auditability
```

The most important architectural decision is therefore not a particular magic number of comparisons.

It is this:

> **Phase 1 does not trust an early ranking enough to eliminate a potentially strong submission without sufficient evidence.**

That principle is what makes the system robust against noisy human pairwise judging.

The numerical budget and exact adaptive thresholds should then be selected empirically through the simulation gate, rather than being justified merely because they "look reasonable." is this algorithm valid

---

# PART II — PHASE 2

> **Scope:** This section defines the Phase 2 judging system for **Type 1 (No Tracks)** after Phase 1 shortlisting.
>
> Phase 1 is defined earlier in this same document. Phase 2 begins only after Phase 1 has produced the exact organizer-defined shortlist.
>
> This section is the implementation specification for Phase 2 assignment, judging, calibration, normalization, final scoring, security, auditability, validation, and failure handling.

# 1. Purpose

Phase 2 takes a shortlisted set of submissions and produces final scores using multiple independent judges.

The system must:

1. Assign every submission to exactly `R` distinct judges.

2. Balance judge workloads exactly.

3. Deliberately create overlap between judges so their scoring tendencies can be estimated.

4. Prevent judges from seeing other judges' work or scores.

5. Collect structured rubric scores.

6. Calibrate judge scoring tendencies using shared submissions.

7. Normalize scores without destroying raw evidence.

8. Calculate final submission scores.

9. Report disagreement without automatically penalizing it.

10. Preserve enough immutable data to reconstruct every final result.

11. Fail closed when hard constraints cannot be satisfied.

12. Produce deterministic assignments from identical inputs.

The core principle is:

> **Assignment is not only about distributing workload. It must deliberately create enough judge overlap to make cross-judge calibration possible.**

---

# 2. Type 1 Definition

Type 1 has:

* `N` submissions

* `J` judges

* `R` required independent reviews per submission

* no track restrictions

Every active judge is eligible to review every submission unless an explicit eligibility rule excludes them.

Hard assignment requirements:

* `R <= J`

* every submission receives exactly `R` judges

* no submission receives the same judge twice

* every judge receives their exact calculated workload quota

* no unknown or inactive judge is assigned

* assignment graph must be connected when global calibration is required

---

**# 3. Phase 2 Pipeline**

The Phase 2 pipeline is:

```text

Shortlisted submissions

        |

        v

Validate configuration

        |

        v

Calculate exact judge quotas

        |

        v

Generate deterministic assignments

        |

        v

Optimize judge overlap

        |

        v

Validate assignment

        |

        v

Judge window opens

        |

        v

Judges submit rubric scores

        |

        v

Lock valid reviews

        |

        v

Build valid overlap graph

        |

        v

Calibrate judge offsets

        |

        v

Normalize judge scores

        |

        v

Calculate final submission scores

        |

        v

Calculate disagreement metrics

        |

        v

Create immutable finalization snapshot

```

---

# 4. Assignment Strategy

## 4.1 Assignment Inputs

The assignment engine receives:

```text

N

J

R

submission_ids

judge_ids

judge eligibility

assignment_version

algorithm_version

seed

```

For Type 1 there are no track restrictions.

---



### Feasibility Invariants

For standard Phase 2 judging:

```text
1 <= R <= J_active
K >= 1
```

A submission cannot be assigned the same judge twice.

If:

```text
R > J_active
```

the assignment must fail closed with:

```text
FAILED_INSUFFICIENT_ACTIVE_JUDGES
```

rather than creating duplicate judge assignments.
# 5. Exact Workload Balancing

Total assignments:

```text

A = N * R

```

Calculate:

```text

baseQuota = floor(A / J)

extra = A mod J

```

Exactly `extra` judges receive:

```text

baseQuota + 1

```

All other judges receive:

```text

baseQuota

```

Therefore:

```text

max(judge_load) - min(judge_load) <= 1

```

This is an exact workload constraint.

There is no organizer-configurable workload tolerance.

### Example

For:

```text

N = 2500

J = 20

R = 4

```

Total assignments:

```text

A = 2500 * 4

A = 10000

```

Therefore:

```text

baseQuota = floor(10000 / 20)

baseQuota = 500

extra = 10000 mod 20

extra = 0

```

Every judge receives exactly:

```text

500 submissions

```

---

# 6. Remaining Quota

During assignment generation:

```text

remainingQuota[j] =

    assignedQuota[j] - currentLoad[j]

```

A candidate assignment may never exceed a judge's remaining quota.

The assignment engine must also maintain a future-feasibility guard.

If:

```text

M = number of submissions still unassigned

```

then:

```text

effectiveCapacity[j] =

    min(remainingQuota[j], M)

```

A candidate state is feasible only if:

```text

sum(effectiveCapacity[j]) >= M * R

```

This prevents the algorithm from creating a state where the remaining assignments cannot possibly be completed.

---

# 7. Judge Overlap

Overlap is intentionally created because calibration requires judges to review some of the same submissions.

For judges `i` and `j`:

```text

O_ij =

number of submissions reviewed by both judge i and judge j

```

Every submission assigned to `R` judges creates:

```text

C(R,2)

```

pair-overlap events.

Therefore:

```text

sum(O_ij for all i < j)

=

N * C(R,2)

```

This is an assignment invariant.

### Example

For:

```text

N = 2500

R = 4

```

each submission creates:

```text

C(4,2) = 6

```

pair-overlap events.

Therefore:

```text

2500 * 6 = 15000

```

Total pair-overlap events:

```text

15000

```

For:

```text

J = 20

```

there are:

```text

C(20,2) = 190

```

possible judge pairs.

The theoretical average overlap is therefore:

```text

15000 / 190

≈ 78.95

```

This is a target for balance, not a requirement that every pair have exactly the same overlap.

---

# 8. Pair-Overlap Objective

Define:

```text

P =

N * C(R,2) / C(J,2)

```

Then:

```text

PairLoss =

sum over all judge pairs (O_ij - P)^2

```

Lower `PairLoss` means more balanced overlap.

The objective is not to maximize overlap for a few judge pairs.

The objective is to distribute overlap broadly enough that the judge graph is connected and pair evidence is reasonably balanced.

---

# 9. Candidate Assignment Generation

For each submission, the engine must construct exactly `R` judges without
enumerating all:

```text
C(J,R)
```

judge groups.

The production constructor is an incremental deterministic greedy algorithm.

For submission `s`:

1. Build the feasible judge set.
2. Select the first judge that minimizes the marginal objective.
3. Add that judge to the group.
4. Recompute the marginal objective for the remaining judges.
5. Repeat until exactly `R` judges are selected.
6. Validate the completed group against all hard constraints.
7. If the completed group is infeasible, invoke assignment repair.

The marginal objective is evaluated using the same priorities as the full
assignment objective:

```text
1. hard feasibility
2. connectivity contribution
3. marginal PairLoss
4. quota spread
5. canonical judge ID
```

For a candidate judge `j`, adding `j` to a partial group creates one new overlap
event with every already-selected judge `k`:

```text
DeltaPairLoss(j) =
sum over k in partial_group
[
    (O_jk + 1 - P)^2
    -
    (O_jk - P)^2
]
```

where `P` is the target pair-overlap level.

The group constructor selects the feasible judge with the smallest
lexicographic tuple:

```text
(
    connectivity_penalty,
    DeltaPairLoss,
    quota_penalty,
    canonical_judge_id
)
```

Connectivity is evaluated as a hard feasibility condition whenever a connected
overlap graph is still required and the remaining assignment capacity can
achieve it.

This reduces candidate generation from combinatorial enumeration to
approximately:

```text
O(K * R * J)
```

marginal evaluations before repair.

The system must record:

```text
candidate_generation_algorithm_version
candidate_count_considered
candidate_group
repair_count
```

The implementation must never silently fall back to full `C(J,R)` enumeration
for large judge pools.

# 10. Deterministic Submission Ordering

Submissions must not be processed according to arbitrary database ordering.

Use:

```text

sortKey =

hash(

    assignment_version

    + submission_id

)

```

Then sort by:

```text

(sortKey, submission_id)

```

This creates a stable deterministic order.

---

# 11. Candidate Group Validation

For each submission, candidate judge groups are evaluated.

A candidate group is rejected if:

1. it contains a duplicate judge

2. a judge is inactive

3. a judge is ineligible

4. a judge exceeds their quota

5. future assignment feasibility fails

6. another hard assignment constraint fails

No candidate that violates a hard constraint may be selected.

---

# 12. Candidate Group Objective

Among valid candidates, compare groups in this deterministic order:

### Priority 1 — Pair overlap balance

Choose the candidate with the smallest:

```text

PairLossChange

```

For an affected pair whose current overlap is `O`:

```text

(O + 1 - P)^2 - (O - P)^2

=

2 * (O - P) + 1

```

The total candidate delta is the sum of the affected pair deltas.

---

### Priority 2 — Connectivity

If pair-loss objectives are tied or connectivity is still being established, prefer the candidate that provides greater judge-graph connectivity.

The assignment must deliberately build a connected overlap graph.

---

### Priority 3 — Quota spread

Prefer the candidate with the smallest workload spread among the judges in that group.

---

### Priority 4 — Deterministic lexical order

If candidates remain tied:

```text

lexicographically smallest canonical judge group

```

is selected.

This guarantees deterministic behavior.

---

# 13. Connectivity

Construct a judge graph:

```text

node = judge

edge(i,j) exists if O_ij > 0

```

The graph must be connected when global calibration is required.

A connected graph means that scoring relationships can propagate across the judge population.

Example:

```text

Judge 1 <-> Judge 2

Judge 2 <-> Judge 3

Judge 3 <-> Judge 4

```

is connected even if Judge 1 and Judge 4 never directly reviewed the same submission.

This is sufficient for global offset calibration because the system estimates offsets through the connected overlap structure.

---

# 14. Bootstrap Connectivity

At the beginning of assignment generation, the overlap graph may be disconnected because no overlap exists yet.

Early assignments should therefore prioritize creating useful connectivity.

Once the graph is connected, pair-overlap balance becomes the primary optimization objective.

The implementation should prefer redundant connections once connectivity exists rather than creating unnecessary fragile bridge structures.

---

# 15. Assignment Repair

After initial assignment generation, run deterministic local repair if necessary.

The primary repair operation is a judge swap between two submissions.

Example:

```text

Submission A:

J1 J2 J3 J4

Submission B:

J1 J5 J6 J7

```

Swap:

```text

J4 <-> J7

```

Result:

```text

Submission A:

J1 J2 J3 J7

Submission B:

J1 J5 J6 J4

```

This preserves:

* number of reviews per submission

* judge workload totals

* assignment count

provided no duplicate judge is created.

---

# 16. Repair Constraints

A repair swap is rejected if:

* it creates a duplicate judge on a submission

* it changes the exact workload quotas

* it assigns an ineligible judge

* it disconnects the judge graph when connectivity is required

* it violates another hard constraint

Hard constraints always have priority over optimization.

---

# 17. Repair Objective

The repair objective is:

```text

1. Hard constraints

2. Required connectivity

3. PairLoss

4. Deterministic tie-break

```

Normal repair accepts only an improving move.

If connectivity is deficient, a move that improves connectivity may be accepted even if:

```text

DeltaPairLoss = 0

```

A move that worsens required connectivity must never be accepted.

Do not use:

* random perturbations

* relaxed quotas

* temporary hard-constraint violations

* intentionally worse moves

---

# 18. Repair Search

Repair must not literally scan every possible submission pair and every judge swap on every pass.

That approach can become unnecessarily expensive.

Instead, repair candidates should be restricted to structurally relevant submissions, such as:

* submissions sharing judges

* submissions contributing to highly imbalanced judge-pair overlaps

* submissions involved in disconnected or weak graph regions

* submissions affected by the most useful potential swap

Within the candidate set:

1. evaluate all valid candidate swaps
2. assign each swap a canonical tuple:

```text
(
    hard_constraint_status,
    connectivity_penalty,
    DeltaPairLoss,
    affected_submission_count,
    canonical_removed_judge_id,
    canonical_added_judge_id,
    canonical_submission_id
)
```

3. select the lexicographically smallest tuple among valid improving swaps
4. apply it
5. restart the pass
6. repeat

`DeltaPairLoss` is minimized first after hard constraints and connectivity.
All remaining ties are resolved by the canonical IDs above.

Do not use first-improvement.

---

# 19. Repair Safety Ceiling

Use:

```text

maxRepairMoves =

max(1000, 10 * N)

```

For:

```text

N = 2500

```

this gives:

```text

maxRepairMoves = 25000

```

Stop repair when:

* no improving valid swap exists

* no valid swap exists

* required connectivity has been achieved and no improvement remains

* safety ceiling is reached

If the ceiling is reached but all hard constraints pass:

```text

SUCCESS_WITH_REPAIR_LIMIT

```

If hard constraints fail:

```text

FAILED_VALIDATION

```

---

# 20. Assignment Determinism

Identical inputs must produce identical assignments.

The deterministic inputs are:

```text

N

J

R

submission IDs

judge IDs

eligibility

assignment_version

algorithm_version

seed

```

Recommended deterministic seed:

```text

seed =

hash(

    competition_id

    + assignment_version

    + algorithm_version

)

```

The seed may be used for deterministic permutations, such as selecting which judges receive the extra quota when:

```text

A mod J != 0

```

The system must never depend on:

* database row order

* network response order

* unordered map iteration

* nondeterministic parallel winner selection

* current time

* random unseeded operations

Judge IDs and candidate groups must use canonical ordering.

---

# 21. Assignment Fingerprint

After assignment generation:

1. canonicalize all assignments

2. sort them deterministically

3. serialize them

4. calculate SHA-256

Store:

```text

assignment_hash

```

Also store:

```text

assignment_version

algorithm_version

seed

created_at

N

J

R

judge quotas

```

The assignment can therefore be independently verified later.

---

# 22. Assignment Validation

Before assignments become usable, validate all of the following:

```text

Every submission has exactly R judges

Every submission has R distinct judges

Every judge receives exactly their calculated quota

Total assignments = N * R

No unknown judge

No inactive judge

No duplicate assignment

No missing submission

Pair-overlap invariant holds

Judge graph is connected when required

Assignment fingerprint is valid

```

The overlap invariant must satisfy:

```text

sum(O_ij)

=

N * C(R,2)

```

Failure of a hard invariant means the assignment is not usable.

---

# 23. Assignment Lifecycle

Use:

```text

DRAFT

   |

   v

GENERATING

   |

   v

VALIDATING

   |

   v

READY

```

Failure at any stage:

```text

FAILED

```

Only:

```text

READY

```

assignments may be used for judging.

A failed new assignment generation must never overwrite a previously valid `READY` assignment.

---

# 24. Assignment Failure Behavior

The system must fail closed.

Never silently relax:

* required review count

* distinct-judge requirement

* exact workload quotas

* required connectivity

If no valid candidate group exists for a submission, stop generation.

The failure report should contain:

```text

submission_id

remaining submissions

remaining assignments

current judge loads

remaining quotas

candidate rejection reasons

assignment_version

algorithm_version

seed

```

Failure reason:

```text

NO_VALID_CANDIDATE

```

---

# 25. Review Requirements

Each assigned judge must submit a score for every active rubric criterion.

Criteria have:

```text

minimum score

maximum score

weight

```

Weights must satisfy:

```text

sum(weights) = 1

```

Scores are numeric integers at input time.

For example:

```text

0–10

```

The backend must validate all ranges.

Missing does not mean zero.

An incomplete review is not treated as a zero-score review.

---

# 26. Rubric Versioning

Every review references a:

```text

rubric_version

```

The rubric becomes locked once judging begins.

The system must not mutate the rubric underneath existing reviews.

If the rubric must change:

```text

create a new rubric version

```

rather than modifying the old version.

---

# 27. Raw Score Storage

Store raw criterion-level scores separately from calculated values.

At minimum store:

```text

submission_id

judge_id

review_version

rubric_version

criterion_id

criterion_max

criterion_weight

raw_score

```

Also store:

```text

raw_weighted_score

```

Raw judge input must never be silently overwritten.

---

# 28. Review Lifecycle

Normal lifecycle:

```text

DRAFT

  |

  v

SUBMITTED

  |

  v

LOCKED

```

Deadline behavior:

```text

DRAFT -> EXPIRED

SUBMITTED -> LOCKED

```

Organizer invalidation:

```text

SUBMITTED -> INVALIDATED

LOCKED -> INVALIDATED

```

A judge cannot invalidate their own review.

---

# 29. Review Versioning

If a judge edits a review before the deadline:

```text

create a new review version

```

Do not silently mutate the old version.

The system must retain the audit history.

Only the latest valid review version participates in final scoring.

Invalidated or expired versions do not contribute to scoring.

---

# 30. Transactional Review Submission

Review submission must be transactional.

Before accepting a review, the backend verifies:

```text

authenticated user is the assigned judge

assignment is active

judge is eligible

submission is assigned to judge

rubric version is correct

all active criteria are present

all scores are within range

no unexpected criteria are present

review is not locked

```

If any validation fails:

```text

reject entire submission

```

Do not partially save an invalid review.

---

# 31. Raw Weighted Score

For criterion `c`:

```text

q(s,j,c) = score(s,j,c) / max_c

```

The normalized criterion value is therefore between:

```text

0 and 1

```

The raw weighted review score is:

```text

r(s,j)

=

100 * sum(

    weight_c * q(s,j,c)

)

```

Equivalent:

```text

r(s,j)

=

100 * sum(

    weight_c * score(s,j,c) / max_c

)

```

Therefore:

```text

0 <= r(s,j) <= 100

```

---

# 32. Example Raw Score

Suppose four criteria have:

```text

Criterion 1:

score = 8

max = 10

weight = 0.40

Criterion 2:

score = 9

max = 10

weight = 0.30

Criterion 3:

score = 7

max = 10

weight = 0.20

Criterion 4:

score = 8

max = 10

weight = 0.10

```

Then:

```text

r

=

100 * (

    0.40*(8/10)

  + 0.30*(9/10)

  + 0.20*(7/10)

  + 0.10*(8/10)

)

r = 81

```

The raw review score is:

```text

81

```

---

# 33. Valid Review Definition

A review is valid for final scoring only if:

* it belongs to the assigned judge

* it belongs to the assigned submission

* it uses the correct rubric version

* all required criteria are valid

* it passed backend validation

* the latest version is not invalidated

* it is locked/finalized for judging

Normal finalization requires:

```text

valid_reviews(submission) = R

```

---

# 34. Incomplete Judging

If:

```text

valid_reviews < R

```

normal finalization is blocked.

Status:

```text

INCOMPLETE_JUDGING

```

The system should attempt to replace the missing judge.

A replacement judge must:

* be eligible

* not already be assigned to the submission

* respect exact workload quotas

* preserve assignment validity

* preserve connectivity as much as possible

* preserve overlap balance as much as possible

If no valid replacement can be made in time, normal finalization remains blocked.

---

# 35. Reduced Review Exception

Reduced-review finalization is not the normal path.

If organizers explicitly authorize it, the final result must carry:

```text

REDUCED_REVIEW_COUNT

```

and record:

```text

actual review count

required review count

reason

organizer

timestamp

```

It must never appear indistinguishable from a normal fully reviewed submission.

---

# 36. Judge Removal

Completed valid reviews are immutable assignments. A judge removal must never
retroactively rewrite completed assignment history.

If a judge becomes unavailable or is removed:

1. freeze all completed valid reviews from that judge
2. identify every submission with a missing future review slot
3. mark those slots `REPLACEMENT_REQUIRED`
4. remove the judge from the active eligibility pool
5. calculate:

```text
remaining_slots = total_required_reviews - valid_completed_reviews
```

6. distribute the remaining slots across active eligible judges using quotas:

```text
q_min = floor(remaining_slots / active_judges)
q_max = ceil(remaining_slots / active_judges)
```

subject to submission-specific eligibility and no-duplicate constraints
7. assign replacement reviews using the Phase 2 deterministic greedy constructor
8. repair connectivity if calibration requires it
9. freeze the replacement assignment snapshot
10. recompute overlap and calibration using the new valid-review set

The original assignment snapshot remains immutable and is retained in the audit
record. The replacement snapshot references it and records only the changed
outstanding slots.

A replacement judge may not receive a submission they already reviewed.

If the remaining active judge pool cannot satisfy all missing review slots while
preserving hard constraints:

```text
FAILED_REPLACEMENT_CAPACITY
```

is emitted.

The system may enter `REDUCED_REVIEW_COUNT` only when the organizer explicitly
authorizes the reduced-review exception defined above.

Quota balancing applies to the **remaining replacement work**, not to completed
historical work. This avoids retroactively changing immutable assignments.

---

# 37. Calibration Philosophy

Judges may have different scoring tendencies.

One judge may consistently give higher scores.

Another may consistently give lower scores.

The system therefore needs to estimate relative judge offsets.

The Type 1 baseline uses:

> **Overlap-based additive judge-offset calibration.**

It does not use z-score normalization.

It does not assume judges receive identical difficulty distributions.

---

# 38. Why No Independent Z-Score Baseline

A judge's score distribution depends partly on the submissions they receive.

Because judges may receive different mixtures of difficult and easy submissions, independently standardizing each judge's entire score distribution can remove legitimate information.

Therefore Type 1 uses shared submissions to estimate relative scoring offsets.

The calibration model asks:

> When two judges evaluate the same submissions, how systematically different are their scores?

---

# 39. Pairwise Calibration Evidence

For every valid shared submission `s` reviewed by judges `i` and `j`:

```text

d_ij,s =

r(s,i) - r(s,j)

```

Calculate the mean difference:

```text

bar_d_ij =

average(

    d_ij,s

)

```

over all valid shared submissions.

Let:

```text

omega_ij =

number of valid shared submissions

```

---

# 40. Global Judge Offset Model

Let:

```text
b_j
```

be the estimated scoring offset for judge `j`.

For judge pair `(i,j)`, let:

```text
omega_ij = number of valid shared submissions
```

and:

```text
bar_d_ij =
(1 / omega_ij)
*
sum over shared submissions s of
(r(s,i) - r(s,j))
```

Estimate offsets by minimizing:

```text
L_cal =
sum over judge pairs i<j
[
    omega_ij *
    ((b_i - b_j) - bar_d_ij)^2
]
```

subject to:

```text
sum_j b_j = 0
```

The zero-sum constraint fixes the arbitrary global origin.

Only relative differences between judges are identifiable.

## 40.1 Why the `omega_ij` weight is correct

Under the baseline additive-noise model with independent, approximately
homoscedastic review noise:

```text
Var(bar_d_ij) = sigma^2 / omega_ij
```

Therefore the inverse-variance weight is proportional to:

```text
1 / Var(bar_d_ij) = omega_ij / sigma^2
```

Since the common factor `1 / sigma^2` does not affect the minimizer, weighting
the squared pair residual by `omega_ij` is the correct inverse-variance WLS
form under this model.

The implementation must **not** replace `omega_ij` with `sqrt(omega_ij)` merely
to address an alleged weighting error.

This does not mean the additive model is universally correct. Heavy-tailed noise,
judge-specific variance, collusion, or scale differences can still make the
baseline model imperfect. Those are handled through diagnostics and explicitly
versioned extensions rather than by silently changing the estimator.

# 41. Interpretation of Calibration Offset

If:

```text

b_j > 0

```

the model estimates that judge `j` tends to score higher relative to the judge population.

If:

```text

b_j < 0

```

the model estimates that judge `j` tends to score lower relative to the judge population.

This is a calibration parameter, not a judgment of whether the judge is good or bad.

---

# 42. Contradictory Pair Evidence

Pairwise evidence will not always agree.

Example:

```text

Judge A vs Judge B:

submission 1 -> A higher

submission 2 -> B higher

submission 3 -> A higher

submission 4 -> B higher

```

Do not force the pair difference to be exact.

The global least-squares model finds the best overall set of offsets.

Residual error is retained as evidence about inconsistency.

---

# 43. Overlap Evidence Levels

For a judge pair:

```text

k = number of valid shared submissions

```

Diagnostic labels:

```text

k < 3     -> LOW_EVIDENCE

3 <= k < 10 -> LIMITED_EVIDENCE

k >= 10   -> ADEQUATE_EVIDENCE

```

These labels are diagnostics only.

The system must not automatically discard pair evidence solely because:

```text

k < 3

```

A pair with fewer observations simply has less empirical evidence.

Judge-level evidence should be considered when assessing calibration stability.

---

# 44. Disconnected Calibration Graph

If there is no overlap path connecting the judges, relative calibration across components cannot be reliably estimated.

If global calibration is required:

```text

disconnected graph

=

FAILED_CALIBRATION_CONNECTIVITY

```

Do not invent arbitrary offsets to connect disconnected groups.

This is one reason assignment deliberately creates a connected overlap graph.

---

# 45. Calibration Residuals

For each observed judge pair, retain the residual:

```text

residual_ij =

(b_i - b_j) - bar_d_ij

```

Residuals should be retained for audit and diagnostics.

High residuals do not automatically mean a judge is invalid.

They indicate that simple additive offsets do not perfectly explain all shared-review differences.

---

# 46. Large Judge Offset

A large estimated offset should:

1. be applied by the calibration model

2. be retained in the audit record

3. trigger an optional organizer audit flag

It must not automatically invalidate the judge.

A judge may legitimately evaluate a difficult or unusual set of submissions.


# 46.1 Robustness and Adversarial Diagnostics

The baseline estimator is deterministic weighted least squares. It is not a
robust estimator against coordinated collusion or arbitrary outliers.

The system must therefore retain:

```text
pair residuals
judge score variance
judge score range
shared-review counts
```

and expose configurable audit flags for:

```text
EXTREME_OFFSET
EXTREME_RESIDUAL
LOW_VARIANCE
COMPRESSED_RANGE
HIGH_INFLUENCE_OVERLAP
```

These flags do not automatically alter scores.

A future robust calibration mode may use a versioned Huber or other robust loss,
but that mode must be separately specified and simulation-validated before use.
The production baseline must not silently switch estimators during finalization.
---

# 47. Narrow Score Range

A narrow scoring range is not automatically evidence of bias.

Do not invalidate a judge simply because their scores have low spread.

The baseline calibration model is about relative offsets between judges on shared submissions.


# 47.1 Scale / Range Compression Diagnostic

The baseline Type 1 calibration model corrects additive location differences:

```text
r(s,j) = quality_s + b_j + error
```

It does not estimate a judge-specific multiplicative scale.

Therefore a judge who systematically uses a compressed range can remain
compressed after offset calibration.

The system must diagnose, but must not silently repair, scale differences.

For each judge, retain:

```text
mean(raw_scores)
variance(raw_scores)
minimum(raw_scores)
maximum(raw_scores)
```

and compare these diagnostics against the population and shared-submission
evidence.

A scale diagnostic is an audit signal, not an automatic invalidation rule.
Any future affine calibration extension must be separately versioned, simulated,
and accepted before production use.
---

# 48. Zero Variance

Type 1 does not require division by a judge's standard deviation.

Therefore zero-variance judges do not create a z-score division-by-zero problem.

If a future calibration model introduces scale normalization, it must explicitly define a zero-variance fallback.

---

# 49. Applying Calibration

For a valid raw review score:

```text
r = raw weighted score
```

and judge offset:

```text
b_j
```

calculate:

```text
n(s,j) = r(s,j) - b_j
```

This is the **uncapped calibrated review score**.

The system must preserve this value exactly as calculated. It must not clamp each
review independently.

If a review is invalidated or a judge is replaced, the calibration graph can change.
Calibration must then be recomputed using the current valid review set.

Per-review clamping is prohibited because it makes the calibration operation
non-linear before aggregation and can systematically compress legitimate scores
near 0 and 100.

# 50. Why Clamping Happens After Calibration

The raw rubric score is naturally bounded:

```text

0 to 100

```

An additive correction can theoretically produce:

```text

< 0

```

or:

```text

> 100

```

Therefore calibration is applied first and the result is then clamped.

This preserves the intended scoring range while retaining the exact uncapped model output for auditability.

---

# 51. Calibration Versioning

Every calibration run must have:

```text

calibration_version

```

Store:

```text

judge_id

calibration_version

offset

overlap evidence

residual metrics

evidence status

```

If a review is invalidated or a judge is replaced, the calibration graph can change.

Calibration must then be recomputed using the current valid review set.

---

# 51.1 Finalization Invariant

Finalization must use one immutable calibration snapshot and one immutable set
of valid reviews.

The calculation order is exactly:

```text
raw criterion scores
    ↓
raw weighted review scores r(s,j)
    ↓
judge-overlap calibration b_j
    ↓
uncapped calibrated review scores n(s,j)
    ↓
per-submission mean F_raw(s)
    ↓
deterministic ranking by F_raw(s)
    ↓
single publication clamp F_s ∈ [0,100]
```

No score may be clamped, rounded, or rank-truncated earlier in this pipeline.


# 52. Final Score

For submission `s`, let the uncapped calibrated review scores be:

```text
n_1
n_2
...
n_R
```

First aggregate the calibrated evidence:

```text
F_raw(s) =
(1 / R) * sum_j n_j
```

Every valid judge contributes equally to the aggregate.

No judge receives a larger final-score weight simply because they are considered
more experienced.

Both values must be stored:

```text
F_raw(s)
F_s
```

The published score is bounded exactly once:

```text
F_s =
min(
    100,
    max(0, F_raw(s))
)
```

The **ranking score is `F_raw(s)`, not `F_s`**.

Therefore:

```text
F_raw(A) = 108
F_raw(B) = 101
```

still produces:

```text
rank(A) > rank(B)
```

even though both published scores are:

```text
100
```

This prevents the public score-range clamp from destroying ordering information.

If competition policy requires the displayed score itself to determine ranking,
that must be explicitly declared as a separate competition rule; it is not the
Type 1 judging-engine default.

The system must not clamp individual `n_j` values before averaging.

# 53. Final Score Example

Suppose four judges produce calibrated normalized scores:

```text

82.1

79.3

86.0

81.5

```

Then:

```text

F =

(82.1 + 79.3 + 86.0 + 81.5) / 4

F = 82.225

```

Display:

```text

82.23

```

Store:

```text

82.225

```

or the system's full-precision equivalent.

Do not use displayed rounding for ranking.

---

# 54. Disagreement

Disagreement is a diagnostic measurement.

It is not an automatic penalty.

For a submission with `R` valid calibrated scores:

```text

SD_s =

sqrt(

    sum(

        (n'_j - F_s)^2

    ) / (R - 1)

)

```

This is the sample standard deviation.

Also store:

```text

minimum score

maximum score

range

review_count

standard deviation

```

---

# 55. Disagreement Policy

High disagreement:

* is reported

* may trigger an optional audit flag

* does not automatically reduce the final score

* does not automatically invalidate a judge

* does not automatically remove an outlier

Do not use:

```text

lowest-score removal

highest-score removal

automatic outlier removal

median-only scoring

automatic judge exclusion

```

---

# 56. Why Disagreement Is Not a Penalty

Different judges can legitimately disagree about a submission.

A disagreement signal answers:

> How much did the judges differ?

It does not by itself answer:

> Which judge was wrong?

Therefore disagreement and final score remain separate concepts.

The architecture is:

```text

Calibration

    =

judge scoring tendency

Final Score

    =

average calibrated assessment

Disagreement

    =

amount of judge difference

Audit Flag

    =

unusual evidence requiring possible review

```

---

# 57. Exact Ties

The scoring engine ranks submissions by:

```text
F_raw(s)
```

at full stored precision.

The published bounded score:

```text
F_s
```

is not used to create false ties caused only by the `[0,100]` display clamp.

If:

```text
F_raw(A) = F_raw(B)
```

at full stored precision, the engine orders the tied submissions using the
versioned deterministic key:

```text
tie_key =
SHA-256(
    canonical_encode(
        competition_id,
        judging_version,
        submission_id
    )
)
```

The lexicographic byte ordering of `tie_key` is used.

The canonical serialization format must be explicitly versioned and use UTF-8
with length-delimited fields. Database insertion order, timestamps, judge
identity, random unseeded values, and organizer preference are forbidden.

The published score remains tied; the deterministic key exists only to make
machine ordering, pagination, exports, and cutoff handling reproducible.

If competition policy requires a human-facing shared rank instead of a total
machine ordering, the UI may display the tie while retaining the same internal
deterministic key.

# 58. Ranking Precision

Ranking must use full stored precision.

For example:

```text

82.22491

```

must be compared against:

```text

82.22489

```

using the full values, even if both display as:

```text

82.22

```

Displayed rounding must never alter ranking.

---

# 59. Judge Visibility and Isolation

Judges must only see the submissions assigned to them.

A judge must not see:

* another judge's assignments

* another judge's scores

* another judge's reviews

* another judge's workload

* calibration offsets

* normalized scores

* current aggregate scores

* ranking based on current judging data

The backend must enforce this.

This cannot be treated as merely a UI restriction.

---

# 60. Submission Identity Protection

Judges receive an anonymized judging-safe submission representation.

Judges must not receive hidden participant PII in API responses or page payloads.

Organizer views may contain full submission identity.

The backend must use separate access-controlled representations rather than sending hidden fields to the browser and hiding them with UI code.

---

# 61. Judge Authorization

Judge access requires:

```text

authenticated user

+

active judge role

+

active assignment

```

A judge must not be able to:

* score an unassigned submission

* score another judge's submission

* modify another judge's review

* change their judge ID

* request another judge's scores

* access participant identity information

* modify locked reviews

* view current normalized/final scores during judging

Authorization must be checked server-side.

---

# 62. Organizer Permissions

The organizer is the event authority.

Organizer capabilities include:

### Before judging

* configure rubric

* manage judges

* generate assignments

* validate assignments

### During judging

* monitor judging status

* investigate issues

* replace unavailable judges

* invalidate reviews when justified

* manage assignment changes

### After judging

* view results

* inspect calibration

* inspect disagreement

* perform audited corrections

* finalize results

Organizer actions that affect judging must be audited.

---

# 63. Organizer Corrections

An organizer should not simply overwrite a judge's score because they disagree with it.

Instead:

1. investigate

2. preserve the original review

3. record the reason

4. invalidate the review if invalid under event rules

5. replace the missing review when required

6. recompute calibration

7. recompute affected final scores

This preserves auditability and prevents silent result manipulation.

---

# 64. Security Tests

The implementation must include automated tests for:

```text

Judge accesses another judge's submission

    -> 403 / authorization rejection

Judge modifies another judge's review

    -> rejection

Judge scores unassigned submission

    -> rejection

Judge changes judge identity

    -> rejection

Judge requests another judge's score

    -> rejection

Judge requests participant identity

    -> hidden/rejected

Organizer invalidates review

    -> original retained

    -> audit record created

    -> review excluded

    -> recalculation triggered when necessary

Judge removed

    -> affected submissions identified

    -> replacement workflow triggered

Finalized result mutation

    -> restricted

    -> audited

```

---

# 65. Audit Logging

Audit events must record, where applicable:

```text

actor_id

actor_role

action

target_type

target_id

timestamp

reason

before_state

after_state

```

Important actions include:

* authentication

* assignment access

* assignment generation

* assignment changes

* review creation

* review submission

* review edit

* review lock

* review invalidation

* judge replacement

* judge removal

* calibration generation

* finalization

* post-finalization correction

---

# 66. Immutable Raw Data

Raw judging data must be preserved.

Do not silently overwrite:

* raw criterion scores

* raw weighted scores

* review versions

* invalidated reviews

* assignment snapshots

* calibration versions

* finalization records

Derived values may be recomputed, but raw evidence must remain reconstructable.

---

# 67. Versioning

Version everything that can affect results:

```text

assignment_version

rubric_version

algorithm_version

calibration_version

scoring_version

```

A final result must reference the exact versions used to produce it.

---

# 68. Reconstructability

The final result must be reconstructable through this chain:

```text

Submission

    |

    v

Assigned judges

    |

    v

Raw criterion scores

    |

    v

Raw weighted review scores

    |

    v

Valid overlap graph

    |

    v

Calibration parameters

    |

    v

Normalized scores

    |

    v

Final score

    |

    v

Disagreement metrics

```

A reviewer should be able to determine exactly how a final score was produced.

---

# 69. Assignment Snapshot

Store an immutable assignment snapshot containing:

```text

submission_id

assigned_judge_ids

assignment_version

algorithm_version

seed

timestamp

```

Global assignment metadata:

```text

N

J

R

judge quotas

assignment_hash

```

---

# 70. Calibration Snapshot

Store:

```text

calibration_version

judge_id

offset

overlap evidence

residual

evidence status

```

The calibration snapshot must identify the exact review set from which it was generated.

---

# 71. Finalization Record

Finalization creates an immutable record containing:

```text

finalization_id

scoring_version

assignment_version

rubric_version

calibration_version

finalized_at

finalized_by

optional snapshot hash

```

After finalization, corrections create a new scoring version.

They must not silently overwrite the previous final result.

---

# 72. Application Data vs Audit Data

Conceptually separate:

```text

application state

```

from:

```text

audit/history

```

Application state contains current usable results.

Audit data preserves how those results were produced and changed.

Both are required for trustworthy reconstruction.

---

# 73. Finalization Rules

Normal finalization is allowed only when:

```text

every submission has R valid locked reviews

```

Then:

```text

valid reviews

    ->

overlap graph

    ->

calibration

    ->

normalized scores

    ->

final scores

    ->

disagreement

    ->

finalization

```

If any required review is missing:

```text

INCOMPLETE_JUDGING

```

and normal finalization is blocked.

---

# 74. Calibration Finalization Procedure

The locked calibration pipeline is:

```text

Valid locked reviews

        |

        v

Valid overlap graph

        |

        v

Connectivity check

        |

        v

Pairwise score differences

        |

        v

Global additive offset estimation

        |

        v

Residual/evidence metrics

        |

        v

Normalized scores

        |

        v

Clamp to [0,100]

        |

        v

Final submission scores

        |

        v

Disagreement metrics

        |

        v

Finalization

```

---

# 75. Failure Codes

Use machine-readable lifecycle and failure states.

Lifecycle:

```text

DRAFT

GENERATING

VALIDATING

READY

FAILED

```

Failure reasons:

```text

CONFIGURATION_INVALID

NO_VALID_CANDIDATE

CONNECTIVITY_FAILURE

FAILED_CALIBRATION_CONNECTIVITY

VALIDATION_FAILURE

EXECUTION_ERROR

TIMEOUT

```

Judging status:

```text

INCOMPLETE_JUDGING

REDUCED_REVIEW_COUNT

```

---

# 76. Fail-Closed Principle

The system must never silently make the scoring model weaker in order to produce a result.

Never silently:

* reduce review count

* duplicate judges

* relax quotas

* disconnect calibration components

* discard inconvenient reviews

* remove outliers

* change rubric weights

* overwrite raw scores

* change final scores without an audit record

If a hard requirement cannot be satisfied:

```text

fail explicitly

```

---

# 77. Performance Targets

For the reference case:

```text

N = 2500

J = 20

R = 4

```

initial assignment generation evaluates approximately:

```text

12.1 million candidate groups

```

with an optimized implementation.

This is a backend computation, not a browser computation.

The assignment process should run asynchronously.

The UI should show progress such as:

```text

Assigning judges...

✓ Calculating judge quotas

✓ Generating assignments

✓ Optimizing overlap

✓ Validating assignments

Assignment ready

```

---

# 78. Assignment Runtime Target

The engineering target is:

```text

Assignment generation + optimization + validation:

ideally < 60 seconds

```

A hard engineering acceptance ceiling may be:

```text

< 5 minutes

```

Five minutes is **not** the expected normal user experience.

If the implementation routinely approaches five minutes for the reference configuration, it should be profiled and optimized.

The browser should not block on one synchronous request.

---

# 79. Calibration Runtime

For:

```text

J = 20

```

there are only:

```text

190

```

possible judge pairs.

For:

```text

N = 2500

R = 4

```

there are:

```text

10000

```

reviews.

Calibration and final scoring should therefore be computationally small relative to human judging and should normally complete within seconds.

Target:

```text

calibration + normalization < 1 minute

```

---

# 80. Finalization Runtime

Finalization, validation, and audit snapshot generation should normally complete quickly.

Target:

```text

finalization + audit snapshot < 1 minute

```

The primary elapsed time in the system should be human judging, not score computation.

---

# 81. Human Judging Capacity

For the reference configuration:

```text

10000 total reviews

20 judges

500 reviews per judge

```

Approximate judge effort:

| Time per review | Time per judge | Total judge-hours |

| --------------- | -------------: | ----------------: |

| 2 min           |         16.7 h |           333.3 h |

| 3 min           |         25.0 h |             500 h |

| 5 min           |         41.7 h |           833.3 h |

| 10 min          |         83.3 h |          1666.7 h |

Because 20 judges work in parallel, ideal wall-clock time is approximately the per-judge workload, subject to real-world availability and breaks.

For a 48-hour judging window:

```text

20 judges * 48 hours

=

960 judge-hours

```

This shows why review duration and judge availability matter operationally.

---

# 82. Reference End-to-End Test Configuration

Use:

```text

N = 2500

J = 20

R = 4

```

with four rubric criteria.

Expected:

```text

Total assignments = 10000

Reviews per judge = 500

Pair-overlap events = 15000

Judge pairs = 190

Average pair overlap ≈ 78.95

```

---

# 83. Required Test Suite

## Test 1 — Configuration Validation

Verify:

```text

R <= J

N > 0

J > 0

R > 0

judge IDs unique

submission IDs unique

```

Expected:

```text

PASS

```

Invalid configuration must fail before assignment generation.

---

## Test 2 — Deterministic Assignment

Run assignment twice with identical:

```text

inputs

versions

seed

```

Expected:

```text

assignment 1 == assignment 2

assignment_hash 1 == assignment_hash 2

```

---

## Test 3 — Assignment Validation

Verify:

```text

every submission has R judges

every judge has exact quota

no duplicates

total assignments = N*R

overlap invariant holds

graph connected

```

Expected:

```text

PASS

```

---

## Test 4 — Judge Isolation

Attempt unauthorized access to:

* another judge's submission

* another judge's review

* another judge's score

* participant identity

Expected:

```text

403 / rejection / hidden data

```

---

## Test 5 — Normal Review

Submit valid scores for all criteria.

Expected:

```text

review accepted

raw scores preserved

raw weighted score calculated

review becomes SUBMITTED

```

---

## Test 6 — Invalid Review Input

Attempt:

* missing criterion

* out-of-range score

* unexpected criterion

* wrong rubric version

* unassigned submission

* duplicate submission

* locked review edit

Expected:

```text

rejected

no partial invalid review saved

```

---

## Test 7 — Judge Isolation During Scoring

Verify a judge cannot discover:

```text

other assignments

other scores

other reviews

current aggregates

calibration

ranking

```

Expected:

```text

PASS

```

---

## Test 8 — Complete Judging

Complete all:

```text

2500 * 4 = 10000

```

valid reviews.

Expected:

```text

all submissions have 4 valid locked reviews

```

---

## Test 9 — Calibration

Create controlled judge offsets.

Verify that the calibration system estimates relative additive offsets.

Verify:

```text

sum(b_j) ≈ 0

```

subject to numerical precision.

---

## Test 10 — Low Overlap

Create judge pairs with low overlap.

Verify:

```text

LOW_EVIDENCE

```

is reported where applicable.

Verify evidence is not automatically discarded solely because:

```text

k < 3

```

---

## Test 11 — Contradictory Evidence

Create shared submissions where pairwise score differences conflict.

Expected:

```text

global least-squares calibration succeeds

residuals retained

no automatic judge invalidation

```

---

## Test 12 — Judge Removal

Remove a judge after assignments exist.

Expected:

```text

affected submissions identified

replacement judges assigned

R restored

quotas preserved

no duplicate judge

calibration recomputed

```

---

## Test 13 — Organizer Invalidation

Invalidate a valid review.

Expected:

```text

original review retained

audit record created

review excluded from scoring

replacement triggered if required

calibration recomputed

```

---

## Test 14 — High Disagreement

Create a submission with very different judge scores.

Expected:

```text

high SD recorded

range recorded

no automatic score penalty

no automatic judge removal

```

---

## Test 15 — Final Score

Verify:

```text

F_s = average(calibrated normalized scores)

```

and that full precision is used internally.

---

## Test 16 — Audit Reconstruction

Given a final score, reconstruct:

```text

assignment

raw criteria

raw weighted scores

overlap

calibration

normalized scores

final score

disagreement

```

Expected:

```text

exact reconstruction

```

---

## Test 17 — Reproducibility

Rerun the complete calculation using identical:

```text

raw data

assignment snapshot

rubric version

algorithm version

calibration version

scoring version

seed

```

Expected:

```text

identical final scores

identical calibration parameters

identical normalized scores

```

subject to explicitly defined floating-point precision.

---

## Test 18 — Finalization Integrity

Attempt to mutate finalized results.

Expected:

```text

rejected or creates a new scoring version

```

Old finalization must remain reconstructable.

---

# 84. Core Invariants

The implementation must enforce these invariants.

### Assignment count

```text

total assignments = N * R

```

### Submission review count

```text

every submission = exactly R judges

```

### Distinctness

```text

no submission contains duplicate judge

```

### Workload

```text

each judge = exact calculated quota

```

### Pair overlap

```text

sum(O_ij)

=

N * C(R,2)

```

### Connectivity

```text

judge overlap graph connected

```

when global calibration is required.

### Raw score bounds

```text

0 <= score <= criterion_max

```

### Raw weighted score

```text

0 <= r <= 100

```

### Normalized score

```text

0 <= n' <= 100

```

### Final score

```text

0 <= F <= 100

```

### Reproducibility

```text

same inputs + same versions + same seed

=

same output

```

---

# 85. What This Architecture Does Not Do

Type 1 Phase 2 does **not**:

* use Phase 1 scores as final scores

* use Phase 1 pairwise results as judge calibration

* use independent judge z-scores

* automatically remove score outliers

* automatically penalize disagreement

* automatically invalidate judges for high/low scoring

* silently reduce review counts

* relax exact workload constraints

* expose calibration parameters to judges

* expose other judges' scores

* overwrite raw review data

* mutate the rubric after judging begins

* use displayed rounded scores for ranking

---

# 86. Design Defense

## Why multiple judges?

A single judge creates excessive dependence on one person's interpretation.

Multiple independent reviews provide:

* independent evidence

* disagreement measurement

* cross-judge calibration

* greater robustness to individual scoring tendencies

---

## Why deliberate overlap?

Without overlap, judges cannot be empirically compared.

If Judge A and Judge B never evaluate the same submission, the system cannot determine whether their score difference comes from:

```text

judge tendency

```

or:

```text

different submission difficulty

```

Overlap provides the common reference needed for calibration.

---

## Why exact workload balancing?

Large workload differences can introduce operational unfairness and increase the chance that some judges rush while others have substantially less work.

The exact quota model makes the workload difference at most one assignment.

---

## Why connected calibration?

A disconnected judge graph creates independent scoring components.

There is no empirical bridge between those components.

Therefore global calibration cannot be justified.

The system should fail rather than invent the relationship.

---

## Why additive offsets?

The Type 1 baseline needs a simple, explainable calibration model.

An additive offset directly represents:

```text

Judge tends to score approximately X points higher/lower

```

It is easy to audit and explain.

More complex models can be introduced later if evidence shows that scale differences need modeling.

---

## Why not z-score each judge?

Because judges may receive different distributions of submissions.

A judge seeing a difficult batch could legitimately have a lower score distribution.

Independent z-scoring can remove meaningful information.

Shared-submission comparison is therefore the baseline.

---

## Why not remove outliers?

A high or low score may be the correct assessment.

Automatically removing it assumes the system knows which judge is wrong.

The architecture instead preserves disagreement and allows audit where necessary.

---

## Why not penalize disagreement?

Disagreement measures uncertainty or genuine difference of opinion.

It does not establish error.

Therefore disagreement is reported rather than converted into an arbitrary penalty.

---

## Why preserve raw scores?

Because the final score must be reconstructable.

If only the calibrated final score is stored, organizers cannot later determine:

```text

what the judge actually entered

```

or:

```text

how calibration changed it

```

Raw data therefore remains immutable.

---

# 87. Final Type 1 Phase 2 Model

The complete mathematical model is:

### Assignment

```text

A = N * R

```

Exact judge quotas:

```text

baseQuota = floor(A/J)

extra = A mod J

```

Overlap:

```text

O_ij =

number of shared submissions between judges i and j

```

Pair-overlap invariant:

```text

sum(O_ij)

=

N * C(R,2)

```

Pair target:

```text

P =

N * C(R,2) / C(J,2)

```

Pair loss:

```text

PairLoss =

sum((O_ij - P)^2)

```

---

### Raw rubric score

```text

r(s,j)

=

100 * sum(

    weight_c * score(s,j,c) / max_c

)

```

---

### Pairwise calibration evidence

```text

d_ij,s =

r(s,i) - r(s,j)

```

```text

bar_d_ij =

average(d_ij,s)

```

```text

omega_ij =

number of valid shared submissions

```

---

### Global calibration

```text

minimize:

sum(

    omega_ij *

    ((b_i - b_j) - bar_d_ij)^2

)

```

subject to:

```text

sum(b_j) = 0

```

---

### Normalization

```text

n(s,j) =

r(s,j) - b_j

```

```text

n'(s,j) =

min(100, max(0, n(s,j)))

```

---

### Final score

```text

F_s =

sum(n'(s,j)) / R

```

---

### Disagreement

```text

SD_s =

sqrt(

    sum((n'(s,j) - F_s)^2)

    / (R - 1)

)

```

---

# 88. Final Architecture Verdict

For Type 1 Phase 2, the implementation should be considered complete only when it satisfies all of the following:

```text

Exact R reviews per submission

+

Exact balanced judge quotas

+

Deliberate overlap

+

Connected judge graph

+

Deterministic assignment

+

Validated review input

+

Immutable raw scores

+

Overlap-based additive calibration

+

No independent z-score baseline

+

Calibration-aware normalization

+

Equal-weight final averaging

+

Disagreement reporting

+

Judge isolation

+

Organizer auditability

+

Replacement workflow

+

Versioned calculations

+

Reproducible finalization

+

Fail-closed behavior

```

The resulting architecture is:

```text

                 TYPE 1 — PHASE 2

                     SUBMISSIONS

                          |

                          v

              +-----------------------+

              | Deterministic Judge   |

              | Assignment Engine     |

              +-----------------------+

                          |

             exact R + exact quotas

                          |

                          v

              +-----------------------+

              | Deliberate Judge      |

              | Overlap / Graph       |

              +-----------------------+

                          |

                          v

                 JUDGE REVIEW WINDOW

                          |

                          v

              +-----------------------+

              | Raw Rubric Scores     |

              | + Review Versions     |

              +-----------------------+

                          |

                          v

              +-----------------------+

              | Valid Overlap Graph   |

              +-----------------------+

                          |

                          v

              +-----------------------+

              | Additive Judge        |

              | Calibration           |

              +-----------------------+

                          |

                          v

              +-----------------------+

              | Normalization         |

              | + Clamp [0,100]       |

              +-----------------------+

                          |

                          v

              +-----------------------+

              | Final Average Score    |

              +-----------------------+

                          |

                          +-------> Disagreement

                          |

                          v

              +-----------------------+

              | Immutable Finalization|

              | + Full Audit Trail    |

              +-----------------------+

```

**Type 1 Phase 2 is therefore defined as a multi-review, overlap-calibrated, deterministic, auditable scoring system in which assignment quality is treated as part of scoring-system correctness rather than merely workload distribution.**

---

# PART III — PHASE 1 → PHASE 2 CONTRACT

## 1. Handoff Condition

Phase 1 is complete only when:

1. the Phase 1 population was validated;
2. the Phase 1 comparison budget was consumed or its stability criterion was met;
3. the final Bradley–Terry model was fitted successfully;
4. the final evidence/uncertainty state was recorded;
5. exactly `K` submissions were selected;
6. the shortlist and its finalization metadata were immutably recorded.

The Phase 2 assignment engine must consume this finalized shortlist rather than recomputing the Phase 1 ranking.

## 2. Handoff Data

At minimum, the Phase 1 finalization record must identify:

```text
competition_id
phase1_version
algorithm_version
N
J_phase1
K
selected_submission_ids
final_model_version
final_model_parameters
evidence_summary
boundary_status
assignment/budget metadata
finalized_at
```

Phase 2 must additionally create its own assignment and rubric versions. Phase 1 and Phase 2 versions must remain independently auditable.

## 3. What Carries Forward

The following carry forward:

- the exact selected submission IDs;
- submission eligibility state;
- submission version/content version;
- competition/version identity;
- anonymization/identity protections required by the judging UI;
- audit and finalization references.

The following do **not** carry forward as Phase 2 scores:

- Phase 1 Bradley–Terry `theta` values;
- Phase 1 pairwise win counts;
- Phase 1 rank positions;
- Phase 1 uncertainty values.

Those are Phase 1 evidence and may be retained for auditability, but Phase 2 starts its own independent scoring process.

## 4. No Cross-Phase Contamination

Phase 2 judge assignment must not depend on:

- Phase 1 rank;
- Phase 1 Bradley–Terry estimate;
- Phase 1 winner/loser counts;
- organizer preferences about which shortlisted submission should receive more attention.

Every shortlisted submission receives the Phase 2 review count required by the Type 1 configuration.

## 5. Final Type 1 Result

The complete Type 1 result is therefore:

```text
Phase 1:
    pairwise evidence
        -> Bradley–Terry estimate
        -> protected investigation
        -> exact K shortlist

Phase 2:
    exact R reviews/submission
        -> raw rubric scores
        -> overlap calibration
        -> normalized scores
        -> final score
```

This separation is intentional: Phase 1 protects against premature exclusion at scale, while Phase 2 provides the detailed, calibrated evaluation used for final scoring.

---

# PART IV — IMPLEMENTATION ACCEPTANCE GATE

A Type 1 judging implementation is not production-ready unless all of the following pass:

### Phase 1

- deterministic bootstrap assignment;
- minimum evidence guarantee;
- connected comparison graph;
- Bradley–Terry fit;
- uncertainty calculation;
- protected adaptive investigation;
- exact `K` shortlist;
- deterministic tie handling;
- budget and workload validation;
- simulation acceptance gate;
- security/isolation tests;
- audit reconstruction.

### Phase 2

- exact `R` reviews per submission;
- distinct judges per submission;
- exact workload quotas;
- overlap invariant;
- connected judge graph;
- deterministic assignment;
- assignment fingerprint;
- rubric validation;
- transactional review submission;
- raw-score preservation;
- incomplete-review handling;
- overlap-based calibration;
- calibration residual diagnostics;
- normalization;
- final score calculation;
- disagreement reporting;
- immutable finalization snapshot;
- audit reconstruction.

### End-to-end

The implementation must demonstrate that:

```text
valid submissions
    -> Phase 1
    -> exactly K shortlisted submissions
    -> Phase 2
    -> exactly R valid reviews per submission
    -> calibrated final scores
    -> reproducible finalization
```

No hard constraint may be silently relaxed to make a run succeed.

---

# PART V — FINAL DESIGN PRINCIPLE

> **Type 1 judging is a two-stage evidence pipeline, not a single ranking formula.**

Phase 1 uses adaptive pairwise evidence to safely reduce a large population to an exact shortlist.

Phase 2 uses independent rubric reviews and deliberate overlap to estimate judge scoring offsets, normalize scores, and calculate final results.

The system is designed around three properties:

1. **fair evidence collection** — strong submissions should not be prematurely discarded;
2. **statistical comparability** — judges must be comparable through shared evidence rather than independent score-distribution assumptions;
3. **auditability** — every assignment, score, calibration parameter, derived result, and final decision must be reconstructable from versioned immutable evidence.



---

# APPENDIX A — AUDIT RESOLUTION RECORD

This appendix records how the accompanying mathematical audit is incorporated.

## A.1 Findings adopted

The specification now explicitly addresses:

1. **Bradley–Terry separation failure**
   - Production Phase 1 fitting uses positive L2 regularization.
   - Solver convergence settings are versioned and recorded.

2. **Per-review calibration clamping**
   - Prohibited.
   - Calibrated review values remain uncapped until the submission-level aggregate.
   - The final aggregate is clamped once to `[0,100]`.

3. **Phase 2 exact-score tie handling**
   - Full-precision score equality is preserved.
   - A canonical SHA-256 tie key provides deterministic machine ordering.

4. **Budget-engine ambiguity**
   - The budget is an explicit deterministic function of `N`, `J`, `K`, and
     versioned configuration.
   - Capacity/connectivity adjustments are recorded rather than silently applied.

5. **Scale compression**
   - The baseline additive model is explicitly documented as unable to correct
     multiplicative judge scale.
   - Scale diagnostics are retained without silently introducing a new estimator.

6. **Adversarial calibration diagnostics**
   - Residual, variance, range, overlap, and influence diagnostics are retained.
   - Robust estimators are treated as future versioned extensions rather than
     implicit behavior.

## A.2 Finding rejected as mathematically incorrect

The audit claim that weighting the calibration loss by `omega_ij` is an error
is rejected.

Under the stated homoscedastic independent-noise model:

```text
Var(bar_d_ij) = sigma^2 / omega_ij
```

so inverse-variance weighting is proportional to:

```text
omega_ij
```

Therefore the existing objective:

```text
sum_ij omega_ij *
    ((b_i - b_j) - bar_d_ij)^2
```

is the correct WLS form for that model.

Changing the weight to `sqrt(omega_ij)` would not be an inverse-variance correction.

## A.3 Findings that remain deliberate limitations

The following are documented limitations rather than hidden defects:

- additive calibration does not correct judge-specific scale;
- sparse overlap increases calibration uncertainty;
- robust collusion resistance is not guaranteed by ordinary least squares;
- final rankings can remain sensitive when submissions are genuinely very close;
- Type 1 remains a no-tracks architecture.

These limitations must be visible in implementation diagnostics and test reports.


---

# APPENDIX B — IMPLEMENTATION GAP CLOSURE

## B.1 Phase 1 judge assignment

Logical comparisons are assigned to judges by a deterministic hash-seeded cyclic
capacity scheduler. Load spread is at most one whenever the active judge pool
can support that balance.

## B.2 Phase 1 adaptive objective

Adaptive pair priority is exactly:

```text
P_ij =
0.40*S_ij +
0.30*B_ij +
0.20*I_ij +
0.10*E_ij
```

with all component definitions in Section 26.1.

## B.3 Phase 1 stability

The default stability test is:

```text
theta_K - theta_K1
>
1.96 * sqrt(se_K^2 + se_K1^2)
```

plus minimum-evidence protection for potentially advancing submissions.

## B.4 Phase 2 scalable assignment

The production assignment engine does not enumerate `C(J,R)` groups. It uses
incremental greedy construction followed by deterministic local repair.

## B.5 Phase 2 dropout

Completed assignments are immutable. Only outstanding review slots are
reallocated. Replacement quotas are computed from remaining demand and the
current active judge pool.

## B.6 Local-repair determinism

Equal repair moves are resolved by a canonical lexicographic tuple containing
connectivity, pair-loss delta, affected-submission count, removed judge ID,
added judge ID, and submission ID.

## B.7 Scope

These additions apply only to Type 1 (No Tracks). Track-specific calibration
requires a separate architecture because disconnected track judge graphs do not
identify a single global judge-offset vector.


## A.4 Ranking-vs-publication score invariant

The bounded publication score and the ranking score are intentionally different
representations of the same calibrated aggregate:

```text
ranking_score = F_raw
published_score = clamp(F_raw, 0, 100)
```

This eliminates artificial ties caused solely by the public score range.

The raw aggregate remains auditable and is immutable after finalization.
