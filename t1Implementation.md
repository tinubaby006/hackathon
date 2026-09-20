# DOGFOOD HACKATHON

## Agent Implementation Prompt & Tier 1 Execution Plan

> **Purpose:** This document is the implementation contract for building the first working version of the DOGFOOD HACKATHON platform.
>
> The agent should implement the functionality explicitly defined below, keep the architecture simple, and avoid inventing additional product workflows that are not required.

---

# 1. Implementation Goal

Build a complete, locally runnable hackathon platform that supports this lifecycle:

```text
Authenticated User
      │
      │ creates event
      ▼
Event Proposal
      │
      │ Admin approval
      ▼
Approved Event
      │
      │ creator becomes Organizer
      ▼
Organizer configures event
      │
      ▼
Participants join / create teams
      │
      ▼
Teams submit projects
      │
      ▼
Submission deadline locks projects
      │
      ▼
Projects become publicly visible
      │
      ▼
Public gallery with search/filter
```

The implementation must work locally without requiring hosted authentication, a hosted database, cloud storage, or external API keys.

---

# 2. Core Implementation Rule

Implement **only the functionality explicitly required by this document**.

Do not invent additional:

* roles
* dashboards
* workflows
* scoring systems
* moderation systems
* integrations
* external services
* APIs
* notifications
* analytics
* automation
* product-management features

unless they are directly required below.

Prefer a simple working implementation over abstraction or speculative extensibility.

---

# 3. Technology Direction

Use a simple full-stack TypeScript application.

Recommended stack:

* Next.js
* TypeScript
* React
* App Router
* Tailwind CSS
* SQLite
* Drizzle ORM
* Zod
* Argon2id for password hashing
* Server-side sessions
* Vitest
* Playwright

Use a clean architecture:

```text
UI
 ↓
Route/API layer
 ↓
Authentication & Authorization
 ↓
Business logic
 ↓
Drizzle ORM
 ↓
SQLite
```

Do not introduce unnecessary microservices.

---

# 4. Authentication

Users must be able to:

* sign up
* log in
* log out
* remain authenticated through a server-side session
* access their authenticated platform experience

Required:

* email
* password
* display name
* secure password hashing
* secure session cookie
* session expiration
* logout invalidation

Do not use JWT authentication.

Do not require OAuth.

Do not require a hosted authentication provider.

Basic rate limiting should exist for sensitive operations such as:

* login
* signup
* invite acceptance

---

# 5. Role Model

There are five concepts:

```text
Visitor
Participant
Judge
Organizer
Admin
```

## Visitor

Unauthenticated user.

Can access public content such as:

* homepage
* approved public events
* public gallery
* public project pages

Cannot perform authenticated actions.

---

## Participant

Participant is an **event-scoped role**.

A user becomes a Participant when they join an event.

A participant can:

* view the event
* create a team
* join a team through an invitation
* view their team
* create/edit the team's project
* submit/resubmit the project before the deadline

### Joining an event

A user becomes a Participant by explicitly joining an approved event.

A user may join an event only when:

* the user is authenticated
* the event exists
* the event has `approval_status = APPROVED`
* the submission deadline has not passed
* the user does not already have a membership in that event

Joining an event creates:

```text
event_memberships(
  event_id = event.id,
  user_id = current_user.id,
  role = PARTICIPANT
)
```

---

## Judge

Judge is an **event-scoped role**.

A judge can:

* log in
* see the event they are associated with
* view submitted project information relevant to the event

Judges must not receive participant/team-management permissions unless they separately have that role in another event.

### Judge assignment

A Judge is assigned explicitly to an approved event by an Organizer of that event.

The target user must:

* already have an authenticated platform account
* have no existing membership in the event

Judge assignment creates:

```text
event_memberships(
  event_id = event.id,
  user_id = target_user.id,
  role = JUDGE
)
```

---

## Organizer

Organizer is an **event-scoped role**.

There is no global Organizer account type.

A user becomes Organizer of an event through the event approval process described below.

