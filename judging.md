# JUDGING.md — assignment strategy, scoring maths, normalization method

> **Scope:** This document is the definitive end-to-end judging specification for the Dogfood Hackathon platform architecture (`Type 1` Single Pool and `Type 2` Multi-Track).
> It defines the mathematical, statistical, operational, and security mechanisms across two distinct, non-overlapping phases:
> 1. **Phase 1 — Pairwise Relative Triage & Exact-K Shortlisting**
> 2. **Phase 2 — Multi-Judge Rubric Evaluation, Overlap-Based Calibration, & Final Scoring**
> 
> 
> **Phase 1 asks:** *Which submissions possess sufficient relative merit to earn detailed evaluation?*
> **Phase 2 asks:** *Given the shortlisted finalists, what are their absolute, calibrated, and normalized rubric scores?*

---

# 0. System Pipeline & Structural Invariants

```text
ALL VALID SUBMISSIONS (N)
           │
           ├─────────────────────────────────────────┐
           ▼                                         ▼
TYPE 1: NO TRACKS                           TYPE 2: MULTI-TRACK
(Global Submission Pool)                    (Isolated Track Panels)
           │                                         │
           ▼                                         ▼
PHASE 1: PAIRWISE TRIAGE                    INDEPENDENT TRACK EVALUATION
  ├── Anonymization & Hashing                 ├── Track Feasibility Checks
  ├── Balanced Cyclic Assignment              ├── Isolated Panel Assignments
  ├── Regularized Bradley–Terry Fitting       ├── Exact R Reviews & Workloads
  ├── Protected Adaptive Exploration          ├── Track-Scoped Overlap Graph
  └── Exact-K Shortlist Output                └── Track WLS Offset Calibration
           │                                         │
           ▼                                         ├─────────────────────────┐
PHASE 2: DETAILED RUBRIC                             ▼                         ▼
  ├── Feasibility Guards                    TYPE 2A RESULTS           TYPE 2B RESULTS
  ├── Exact Workload Quotas                 (Track Winners Only)      (Overall Finalist Panel)
  ├── Connected Overlap Graph                        │                         │
  ├── Immutable Raw Scoring                          └────────────┬────────────┘
  ├── WLS Judge Offset Calibration                                │
  ├── Single-Aggregate Normalization                              ▼
  └── Deterministic Tie Resolution                       FINALIZED RESULTS

```

## 0.1 Core Invariants

* **Phase Isolation:** Phase 1 latent scores ($\theta$) never serve as Phase 2 rubric scores.
* **Fail-Closed Governance:** Invalid configurations or unresolvable constraints abort explicitly rather than silently degrading statistical models.
* **Workload Parity:** Judge assignment distributions satisfy $\max(\text{load}) - \min(\text{load}) \le 1$ across all execution units.
* **Immutable Evidence:** Raw judge reviews, timestamped submissions, and system snapshots are immutable; calibration modifies derived metrics only.
* **Determinism:** Given identical seeds, algorithm versions, and input datasets, execution produces byte-identical outputs.

---

# PART I — FEASIBILITY & CONFIGURATION GUARDS

Before assignment generation or model fitting occurs in either Phase 1 or Phase 2, the setup parameters are validated against strict feasibility guards to prevent runtime deadlocks, infinite mathematical solutions, or sparse graph disconnections.

## 1. Mathematical Feasibility Guards

Given total submissions $N$, active judge panel $J$, required independent reviews per submission $R$, and Phase 1 shortlist target $K$:

### Guard 1: Active Judge Feasibility

Every submission requires $R$ distinct judges. The system requires $J \ge R$.


$$\text{If } R > J \implies \text{THROW } \text{FEASIBILITY\_TOO\_FEW\_JUDGES}$$

### Guard 2: Workload Capacity & Idle Judge Prevention

Total required review slots $A = N \times R$ must equal or exceed active judges $J$ so that every active judge receives at least one review task.


$$\text{If } (N \times R) < J \implies \text{THROW } \text{FEASIBILITY\_TOO\_MANY\_JUDGES}$$

### Guard 3: Overlap & Calibration Feasibility

For $R \ge 2$, every submission evaluated by $R$ judges creates $\binom{R}{2}$ pair-overlap events. Global additive calibration requires sufficient pairwise overlap events across the panel to form a connected projection graph $G_J$ (requiring at least $J - 1$ edges).


