# T2 Judging System

## Assignment, Judge Overlap, Cross-Judge Calibration, Normalization and Final Scoring

---

## 1. System Objective

The T2 judging system is designed around one principle:

> **Assignment should not only distribute judging workload; it should deliberately create the statistical overlap required to calibrate judges.**

The complete judging pipeline is:

```text
Organizer Configuration
        ↓
Judge Assignment
        ↓
Repeated Reviews
        ↓
Judge Overlap Graph
        ↓
Cross-Judge Calibration
        ↓
Normalized Scores
        ↓
Final Submission Score
        ↓
Audit / Review
```

The important architectural decision is that **assignment and normalization are connected**.

If judges are assigned completely independently, the system may have no reliable way to determine whether differences in scores come from:

* differences in submission quality, or
* differences in how judges use the scoring scale.

By deliberately assigning multiple judges to the same submissions, the system obtains direct observations of judges evaluating common work.

---

# 2. Configuration

The organizer specifies:

* `N` = number of submissions
* `J` = number of judges
* `R` = required number of independent reviews per submission
* rubric criteria
* criterion weights
* judge eligibility rules
* optional tracks

The organizer chooses `R`.

The assignment algorithm does **not** automatically decide `R`.

The algorithm's responsibility is to determine whether the chosen configuration is feasible and then produce an assignment satisfying it.

---

# 3. Example System Size

For the example:

$$
N=2500
$$

$$
J=20
$$

$$
R=4
$$

Total review assignments:

$$
A=N\times R
$$

Therefore:

$$
A=2500\times4=10000
$$

The average target workload per judge is:

$$
W_{target}=\frac{A}{J}
$$

$$
W_{target}=\frac{10000}{20}=500
$$

Therefore the assignment system aims for approximately:

$$
\boxed{500\text{ reviews per judge}}
$$

The actual distribution should be as close to equal as the constraints permit.

---

# 4. Two Assignment Modes

The system should be designed as **one assignment engine with two constraint modes**, rather than two unrelated implementations.

```text
                    Assignment Engine
                           │
             ┌─────────────┴─────────────┐
             │                           │
         No Tracks                    Tracks
             │                           │
      All judges eligible       Track eligibility applied
```

## Algorithm 1 — No-Track Assignment

This is the baseline algorithm.

When there are no tracks:

$$
Eligible(s)=G
$$

where `G` is the complete judge set.

The algorithm optimizes:

1. exact review count,
2. distinct judges per submission,
3. balanced workload,
4. connected judge-overlap graph,
5. balanced judge-pair overlap.

---

## Algorithm 2 — Track-Constrained Assignment

When tracks are introduced, the same core assignment engine is retained, but the candidate judge pool becomes constrained.

For submission `s`:

$$
Eligible(s)
=
\{j\in G:E(j,s)=true\}
$$

The exact track strategy must be defined separately before implementation.

**Important:** the track algorithm should not be assumed to be identical to the no-track case merely by filtering judges. Track restrictions can fundamentally affect the overlap graph and therefore the ability to perform global cross-judge calibration.

This document therefore fully specifies the no-track algorithm and establishes the mathematical requirements that the track-constrained algorithm must preserve.

---

# 5. Algorithm 1 — No-Track Assignment

## 5.1 Inputs

```text
N = number of submissions
J = number of judges
R = reviews required per submission
```

There are no track restrictions.

Therefore every judge is eligible for every submission.

---

# 6. Hard Assignment Constraints

These conditions must always be satisfied.

## 6.1 Exactly R Reviews

For every submission `s`:

$$
|\text{Judges}(s)|=R
$$

For example, if:

$$
R=4
$$

every submission must receive exactly four reviews.

---

## 6.2 Distinct Judges

For every submission:

$$
j_a\neq j_b
$$

for every pair of assigned judges.

A judge cannot review the same submission twice.

---

## 6.3 Feasibility

At minimum:

$$
J\ge R
$$

If:

$$
R>J
$$