An Organizer can manage their approved event, including:

* event configuration
* tracks
* prizes
* custom questions
* participant/team context
* project/event administration

---

## Admin

Admin is platform-level.

Admin can:

* access platform administration
* review event proposals
* approve events
* manage platform-level administration

Admin is **not** responsible for creating every event.

---

# 6. Important Role Rule

A user can have different roles across different events.

Example:

```text
Event A → Organizer
Event B → Participant
Event C → Judge
```

Within a single event, a user may have only one event membership role.

Enforce this on the backend.

Do not rely on frontend visibility for authorization.

---

# 7. Event Creation and Approval

This is the required event ownership model.

## Anyone authenticated can propose an event

Any authenticated platform user may create an event proposal.

This includes someone who is currently:

* a participant in another event
* a judge in another event
* an organizer of another event
* simply an authenticated user with no event role

They do not need a global Organizer role.

---

## Event creation flow

```text
Authenticated User
        ↓
Create Event
        ↓
PENDING_APPROVAL
        ↓
Admin reviews
        ↓
Admin approves
        ↓
Creator becomes ORGANIZER
        ↓
Organizer manages event
```

The event creator is stored as `created_by`.

Before approval, the creator is **not** an Organizer of that event.

---

## Pending event

A pending event:

* is not an active public event
* does not allow normal participant registration
* does not appear as a normal public event
* does not allow the creator to use Organizer permissions
* can be viewed/edited by its creator as their own event proposal
* can be reviewed by an Admin

---

## Admin approval

When an Admin approves the event:

1. The event approval state changes to approved.
2. The creator receives an `ORGANIZER` membership for that event.
3. The creator can access Organizer functionality.
4. The event can proceed through its normal lifecycle.

The approval operation must be protected by backend authorization.

Only an Admin may approve an event.

---

# 8. Event Lifecycle

Use two concepts:

Approval state
PENDING
APPROVED
Event lifecycle

The event lifecycle is derived from the approval state and the event's configured dates.

For an approved event:

Before start_at
    ↓
DRAFT

start_at <= current_time < end_at
    ↓
ONGOING

current_time >= end_at
    ↓
ENDED

An event must satisfy:

start_at < submission_deadline <= end_at

Lifecycle rules:

PENDING approval status always means the event has not yet been approved.
An approved event remains DRAFT until start_at.
At or after start_at, an approved event is ONGOING.
At or after end_at, an approved event is ENDED.
An event must never become ONGOING before it is approved.
An event must never return from ENDED to DRAFT or ONGOING.
The server's current time is authoritative.
The client must not be able to manually set the lifecycle status.
No background scheduler is required to transition events.
The backend should determine the effective lifecycle status whenever the event is read or a lifecycle-dependent operation is performed.

The submission deadline is independent of the event lifecycle status and is enforced separately according to the submission deadline rules.

---

# 9. Event Configuration

An approved Organizer can configure:

* event name
* description
* start date/time
* submission deadline
* end date/time
* maximum team size
* tracks
* prizes
* custom project questions

Required date constraint:

```text
start_at < submission_deadline <= end_at
```

Maximum team size must be configurable per event.

Do not hard-code the DOGFOOD event's own team size as a platform-wide rule.

---

# 10. Tracks

Tracks are optional.

An Organizer can create tracks for an event.

Each track has:

* name
* description

If an event has tracks:

* participants may select a track for their project
* the selected track must belong to that event

If an event has no tracks:

* participants do not need to select one

Participants cannot create or modify event tracks.

---

# 11. Prizes

An Organizer can define prizes.

Each prize may contain:

* name
* description
* amount

Keep prizes simple.

No payment or prize-distribution system is required.

---

# 12. Custom Project Questions

Organizers can define questions that participants answer as part of their project.

Supported question types:

```text
SHORT_TEXT
LONG_TEXT
URL
```

Each question contains:

* question text
* type
* required flag
* display order

Questions belong to an event.

Project answers belong to a project and question.

Required questions must be validated before submission.

