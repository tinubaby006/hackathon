# DOGFOOD HACKATHON

# TIER 3 — PUBLIC VOTING & COMMUNITY IMPLEMENTATION PLAN

## Agent Implementation Prompt & Tier 3 Execution Plan

> **Purpose:** This document is the authoritative implementation contract for building **Tier 3 — Public Voting & Community Participation** of the DOGFOOD HACKATHON platform.
>
> You are the senior implementation agent responsible for building a **real, production-shaped Tier 3 community and voting system**, not a mock/demo.
>
> The implementation must integrate cleanly with the existing Tier 1 (Core) and Tier 2 (Judging) systems, preserve local-first self-hosting (`docker compose up`), enforce strict backend authorization, and satisfy all requirements specified by `specifications.md` and `https://dogfoodhack.com/`.

---

# 0. SOURCE OF TRUTH & HIERARCHY OF CONSTRAINTS

Before writing code, inspect the repository thoroughly and conform to the following priority hierarchy:

```text
1. DOGFOOD Specification & Rules (https://dogfoodhack.com/ & specifications.md)
2. Existing Repository Architecture & Data Model (T1 Core & T2 Judging contracts)
3. JUDGING.md (Statistical and architectural invariants)
4. Tier 3 Implementation Plan (This document)
5. Standard Software Engineering & Security Best Practices
```

### Absolute Non-Negotiables:
1. **Never Break Tier 1 or Tier 2:** Existing authentication, team formation, project submission, deadline locking, and Tier 2 WLS judge rubric normalization must remain 100% operational.
2. **Self-Hosted & Offline First:** The platform must execute locally using `docker compose up` with zero dependencies on external cloud services, hosted databases, third-party auth, or SaaS APIs.
3. **Backend Authorization over UI Hiding:** Hiding UI controls is not security. All voting result privacy, rate limits, audit logs, and moderation controls MUST be enforced at the server/API layer.
4. **No Synthetic/Dummy Fallbacks:** Do not swallow security exceptions, mask fraud, or return dummy fallback votes.

---

# 1. IMPLEMENTATION GOAL

Build a complete, locally runnable Tier 3 community participation engine supporting this extended lifecycle:

```text
Approved Event (Tier 1)
      │
      ▼
Organizer configures Tier 3 Community Voting & Comments
      │
      ▼
Public Ballot (Session-Randomized Order & Stable Pagination)
      │
      ├──────────────────────────────┐
      ▼                              ▼
Project Comments               Community Voting
(Auth / Verified Users)     (Open / Email / Auth Modes)
      │                              │
      ▼                              ▼
Moderation Pipeline           Voting Engine (1-Person-1-Vote or Quadratic)
(Flag, Hide, Mute)                   │
                                     ▼
                             Anti-Abuse Engine
                       (Rate Limit, Duplicate Check, Risk Score)
                                     │
                                     ▼
                           Immutable Audit Trail
                                     │
                      ┌──────────────┴──────────────┐
                      ▼                             ▼
            Organizer Audit UI             Vote Result Privacy
          (Live Audit & Override)         (Hidden while Active)
```

---

# 2. CRITICAL CONSTRAINTS & HARNESS REALITIES

## 2.1 Official Test Runner Limitation (`run.py`) & Config (`.dogfood.toml`)

> **Testing Harness & Auth Constraints:**
> * The official hackathon test suite (`run.py`) executes **7 automated checks covering Tier 1 and Tier 2 functionality ONLY**. `run.py` contains NO official automated check for Tier 3.
> * **Do NOT add `"T3"` to claimed tiers in `.dogfood.toml`** until Section 8's acceptance checklist is 100% green.
> * **Configured Auth Non-Collision:** Whatever authentication credential format `.dogfood.toml` points `run.py` at for actors (`organizer`, `judge_a`, `judge_b`, `participant` — e.g. cookies, bearer tokens, basic auth, or custom headers) MUST continue working unmodified once Tier 3 voter session handling (cookies, OTP tokens, fingerprint hashes) is added.
> * The coding agent MUST write custom self-contained test suites (Vitest unit tests, Playwright E2E tests, and a custom runner script `scripts/test-tier3.ts`) to verify all Tier 3 requirements.
> * The agent MUST execute `python run.py` to ensure all 7 existing T1/T2 checks continue to PASS with 100% zero regressions.

## 2.2 Shared Fixture Limitation (`fixtures.json`)