$$\text{If } R \ge 2 \text{ and } N \cdot \frac{R(R - 1)}{2} < (J - 1) \implies \text{THROW } \text{FEASIBILITY\_SPARSE\_OVERLAP}$$

### Guard 4: Single Review Mode Exception ($R = 1$)

When $R = 1$, zero pair-overlap events are created ($\binom{1}{2} = 0$). The system permits execution for simple single-evaluator pass-throughs but bypasses cross-judge WLS calibration ($b_j = 0$ for all judges).

---

# PART II — PHASE 1: PAIRWISE TRIAGE & EXACT-K SHORTLISTING

## 2. Purpose & Principles

Phase 1 rapidly triages a large submission population down to an exact shortlist $K$ for detailed evaluation.

* **Primary Objective:** Minimize the **False Exclusion Rate** (the accidental elimination of genuinely strong submissions due to noisy early comparisons).
* **Forced Choice:** Judges perform binary comparisons between two anonymized submissions ($A$ vs $B$). No numerical scores, ties, skips, or confidence sliders are exposed.

## 3. Assignment Strategy & Deterministic Cyclic Scheduler

Initial Phase 1 comparisons are generated independently of initial submission rankings to avoid feedback loops where early lucky projects hog comparisons.

Logical comparison pairs are generated deterministically and assigned to judges via a hash-seeded cyclic scheduler:


$$\text{preferred\_judge\_index} = \text{hash}(\text{seed} \parallel \text{comparison\_id}) \bmod J$$

The scheduler assigns tasks sequentially, maintaining workload balance $\max(\text{load}) - \min(\text{load}) \le 1$.

## 4. Regularized Bradley–Terry Relative Scoring Model

Pairwise outcomes fit a regularized Bradley–Terry latent strength model. The probability that submission $i$ defeats submission $j$ is:


$$P(i \succ j) = \frac{e^{\theta_i}}{e^{\theta_i} + e^{\theta_j}} = \frac{1}{1 + e^{-(\theta_i - \theta_j)}}$$

### Mathematical Defense of $L_2$ Regularization

In unregularized maximum likelihood estimation, if a submission wins all its comparisons ($3\text{ wins}, 0\text{ losses}$), the likelihood maximization drives $\theta_i \to +\infty$. Conversely, winless projects drive $\theta_i \to -\infty$, causing solver divergence or floating-point overflow.

Phase 1 solves a regularized objective with parameter $\lambda = 0.1$:


$$\max_{\boldsymbol{\theta}} \left[ \sum_{(i \succ j)} \log \left( \frac{1}{1 + e^{-(\theta_i - \theta_j)}} \right) - \lambda \sum_{k=1}^N \theta_k^2 \right] \quad \text{subject to} \quad \sum_{k=1}^N \theta_k = 0$$

* **Mathematical Guarantees:**
1. **Coercivity:** The penalty term $-\lambda \Vert{}\boldsymbol{\theta}\Vert{}_2^2 \to -\infty$ as $\Vert{}\boldsymbol{\theta}\Vert{} \to \infty$, forcing the objective peak to remain within a finite boundary.
2. **Strict Concavity:** Adding $2\lambda \mathbf{I}$ to the Hessian matrix guarantees a strictly positive definite system, proving the existence of **exactly one unique, finite global solution**.


## 5. Protected Adaptive Exploration & Graph Protection

Remaining comparison budgets are allocated dynamically to protect top contenders and resolve the shortlist boundary $K$.

### Adaptive Priority Formula

Candidate pairs $(i, j)$ are evaluated using normalized component scores in $[0, 1]$:

$$P_{ij} = 0.40 \cdot S_{ij} + 0.30 \cdot B_{ij} + 0.20 \cdot I_{ij} + 0.10 \cdot E_{ij}$$

* **$S_{ij}$ (Strong-Project Protection):** Measures whether upper evidence bounds reach the boundary cutoff strength $\theta_K$. Single-submission bounds $S_i = \max(0, \theta_i + 1.96 \cdot \text{SE}_i - \theta_K)$ are mapped to candidate pairs via arithmetic mean:

$$S_{ij} = \frac{S_i + S_j}{2}$$

* **$B_{ij}$ (Boundary Relevance):** Concentrates evidence around the cutoff threshold $\theta_K$. Single-submission relevance $B_i = \exp \left( -\frac{|\theta_i - \theta_K|}{\text{median\_SE}} \right)$ is mapped to candidate pairs via arithmetic mean:

$$B_{ij} = \frac{B_i + B_j}{2}$$

* **$I_{ij}$ (Information Value):** Maximizes variance gain where the predicted pairwise outcome $p_{ij} \approx 0.5$:

$$I_{ij} = 4 \cdot p_{ij}(1 - p_{ij})$$

* **$E_{ij}$ (Evidence Balancing):** Prioritizes pairs with lower combined total comparison counts $C_i$ and $C_j$:

$$E_{ij} = \frac{1}{1 + C_i + C_j}$$

### Graph-Connectivity Override

If adaptive priority focuses comparisons within localized score clusters, the bipartite graph risks splitting into disconnected components ($C > 1$). Whenever $C > 1$, the priority $P_{ij}$ is overridden to force comparison between disconnected components, guaranteeing global graph connectivity ($C = 1$).

## 6. Phase 1 Stability & Finalization

Phase 1 finalizes when the shortlist boundary separation $\Delta_K = \theta_K - \theta_{K+1}$ satisfies:


$$\Delta_K > 1.96 \cdot \sqrt{\text{SE}_K^2 + \text{SE}_{K+1}^2}$$


If the budget exhausts prior to reaching stability, the system flags `SHORTLIST_BOUNDARY_UNCERTAIN` and selects the top $K$ submissions using the model estimate, breaking exact ties with a SHA-256 canonical hash key.

---

# PART III — PHASE 2: RUBRIC EVALUATION & OVERLAP-BASED CALIBRATION

## 7. Purpose & Inputs

Phase 2 evaluates the $K$ shortlisted finalists using structured rubrics, assigning exactly $R$ independent judges per submission.

* **Inputs:** Shortlisted submissions $K$, judge panel $J$, review count $R$, rubric weights $w_c$, deterministic seed.
* **Total Assignments:** $A = K \times R$.

## 8. Exact Workload & Overlap Assignment Strategy

Workloads are assigned deterministically:


$$\text{baseQuota} = \left\lfloor \frac{K \cdot R}{J} \right\rfloor, \quad \text{extra} = (K \cdot R) \bmod J$$


Exactly $\text{extra}$ judges receive $\text{baseQuota} + 1$; all others receive $\text{baseQuota}$.

### Overlap Objective

For judges $i$ and $j$, let $O_{ij}$ be the number of shared submissions. The assignment engine minimizes pair-overlap variance against the target $P$:


$$P = \frac{K \cdot \binom{R}{2}}{\binom{J}{2}}, \quad \text{PairLoss} = \sum_{i < j} (O_{ij} - P)^2$$


Candidate judge groups are generated via greedy deterministic construction and refined using deterministic local swaps, preserving $\max(\text{load}) - \min(\text{load}) \le 1$ and graph connectivity $C = 1$.

## 9. Immutable Raw Rubric Scoring

For a submission $s$ and judge $j$, given criterion score $\text{score}(s,j,c)$, criterion maximum $\max_c$, and weight $w_c$ where $\sum w_c = 1$:


$$r(s,j) = 100 \cdot \sum_c w_c \cdot \frac{\text{score}(s,j,c)}{\max_c}$$


This produces a raw weighted review score $0 \le r(s,j) \le 100$. Raw scores are saved immutably.
s
## 10. Overlap-Based Additive Judge Calibration (WLS)

Judges exhibit inherent leniency or severity tendencies. Rather than assuming judges receive identical submission quality distributions (which invalidates standard z-score normalization), the system estimates relative judge offsets $b_j$ using shared submissions.

### Pairwise Evidence Aggregation

For all pairs of judges $(i, j)$ with at least one shared submission ($\omega_{ij} \ge 1$):

$$\bar{d}_{ij} = \frac{1}{\omega_{ij}} \sum_{s \in \text{Shared}} (r(s,i) - r(s,j))$$

where $\omega_{ij}$ is the exact count of shared submissions evaluated by both Judge $i$ and Judge $j$.

> **Connectivity & Edge Preservation Rule:** Every shared submission pair ($\omega_{ij} \ge 1$) forms a valid edge in the projection graph $G_J$. Filtering or dropping edges where $\omega_{ij} = 1$ is strictly forbidden, as doing so can disconnect $G_J$ even when Guard 3 passes.