---

# 13. Teams

Participants form teams using invitation links.

## Team creation

A Participant can create a team.

The creator becomes the team owner/captain.

The owner cannot transfer ownership.

A participant can belong to only one team in a given event.

---

## Team size

The event's configured `max_team_size` must be enforced by the backend.

Do not hard-code a platform-wide team size.

---

## Team invitations

Invitations use a secure random token.

An invitation must support:

* expiry
* revocation
* capacity checking
* event/team association
* server-side validation

Store a secure hash of the invite token rather than relying on storing the raw token.

---

# 14. Invite Authentication Flow

This flow is important.

A user may open an invite link while logged out.

Required flow:

```text
Invite Link
    ↓
Not authenticated
    ↓
Signup / Login
    ↓
Invite context preserved
    ↓
Return to invitation
    ↓
User explicitly accepts
    ↓
Server validates invitation
    ↓
User joins team
```

Opening an invitation must **not automatically join the team**.

After signup/login, the server must revalidate:

* invitation validity
* expiration
* revocation
* event membership
* existing team membership
* team capacity
* submission deadline

Do not rely only on frontend state to preserve the invitation.

---

# 15. Team Rules

Before the submission deadline:

* team owner can remove members
* team owner can delete the team
* invitations can be created/accepted
* eligible participants can join

After the submission deadline:

* no team membership changes
* no new invitations
* no invitation acceptance
* no team deletion by participants

A participant cannot leave their team themselves.

The owner cannot leave their own team.

---

# 16. Project Model

Each team can have exactly one project for an event.

Any member of the team can edit the shared project.

There is no separate project for each team member.

Project lifecycle:

```text
DRAFT
  ↕
SUBMITTED
  ↓
LOCKED
```

Before the deadline, submitting is reversible.

A participant may:

```text
Draft
 ↓
Submit
 ↓
Reopen
 ↓
Edit
 ↓
Resubmit
```

The project is not permanently locked when the user clicks Submit.

The actual deadline is what locks the project.

---

# 17. Project Fields

A project must support:

* name
* tagline
* long description
* thumbnail
* image gallery
* hosted demo video URL
* repository URL
* live/demo URL
* technology tags
* track
* answers to organizer-defined questions

Technology tags can be free-form.

The video is URL-only.

Do not implement video file hosting.

URLs should be validated before storage.

---

# 18. Image Uploads

Projects support:

* one thumbnail
* multiple gallery images

Use local filesystem storage.

Do **not** store uploaded images as base64 blobs inside SQLite.

The database should store file paths/URLs.

The upload implementation must be compatible with persistent container storage.

Example deployment expectation:

```text
./uploads:/app/public/uploads
```

The exact Docker configuration can be finalized during packaging, but the application must already use a persistent-volume-compatible upload directory.

Required validation:

* file type
* file size
* safe filename/path handling

---

# 19. Submission Deadline Enforcement

The submission deadline is a hard backend boundary.

After the deadline, participants must not be able to:

* edit projects
* create projects
* submit projects
* reopen projects
* resubmit projects
* modify project answers
* modify team membership
* accept team invitations
* create team invitations
* delete teams

Do not rely on the UI disabling buttons.

Every relevant backend operation must independently check the event deadline.

The server's current time is authoritative.

---

# 20. Public Project Visibility

Before the submission deadline:

```text
Project = private
```

After the submission deadline:

```text
Valid submitted project = public
```

Only projects that have actually been submitted should become public.

Teams without a submitted project do not receive a public project page.

Public projects remain accessible after the event ends.

---

# 21. Public Gallery

Build a public gallery that does not require authentication.

The gallery must support:

* browsing projects
* searching projects
* filtering by event
* filtering by track when tracks exist

Each public project should have a dedicated project page.

Project pages should display the submitted project information and relevant team/event context.

---

# 22. Cross-Event Isolation

Backend authorization must enforce event boundaries.

A user must not be able to access another event's protected resources simply by changing an ID in a URL or API request.