the configuration is impossible because there are not enough judges to provide `R` distinct reviews.

---

# 7. Total Assignment Mathematics

Every submission generates exactly `R` assignments.

Therefore:

$$
A=N\times R
$$

For:

$$
N=2500,\quad R=4
$$

$$
A=10000
$$

This gives a useful invariant:

$$
\boxed{\sum_{j=1}^{J}W_j=NR}
$$

where `W_j` is the number of submissions assigned to judge `j`.

For the example:

$$
\sum_{j=1}^{20}W_j=10000
$$

This is an automated correctness check.

---

# 8. Workload Balancing

The ideal workload is:

$$
W_{target}=\frac{NR}{J}
$$

For the example:

$$
W_{target}=500
$$

If the target is not an integer, the best possible balanced distribution consists of judges receiving:

$$
\lfloor W_{target}\rfloor
$$

or:

$$
\lceil W_{target}\rceil
$$

reviews, subject to constraints.

A useful optimization objective is:

$$
L_{work}
=
\sum_{j=1}^{J}
(W_j-W_{target})^2
$$

The ideal value is:

$$
L_{work}=0
$$

when perfectly equal workload is possible.

---

# 9. Why Random Assignment Is Not Enough

A purely random assignment may satisfy:

$$
|\text{Judges}(s)|=R
$$

but that alone is insufficient.

Random assignment can produce undesirable distributions of:

* judge workload,
* judge-pair overlap,
* graph connectivity.

The problem is therefore not simply:

> "Give every submission R random judges."

It is:

> "Give every submission R valid judges while simultaneously constructing a useful judge-overlap network."

This is the key distinction of the system.

---

# 10. Judge-Overlap Graph

Create a graph:

$$
H=(G,E_H)
$$

where:

* every judge is a vertex,
* an edge exists between judges `i` and `j` if they have reviewed at least one common submission.

Formally:

$$
(i,j)\in E_H
\iff
|O_{ij}|>0
$$

where:

$$
O_{ij}
=
\{s:i\text{ and }j\text{ both reviewed }s\}
$$

is the set of submissions reviewed by both judges.

---

# 11. Why the Graph Must Be Connected

Suppose the graph becomes:

```text
J1 — J2 — J3 — J4

J5 — J6 — J7 — J8
```

There is no overlap between the two groups.

The system has evidence about relative scoring behavior within each group, but no overlap-based evidence connecting the groups.

Therefore the relative scale between the two groups is not identifiable from overlap.

The desired condition is:

$$
\boxed{\text{components}(H)=1}
$$

A connected graph means there is an overlap path between every pair of judges.

For example:

```text
J1 — J2 — J5
│         │
J3 — J4 — J6
```

There does not need to be a direct overlap between every pair.

The network simply needs sufficient connections to allow relative calibration information to propagate.

---

# 12. Connectivity vs Redundant Connectivity

A mathematically connected graph requires at least:

$$
J-1
$$

edges.

For 20 judges:

$$
20-1=19
$$

However, the system should not deliberately construct only a single chain:

```text
J1 — J2 — J3 — ... — J20
```

A single chain is technically connected but fragile.

When sufficient assignments exist, the algorithm should create redundant connections.

The objective is therefore:

> **Connected and well-distributed overlap, not merely the minimum number of edges required for connectivity.**

---

# 13. Pairwise Overlap Mathematics

Every submission reviewed by `R` judges creates:

$$
\binom{R}{2}
$$

judge-pair overlap events.

For:

$$
R=4
$$

we have:

$$
\binom42
=
\frac{4\times3}{2}
=
6
$$

Therefore every submission contributes six pairwise overlap events.

For:

$$
N=2500
$$

total pair-overlap events are:

$$
P=N\binom{R}{2}
$$

$$
P=2500\times6
$$

$$
\boxed{P=15000}
$$

---

# 14. Number of Possible Judge Pairs

With `J` judges, the number of possible judge pairs is:

$$
\binom{J}{2}
$$

For:

$$
J=20
$$