### Weighted Least Squares (WLS) Objective

Judge offsets $b_j$ are solved by minimizing:

$$\min_{\mathbf{b}} \sum_{i < j} \omega_{ij} \left( (b_i - b_j) - \bar{d}_{ij} \right)^2 \quad \text{subject to} \quad \sum_{j=1}^J b_j = 0$$

### Mathematical Proof of WLS Weighting ($w_{ij} = \omega_{ij}$)

Let $r(s,j) = \theta_s + b_j + \epsilon_{s,j}$ with independent, homoscedastic review noise $\epsilon_{s,j} \sim \mathcal{N}(0, \sigma^2)$.

The variance of the difference on a single shared submission is:

$$\text{Var}(r(s,i) - r(s,j)) = \text{Var}(\epsilon_{s,i}) + \text{Var}(\epsilon_{s,j}) = 2\sigma^2$$

The sample mean difference $\bar{d}_{ij}$ across $\omega_{ij}$ shared submissions has variance:

$$\text{Var}(\bar{d}_{ij}) = \frac{2\sigma^2}{\omega_{ij}}$$

In Gauss-Markov Weighted Least Squares, the optimal weight assigned to residual $((b_i - b_j) - \bar{d}_{ij})$ is the inverse variance:

$$w_{ij} = \frac{1}{\text{Var}(\bar{d}_{ij})} = \frac{\omega_{ij}}{2\sigma^2} \propto \omega_{ij}$$

Omitting the global constant factor $2\sigma^2$ proves that weighting each edge linearly by its overlap count $\omega_{ij}$ (including $\omega_{ij} = 1$) is the mathematically optimal WLS formulation.
## 11. Normalization & Single-Aggregate Clamping

### Uncapped Review Score

For judge $j$ with estimated offset $b_j$:


$$n(s,j) = r(s,j) - b_j$$

### Mathematical Defense of Single-Aggregate Clamping