> **Fixture Data Constraint:**
> * The shared dataset (`fixtures.json`) ships data **strictly for Tier 1 and Tier 2** (`events`, `tracks`, `judges`, `teams`, `projects`, `scores`).
> * **`fixtures.json` contains NO voter or vote data for Tier 3.**
> * The coding agent MUST NOT expect `fixtures.json` to populate Tier 3 tables (`voting_configurations`, `voters`, `votes`, `vote_audit_logs`, `project_comments`).
> * The agent MUST create custom application seed data (`src/db/seed-tier3.ts` or extend `src/db/seed.ts`) to populate sample voting configurations, quadratic budgets, simulated votes, and comments for local development and Playwright E2E testing.
> * Loading canonical T1/T2 fixtures during test execution must continue to work cleanly without foreign key or schema conflicts.

## 2.3 Local & Self-Hosted Security Controls
* **Captcha & Bot Defense:** Do NOT depend on external Cloudflare Turnstile or Google reCAPTCHA. Implement a self-contained local visual/math challenge fallback (e.g., SVG arithmetic captcha or cryptographic proof-of-work header) when open voting is enabled.
* **Email Verification in Local Mode:** For email-gated voting, when external SMTP is unconfigured, verification codes/tokens MUST be logged to the server console and stored in SQLite for easy local test execution.

## 2.4 Comprehensive Security Specification & Threat Model

> **VITAL SECURITY REQUIREMENT:** The coding agent MUST enforce all server-side security controls below. Frontend hiding is strictly NOT authorization.

1. **CSRF Protection & Secure Cookies:** All state-changing endpoints (`POST`, `PUT`, `DELETE`) MUST validate a cryptographically secure CSRF token tied to the session. Session cookies MUST use `HttpOnly`, `SameSite=Lax` (or `Strict`), and `Path=/`.
2. **Input Sanitization & XSS Prevention:** Public project comments MUST be sanitized server-side (HTML entity encoding / DOMPurify) before database insertion to prevent stored XSS attacks.
3. **Parameter Tampering & Negative Vote Rejection:** Server validation MUST enforce integer constraints $\text{vote\_weight}_i \ge 0$. Negative vote weights, non-integer inputs, or quadratic point budget overflows ($\sum v_i^2 > B$) MUST be rejected immediately with `HTTP 400 Bad Request`.
4. **Self-Voting Blocking:** In `AUTHENTICATED` voting mode, the backend MUST verify `voter.userId !== project.authorId` and that the voter is not a member of the project team. Self-voting attempts MUST return `HTTP 403 Forbidden`.
5. **IDOR & Organizer Privilege Scoping:** Organizer endpoints (`/api/organizer/...`) MUST verify server-side that the authenticated user holds explicit organizer permissions for the specific `event_id` in the route before allowing configuration updates, audit log views, or bulk vote overrides.
6. **OTP Token Security & Rate Limiting:** Email OTP verification tokens MUST be 6 digits, cryptographically random, expire in 10 minutes, lock out after 5 failed validation attempts, and invalidate immediately upon successful verification. OTP request endpoints MUST be rate-limited (max 3 requests/min per IP).
7. **Vote Result Privacy Leak Seals:** Public endpoints (`GET /api/events/[id]/ballot`, `GET /api/projects/[id]`) MUST strip all vote tallies and rankings server-side while `voting_status == 'ACTIVE'`. Aggregate queries by non-organizers MUST yield `HTTP 403 Forbidden`.
8. **Parameterized SQL Queries:** All database access MUST use Drizzle ORM parameterized queries or SQLite prepared statements to prevent SQL injection.

---

# 3. ROLE MODEL & VOTER IDENTITY ARCHITECTURE

Preserve the core platform account roles:

```text
Visitor
Participant
Judge
Organizer
Admin
```

## 3.0 Voter Identity Model vs. Platform Account Roles

> **Architectural Rule:**
> **Voter is NOT a separate permanent user account role** in `users.role` (which strictly remains: `VISITOR`, `PARTICIPANT`, `JUDGE`, `ORGANIZER`, `ADMIN`).
>
> Instead, **Voter is an event-scoped identity session** represented in the `voters` database table.
>
> * **Participants as Voters:** Registered participants (hackers/builders) are eligible to participate in community voting.
> * **Voters Extend Beyond Participants:** Depending on the organizer's configured voting mode (Open Link or Email-Gated), unauthenticated visitors and general community members can act as voters without requiring a `PARTICIPANT` user profile.
> * **Session-Scoped Identity:** A `voter` database record links either to an authenticated user account (`userId`) or to a verified transient session (`identityHash` / `email`).

