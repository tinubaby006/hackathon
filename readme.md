# DOGFOOD HACKATHON Platform

A self-hostable, local-first hackathon management platform built for the DOGFOOD HACKATHON.

The platform covers the complete core hackathon lifecycle: authentication, event configuration, organizer assignment, participant/team management, project submission, deadline enforcement, and a public project gallery.

**Current implementation target: Tier 1 (Core).**

---

## 1. Current Status

### Tier 1 — Core

The current build focuses on the mandatory core platform:

* Authentication and server-side sessions
* Five-role authorization model
* Admin-created events
* Event-scoped organizer assignment
* Organizer event management
* Participant membership
* Team creation and invite links
* Project creation and editing
* Draft / submit / reopen / resubmit workflow
* Hard submission deadline enforcement
* Organizer-defined tracks, prizes, and custom questions
* Local thumbnail and gallery image uploads
* Public gallery with search and filtering
* Event-scoped backend authorization
* Cross-event data isolation
* Real judge role and read-only judge event/submission view

T2/T3/T4 functionality is intentionally excluded from the active Tier 1 implementation.

---

# 2. Core Roles

The platform has five conceptual roles.

| Role        | Scope    | Purpose                                  |
| ----------- | -------- | ---------------------------------------- |
| Visitor     | Public   | Browse public events and projects        |
| Participant | Event    | Join teams and submit projects           |
| Judge       | Event    | Access assigned event/submission context |
| Organizer   | Event    | Manage an event                          |
| Admin       | Platform | Create events and assign organizers      |

### Role model

Platform administrators are stored separately from event roles.

```text
users.platform_role
    NONE
    ADMIN

event_memberships.role
    PARTICIPANT
    JUDGE
    ORGANIZER
```

A user may have different roles in different events.

For example:

```text
Event A → Participant
Event B → Organizer
Event C → Judge
```

Within a single event, a user has only one event-scoped role.

The backend is authoritative for authorization. Frontend role checks only control the UI and are never treated as a security boundary.

---

# 3. Authentication

Tier 1 uses:

* Email/password authentication
* Secure password hashing
* Server-side sessions
* Secure session cookies
* Logout/session invalidation
* Basic rate limiting for authentication-sensitive endpoints

Tier 1 does **not** require:

* JWT authentication
* OAuth
* Hosted authentication providers
* Password reset
* Email verification
* Account deletion workflows

These can be added later.

---

# 4. Event Lifecycle

Events are created by an **Admin**.

The lifecycle is:

```text
DRAFT
  ↓
ONGOING
  ↓
ENDED
```

### DRAFT

Used for event configuration and organizer assignment.

### ONGOING

Participants can:

* join/form teams
* accept team invitations
* create projects
* edit projects
* submit/reopen/resubmit projects

### ENDED

Participation and submissions are closed.

Projects that were validly submitted before the deadline remain publicly available.

---

# 5. Event Creation and Ownership

The event ownership flow is intentionally explicit.

```text
Admin
  ↓
Creates Event
  ↓
Assigns / Invites Organizer
  ↓
Organizer Accepts
  ↓
Organizer Manages Event
```

An event can have multiple organizers.

Organizers are event-scoped rather than global platform administrators.

An Admin remains responsible for platform-level administration and event creation.

---

# 6. Event Configuration

Each event can define:

* Name
* Description
* Start date/time
* Submission deadline
* End date/time
* Maximum team size
* Tracks
* Prizes
* Organizer-defined project questions

The basic time constraint is:

```text
start_at < submission_deadline <= end_at
```

There is no separate Tier 1 registration deadline.

Team formation and participation close at the submission deadline.

---

# 7. Tracks

Tracks are optional.

If an organizer configures tracks:

* Participants select a track for their project.
* The selected track must belong to the current event.
* Participants cannot create their own tracks.

If no tracks are configured, projects do not require track selection.

---

# 8. Prizes

Prizes are simple event configuration records.

A prize contains:

* Name
* Description
* Optional amount

Advanced prize logic is outside Tier 1.

---

# 9. Custom Project Questions

Organizers can define additional project questions.

Tier 1 supports:

* Short text
* Long text
* URL

Each question can define:

* Question text
* Type
* Required/optional
* Sort order

Answers are stored against the project's question definitions.

---

# 10. Teams

A participant can belong to only **one team per event**.

The participant who creates the team becomes its owner/captain.

Example:

```text
Participant
    ↓
Create Team
    ↓
Team Owner
    ↓
Generate Invite Link
    ↓
Other Participant Opens Link
    ↓
Signup / Login if necessary
    ↓
Explicitly Accept Invitation
    ↓
Join Team
```

