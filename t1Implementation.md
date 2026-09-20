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

CSRF protection must be implemented for browser-based state-changing requests.

State-changing requests such as POST, PATCH, PUT, and DELETE must require a valid CSRF token associated with the authenticated session.

The CSRF token must be generated using a cryptographically secure random source and must not be accepted from an untrusted URL parameter.

The server must validate the CSRF token before performing the protected mutation.

Session cookies must use:

- HttpOnly
- SameSite=Lax or stricter
- Path=/
- Secure when the application is served over HTTPS

The session identifier must be opaque and unpredictable.

A successful login must establish a new authenticated session rather than trusting a session created before authentication.

Logout must invalidate the server-side session so the previous session identifier cannot be reused.

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

An approved Organizer may modify event structure only while the event lifecycle is DRAFT.

The following fields are considered event structure:

start date/time
submission deadline
end date/time
maximum team size
tracks
prizes
custom project questions

Once the event reaches ONGOING, these fields are frozen and cannot be created, removed, reordered, or modified.

The server must reject structural event updates when the effective lifecycle is ONGOING or ENDED.

The server's current time is authoritative for determining whether the structure is frozen.

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

Team creation must be performed atomically.

Creating a team and creating its owner/captain membership must occur in the same database transaction.

If either operation fails, the entire team creation operation must be rolled back.

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
A revoked invitation must always be rejected by the server, even when the invitation token itself is otherwise valid.

Revocation is represented by `revoked_at` being set on the invitation record.
* event membership
* existing team membership
* team capacity
* submission deadline

Invitation acceptance must be performed atomically.

The server must validate the invitation, verify event membership, verify existing team membership, verify team capacity, and create the team membership within one database transaction.

The capacity check and membership creation must occur within the same transaction so concurrent requests cannot bypass the configured team size.

If the membership cannot be created, the transaction must roll back without creating a partial membership state.

Do not rely only on frontend state to preserve the invitation.

---

# 15. Team Rules

Before the submission deadline:

* team owner can remove members
* team owner can delete the team
* invitations can be created/accepted
* team owner can revoke outstanding invitations
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

Project creation and submission state changes must be performed atomically.

Project creation must verify the team/event relationship and create the project within one database transaction.

Project submission must validate all required project data and custom answers before changing the project to SUBMITTED.

If any required validation or database operation fails, no partial submission state may be persisted.

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
./uploads:/app/data/uploads
```

The exact Docker configuration can be finalized during packaging, but the application must already use a persistent-volume-compatible upload directory.

Uploaded project images must never be exposed through a directly guessable static file path.

Image access must be controlled by the application:

* before the submission deadline, project images are private and accessible only to authorized project/team users and appropriate event administration users
* after the submission deadline, images belonging to valid submitted public projects may be served publicly
* images belonging to non-submitted projects must remain inaccessible to unauthenticated users

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

The submission deadline must be evaluated directly from the server's current time.

Use:

current_time < submission_deadline
    → submission-dependent mutations are allowed

current_time >= submission_deadline
    → submission-dependent mutations are rejected

The backend must not rely on a stored project or event status to determine whether the deadline has passed.

The deadline check must run on every participant-controlled mutation that is restricted by the submission deadline.

No background scheduler or cron job is required for deadline enforcement.

A project is considered effectively LOCKED once current_time >= submission_deadline, even if a persisted status field has not yet been updated.

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
csrf_token_hash
```

csrf_token_hash stores only a secure hash of the session's CSRF token.

The raw CSRF token must never be persisted in the database.

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
GET  /api/auth/csrf
````

GET /api/auth/csrf returns a CSRF token for the currently authenticated session.

State-changing authenticated endpoints must require the corresponding CSRF token.

The CSRF token must be validated server-side before authorization-sensitive mutations are executed.

**---**

## Event creation

Authenticated users can create event proposals:

```text
POST  /api/events
GET   /api/events
GET   /api/events/:eventId
PATCH /api/events/:eventId
GET   /api/me/events
```

Creation creates a pending event proposal.

The creator may edit their own pending proposal.

The event lifecycle status is determined by the configured dates and server time.

Clients must not directly set or transition `status`.

Protected Organizer operations require an approved `ORGANIZER` membership for the event.

For PATCH /api/events/:eventId:

pending proposals may be edited by their creator
approved events may have their event structure configured while the lifecycle is DRAFT
structural changes are rejected once the effective lifecycle is ONGOING
structural changes remain rejected while the lifecycle is ENDED
the backend must determine the effective lifecycle from server time
the client must not bypass the freeze by sending a lifecycle/status value

GET /api/events returns public approved events suitable for the public event listing.

It must not expose pending events as normal public events.

GET /api/me/events requires authentication and returns the authenticated user's event relationships, including event-scoped memberships and their own pending event proposals.

GET /api/events/:eventId returns only information the requester is authorized to view for that event.