---

# 4. DATA MODEL SPECIFICATION

Extend the existing database schema (`src/db/schema.ts`) with five new tables.

## 4.1 Table: `voting_configurations`
Stores Tier 3 voting parameters per event. One configuration per event.

| Field | Type | Modifiers | Description |
| :--- | :--- | :--- | :--- |
| `id` | text | primary key | Unique UUID |
| `event_id` | text | not null, foreign key -> `events.id` (cascade) | Target event |
| `mode` | text | not null, enum (`OPEN`, `EMAIL_GATED`, `AUTHENTICATED`) | Identity requirement |
| `mechanism` | text | not null, enum (`ONE_PERSON_ONE_VOTE`, `QUADRATIC`) | Voting algorithm |
| `max_votes_per_voter` | integer | not null, default 1 | Allowed votes in One-Person-One-Vote mode |
| `quadratic_budget` | integer | not null, default 100 | Point credit budget in Quadratic mode |
| `voting_status` | text | not null, enum (`DRAFT`, `ACTIVE`, `PAUSED`, `FINALIZED`) | Lifecycle state |
| `start_at` | text | optional | Voting start timestamp (ISO string) |
| `end_at` | text | optional | Voting end timestamp (ISO string) |
| `created_at` | text | not null | Creation timestamp |
| `updated_at` | text | not null | Last update timestamp |

## 4.2 Table: `voters`
Stores verified voter sessions for an event.

| Field | Type | Modifiers | Description |
| :--- | :--- | :--- | :--- |
| `id` | text | primary key | Unique UUID |
| `event_id` | text | not null, foreign key -> `events.id` (cascade) | Event scope |
| `identity_type` | text | not null, enum (`OPEN`, `EMAIL`, `AUTH`) | Voter identity origin |
| `identity_hash` | text | not null | `SHA256(IP + UserAgent)` or `SHA256(email)` or `userId` |
| `user_id` | text | optional, foreign key -> `users.id` (set null) | Platform user account link (if AUTH mode) |
| `email` | text | optional | Email address (if EMAIL mode) |
| `verification_token` | text | optional | 6-digit OTP or verification hash |
| `verified_at` | text | optional | Verification timestamp |
| `created_at` | text | not null | Creation timestamp |

## 4.3 Table: `votes`
Stores individual project vote allocations cast by verified voters.

| Field | Type | Modifiers | Description |
| :--- | :--- | :--- | :--- |
| `id` | text | primary key | Unique UUID |
| `event_id` | text | not null, foreign key -> `events.id` (cascade) | Event scope |
| `voter_id` | text | not null, foreign key -> `voters.id` (cascade) | Voter session |
| `project_id` | text | not null, foreign key -> `projects.id` (cascade) | Voted project |
| `vote_weight` | integer | not null, default 1 | Allocated votes count for this project |
| `points_spent` | integer | not null, default 1 | Voice credits spent ($v^2$ in QV, $v$ in 1-Person-1-Vote) |
| `status` | text | not null, enum (`VALID`, `FLAGGED`, `INVALIDATED`) | Vote state |
| `created_at` | text | not null | Timestamp |

## 4.4 Table: `vote_audit_logs`
Immutable audit log recording every vote attempt, fraud evaluation, and administrative override.

| Field | Type | Modifiers | Description |
| :--- | :--- | :--- | :--- |
| `id` | text | primary key | Unique UUID |
| `event_id` | text | not null, foreign key -> `events.id` (cascade) | Event scope |
| `voter_id` | text | optional, foreign key -> `voters.id` (set null) | Voter session |
| `ip_hash` | text | not null | `SHA256(IP)` for subnet analysis |
| `user_agent_hash` | text | not null | `SHA256(UserAgent)` |
| `action` | text | not null | `VOTE_SUBMITTED`, `VOTE_FLAGGED`, `VOTE_REJECTED`, `ADMIN_INVALIDATED` |
| `risk_score` | integer | not null, default 0 | Fraud Risk Engine score (0 to 100) |
| `risk_factors` | text | optional | JSON array of triggered risk rule names |
| `metadata` | text | optional | JSON string containing vote payload snapshot |
| `created_at` | text | not null | Log timestamp |

## 4.5 Table: `project_comments`
Public comments and nested replies on project showcase pages.