Examples:

```text
Event A participant
        ↓
cannot edit
        ↓
Event B project
```

Every protected operation must verify:

1. authenticated user
2. requested resource
3. resource's event
4. user's membership/role in that event
5. required permission

---

# 23. HTTP Authorization Rules

Use:

```text
401 Unauthorized
```

when authentication is required but the user is not authenticated.

Use:

```text
403 Forbidden
```

when the user is authenticated but does not have permission.

Do not expose privileged functionality merely because a frontend route is hidden.

---

# 24. Suggested Database Model

Use these tables.

### users

```text
id
email
password_hash
name
platform_role
created_at
```

`platform_role`:

```text
NONE
ADMIN
```

---

### sessions

```text
id
user_id
expires_at
created_at
last_used_at
```

---

### events

```text
id
name
description
start_at
submission_deadline
end_at
status
approval_status
max_team_size
created_by
approved_by
approved_at
created_at
updated_at
```

`status`:

```text
DRAFT
ONGOING
ENDED
```

`approval_status`:

```text
PENDING
APPROVED
```

The event creator is identified by `created_by`.

`approved_by` references the Admin who approved it.

---

### event_memberships

```text
id
event_id
user_id
role
created_at
```

Roles:

```text
PARTICIPANT
JUDGE
ORGANIZER
```

Constraint:

```text
UNIQUE(event_id, user_id)
```

This enforces one event role per user.

When an event is approved, create:

```text
event_memberships(
  event_id = event.id,
  user_id = event.created_by,
  role = ORGANIZER
)
```

---

### event_tracks

```text
id
event_id
name
description
created_at
```

---

### event_prizes

```text
id
event_id
name
description
amount
created_at
```

---

### event_questions

```text
id
event_id
question
type
required
sort_order
created_at
```

---

### teams

```text
id
event_id
name
owner_id
created_at
```

---

### team_members

```text
team_id
user_id
joined_at
```

Enforce one team per user per event in application/database logic.

---

### team_invites

```text
id
team_id
token_hash
expires_at
revoked_at
created_at
```

---

### projects

```text
id
event_id
team_id
name
tagline
description
thumbnail_url
demo_video_url
repository_url
live_demo_url
track_id
status
created_at
updated_at
submitted_at
locked_at
```

Constraint:

```text
UNIQUE(event_id, team_id)
```

---

### project_images

```text
id
project_id
image_url
sort_order
created_at
```

---

### project_technologies

```text
id
project_id
name
```

---

### project_answers

```text
id
project_id
question_id
answer
```

Constraint:

```text
UNIQUE(project_id, question_id)
```

---

# 25. API Surface

The exact URL naming may be adapted to the chosen framework, but the capabilities must exist.

## Authentication