### Team rules

* Maximum team size is configured per event.
* Team ownership cannot be transferred.
* The owner cannot leave their own team.
* The owner can remove members before the deadline.
* Members cannot remove themselves.
* The owner can delete the team before the deadline.
* Team changes are blocked at/after the submission deadline.
* A participant cannot join a second team in the same event.

---

# 11. Team Invite Links

Team invitations use secure, random invitation tokens.

Opening an invite link does **not** automatically join the team.

The invitation must be explicitly accepted.

### Authentication edge case

Invite context must survive signup/login.

Example:

```text
/invite/<token>
       ↓
Not authenticated
       ↓
Signup / Login
       ↓
Return to invitation
       ↓
Explicit Accept
       ↓
Server validates invitation
       ↓
Join team
```

The server revalidates the invitation when it is accepted.

Validation includes:

* Token validity
* Expiry
* Revocation
* Event membership
* Existing team membership
* One-team-per-event rule
* Team capacity
* Submission deadline

Frontend state alone is not trusted for invitation acceptance.

---

# 12. Projects

Each team has at most **one project per event**.

Any team member can create or edit the shared team project.

The project lifecycle is:

```text
DRAFT ↔ SUBMITTED
          ↓
     DEADLINE
          ↓
        LOCKED
```

Submitting a project before the deadline does **not** permanently lock it.

Participants can:

```text
Create
  ↓
Edit
  ↓
Submit
  ↓
Reopen
  ↓
Edit
  ↓
Resubmit
```

This remains possible until the submission deadline.

At or after the deadline:

* project edits are blocked
* new submissions are blocked
* reopening is blocked
* resubmission is blocked
* team membership changes are blocked

The backend enforces these rules regardless of the frontend state.

---

# 13. Project Fields

Tier 1 supports the required project information:

* Project name
* Tagline
* Long description
* Thumbnail
* Image gallery
* Hosted demo video URL
* Repository URL
* Live/demo URL
* Technology tags
* Track
* Organizer-defined custom question answers

URLs are stored and displayed; Tier 1 does not verify or integrate with external services.

Technology tags are free-form.

---

# 14. Image Uploads

Project images use local filesystem storage.

The database stores **relative file paths**, not image binaries or base64 data.

Example:

```text
public/uploads/projects/<project-id>/thumbnail.webp
public/uploads/projects/<project-id>/gallery-1.webp
```

The implementation is designed to work with persistent container storage.

For final local/containerized deployment, the uploads directory should be mounted to a persistent volume so that:

```text
App restart
    ↓
Database remains
    ↓
Uploaded images remain
```

Tier 1 should therefore be tested by:

1. Uploading a thumbnail/gallery image.
2. Confirming the image displays.
3. Restarting the application.
4. Confirming the image still displays.
5. Confirming SQLite stores only file paths, not base64/blob image data.

Basic file type and size validation is required.

---

# 15. Submission Visibility

Projects are private during the active hackathon/submission period.

Before the deadline:

```text
Project → Private
```

At the deadline:

```text
Project → Locked
```

After the deadline:

```text
Valid submitted project → Public
```

Public projects can be viewed without authentication.

A team that has not produced a valid submitted project by the deadline does not have a public submission.

Project visibility is separate from event lifecycle.

An event can be `ENDED` while its submitted projects remain publicly accessible indefinitely.

---

# 16. Public Gallery

The public gallery provides:

* Project listing
* Project detail pages
* Search
* Event filtering
* Track filtering when applicable

Public project pages do not require login.

Only projects that became public after a valid submission are shown.

---

# 17. Judge Role in Tier 1

`JUDGE` is a real event-scoped role in Tier 1.

A judge can:

* Log in
* See the event in the unified **My Events** experience
* Open their event context
* View relevant submitted project information
* Access projects in a read-only judging context

Tier 1 does **not** implement the judging/scoring system.

Specifically excluded from Tier 1:

* Rubrics
* Weighted criteria
* Score submission
* Judge scoring UI
* Score aggregation
* Cross-judge normalization
* CSV score export

These belong to Tier 2.

The Tier 1 judge view must not expose:

* Other judges' scores
* Aggregate scores
* Normalization information
* Organizer/admin-only configuration

The complete judge invitation/assignment workflow is a Tier 2 feature; Tier 1 nevertheless keeps the judge role real and functional so the role model does not contain a dead-end role.

---

# 18. Unified My Events Experience

Authenticated users use one unified event experience.

A user may see:

```text
My Events

Event A
Role: Participant

Event B
Role: Organizer

Event C
Role: Judge
```

The backend determines the user's event role.

The UI presents the appropriate event capabilities for that role.

There are not separate global user accounts or separate platform identities for each role.

---

# 19. Authorization and Security

Security is enforced server-side.

Every protected API must perform:

1. Authentication check
2. Event membership lookup where applicable
3. Role authorization
4. Resource ownership/access validation
5. Deadline validation where applicable
6. Input validation

### HTTP behavior

```text
Unauthenticated → 401
Authenticated but unauthorized → 403
```

Frontend hiding is never considered sufficient authorization.

### Cross-event isolation

Changing an event ID, team ID, project ID, or other resource identifier must not allow a user to access another event's protected resources.

---

# 20. Deadline Enforcement

The submission deadline is a backend security boundary.

After the deadline, the backend must reject:

* Project creation
* Project editing
* Project submission
* Project reopening
* Project resubmission
* Team membership changes
* Team invitations
* Invitation acceptance

The frontend may disable controls earlier for usability, but the backend remains authoritative.

---

# 21. Organizer Intervention

Organizers can intervene in teams/projects where necessary for event administration or rule enforcement.

Tier 1 keeps moderation intentionally simple.

Advanced moderation systems such as:

* sophisticated project disabling
* archival workflows
* moderation queues
* advanced audit systems
* automated abuse detection

are post-Tier-1 work.

---

# 22. Architecture

The application follows a simple full-stack architecture:

```text
Browser
   ↓
Next.js UI
   ↓
API / Route Handlers
   ↓
Authentication & Authorization
   ↓
Business Services
   ↓
Drizzle ORM
   ↓
SQLite
```

Local project images are stored on the filesystem rather than inside SQLite.

The architecture is intentionally simple so the platform can run locally without hosted infrastructure.

---

# 23. Data Model

Core Tier 1 entities:

```text
users
sessions

events
event_memberships
event_tracks
event_prizes
event_questions

teams
team_members
team_invites

projects
project_images
project_technologies
project_answers
```

Important relationships:

```text
User
 ├── Sessions
 ├── Event Memberships
 └── Team Memberships

Event
 ├── Memberships
 ├── Tracks
 ├── Prizes
 ├── Questions
 ├── Teams
 └── Projects

Team
 ├── Members
 ├── Invites
 └── Project

Project
 ├── Images
 ├── Technologies
 └── Question Answers
```

A project's team relationship is unique per event.

---

# 24. Technology Stack

The planned Tier 1 stack is:

* Next.js
* React
* TypeScript
* App Router
* Tailwind CSS
* SQLite
* Drizzle ORM
* Zod
* Argon2id
* Vitest
* Playwright

Storage:

* SQLite for application data
* Local filesystem for uploaded project images

No cloud database, hosted authentication provider, or external storage service is required.

---

# 25. Tier 1 Scope Checklist

## Authentication

* [ ] Signup
* [ ] Login
* [ ] Logout
* [ ] Server-side sessions
* [ ] Secure password hashing
* [ ] Basic authentication rate limiting

## Roles

* [ ] Visitor
* [ ] Participant
* [ ] Judge
* [ ] Organizer
* [ ] Admin
* [ ] Event-scoped role enforcement
* [ ] Cross-event isolation

## Events

* [ ] Admin creates events
* [ ] Admin assigns/invites organizer
* [ ] Organizer accepts assignment
* [ ] Event lifecycle
* [ ] Configurable dates
* [ ] Configurable maximum team size
* [ ] Tracks
* [ ] Prizes
* [ ] Custom questions

## Teams

* [ ] Team creation
* [ ] Team owner/captain
* [ ] Invite links
* [ ] Invite acceptance
* [ ] Invite state survives authentication
* [ ] Capacity enforcement
* [ ] One team per participant per event
* [ ] Pre-deadline membership management
* [ ] Post-deadline lock

## Projects

* [ ] One project per team/event
* [ ] Draft editing
* [ ] Submit
* [ ] Reopen
* [ ] Resubmit
* [ ] Hard deadline enforcement
* [ ] Required project fields
* [ ] Custom question answers
* [ ] Thumbnail upload
* [ ] Gallery image upload
* [ ] Persistent upload-compatible storage
* [ ] Project locking
* [ ] Public visibility after deadline

## Gallery

* [ ] Public gallery
* [ ] Project detail page
* [ ] Search
* [ ] Event filtering
* [ ] Track filtering

## Judge