$$
\binom{20}{2}
=
\frac{20\times19}{2}
=
190
$$

Therefore there are 190 possible judge pairs.

---

# 15. Target Pairwise Overlap

If pair overlap were perfectly balanced, the average overlap would be:

$$
O_{target}
=
\frac{
N\binom{R}{2}
}{
\binom{J}{2}
}
$$

For the example:

$$
O_{target}
=
\frac{2500\binom42}{\binom{20}{2}}
$$

$$
=
\frac{15000}{190}
$$

$$
\approx78.947
$$

Therefore:

$$
\boxed{O_{target}\approx79}
$$

shared submissions per judge pair on average.

This is an **optimization target**, not a hard equality requirement.

Exact equality may not be possible because assignments are discrete.

---

# 16. Critical Overlap Invariant

The total pair-overlap count must satisfy:

$$
\boxed{
\sum_{i<j}O_{ij}
=
N\binom{R}{2}
}
$$

For:

$$
N=2500,\quad R=4
$$

the right-hand side is:

$$
15000
$$

Therefore:

$$
\boxed{
\sum_{i<j}O_{ij}=15000
}
$$

This is an extremely useful automated test.

If the assignment system says it created 10,000 reviews but the pair-overlap matrix does not sum to 15,000, the assignment/overlap implementation is incorrect.

---

# 17. Assignment Optimization

The assignment engine has hard constraints and optimization objectives.

## Hard constraints

```text
exactly R judges per submission
no duplicate judge/submission
J >= R
```

## Optimization objectives

```text
minimize workload imbalance
minimize pair-overlap imbalance
maximize useful connectivity
```

A conceptual combined objective is:

$$
L=
\lambda_wL_{work}
+
\lambda_oL_{overlap}
+
\lambda_cL_{connectivity}
$$

where:

* `L_work` measures workload imbalance,
* `L_overlap` measures overlap imbalance,
* `L_connectivity` penalizes weak/disconnected overlap structure,
* `λ` values determine implementation priorities.

The exact numerical weights are implementation parameters.

---

# 18. Assignment Procedure

## Phase 1 — Feasibility

Validate:

$$
J\ge R
$$

Calculate:

$$
A=NR
$$

Calculate:

$$
W_{target}=\frac{NR}{J}
$$

Initialize:

```text
judgeLoad[j]
pairOverlap[i][j]
judgeGraph
submissionAssignments[s]
```

---

## Phase 2 — Establish Judge Connectivity

Before simply filling all review slots, ensure the assignment process creates overlap relationships connecting the judges.

The algorithm should deliberately introduce shared submissions between judges.

The system should prefer redundant connections when possible.

---

## Phase 3 — Fill Remaining Slots

For each submission:

1. identify currently valid candidate judges,
2. remove judges already assigned to that submission,
3. consider workload,
4. consider pairwise overlap,
5. consider graph connectivity,
6. select the best candidate.

A conceptual candidate score is:

$$
Score(j,s)
=
-\alpha LoadPenalty(j)
-\beta OverlapPenalty(j,s)
+\gamma ConnectivityGain(j,s)
$$

The candidate with the best score is selected, subject to all hard constraints.

---

# 19. Assignment Repair

After the initial assignment, calculate:

```text
workload per judge
pairwise overlap matrix
judge graph connectivity
review counts
duplicate assignments
```

If the result is not sufficiently balanced, perform valid swaps.

Example:

```text
Before:

Submission S
→ J1
→ J5
→ J9
→ J14

Candidate repair:

J14 → J17
```

The swap is valid only if:

* `J17` is eligible,
* `J17` is not already reviewing `S`,
* review count remains exactly `R`,
* no hard constraint is violated,
* connectivity is preserved,
* the swap improves or maintains the optimization objective.

---

# 20. Why Assignment and Normalization Are Connected

After assignment, judges will have reviewed overlapping submissions.

That creates direct observations:

```text
Submission A → J1, J4, J8, J12
Submission B → J1, J4, J9, J15
Submission C → J4, J8, J9, J12
```