| Field | Type | Modifiers | Description |
| :--- | :--- | :--- | :--- |
| `id` | text | primary key | Unique UUID |
| `project_id` | text | not null, foreign key -> `projects.id` (cascade) | Project scope |
| `user_id` | text | optional, foreign key -> `users.id` (set null) | Comment author (if authenticated) |
| `voter_id` | text | optional, foreign key -> `voters.id` (set null) | Comment author (if verified voter) |
| `parent_id` | text | optional, foreign key -> `project_comments.id` | Parent comment ID for replies |
| `content` | text | not null | Comment text (Sanitized HTML/Markdown) |
| `status` | text | not null, enum (`VISIBLE`, `HIDDEN`, `FLAGGED`) | Moderation state |
| `created_at` | text | not null | Creation timestamp |
| `updated_at` | text | not null | Last edit timestamp |

---

# 5. DOMAIN FEATURES & ALGORITHMIC MECHANICS

## 5.1 Community Voting Identity Modes
Organizers configure one of three identity modes per event:
1. **Open Link Mode:** Anyone with a public link can vote. Identity is derived via client fingerprint hash (`SHA-256(IP + UserAgent + Cookie)`).
2. **Email-Gated Mode:** Voter inputs an email address, receives a 6-digit OTP token via email (logged to server console when external SMTP is unconfigured), and validates identity before submitting votes.
3. **Authenticated Mode:** Only logged-in platform account holders can vote. Self-voting by team members on their own project is blocked server-side (`voter.userId !== project.authorId`).

## 5.2 Voting Mechanics & Mathematical Validation

### Mechanism A: One-Person-One-Vote
* Each verified voter receives a fixed vote allowance $N$ (default $N=1$ or $N=3$).
* Constraints enforced on ballot submission:
  $$\sum_{i} \text{vote\_weight}_i \le N \quad \text{and} \quad \text{vote\_weight}_i \in \{0, 1\} \quad \forall i$$

### Mechanism B: Quadratic Voting (QV)
* Each verified voter receives a voice credit budget $B$ (default $B = 100$).
* Allocating $v_i$ votes to project $i$ incurs a point cost of $v_i^2$.
* Constraints enforced on ballot submission:
  $$\text{points\_spent}_i = v_i^2 \quad \text{and} \quad \sum_{i} v_i^2 \le B$$

## 5.3 Voting Result Privacy (Dark Period / Blind Voting)
While voting is `ACTIVE`, vote tallies and community rankings are strictly private:
* Public ballot queries (`GET /api/events/[id]/ballot`) omit `voteCount` and community rank fields.
* **Existing T1 Route Seam Protection:** The pre-existing Tier 1 gallery detail endpoint `GET /api/projects/[id]` MUST ALSO omit/strip `voteCount`, `votes`, and community rank metrics while `voting_status == 'ACTIVE'`.
* Direct API calls attempting to query aggregate vote counts yield HTTP `403 Forbidden` for non-organizers.
* Organizers and Admins retain real-time access to live metrics via `/api/organizer/events/[id]/voting/dashboard`.
* Upon transition to `FINALIZED`, vote tallies become publicly readable across all endpoints.

## 5.4 Session-Randomized Ballot Order & Position Bias Mitigation
To prevent top-of-list position bias:
* **Session Seed Derivation:** Generate a deterministic seed per voter session:
  $$\text{ballot\_seed} = \text{SHA-256}(\text{voter\_session\_id} \parallel \text{event\_id})$$
* **Fisher-Yates Shuffle:** Use `ballot_seed` to reproducibly shuffle project IDs for paginated ballot requests.
* **Pagination Stability:** A voter navigating across pages (e.g. Page 1 to Page 2) sees a consistent, non-duplicating sequence of projects.

## 5.5 Anti-Abuse, Fraud Scoring & Risk Engine

Every vote attempt undergoes multi-stage server-side evaluation:

```text
Vote Submission ──► Rate Limiter ──► Duplicate Check ──► Fraud Risk Engine ──► Audit Log & State
```

1. **Sliding Window Rate Limiter:** Max 5 vote submissions per minute per IP / fingerprint. Exceeding yields HTTP `429 Too Many Requests`.
2. **Duplicate Vote Check:** Checks if `identity_hash` or `email` has already submitted a ballot for the event. Duplicate attempts yield HTTP `400 Bad Request`.
3. **Fraud Risk Scoring Engine:** Computes a composite risk score $R \in [0, 100]$:
   * **IP Subnet Velocity ($+30\text{ pts}$):** $>10$ votes from the same `/24` IP range within 5 minutes.
   * **Disposable Email Domain ($+40\text{ pts}$):** Email domain matches temporary email blocklist.
   * **User-Agent Anomaly ($+20\text{ pts}$):** Missing, headless, or script-like User-Agent string.
   * **Rapid Sequence ($+20\text{ pts}$):** Vote submitted $<3$ seconds after loading ballot page.