```text
POST /api/auth/signup
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
````

**---**

## Event creation

Authenticated users can create event proposals:

```text
POST  /api/events
GET   /api/events/:eventId
PATCH /api/events/:eventId
```

Creation creates a pending event proposal.

The creator may edit their own pending proposal.

The event lifecycle status is determined by the configured dates and server time.

Clients must not directly set or transition `status`.

Protected Organizer operations require an approved `ORGANIZER` membership for the event.

**---**

## Event participation

Authenticated users may explicitly join an approved event:

```text
POST /api/events/:eventId/join
```

Joining must:

* require authentication
* verify that the event exists
* verify `approval_status = APPROVED`
* verify that the submission deadline has not passed
* verify that the user has no existing membership in the event
* create an `event_memberships` record with `role = PARTICIPANT`

The operation must be atomic.

Joining an event must be performed by an explicit user action.

Opening or viewing an event must not automatically create an event membership.

Possible outcomes:

```text
201 Created
```

when the participant membership is successfully created.

```text
401 Unauthorized
```

when the user is not authenticated.

```text
404 Not Found
```

when the event does not exist.

```text
403 Forbidden
```

when the event exists but participation is not currently permitted.

```text
409 Conflict
```

when the user already has an event membership or the event can no longer accept participants.

**---**

## Admin approval

```text
GET  /api/admin/events/pending
POST /api/admin/events/:eventId/approve
```

Approval must be Admin-only.

Approval creates the `ORGANIZER` membership for the event creator.

**---**

## Judge assignment

An Organizer may assign an existing platform user as a Judge for their approved event.

```text
POST /api/events/:eventId/judges
GET  /api/events/:eventId/judges
DELETE /api/events/:eventId/judges/:userId
```

Judge assignment must:

* require authentication
* require an approved event
* require Organizer authorization for that event
* verify that the target user exists
* verify that the target user has no existing membership in the event
* create an `event_memberships` record with `role = JUDGE`
* perform the membership creation atomically

Removing a Judge deletes only that Judge's membership for the specified event.

Possible outcomes:

```text
201 Created
```

when the Judge membership is created successfully.

```text
401 Unauthorized
```

when the requester is not authenticated.

```text
403 Forbidden
```

when the requester is authenticated but is not an Organizer of the event.

```text
404 Not Found
```

when the event or target user does not exist.

```text
409 Conflict
```

when the target user already has a membership in the event.

Judge assignment does not change the user's roles in other events.


## Additional organizer administration

If additional event organizers are supported, keep this separate from the event-creation flow.

The initial Organizer is always the approved event creator.

**---**

## Event configuration

```text
POST   /api/events/:eventId/tracks
PATCH  /api/events/:eventId/tracks/:trackId
DELETE /api/events/:eventId/tracks/:trackId

POST   /api/events/:eventId/prizes
PATCH  /api/events/:eventId/prizes/:prizeId
DELETE /api/events/:eventId/prizes/:prizeId

POST   /api/events/:eventId/questions
PATCH  /api/events/:eventId/questions/:questionId
DELETE /api/events/:eventId/questions/:questionId
```

All require appropriate Organizer authorization.

**---**

## Teams

```text
POST   /api/events/:eventId/teams
GET    /api/events/:eventId/teams/:teamId
POST   /api/teams/:teamId/invites
POST   /api/team-invites/:token/accept
DELETE /api/teams/:teamId/members/:userId
DELETE /api/teams/:teamId
```

Deadline rules apply to all participant-controlled team mutations.

**---**

## Projects

```text
POST   /api/teams/:teamId/project
GET    /api/projects/:projectId
PATCH  /api/projects/:projectId
POST   /api/projects/:projectId/submit
POST   /api/projects/:projectId/reopen
POST   /api/projects/:projectId/images
DELETE /api/projects/:projectId/images/:imageId
```

Every mutation must verify team membership and deadline state.

**---**

## Public gallery

```text
GET /api/gallery
GET /api/gallery/projects/:projectId
```

Support:

```text
q
event
track
```

# 26. Required UI

## Public

Build:

* homepage
* public event page
* public gallery
* public project page

---

## Authentication

Build:

* signup
* login
* logout
* authenticated session handling

---

## Unified authenticated experience

Build a single authenticated area such as:

```text
My Events
```

A user should be able to see their relationship to different events.

Example:

```text
Hackathon A — Organizer
Hackathon B — Participant
Hackathon C — Judge
My Event Proposal — Pending Approval
```

Do not create completely separate account systems for each role.

---

# 27. Event Proposal UI

Authenticated users must have a clear way to create an event.

The flow should be:

```text
Create Event
     ↓
Fill event information
     ↓
Submit for approval
     ↓
Pending Approval
```

The creator should be able to see the proposal status.

After approval:

```text
Pending Approval
     ↓
Approved
     ↓
Organizer
```

The UI must clearly distinguish:

```text
Creator of pending proposal
```

from:

```text
Organizer of approved event
```

---

# 28. Admin UI

Admin must have an approval queue.

The queue should show pending event proposals.

Admin can:

* inspect proposal
* approve event

Approval must happen through a protected backend operation.

---

# 29. Participant UI

Participant experience must support:

```text
Approved Event
 ↓