For example, J1 and J4 have common work.

If this happens repeatedly across the judging population, the system can estimate whether their scoring scales systematically differ.

Therefore:

$$
\boxed{
\text{Assignment creates the data required for calibration}
}
$$

This is the core reason the overlap structure is deliberately optimized.

---

# 21. Rubric Scoring

Suppose there are `C` judging criteria.

For criterion `c`:

* `x_s,j,c` = score given by judge `j`,
* `M_c` = maximum score,
* `w_c` = criterion weight.

Weights satisfy:

$$
\sum_{c=1}^{C}w_c=1
$$

Normalize the individual criterion:

$$
q_{s,j,c}
=
\frac{x_{s,j,c}}{M_c}
$$

The judge's weighted raw score is:

$$
r_{s,j}
=
100
\sum_{c=1}^{C}
w_cq_{s,j,c}
$$

---

# 22. Example Weighted Score

Suppose:

| Criterion    | Maximum | Weight | Judge Score |
| ------------ | ------: | -----: | ----------: |
| Technical    |      10 |   0.40 |           8 |
| Innovation   |      10 |   0.30 |           9 |
| UX           |      10 |   0.20 |           7 |
| Presentation |      10 |   0.10 |           8 |

Then:

$$
r=
100[
0.40(8/10)
+
0.30(9/10)
+
0.20(7/10)
+
0.10(8/10)
]
$$

$$
r=81
$$

This raw score is stored before any cross-judge normalization.

---

# 23. Cross-Judge Calibration

For judges `i` and `j`, their overlap set is:

$$
O_{ij}
=
\{s:i,j\text{ both reviewed }s\}
$$

For each shared submission:

$$
d_{ij,s}
=
r_{s,i}-r_{s,j}
$$

Average difference:

$$
\bar d_{ij}
=
\frac{1}{|O_{ij}|}
\sum_{s\in O_{ij}}
(r_{s,i}-r_{s,j})
$$

Interpretation:

If:

$$
\bar d_{ij}>0
$$

judge `i` has, on their common submissions, tended to give higher scores than judge `j`.

If:

$$
\bar d_{ij}<0
$$

judge `i` has tended to give lower scores.

This does **not** establish which judge is correct.

It only measures the observed relative scoring tendency.

---

# 24. Why We Do Not Independently Z-Score Every Judge

A naive normalization would calculate:

$$
z_{s,j}
=
\frac{r_{s,j}-\mu_j}{\sigma_j}
$$

for each judge independently.

The problem is that judges may have reviewed different populations of submissions.

A judge could have a higher average simply because that judge happened to receive stronger submissions.

Therefore:

> **Overall judge averages alone do not establish judge severity.**

The overlap structure gives a stronger source of evidence because judges are being compared on submissions they actually had in common.

Independent z-scoring may be used as a fallback or supplementary statistic, but it should not be the fundamental calibration mechanism.

---

# 25. Global Calibration Through the Overlap Graph

Let:

$$
b_j
$$

be the estimated scoring offset for judge `j`.

For each overlapping judge pair:

$$
b_i-b_j
\approx
\bar d_{ij}
$$

We estimate all judge offsets simultaneously by minimizing:

$$
L_{cal}
=
\sum_{(i,j)\in E_H}
\omega_{ij}
\left[
(b_i-b_j)-\bar d_{ij}
\right]^2
$$

where:

$$
\omega_{ij}=|O_{ij}|
$$

is the number of shared submissions.

The more common submissions two judges have, the more evidence their pairwise relationship contains.

---

# 26. Identifiability Constraint

Only relative offsets matter.

If every judge's offset were increased by the same constant `k`:

$$
b'_j=b_j+k
$$

then:

$$
b'_i-b'_j
=
(b_i+k)-(b_j+k)
=
b_i-b_j
$$

Therefore the offsets have an arbitrary global translation.

To make the solution unique, impose:

$$
\boxed{
\sum_{j=1}^{J}b_j=0
}
$$

