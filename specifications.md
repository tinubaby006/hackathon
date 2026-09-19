# DOGFOOD Hackathon — Specification

## Tier 1 — Core

            Tier 1 is mandatory.

            ### Authentication & Sessions

            * Authentication
            * Session management
            * Role-based access control

            ### Roles

            * Visitor
            * Participant
            * Judge
            * Organizer
            * Admin

            ### Event Management

            * Create events
            * Configure event dates
            * Configure tracks
            * Configure prizes

            ### Team Management

            * Create/form teams
            * Invite members using invite links

            ### Project Submission

            * Create project submissions
            * Save submissions as drafts
            * Edit submissions until the deadline
            * Enforce submission deadline on the backend

            ### Public Project Gallery

            * Browse projects
            * Search projects
            * Filter projects

            ### Project Fields

            Each project should support:

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
            * Organizer-defined custom questions

            ---

# Tier 2 — Judging

            Tier 2 adds the judging system.

            ### Judge Management

            * Invite judges
            * Assign judges to projects
            * Support manual/batch assignment
            * Support algorithmic assignment

            ### Judging Rubric

            * Organizer-configurable judging criteria
            * Weighted criteria
            * Configurable weights
            * Judge scoring

            ### Role Isolation

            Backend/API authorization must enforce isolation.

            Judges must:

            * See projects assigned to them
            * See their own scores

            Judges must NOT:

            * See another judge's scores
            * See scores from another track when unauthorized
            * Access unauthorized judging data through direct API requests

            Organizers/Admins may access aggregate judging information according to their permissions.

            ### Judging Progress

            Provide a live organizer dashboard showing:

            * Judges who have started judging
            * Judges who have not started
            * Judging progress

            ### Score Normalization

            * Implement cross-judge score normalization
            * Document the normalization methodology
            * Defend/explain why the chosen method is appropriate

            ### Export

            * CSV export
            * Export available at every applicable judging stage

            ---

# Tier 3 — Public

        Tier 3 adds community participation.

        ### Community Voting

        Support public/community voting through one of:

        * Open link
        * Email-gated voting
        * Authenticated voting

        ### Voting Mechanism

        Support:

        * One-person-one-vote

        Or an alternative mechanism such as:

        * Quadratic voting

        ### Comments

        * Users can comment on public gallery projects

        ### Voting Result Privacy

        While voting is active:

        * Voting results are hidden from the public
        * Organizers can view results

        ### Ballot Randomization

        * Randomize project/ballot ordering
        * Reduce position bias

        ### Anti-Abuse

        Implement mechanisms such as:

        * Rate limiting
        * Duplicate-vote detection
        * Audit trail

        ### Audit Trail

        Organizers must be able to:

        * Review voting activity
        * Investigate suspicious activity
        * Access the audit information without directly querying the database

        ---

# Tier 4 — Stretch

        Tier 4 adds advanced integrations and extensibility.

        ### REST API

        Provide a REST API covering actions available through the UI.

        ### Webhooks

        * Provide webhook support
        * Expose relevant platform events/actions through webhooks

        ### Certificates

        * Generate certificates

        ### Records

        * Generate participation/judging records

        ### Signed Judge Records

        * Generate signed judge-participation records
        * Make records publicly verifiable

        ### Embeddable Gallery

        * Provide an embeddable project gallery widget
        * Allow organizers to embed the gallery on external websites

        ### Bulk Migration

        Support:

        * Bulk import
        * Bulk export
        * Easy migration into the platform
        * Easy migration out of the platform

        ---

        # Local Development Requirements

        The platform should run locally using:

        ```bash
        docker compose up
        ```

        The application should be usable without requiring:

        * A cloud account
        * A hosted database
        * External APIs
        * External signup

        The project should provide seeded data sufficient to exercise the required functionality.

---

# Tier Progression

```text
TIER 1
Core Platform
    │
    ▼
TIER 2
Judging System
    │
    ▼
TIER 3
Public Voting & Community
    │
    ▼
TIER 4
API & Advanced Integrations
```

## Priority

Build in this order:

1. Tier 1 — Core
2. Tier 2 — Judging
3. Tier 3 — Public
4. Tier 4 — Stretch

A correctly implemented lower tier is more valuable than a higher tier with broken or incomplete functionality.