4. **Automated Action Based on Risk Score:**
   * $R < 50$: Vote accepted (`status = 'VALID'`).
   * $50 \le R < 80$: Vote accepted but flagged (`status = 'FLAGGED'`). Excluded from live tallies until approved by organizer.
   * $R \ge 80$: Vote rejected with HTTP `422 Unprocessable Entity` (`status = 'REJECTED'`).

## 5.6 Project Comments & Moderation
* Authenticated users or verified voters can post top-level comments and nested replies on public project showcase pages.
* Rate limited to max 3 comments per minute.
* Organizers and Admins can hide comments, flag abusive content, or mute problematic voters per event.

---

# 6. REST API ROUTE SPECIFICATION

## Public / Voter Endpoints

### 1. `GET /api/events/[id]/ballot`
* **Query Parameters:** `page` (default 1), `limit` (default 10)
* **Headers/Cookies:** Voter session cookie / fingerprint header
* **Behavior:** Returns session-randomized paginated project list.
* **Privacy Rule:** Omits `voteCount` and community rank when `voting_status == 'ACTIVE'`.

### 2. `GET /api/projects/[id]` (Existing Tier 1 Route Seam Protection)
* **Behavior:** Returns project details.
* **Privacy Rule:** MUST strip `voteCount`, `votes`, and community rank fields when `voting_status == 'ACTIVE'`.

### 3. `POST /api/events/[id]/voter/request-verification`
* **Request Body:** `{ email: string }`
* **Behavior:** Generates 6-digit OTP verification token. Logs token to console in local dev mode.
* **Response:** `{ success: true, message: "Verification token sent" }`

### 4. `POST /api/events/[id]/voter/verify`
* **Request Body:** `{ email: string, token: string }`
* **Behavior:** Validates OTP token, creates/fetches `voters` record, sets voter session cookie.
* **Response:** `{ success: true, voterId: string }`

### 5. `POST /api/events/[id]/votes`
* **Request Body:** `{ allocations: Array<{ projectId: string, voteWeight: number }> }`
* **Behavior:** Enforces identity check, voting mechanism math (1-person-1-vote or QV), rate limits, and Fraud Risk Engine scoring.
* **Response:** `{ success: true, status: 'VALID' | 'FLAGGED', riskScore: number }`

### 6. `GET /api/projects/[id]/comments`
* **Behavior:** Returns visible comments and nested replies for the project.

### 7. `POST /api/projects/[id]/comments`
* **Request Body:** `{ content: string, parentId?: string }`
* **Behavior:** Creates new comment (Rate limited to max 3/min).

## Organizer & Admin Endpoints (Auth Required)

### 1. `GET /api/organizer/events/[id]/voting/config`
* **Authorization:** Organizer / Admin
* **Behavior:** Returns current voting configuration.

### 2. `PUT /api/organizer/events/[id]/voting/config`
* **Authorization:** Organizer / Admin
* **Request Body:** `{ mode, mechanism, maxVotesPerVoter, quadraticBudget, votingStatus, startAt, endAt }`
* **Behavior:** Updates voting configuration.

### 3. `GET /api/organizer/events/[id]/voting/dashboard`
* **Authorization:** Organizer / Admin
* **Behavior:** Returns real-time vote tallies, voter turnout counts, flagged vote counts, and top community rankings.

### 4. `GET /api/organizer/events/[id]/audit-logs`
* **Authorization:** Organizer / Admin
* **Query Parameters:** `riskMin`, `action`, `page`, `limit`
* **Behavior:** Returns paginated audit trail logs.

### 5. `POST /api/organizer/events/[id]/votes/bulk-override`
* **Authorization:** Organizer / Admin
* **Request Body:** `{ voterIds?: string[], ipHash?: string, action: 'INVALIDATE' | 'APPROVE' }`
* **Behavior:** Bulk overrides vote statuses and appends an entry to audit logs.

### 6. `GET /api/organizer/events/[id]/votes/export-csv`
* **Authorization:** Organizer / Admin
* **Behavior:** Streams CSV file containing raw votes, voter identity metadata, and risk scores.