Protected event data must not be exposed through the public event read.

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
GET    /api/events/:eventId/tracks
POST   /api/events/:eventId/tracks
PATCH  /api/events/:eventId/tracks/:trackId
DELETE /api/events/:eventId/tracks/:trackId

GET    /api/events/:eventId/prizes
POST   /api/events/:eventId/prizes
PATCH  /api/events/:eventId/prizes/:prizeId
DELETE /api/events/:eventId/prizes/:prizeId

GET    /api/events/:eventId/questions
POST   /api/events/:eventId/questions
PATCH  /api/events/:eventId/questions/:questionId
DELETE /api/events/:eventId/questions/:questionId
```

Read endpoints must enforce the same event boundaries as mutation endpoints.

Publicly readable event configuration may include tracks, prizes, and questions associated with an approved public event.

Pending-event and protected organizer data must remain restricted.

All mutation endpoints require appropriate Organizer authorization.

Read endpoints follow the read-access rules defined above.

**---**

## Event administration reads

GET /api/events/:eventId/members
GET /api/events/:eventId/teams
GET /api/events/:eventId/projects

These endpoints require an appropriate event-scoped role.

Organizer:
- may inspect participants, teams, and submitted projects for their event

Judge:
- may view the event context and submitted project information for their assigned event
- must not receive participant/team-management permissions

Participant:
- may access only resources permitted by their participant membership and team membership

Responses must not expose passwords, session identifiers, CSRF tokens, raw invitation tokens, or other private security data.

## Teams

```text
POST   /api/events/:eventId/teams
GET    /api/events/:eventId/teams
GET    /api/events/:eventId/teams/:teamId

GET    /api/teams/:teamId
GET    /api/teams/:teamId/project

POST   /api/teams/:teamId/invites
GET    /api/teams/:teamId/invites

GET    /api/team-invites/:token
POST   /api/team-invites/:token/accept

DELETE /api/teams/:teamId/members/:userId
DELETE /api/teams/:teamId
DELETE /api/team-invites/:inviteId
```

GET /api/team-invites/:token may be accessed while logged out and returns only the invitation context required to display the invitation flow.

It must not expose the raw stored token, private team data, or security-sensitive information.

Invitation acceptance remains an explicit POST action.

Invite revocation must:

* require authentication
* require the requester to be the owner/captain of the associated team
* verify that the invite exists
* verify that the invite belongs to the specified team
* mark the invite as revoked using `revoked_at`
* prevent the invite from being accepted after revocation
* be subject to the submission deadline

Revoking an invite must not remove existing team members.

Deadline rules apply to all participant-controlled team mutations.

Team creation, invite acceptance, member removal, and team deletion must use database transactions whenever multiple related records are created, updated, or deleted.

The server must perform authorization, deadline checks, capacity checks, and the related database mutation as one atomic operation where applicable.

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
GET    /api/projects/:projectId/images/:imageId
GET /api/events/:eventId/projects
GET /api/teams/:teamId/project
```
GET /api/events/:eventId/projects is restricted to appropriate Organizer and Judge event roles.

Organizers may inspect projects for their event.

Judges may view submitted projects for their assigned event.

Participants may access their team's project through GET /api/teams/:teamId/project when authorized.

Private project data must remain subject to the project's visibility and authorization rules.

Image access must be authorization-controlled.

The image endpoint must:

* verify that the image belongs to the requested project
* determine the project's current visibility from the server-side deadline and submission state
* allow authorized private access before the deadline
* allow public access only when the project is publicly visible
* reject unauthorized requests

Do not return the underlying filesystem path to clients.

Do not expose the upload directory through a public static route.

Every mutation must verify team membership and deadline state.

Project creation, submission, reopening, and resubmission must use database transactions when the operation changes multiple related records or requires validation followed by a state transition.

Validation and state mutation must be completed against the same authoritative server state.

A failed transaction must not leave a partially updated project, answer set, or submission state.

Deadline-dependent project mutations must evaluate the submission deadline using the server's current time at the moment of the request.

The API must reject the mutation when:

```text
current_time >= submission_deadline
```

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
* CSRF protection for state-changing requests
* HttpOnly session cookies
* SameSite cookie protection
* Secure cookies when served over HTTPS
* session rotation on successful authentication

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

Use a dedicated upload directory outside the application's public web root.

Uploaded files must be served through application-controlled routes rather than direct static filesystem access.

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
* state-changing request without CSRF token is rejected
* invalid CSRF token is rejected
* valid CSRF token allows the authenticated mutation
* CSRF token from another session is rejected
* session is rotated on successful login
* logout invalidates the previous session
* session cookie uses the required security attributes

## API read operations