View Event
 ↓
Explicitly Join Event
 ↓
Participant Membership Created
 ↓
Create or join team
 ↓
Invite teammates
 ↓
Create project
 ↓
Edit project
 ↓
Submit
 ↓
Reopen/edit/resubmit before deadline
 ↓
Locked at deadline
```

---

# 30. Judge UI

Judge must have a real authenticated event experience.

At minimum, the Judge can:

* log in
* see their event
* access the event context
* view submitted project information

Keep this view read-only with respect to participant submissions.

Do not expose participant editing controls to judges.

---

# 31. Organizer UI

After approval, the event creator receives Organizer access.

Organizer must be able to configure:

* event information
* dates
* maximum team size
* tracks
* prizes
* custom questions

The Organizer must also be able to inspect the event's participants, teams, and submitted projects as needed for event administration.

The Organizer must also be able to:

* view the event's assigned Judges
* assign an existing platform user as Judge
* remove an assigned Judge

---

# 32. Security Requirements

Implement backend enforcement for:

### Authentication

* secure password hashing
* server-side sessions
* secure cookies
* session invalidation on logout

### Authorization

* check authorization on every protected endpoint
* enforce event-scoped roles
* prevent cross-event access

### Deadline

* enforce server-side
* never trust frontend state

### Invitations

* cryptographically secure tokens
* hashed token storage
* expiration
* revocation
* capacity validation
* explicit acceptance

### Input validation

Validate:

* required fields
* field types
* URLs
* image uploads
* IDs
* event ownership/membership

Escape/sanitize user-generated content appropriately.

---

# 33. Critical Architectural Pitfall: Local Upload Persistence

Do not build uploads in a way that only works inside one temporary container.

Use a dedicated upload directory.

The application should behave correctly when:

```text
application restarts
```

or when:

```text
container restarts
```

provided the upload directory is mounted as persistent storage.

Test:

1. Upload an image.
2. Restart the application.
3. Confirm the image still loads.
4. Confirm SQLite contains only the image path/reference, not the image binary.

---

# 34. Critical Architectural Pitfall: Invite + Authentication

Test the complete flow:

```text
User A creates team
       ↓
User A generates invite
       ↓
Logged-out User B opens invite
       ↓
User B signs up/logs in
       ↓
Invite context survives authentication
       ↓
User B explicitly accepts
       ↓