* [ ] Real event-scoped judge role
* [ ] Judge login
* [ ] Judge event visibility in My Events
* [ ] Read-only project/submission context
* [ ] No T2 scoring functionality

## Security

* [ ] Backend authorization
* [ ] 401/403 behavior
* [ ] Cross-event access tests
* [ ] Deadline enforcement tests
* [ ] Invite validation tests
* [ ] Input validation
* [ ] URL validation
* [ ] Image validation
* [ ] Basic rate limiting
* [ ] Secure sessions

---

# 26. Explicitly Post-Tier-1

The following are intentionally not part of the current Tier 1 implementation.

### Tier 2 — Judging

* Judge invitation/assignment system
* Weighted organizer-configurable rubric
* Score submission
* Score isolation
* Progress dashboard
* Cross-judge normalization
* CSV export

### Tier 3 — Community

* Community voting
* Comments
* Hidden results
* Randomized ordering
* Advanced anti-abuse mechanisms
* Advanced audit functionality

### Tier 4 — Integrations and Records

* REST API
* Webhooks
* Certificates
* Participation records
* Signed judge records
* Embeddable gallery
* Bulk import/export

### Other deferred functionality

* Password reset
* Email verification
* OAuth
* Email notifications
* External object storage
* Advanced moderation
* Advanced analytics
* Advanced search

---

# 27. End-to-End Tier 1 Flow

The intended complete flow is:

```text
Admin
  ↓
Create Event
  ↓
Assign Organizer
  ↓
Organizer Accepts
  ↓
Configure Dates / Teams / Tracks / Prizes / Questions
  ↓
Participants Sign Up / Log In
  ↓
Participants Join Event
  ↓
Participant Creates Team
  ↓
Team Generates Invite Link
  ↓
Other Participant Opens Invite
  ↓
Signup / Login if Needed
  ↓
Explicitly Accept Invite
  ↓
Team Formed
  ↓
Team Creates Project
  ↓
Project Drafted
  ↓
Project Submitted
  ↓
Project Can Be Edited / Reopened / Resubmitted
  ↓
Submission Deadline
  ↓
Backend Locks Participation Changes
  ↓
Valid Submitted Projects Become Public
  ↓
Public Gallery
```

Judge role:

```text
Judge
  ↓
Login
  ↓
My Events
  ↓
Judge Event View
  ↓
Read-Only Submitted Project Context
```

No scoring occurs in this Tier 1 flow.

---

# 28. Development Priorities

The implementation should prioritize:

1. Correct data model
2. Authentication/session correctness
3. Backend authorization
4. Event-scoped role enforcement
5. Deadline enforcement
6. Team/invite correctness
7. Project lifecycle correctness
8. Persistent local image storage
9. Public gallery
10. Judge role/view
11. End-to-end testing
12. Final packaging and documentation

The primary goal is a **correct, runnable Tier 1 platform**, not breadth of unfinished features.

---

# 29. Local-First Deployment

The final platform is intended to be self-hostable and runnable locally without cloud dependencies.

Final packaging should support the required local deployment workflow, including persistent storage for uploaded project images.

The Docker Compose configuration is treated as part of final compliance/package verification rather than as a reason to expand the active Tier 1 feature scope prematurely.

---

# 30. Definition of Done — Tier 1

Tier 1 is complete when a fresh local deployment can demonstrate:

* A user can sign up and log in.
* An Admin can create an event.
* An organizer can be assigned to the event and manage it.
* Participants can join and form teams.
* Team invitations survive signup/login and require explicit acceptance.
* Team capacity is enforced.
* Participants can create and edit a project.
* Projects can be submitted, reopened, edited, and resubmitted before the deadline.
* The backend prevents prohibited actions after the deadline.
* Project images persist across application restarts.
* Judges can log in and access a real read-only judge event/submission view.
* Judges cannot access protected organizer/admin data or T2 scoring information.
* Submitted projects become publicly visible after the deadline.
* The public gallery supports search/filtering.
* Cross-event authorization boundaries hold.
* Automated tests cover the highest-risk rules.

Anything beyond these requirements should be treated as secondary until Tier 1 is stable.

---

# 31. License

The final repository should use an OSI-approved open-source license as required by the hackathon submission rules.

Preferred options include:

* MIT
* Apache-2.0

The final repository should include the selected `LICENSE` file.

---

# 32. Project Principle

> **Build the smallest correct hackathon platform that fully satisfies Tier 1 before expanding into judging, community features, or integrations.**

Correct authorization, deadlines, team membership, invitation handling, persistence, and project lifecycle behavior take priority over additional features.