- public event listing excludes pending events
- GET /api/me/events returns only the authenticated user's event relationships
- cross-event event reads are rejected
- Organizer can read participants for their event
- Organizer can read teams for their event
- Organizer can read submitted projects for their event
- Judge can read submitted project information for their assigned event
- Participant can read their authorized team and project
- participant cannot read another event's protected team/project data
- event tracks/prizes/questions are correctly scoped to the event
- logged-out user can read allowed invitation context without accepting the invite
- invitation read does not expose raw token or security-sensitive data

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
* team owner can revoke an outstanding invite
* revoked invite cannot be accepted
* revoking an invite does not remove existing team members
* non-owner cannot revoke another team's invite
* concurrent team creation and membership operations do not create invalid partial state
* concurrent invite acceptance cannot exceed max team size
* failed team creation rolls back all related records
* failed invite acceptance does not create a partial membership

## Projects

* create project
* edit draft
* submit
* reopen before deadline
* edit after reopening
* resubmit
* required custom question validation
* failed project submission does not leave the project partially submitted
* failed reopen/resubmit operation does not leave an inconsistent project state

## Deadline

Test at minimum:

current_time < submission_deadline
    → mutation allowed

current_time = submission_deadline
    → mutation rejected and project effectively locked

current_time > submission_deadline
    → mutation rejected and project effectively locked

Deadline tests must use controlled server time so the exact boundary can be tested deterministically.

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
* private project images cannot be accessed anonymously before the submission deadline
* authorized project/team users can access private project images
* submitted public project images become publicly accessible after the deadline
* non-submitted project images remain private after the deadline
* image requests cannot access files belonging to another project

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

Organizer can modify event structure while the approved event is DRAFT
Organizer cannot modify start time, submission deadline, end time, maximum team size, tracks, prizes, or custom questions once the event reaches ONGOING
structural event updates remain rejected after ENDED
exact start_at boundary is treated as ONGOING
rejected structural updates do not partially modify the event

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
9. Confirm authenticated user can view their My Events relationships
10. Confirm Organizer can read event participants, teams, and submitted projects
11. Confirm authenticated state-changing requests require a valid CSRF token
12. Confirm login establishes a new authenticated session
13. Confirm logout invalidates the session
14. Organizer configures event
15. Confirm event is DRAFT before start_at
16. Confirm Organizer can modify event structure while event is DRAFT
17. Participant joins event
18. Participant creates team
19. Participant generates invite
20. Second user opens invite while logged out
21. Second user signs up/logs in
22. Invite context survives authentication
23. Second user explicitly accepts invite
24. Team reaches valid membership state
25. Team creates project
26. Team edits project
27. Team submits project
28. Team reopens and resubmits before deadline
29. Confirm event becomes ONGOING at start_at
30. Confirm structural event configuration is frozen at start_at
31. Confirm Organizer cannot modify frozen event structure while ONGOING
32. Before the submission deadline, confirm project images are not publicly accessible
33. Confirm authorized project users can access their private images
34. Organizer assigns a Judge
35. Confirm Judge membership is created
36. Confirm Judge can access event/project context without participant permissions
37. Current server time reaches the submission deadline
38. Confirm project mutations are rejected at the exact deadline boundary
39. Confirm project is effectively locked
40. Confirm team changes are locked
41. Confirm submitted project is publicly visible
42. Confirm gallery search/filter works
43. Confirm event becomes ENDED at end_at
44. Confirm unauthorized users cannot modify protected resources
45. Confirm Judge cannot edit projects and has read-only project access

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

Organizers/Team owners can revoke outstanding team invitations, and revoked invitations cannot be accepted.

The implementation is complete when:

1. A normal authenticated user can create an event proposal.
2. The proposal enters a pending approval state.
3. An Admin can approve it.
4. Approval automatically makes the creator the Organizer of that event.
5. The Organizer can configure and run the event.
6. Event lifecycle status is correctly derived from approval state and configured dates.
7. Event structure can be configured during DRAFT and is frozen when the event becomes ONGOING.
8. Participants can form teams using invitation links.
9. Invitation state survives signup/login.
10. Teams can create and edit a shared project.
11. Projects can be submitted, reopened, edited, and resubmitted before the deadline.
12. The backend strictly enforces the submission deadline using server time, including the exact deadline boundary, and locks projects and team changes without requiring a scheduler.
13. Local project images persist across application restarts when persistent storage is mounted.
14. Project images are stored outside the public web root and are served according to project visibility and authorization rules.
15. Team creation, invitation acceptance, and project state transitions are atomic and cannot leave partial database state.
16. Submitted projects become publicly visible after the deadline.
17. The public gallery supports search and filtering.
18. Organizers can assign and remove event-scoped Judges.
19. Judge, Participant, Organizer, and Admin permissions are correctly isolated.
20. Cross-event authorization is enforced server-side.
21. The complete lifecycle works from a clean local installation.
22. The implementation remains limited to the functionality specified in this document.
23. Required event, membership, team, invitation, configuration, and project read operations are available with correct authorization and event isolation.

24. Public and protected read endpoints do not expose private or security-sensitive data.
25. State-changing authenticated requests are protected against CSRF.
26. Session cookies use appropriate security attributes and sessions are invalidated on logout.

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