User B joins team
```

Also test failure cases:

* expired invite
* revoked invite
* full team
* already on another team in the event
* already a member
* deadline passed

---

# 35. Critical Architectural Pitfall: Role Isolation

Test:

```text
User A = Participant in Event A
User A = Organizer in Event B
User A = Judge in Event C
```

Each event must expose only the permissions associated with that event membership.

Changing an event ID in a request must not bypass authorization.

---

# 36. Testing Requirements

At minimum, create automated coverage for:

## Authentication

* signup
* login
* logout
* protected route without session
* invalid credentials

## Event approval

* authenticated user creates event
* event starts as pending
* creator is not Organizer before approval
* Admin can see pending event
* non-Admin cannot approve
* Admin approves event
* creator becomes Organizer
* approved event becomes manageable by creator

## Event lifecycle

* approved event before `start_at` is `DRAFT`
* event at `start_at` is `ONGOING`
* event between `start_at` and `end_at` is `ONGOING`
* event at `end_at` is `ENDED`
* event after `end_at` remains `ENDED`
* unapproved event never becomes `ONGOING`
* client cannot manually change lifecycle status

## Judge assignment

* Organizer can assign an existing user as Judge
* non-Organizer cannot assign Judges
* assigned Judge receives `JUDGE` membership
* user already holding a membership in the event cannot be assigned as Judge
* Judge can access the event/project context
* Judge cannot modify participant/team/project resources
* removing a Judge removes only that event membership
* Judge role in one event does not affect roles in another event

## Event isolation

* user can access own event
* unauthorized user receives 403
* cross-event ID manipulation fails

## Teams

* participant creates team
* owner becomes captain
* invite creation
* invite acceptance
* signup/login invite continuation
* max team size
* one team per event
* expired invite
* revoked invite

## Projects

* create project
* edit draft
* submit
* reopen before deadline
* edit after reopening
* resubmit
* required custom question validation

## Deadline

Test at minimum:

```text
before deadline → allowed
at deadline → locked
after deadline → rejected
```

Test both:

* project mutations
* team/invitation mutations

## Public gallery

* submitted project becomes public after deadline
* non-submitted project does not become public
* search works
* event filter works
* track filter works

## Uploads

* valid image upload
* invalid image rejected
* image persists across application restart

---

# 37. Seed Data

Provide useful local seed data.

The seeded environment should make the complete lifecycle easy to demonstrate.

Include:

* Admin account
* several normal users
* at least one approved event
* at least one pending event proposal
* participant memberships
* judge membership
* organizer membership
* example team
* example project
* tracks
* prizes
* custom questions

Do not make the seed data dependent on external services.

Document development credentials clearly for local use.

---

# 38. Recommended Implementation Order

Build in this order.

### Phase 1 — Foundation

* project setup
* database
* schema
* migrations
* authentication
* sessions
* seed data
* base layout

### Phase 2 — Roles and Authorization

* platform Admin
* event memberships
* Participant
* Judge
* Organizer
* backend permission helpers
* cross-event authorization

### Phase 3 — Event Proposal and Approval

* authenticated user creates event
* pending event state
* creator proposal editing
* Admin approval queue
* Admin approval endpoint
* creator becomes Organizer after approval
* event lifecycle

### Phase 4 — Event Configuration

* dates
* team size
* tracks
* prizes
* custom questions

### Phase 5 — Teams

* team creation
* ownership
* invites
* invite acceptance
* signup/login invite continuation
* capacity enforcement
* deadline enforcement

### Phase 6 — Projects

* project creation
* project editing
* custom answers
* technology tags
* image uploads
* submit/reopen/resubmit
* deadline locking

### Phase 7 — Public Gallery

* public event pages
* public projects
* gallery
* search
* filters

### Phase 8 — Testing and Hardening

* authorization tests
* deadline tests
* invite tests
* upload persistence tests
* cross-event isolation tests
* end-to-end lifecycle test

### Phase 9 — Final Packaging

Only after the application works:

* Docker configuration
* persistent upload volume
* local startup flow
* README
* architecture documentation
* data model documentation
* license
* acceptance documentation

---

# 39. Critical End-to-End Test

The most important test should reproduce the complete platform lifecycle.

1. Create Admin

2. Create normal authenticated user

3. Normal user creates event proposal

4. Confirm event is PENDING_APPROVAL

5. Confirm creator is NOT Organizer yet

6. Admin opens approval queue

7. Admin approves event

8. Confirm creator becomes Organizer

9. Organizer configures event

10. Confirm event is DRAFT before start_at

11. Confirm event becomes ONGOING at start_at

12. Confirm event becomes ENDED at end_at

13. Organizer assigns a Judge

14. Confirm Judge membership is created

15. Confirm Judge can access event/project context without participant permissions

16. Participant joins event

17. Participant creates team

18. Participant generates invite

19. Second user opens invite while logged out

20. Second user signs up/logs in

21. Invite context survives authentication

22. Second user explicitly accepts invite

23. Team reaches valid membership state

24. Team creates project

25. Team edits project

26. Team submits project

27. Team reopens and resubmits before deadline

28. Deadline passes

29. Confirm project is locked

30. Confirm team changes are locked

31. Confirm submitted project is publicly visible

32. Confirm gallery search/filter works

33. Confirm unauthorized users cannot modify protected resources

34. Confirm Judge can access event/project context without participant editing permissions

---

# 40. Scope Control Rule

If a requested feature is not necessary for the lifecycle described in this document, do not implement it simply because it might be useful later.

Prioritize:

```text
Correctness
↓
Security
↓
Deadline enforcement
↓
Role isolation
↓
Complete user lifecycle
↓
Usability
↓
Visual polish
```

Do not sacrifice backend correctness for UI polish.

Do not build speculative abstractions.

---

# 41. Local Development Requirement

The application must be runnable locally without cloud dependencies.

The final environment should support:

```text
docker compose up
```

and provide:

* application
* local SQLite database
* persistent uploaded images
* seeded data
* working authentication
* complete core lifecycle

The application must not depend on a hosted database or hosted authentication provider.

---

# 42. Final Verification Checklist

Before considering the implementation complete, verify:

### Authentication

* [ ] Signup works
* [ ] Login works
* [ ] Logout works
* [ ] Sessions work
* [ ] Passwords are securely hashed

### Event creation

* [ ] Any authenticated user can create an event proposal
* [ ] Event starts as pending approval
* [ ] Creator is not Organizer before approval
* [ ] Admin can review pending events
* [ ] Only Admin can approve
* [ ] Approval creates Organizer membership
* [ ] Creator can manage approved event

### Event configuration

* [ ] Dates work
* [ ] Team size is configurable
* [ ] Tracks work
* [ ] Prizes work
* [ ] Custom questions work

### Teams

* [ ] Team creation works
* [ ] Team owner is assigned
* [ ] One team per participant per event
* [ ] Invite links work
* [ ] Invite authentication flow works
* [ ] Capacity is enforced
* [ ] Deadline is enforced

### Projects

* [ ] One project per team
* [ ] Draft editing works
* [ ] Submit works
* [ ] Reopen works before deadline
* [ ] Resubmit works
* [ ] Custom answers work
* [ ] Images work
* [ ] Images persist
* [ ] Deadline locks project

### Public gallery

* [ ] Submitted projects become public after deadline
* [ ] Non-submitted projects remain private
* [ ] Search works
* [ ] Event filtering works
* [ ] Track filtering works
* [ ] Public project pages work

### Roles

* [ ] Participant permissions work
* [ ] Judge can access appropriate event/project context
* [ ] Organizer permissions work
* [ ] Admin approval works
* [ ] Cross-event permissions are isolated

### Security

* [ ] Protected endpoints require authentication
* [ ] Role checks happen server-side
* [ ] Cross-event ID manipulation fails
* [ ] Deadline checks happen server-side
* [ ] Invite tokens are secure
* [ ] Input validation exists
* [ ] Upload validation exists

---

# 43. Definition of Done

The implementation is complete when:

1. A normal authenticated user can create an event proposal.
2. The proposal enters a pending approval state.
3. An Admin can approve it.
4. Approval automatically makes the creator the Organizer of that event.
5. The Organizer can configure and run the event.
6. Event lifecycle status is correctly derived from approval state and configured dates.
7. Participants can form teams using invitation links.
8. Invitation state survives signup/login.
9. Teams can create and edit a shared project.
10. Projects can be submitted, reopened, edited, and resubmitted before the deadline.
11. The backend strictly locks projects and team changes after the deadline.
12. Local project images persist across application restarts when persistent storage is mounted.
13. Submitted projects become publicly visible after the deadline.
14. The public gallery supports search and filtering.
15. Organizers can assign and remove event-scoped Judges.
16. Judge, Participant, Organizer, and Admin permissions are correctly isolated.
17. Cross-event authorization is enforced server-side.
18. The complete lifecycle works from a clean local installation.
19. The implementation remains limited to the functionality specified in this document.

---

# 44. Final Product Principle

The platform should feel like one coherent system rather than separate role-based applications.

A single user account can participate in the platform in different ways:

```text
                    ┌── Participant in Event A
Authenticated User ─┼── Organizer in Event B
                    ├── Judge in Event C
                    └── Creator of pending Event D
```

The event determines the user's role.

The Admin provides platform-level approval.

The creator of an approved event becomes its Organizer.

The backend, not the frontend, is the final authority for every permission and deadline.

Build the smallest complete system that makes this lifecycle reliable.