---

# 7. STEP-BY-STEP IMPLEMENTATION TASKS

1. **Database Schema & Migrations:** Add Tier 3 tables (`voting_configurations`, `voters`, `votes`, `vote_audit_logs`, `project_comments`) to `src/db/schema.ts` and execute migrations.
2. **Voting Services Core:** Implement `src/services/voting-config.service.ts`, `src/services/voter-identity.service.ts`, and `src/services/ballot-randomization.service.ts`.
3. **Voting Mechanics Engine:** Implement `src/services/voting-engine.service.ts` enforcing One-Person-One-Vote and Quadratic Voting point cost validation.
4. **Anti-Abuse & Audit System:** Implement `src/services/anti-abuse.service.ts` (rate limiter, duplicate check, risk scoring) and `src/services/vote-audit.service.ts` (audit logger, bulk override execution).
5. **Comments Service:** Implement `src/services/comments.service.ts` for comments, nested replies, rate limiting, and organizer moderation controls.
6. **API Routes & Seam Protections:** Implement public voter endpoints, organizer endpoints, and update existing T1 `GET /api/projects/[id]` route to strip vote counts while `ACTIVE`.
7. **UI Components & Dashboards:** Build ballot gallery view, voting point controls, comments section, Organizer Live Voting Dashboard, and Organizer Audit Forensic UI.
8. **Test Suites & Verification Script:** Create custom seed data (`src/db/seed-tier3.ts`), Vitest unit tests, Playwright E2E tests, custom runner `scripts/test-tier3.ts`, and run `python run.py`.
9. **Documentation & Manifest Claim Updates:**
   * Update `DATA-MODEL.md` with the 5 new Tier 3 tables.
   * Update `ARCHITECTURE.md` with Tier 3 services and API routes.
   * Update `JUDGING.md` (or append a Threat Model section) documenting the Anti-Abuse Engine, Fraud Risk Score rules ($R \in [0, 100]$), and Sybil protection logic (graded under Judging Integrity & Adoptability).
   * Update `.dogfood.toml` to add `"T3"` to claimed tiers **ONLY after** all tests pass and documentation is complete.

---

# 8. ACCEPTANCE CHECKLIST

```text
[ ] Database migrations run cleanly
[ ] Organizer can configure voting modes (OPEN, EMAIL_GATED, AUTHENTICATED) and mechanisms (ONE_PERSON_ONE_VOTE, QUADRATIC)
[ ] Open, Email-Gated, and Authenticated identity modes work as specified
[ ] One-Person-One-Vote enforces max 1 vote per project
[ ] Quadratic Voting validates sum(v^2) <= budget constraint accurately
[ ] Ballot order is randomized per voter session using Fisher-Yates with stable pagination
[ ] Public vote tallies are hidden while voting is ACTIVE (403 on direct API query)
[ ] Existing T1 route GET /api/projects/[id] strips voteCount while voting is ACTIVE
[ ] Configured actor auth in .dogfood.toml (organizer/judges/participant) remains unaffected by voter session auth
[ ] Rate limiter blocks rapid automated requests (HTTP 429)
[ ] Fraud risk engine flags suspicious IP/disposable email votes (Risk score R >= 50 flagged, R >= 80 rejected)
[ ] Organizer can inspect audit logs and execute bulk invalidation
[ ] Project comments and moderation (hide/flag) work as specified
[ ] CSV export downloads raw vote data and audit trail
[ ] Tier 1 and Tier 2 functionality remains 100% working (python run.py PASSES)
[ ] Documentation updated (ARCHITECTURE.md, DATA-MODEL.md, JUDGING.md threat model)
[ ] .dogfood.toml updated to claim "T3" after checklist is 100% green
[ ] docker compose up works locally offline
[ ] CSRF protection enforced on all state-changing endpoints (POST/PUT/DELETE)
[ ] Input sanitization (XSS prevention) enforced on public project comments
[ ] Parameter tampering rejected (negative votes, non-integers, quadratic budget overflow yield HTTP 400)
[ ] Self-voting blocked in AUTHENTICATED mode (author/team members yield HTTP 403)
[ ] Organizer IDOR protection enforced (routes verify event ownership server-side)
[ ] OTP tokens expire in 10 minutes, lock out after 5 failures, rate limited (max 3/min)
[ ] SQL queries parameterized via Drizzle ORM (SQL injection prevention)
[ ] Unit and E2E tests pass
```