This fixes the global reference point.

---

# 27. Normalized Judge Score

Once the calibration offsets are estimated:

$$
n_{s,j}
=
r_{s,j}-b_j
$$

If the scoring system requires the final score to remain within `[0,100]`, clamp the displayed/derived score:

$$
n'_{s,j}
=
\min(100,\max(0,n_{s,j}))
$$

The system must preserve:

```text
raw_score
normalized_score
calibration_version
```

The raw score must never be replaced.

---

# 28. Judge Scale and Spread

Judges may differ not only in average scoring level but also in how much of the scoring range they use.

A general model is:

$$
r_{s,j}
=
\alpha_j+\beta_jq_s+\epsilon_{s,j}
$$

where:

* `α_j` = judge-specific location,
* `β_j` = judge-specific scale,
* `q_s` = latent submission quality,
* `ε_s,j` = residual variation.

The baseline implementation should use transparent overlap-weighted offset calibration.

A scale adjustment can be introduced if there is sufficient overlap data to estimate it reliably.

This is preferable to blindly applying independent z-scores.

---

# 29. Final Submission Score

After normalization, submission `s` has:

$$
n'_{s,1},n'_{s,2},...,n'_{s,R}
$$

The baseline final score is the arithmetic mean:

$$
\boxed{
F_s
=
\frac{1}{R}
\sum_{j=1}^{R}
n'_{s,j}
}
$$

For `R=4`:

$$
F_s
=
\frac{
n'_{s,1}
+n'_{s,2}
+n'_{s,3}
+n'_{s,4}
}{4}
$$

---

# 30. Example Final Score

Suppose a submission receives:

```text
J3  → 82.1
J7  → 79.3
J12 → 86.0
J18 → 81.5
```

Then:

$$
F=
\frac{82.1+79.3+86.0+81.5}{4}
$$

$$
F=82.225
$$

The system displays:

$$
\boxed{82.23}
$$

but stores the unrounded value internally.

---

# 31. Why Arithmetic Mean Is the Baseline

The arithmetic mean is:

* deterministic,
* easy to explain,
* easy to reproduce,
* inclusive of all assigned judges,
* not dependent on arbitrary outlier removal.

The system should not silently remove a judge because their score differs from others.

Any alternative aggregation rule must be explicitly defined and versioned.

---

# 32. Preserve Judge Disagreement

Normalization should correct systematic scoring-scale differences.

It should not erase genuine disagreement.

For submission `s`:

$$
\bar n_s
=
\frac{1}{R}
\sum_jn'_{s,j}
$$

and:

$$
SD_s
=
\sqrt{
\frac{1}{R-1}
\sum_j(n'_{s,j}-\bar n_s)^2
}
$$

The final score may be the mean, while `SD_s` remains available for audit.

For example:

```text
Submission A:
78, 79, 80, 81

Submission B:
55, 70, 91, 96
```

The means may be similar, but the judging agreement is clearly different.

The system should retain this information.

---

# 33. Complete Mathematical Pipeline

### Step 1 — Criterion score

$$
q_{s,j,c}
=
\frac{x_{s,j,c}}{M_c}
$$

### Step 2 — Weighted raw score

$$
r_{s,j}
=
100\sum_cw_cq_{s,j,c}
$$

### Step 3 — Judge overlap

$$
O_{ij}
=
\{s:i,j\text{ both reviewed }s\}
$$

### Step 4 — Pairwise observed difference

$$
\bar d_{ij}
=
\frac{1}{|O_{ij}|}
\sum_{s\in O_{ij}}
(r_{s,i}-r_{s,j})
$$

### Step 5 — Global calibration

$$
\hat b
=
\arg\min_b
\sum_{(i,j)\in E_H}
|O_{ij}|
\left[
(b_i-b_j)-\bar d_{ij}
\right]^2
$$

subject to:

$$
\sum_jb_j=0
$$

### Step 6 — Normalized score

$$
n'_{s,j}
=
\operatorname{clip}(r_{s,j}-\hat b_j,0,100)
$$

### Step 7 — Final score

$$
\boxed{
F_s=
\frac{1}{R}
\sum_{j\in J_s}
n'_{s,j}
}
$$

### Step 8 — Disagreement

$$
SD_s=
\sqrt{
\frac{1}{R-1}
\sum_{j\in J_s}
(n'_{s,j}-F_s)^2
}
$$

---

# 34. No-Track Example

Given:

$$
N=2500
$$

$$
J=20
$$

$$
R=4
$$

Total reviews:

$$
2500\times4=10000
$$

Average workload:

$$
10000/20=500
$$

Total pair-overlap events:

$$
2500\binom42=15000
$$

Possible judge pairs:

$$
\binom{20}{2}=190
$$

Target average pair overlap:

$$
15000/190\approx78.95
$$

Therefore the assignment engine is designed around:

```text
2,500 submissions
20 judges
4 reviews/submission
10,000 total reviews
~500 reviews/judge
~79 shared submissions/judge pair on average
connected judge-overlap graph
```

---

# 35. Track-Constrained Assignment

The track version must use the same core architecture but introduces eligibility constraints.

For a submission `s`:

$$
Eligible(s)
=
\{j\in G:E(j,s)=true\}
$$

The hard review constraint becomes:

$$
|Eligible(s)|\ge R
$$

before assignment is possible.

However, tracks introduce an additional issue:

> **Track constraints can change the structure of the judge-overlap graph.**

For example:

```text
Track A:
J1 J2 J3 J4

Track B:
J5 J6 J7 J8
```

If no judge can evaluate both tracks, there may be no assignment capable of creating overlap between these judge groups.

Therefore the track algorithm must be designed after defining the actual track/judge eligibility model.

It must not simply assume that the no-track connectivity guarantees remain achievable.

---

# 36. What the Track Algorithm Must Preserve

Once the track rules are defined, Algorithm 2 must preserve the core invariants:

```text
exactly R reviews per submission
distinct judges
eligibility
balanced workload
useful overlap
calibration feasibility
auditability
```

But the exact optimization strategy may need to change because eligibility can constrain which overlap relationships are possible.

The track algorithm should therefore be treated as:

$$
\boxed{
\text{Core Assignment Engine}
+
\text{Track Constraints}
+
\text{Track-Aware Connectivity Strategy}
}
$$

rather than as a completely unrelated system.

---

# 37. Security and Judge Isolation

Assignment correctness is not enough.

The backend must enforce:

$$
JudgeAccess(j,s)
\iff
Assigned(j,s)
$$

A judge must only be able to retrieve submissions assigned to them.

The client/UI must not be trusted to enforce this.

A judge must not be able to access:

* another judge's assignments,
* another judge's scores,
* unauthorized aggregate scores,
* submissions outside their assignment,
* sensitive information about other judges' scoring behavior.

---

# 38. Auditability

For every final score, the system should be able to reconstruct:

```text
Submission
    ↓
Assigned judges
    ↓
Raw criterion scores
    ↓
Raw weighted scores
    ↓
Judge overlap
    ↓
Calibration parameters
    ↓
Normalized scores
    ↓
Final score
```

Store:

```text
assignment_version
rubric_version
submission_id
judge_id
criterion_id
raw_score
raw_weighted_score
calibration_version
calibration_parameter
normalized_score
final_score
disagreement_statistics
```

A final score must never be a number that cannot be reconstructed.

---

# 39. Edge Cases

## 39.1 R > J

Invalid:

$$
R>J
$$

There are not enough judges for `R` distinct reviews.

---

## 39.2 Insufficient Eligible Judges

For any submission:

$$
|Eligible(s)|<R
$$

means the assignment is infeasible.

The system must report this explicitly.

---

## 39.3 Disconnected Judge Graph

If:

$$
components(H)>1
$$

the system should attempt assignment repair.

If connectivity cannot be achieved under the current constraints, the system must report that global overlap-based calibration is not identifiable under the current assignment rules.

It must not pretend that calibration has the same reliability as a connected system.

---

## 39.4 Zero Variance

If a judge has:

$$
\sigma_j=0
$$

a z-score would divide by zero.

The implementation must use an explicit fallback and flag the situation.

---

## 39.5 Low Overlap

If two judges have very little shared work, their pairwise calibration estimate has limited evidence.

The system should expose calibration coverage/confidence.

---

## 39.6 Genuine Disagreement

A judge should not automatically be considered wrong because their score differs from another judge.

The raw review must remain available.

---

# 40. Automated Correctness Tests

The assignment engine should automatically test the following.

## Review count

```text
for every submission:
    assigned_judges.count == R
```

## Distinctness

```text
for every submission:
    assigned_judges are unique
```

## Workload

$$
\boxed{
\sum_jW_j=NR
}
$$

## Pair overlap

$$
\boxed{
\sum_{i<j}O_{ij}
=
N\binom{R}{2}
}
$$

## Connectivity

```text
judge_overlap_graph.is_connected() == true
```

when global calibration is required and connectivity is feasible.

## Final score

```text
final_score =
mean(normalized_scores)
```

according to the active scoring version.

---

# 41. Why This Architecture Is Defensible

## 41.1 It is not arbitrary random assignment

The assignment is mathematically constrained.

Every submission receives exactly `R` independent judges while the system simultaneously manages workload and overlap.

---

## 41.2 It creates direct comparison evidence

Judges are compared through work they actually reviewed in common.

This is stronger than simply comparing their global averages.

---

## 41.3 It separates raw evidence from derived results

Raw judging scores are immutable.

Normalization and final scoring are derived values.

Therefore the system can always answer:

> "How did this final score come from the original reviews?"

---

## 41.4 It explicitly models the calibration network

The judge-overlap graph is not an accidental side effect.

It is a designed component of the judging architecture.

That gives the normalization layer a statistical foundation.

---

## 41.5 It does not pretend normalization creates objective truth

The system estimates relative scoring tendencies.

It does not claim that the calibrated score is an objectively perfect measurement of submission quality.

This is an important limitation and makes the methodology more defensible.

---

## 41.6 It preserves disagreement

The system does not hide disagreement by deleting unusual reviews.

It calculates the final score while retaining dispersion information for audit and review.

---

## 41.7 It is scalable

For:

$$
N=2500,\quad J=20,\quad R=4
$$

the system handles:

$$
10000
$$

review assignments and:

$$
15000
$$

pair-overlap events.

The assignment problem is therefore computationally manageable while still providing substantial overlap evidence.

---

# 42. Core Design Principle

The entire system can be summarized as:

$$
\boxed{
Assignment
\rightarrow
Overlap
\rightarrow
Calibration
\rightarrow
Normalization
\rightarrow
Aggregation
}
$$

The central principle is:

> **Assignment creates the evidence needed for normalization.**

The system therefore does not treat assignment as a simple workload-distribution problem.

It treats assignment as the first statistical stage of the judging system.

---

# 43. Final Architecture

```text
                    ORGANIZER
                        │
                        ▼
              Judging Configuration
                        │
              ┌─────────┴─────────┐
              │                   │
          No Tracks            Tracks
              │                   │
              ▼                   ▼
       Core Assignment      Track Constraints
              │                   │
              └─────────┬─────────┘
                        ▼
                 Judge Assignment
                        │
                        ▼
                Multiple Reviews
                        │
                        ▼
              Judge Overlap Graph
                        │
                        ▼
             Cross-Judge Calibration
                        │
                        ▼
                Normalized Scores
                        │
                        ▼
              Final Score Aggregation
                        │
                        ▼
              Audit / Review / Results
```

The no-track algorithm is the baseline implementation.

The track algorithm should be designed next, once the exact rules governing judge-to-track eligibility are finalized.

The two modes should share the same core assignment, overlap, calibration, scoring, security, and audit infrastructure rather than becoming two unrelated systems.