Clamping individual reviews ($n'(s,j) = \text{clamp}(n(s,j), 0, 100)$) before averaging introduces non-linear threshold compression near boundaries, distorting standard deviation metrics and introducing false score ties.

Instead, individual calibrated review scores $n(s,j)$ **remain uncapped** during aggregation:


$$F_{\text{raw}}(s) = \frac{1}{R} \sum_{j=1}^R n(s,j)$$

The published score $F_s$ is clamped exactly once:


$$F_s = \min \left( 100, \max \left( 0, F_{\text{raw}}(s) \right) \right)$$

### Ranking Invariant

Official contest rankings use full double-precision $F_{\text{raw}}(s)$, **not** $F_s$. This guarantees that a submission with $F_{\text{raw}} = 104.2$ ranks above a submission with $F_{\text{raw}} = 101.5$, even though both present a published score of $100.0$.

## 12. Disagreement Metrics & Exact Tie Resolution

Judge disagreement is tracked as a diagnostic metric using sample standard deviation:


$$\text{SD}_s = \sqrt{\frac{\sum_{j=1}^R (n(s,j) - F_{\text{raw}}(s))^2}{R - 1}}$$


High disagreement triggers audit flags but does not automatically penalize submission scores.

### Exact Tie Resolution

If $F_{\text{raw}}(A) = F_{\text{raw}}(B)$ at full double precision (IEEE 754 float64), ties are broken using a deterministic, canonical SHA-256 hash key:


$$\text{tie\_key} = \text{SHA-256}(\text{canonical\_encode}(\text{competition\_id}, \text{judging\_version}, \text{submission\_id}))$$


Ordering by byte-sequence of `tie_key` guarantees 100% reproducible tie resolution without relying on unseeded randomness or database insertion order.

---

# PART IV — TYPE 2: MULTI-TRACK ARCHITECTURE

In a `Type 2` competition, submissions are partitioned into distinct tracks or categories.

```text
                               TYPE 2 ARCHITECTURE
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼                            ▼                            ▼
   TRACK A SUBMISSIONS          TRACK B SUBMISSIONS          TRACK C SUBMISSIONS
           │                            │                            │
     Panel A Judges               Panel B Judges               Panel C Judges
           │                            │                            │
  Track A Assignment           Track B Assignment           Track C Assignment
           │                            │                            │
   Track A Reviews              Track B Reviews              Track C Reviews
           │                            │                            │
  WLS Calibration (A)          WLS Calibration (B)          WLS Calibration (C)
  (sum b_j,A = 0)              (sum b_j,B = 0)              (sum b_j,C = 0)
           │                            │                            │
    Track A Ranking              Track B Ranking              Track C Ranking
           │                            │                            │
           ├────────────────────────────┴────────────────────────────┤
           ▼                                                         ▼
  TYPE 2A: TRACK WINNERS                                    TYPE 2B: OVERALL STAGE
  (Final Track Results)                                     (Top Track Finalists)
                                                                     │
                                                            Fresh Overall Panel
                                                                     │
                                                            Independent Calibration
                                                                     │
                                                            Overall Winner Results

```

## 13. Track Scope Isolation

The fundamental principle of `Type 2` judging is:

> **A track is an isolated judging universe.**

* **Isolated Panels:** Judges assigned to Panel A evaluate only Track A submissions.
* **Isolated Calibration:** Calibration parameters are calculated independently per track panel:

$$\sum_{j \in \text{Panel A}} b_{j,A} = 0, \quad \sum_{j \in \text{Panel B}} b_{j,B} = 0$$


* **No Cross-Track Normalization:** Raw or calibrated scores are never compared directly across tracks because different track panels evaluate distinct submission populations without shared-review overlap bridges.

## 14. Type 2A vs. Type 2B Variants

### Type 2A — Track Winners Only

Judging terminates at independent track rankings. Each track publishes its own winners based on $F_{\text{raw}}$ within that track scope.

### Type 2B — Overall Winner Stage

Top-ranked finalists from each track advance to a secondary **Overall Judging Scope**:

1. **Fresh Scope:** Track scores do not carry over into numerical overall scores.
2. **Dedicated Overall Panel:** A secondary panel evaluates overall finalists under a fresh Phase 2 instance.
3. **Independent Calibration:** Overall judge offsets are calibrated exclusively using reviews submitted within the overall panel.

---

# PART V — DEFENSE, SECURITY & AUDITABILITY

## 15. Architectural Defense Against Attack Vectors

### Attack Vector 1: Collusive / Extreme Scoring (Phase 2)

* **Threat:** A rogue judge scores target submission $S^*$ as $100/100$ and all competing submissions as $0/100$ to inflate $S^*$.
* **System Defense:** In the WLS linear solve $\mathbf{L} \mathbf{b} = \mathbf{d}$, an extreme score drop creates massive pairwise residuals $\bar{d}_{ij}$. This drives the rogue judge's offset $b_{\text{rogue}}$ significantly negative, reducing their net influence across all assignments. The system logs `EXTREME_OFFSET` ($\vert{}b_j\vert{} > 20.0$) and `COMPRESSED_RANGE` for organizer investigation.

### Attack Vector 2: Unlucky Pairing / Bad Anchor (Phase 1)

* **Threat:** A mediocre project wins early comparisons purely because it was paired against exceptionally weak projects.
* **System Defense:** Bradley–Terry modeling evaluates opponent strength rather than raw win rate. Defeating weak opponents yields minimal increase in $\theta_i$. Furthermore, protected exploration ($S_{ij}$) mandates that any project entering the shortlist boundary must face high-ranked opponents before advancement.

### Attack Vector 3: Mid-Event Judge Dropout

* **Threat:** A judge drops out midway through Phase 2, severing the overlap projection graph ($C > 1$).
* **System Defense:** If an active judge is removed, completed valid reviews remain immutable. Unfinished review slots are reallocated across active judges via deterministic cyclic capacity repair. If the overlap graph becomes disconnected, the WLS solver falls back to component-wise calibration across connected subgraphs rather than aborting.

## 16. Auditability & Reconstructability Record

Every competition finalization snapshot stores the following immutable audit record:

```text
FINALIZATION SNAPSHOT RECORD
├── competition_id
├── algorithm_version
├── assignment_seed
├── feasibility_validation_logs
├── phase1_audit
│   ├── raw_pairwise_outcomes
│   ├── regularized_bt_theta_estimates
│   └── boundary_stability_status
├── phase2_audit
│   ├── raw_criterion_scores
│   ├── wls_judge_offsets_b
│   ├── overlap_matrix_omega
│   ├── uncapped_raw_scores_F_raw
│   └── published_clamped_scores_F_s
└── SHA256_snapshot_hash

