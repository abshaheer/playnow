# PlayNow — Database Design Specification

**Document Type:** Database Design Specification (Database.md)
**Source of Truth:** Product Requirements Document — PlayNow PRD v1.2 (Approved, Frozen); System Architecture Document — Architecture.md v1.1 (Approved, Frozen)
**Audience:** Backend Engineers, Database Administrators, QA
**Status:** Draft v1.0
**Engine:** MySQL 8.x — **Character Encoding:** utf8mb4 — **Timezone:** UTC (all timestamps stored and compared in UTC)
**Scope Note:** This is a design document, not an implementation artifact. It contains **no SQL**, **no ORM models**, and **no schema migration scripts** — only the structural design, rationale, and constraints that a backend developer needs to implement the schema correctly. Every entity maps to a specific PRD requirement (FR-XX/NFR-XX) or Architecture Document module, and no product feature not already present in PRD v1.2 is introduced here.

---

## 1. Database Philosophy

**Why a relational database:** PlayNow's core data has clear, well-defined relationships with strong consistency requirements — a Session cannot reference a nonexistent participant, a Report cannot reference a nonexistent user, and Reputation Metrics must accurately reflect real Session outcomes. A relational model with enforced foreign keys is the natural fit for this kind of interconnected, integrity-sensitive data, and is more appropriate for an MVP of this size than a document or key-value store, which would push referential integrity into application code.

**Why MySQL:** MySQL 8.x is mature, widely documented, has first-class support on the project's chosen hosting platform (Railway, per Architecture.md Section 2), and is a technology most engineering teams — including college-project teams — already have some exposure to, which directly supports the "easy to understand, easy to maintain" goal. MySQL 8.x's JSON column type and window functions also provide useful flexibility for the handful of semi-structured fields in this design (e.g., AuditLogs metadata) without requiring a separate NoSQL store.

**Normalization strategy:** The schema targets **Third Normal Form (3NF)** throughout: every non-key column depends on the whole primary key and nothing but the primary key. Concretely, this means:
- Volatile, frequently-updated operational state (e.g., a user's current matching status) is kept in its own table rather than mixed into the durable Users profile row, to avoid both write contention and mixed-responsibility columns.
- Derived/aggregated facts (e.g., Reputation Metrics counters) live in their own table rather than as columns on Users, since they are facts *about* a user's history, not intrinsic identity attributes.
- No column stores a value that could instead be computed from, or is redundant with, another table — with the narrow, explicitly justified exception of a cached `verification_level` value (Section 4, Verification) kept for query performance.

**Future scalability philosophy:** Rather than over-building for scale the MVP doesn't yet need, this design follows the same principle established in Architecture.md Section 21: keep the schema **shaped correctly** so that future growth (more activities, more users, Group Sessions, multiple cities, internationalization) is additive — new tables or nullable columns — rather than requiring a redesign of existing tables. The clearest example is `SessionParticipants`, modeled as a proper many-to-many bridge table from day one even though the MVP only ever populates two rows per Session.

---

## 2. Entity Overview

| Entity | Responsibility |
|---|---|
| **Users** | Core account identity and durable profile data: credentials, display name, gender (for filtering only), date of birth (for age validation only), Theme Preference, and account status. |
| **Verification** | A user's tiered Verification Level (Level 1 Email / Level 2 Mobile / Level 3 College-University), decoupled from Users so optional steps never gate core functionality (FR-21). |
| **UserStatus** | The volatile, frequently-updated snapshot of a user's current matching state: status (Offline/Searching/Matched/Busy), Availability, selected activity, search radius, gender preference filter, and approximate location (FR-09, FR-22, FR-06, FR-07, FR-08). |
| **Activities** | The extensible Sports Catalog (FR-05) — sports available for selection, stored as data so new sports require no code deployment. |
| **Sessions** | The core Session domain object (Architecture Section 8.0) — a confirmed match's activity, status, meeting point, compatibility score, and expiration timer (FR-11, FR-12, FR-18, FR-23). |
| **SessionParticipants** | The bridge between Users and Sessions — who is part of a given Session, and their individual acceptance/no-show/departure state. Modeled as a collection to remain Group-Session-ready (Architecture Section 21). |
| **Messages** | Ephemeral Anonymous Temporary Match Chat content, scoped to a single Session and intended for short retention only (FR-16). |
| **Reports** | User-submitted reports of another user's behavior, optionally tied to a Session, feeding the Moderation Module (FR-17). |
| **ModerationActions** | Enforcement actions (Temporary Suspension, Permanent Ban) taken against a user as a result of a confirmed report, implementing the two-strike policy (FR-17.3, FR-17.4), with a reserved field for future Appeal Logging. |
| **ReputationMetrics** | Aggregated, objective, system-derived statistics (Matches Completed, Cancelled Matches, No-Shows) shown to a matched user — explicitly not a store of peer ratings, which do not exist in this product (FR-19.5). |
| **AuditLogs** | Append-only record of significant system events (login, Session lifecycle transitions, moderation actions) for debugging, auditing, and monitoring (Architecture Section 1.9/4, Logging Module). |

No entity beyond this list is introduced. In particular, there is deliberately **no separate "Ratings" entity** (peer-to-peer rating is explicitly excluded by the PRD, FR-19.5), and **no "Group" or "Team" entity** (Group Sessions are a Future Enhancement, not MVP scope — see Section 16 and Section 19 for how the current design accommodates them later).

---

## 3. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o| VERIFICATION : has
    USERS ||--o| USER_STATUS : has
    USERS ||--o| REPUTATION_METRICS : has
    USERS ||--o{ SESSION_PARTICIPANTS : participates_in
    SESSIONS ||--o{ SESSION_PARTICIPANTS : includes
    ACTIVITIES ||--o{ SESSIONS : categorizes
    ACTIVITIES ||--o{ USER_STATUS : currently_selected_by
    SESSIONS ||--o{ MESSAGES : contains
    USERS ||--o{ MESSAGES : sends
    USERS ||--o{ REPORTS : files_as_reporter
    USERS ||--o{ REPORTS : is_subject_of
    SESSIONS ||--o{ REPORTS : gives_context_to
    USERS ||--o{ MODERATION_ACTIONS : is_subject_of
    REPORTS ||--o{ MODERATION_ACTIONS : leads_to
    USERS ||--o{ AUDIT_LOGS : generates
    SESSIONS ||--o{ AUDIT_LOGS : generates

    USERS {
        bigint id PK
        uuid public_id
        string email
        string password_hash
        string display_name
        date date_of_birth
        enum gender
        enum theme_preference
        enum account_status
        string photo_url
        timestamp created_at
        timestamp updated_at
    }

    VERIFICATION {
        bigint id PK
        bigint user_id FK
        timestamp email_verified_at
        timestamp mobile_verified_at
        string mobile_number_hash
        timestamp college_verified_at
        string college_domain
        tinyint verification_level
        timestamp updated_at
    }

    USER_STATUS {
        bigint user_id PK_FK
        enum status
        enum availability
        bigint selected_activity_id FK
        int search_radius_km
        enum gender_preference
        decimal approx_latitude
        decimal approx_longitude
        timestamp last_location_updated_at
        timestamp updated_at
    }

    ACTIVITIES {
        bigint id PK
        string name
        string slug
        string icon_key
        boolean is_active
        int sort_order
        timestamp created_at
    }

    SESSIONS {
        bigint id PK
        uuid public_id
        bigint activity_id FK
        enum status
        int compatibility_score
        string meeting_point_name
        decimal meeting_point_latitude
        decimal meeting_point_longitude
        string meeting_point_place_ref
        timestamp expires_at
        timestamp created_at
        timestamp ended_at
        enum ended_reason
    }

    SESSION_PARTICIPANTS {
        bigint id PK
        bigint session_id FK
        bigint user_id FK
        timestamp accepted_at
        timestamp left_at
        boolean no_show
        timestamp created_at
    }

    MESSAGES {
        bigint id PK
        bigint session_id FK
        bigint sender_user_id FK
        text body
        timestamp created_at
    }

    REPORTS {
        bigint id PK
        uuid public_id
        bigint reporter_user_id FK
        bigint reported_user_id FK
        bigint session_id FK
        enum category
        text message_excerpt
        enum status
        timestamp created_at
        timestamp reviewed_at
    }

    MODERATION_ACTIONS {
        bigint id PK
        bigint user_id FK
        bigint report_id FK
        enum action_type
        string reason
        timestamp starts_at
        timestamp ends_at
        enum appeal_status
        timestamp created_at
    }

    REPUTATION_METRICS {
        bigint user_id PK_FK
        int matches_completed
        int cancelled_matches
        int no_shows
        timestamp updated_at
    }

    AUDIT_LOGS {
        bigint id PK
        string event_type
        bigint user_id FK
        bigint session_id FK
        text metadata
        timestamp created_at
    }
```

---

## 4. Table Specifications

*Data types below are described at a high, engine-agnostic level (e.g., "String (short)", "Integer") rather than as exact MySQL column types, consistent with this document's role as a design specification rather than a schema script.*

### 4.1 Users

**Purpose:** The durable identity and profile record for every account. The single source of truth for authentication credentials and the small set of profile attributes the PRD requires (FR-01, FR-02, FR-03, FR-04, FR-15).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | Internal numeric identifier, used for efficient joins. |
| `public_id` | UUID | No | generated at creation | External-facing identifier; never the internal `id` is exposed to clients. |
| `email` | String (short) | No | — | Registration/login credential (FR-01, FR-02). |
| `password_hash` | String (long) | No | — | Salted hash only; plaintext is never stored (NFR-04). |
| `display_name` | String (short) | No | — | User-chosen display name (FR-03.1). |
| `date_of_birth` | Date | No | — | Used only to validate minimum age at registration; never displayed to other users. |
| `gender` | Enum (Male / Female / Other / PreferNotToSay) | No | — | Used only for the Gender Preference Filter (FR-08); explicitly never used for Theme Preference (FR-04.3). |
| `theme_preference` | Enum (Dark / Light / SystemDefault) | No | `SystemDefault` | FR-15. |
| `account_status` | Enum (Active / Suspended / Banned / Deactivated) | No | `Active` | Kept in sync with the most recent applicable `ModerationActions` row. |
| `photo_url` | String (long) | Yes | `NULL` | Optional profile photo reference (FR-03.1 — "optional"). |
| `created_at` | Timestamp (UTC) | No | current time | Also serves as the "Member Since" value in Reputation Metrics display (FR-19.1). |
| `updated_at` | Timestamp (UTC) | No | current time, on update | Standard audit column. |

**Primary Key:** `id`
**Foreign Keys:** None (Users is the root entity).
**Unique Constraints:** `email`, `public_id`.
**Business Rules:**
- `gender` and `theme_preference` are architecturally independent; no application logic may derive one from the other (FR-04.3).
- `account_status = Suspended` must always correspond to an active (`ends_at` in the future) `ModerationActions` row for this user; `Banned` corresponds to a `PermanentBan` row.
**Validation Rules:** `email` must be a syntactically valid address (application-layer check); `date_of_birth` must indicate an age at or above the platform's minimum age at registration time.

---

### 4.2 Verification

**Purpose:** Tracks a user's tiered, largely-optional identity verification (FR-21), kept separate from Users so that optional steps never block or complicate core profile writes.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `user_id` | Integer (FK → Users.id) | No | — | One row per user (1:1). |
| `email_verified_at` | Timestamp (UTC) | Yes | `NULL` | Populated once Level 1 (mandatory) is confirmed. |
| `mobile_verified_at` | Timestamp (UTC) | Yes | `NULL` | Populated once Level 2 (optional) is confirmed. |
| `mobile_number_hash` | String (short) | Yes | `NULL` | A hash of the verified mobile number, retained only to detect duplicate verification attempts — the raw number itself is not retained once verification succeeds (privacy minimization). |
| `college_verified_at` | Timestamp (UTC) | Yes | `NULL` | Populated once Level 3 (optional) is confirmed. |
| `college_domain` | String (short) | Yes | `NULL` | Only the institutional email **domain** (e.g., `university.edu`) is retained as proof of affiliation — never the full institutional email address. |
| `verification_level` | Small Integer (1 / 2 / 3) | No | `1` | A cached, derived summary of the highest level reached, kept for query/display performance (the one intentionally denormalized field in this schema — see Section 1). |
| `updated_at` | Timestamp (UTC) | No | current time, on update | |

**Primary Key:** `id`
**Foreign Keys:** `user_id` → `Users.id`
**Unique Constraints:** `user_id` (enforces 1:1).
**Business Rules:**
- Level 2 and Level 3 are never required for any core matching feature (FR-21.2); their absence must never block a write to `UserStatus`.
- `verification_level` must always equal the highest level for which a `*_verified_at` timestamp is populated; recomputed whenever a new level is confirmed.
**Validation Rules:** A row must exist for every user with `email_verified_at` populated before that user's `UserStatus.status` may transition to `Searching` (application-layer enforcement of FR-01/FR-21.1).

---

### 4.3 UserStatus

**Purpose:** The single, frequently-overwritten row representing a user's *current* matching state — deliberately separated from the durable `Users` row (Section 1) to isolate high-write-frequency operational data (FR-09, FR-22, FR-06, FR-07, FR-08).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `user_id` | Integer (PK & FK → Users.id) | No | — | One row per user (1:1); acts as both primary key and foreign key. |
| `status` | Enum (Offline / Searching / Matched / Busy) | No | `Offline` | FR-09.1. |
| `availability` | Enum (AvailableNow) | Yes | `NULL` | The MVP supports exactly one value (FR-22.1); populated only while `status = Searching`. Modeled as its own column, rather than folded into `status`, so a future "AvailableLater" scheduling value is purely additive. |
| `selected_activity_id` | Integer (FK → Activities.id) | Yes | `NULL` | The sport currently selected (FR-05); required only while Searching. |
| `search_radius_km` | Integer | Yes | `NULL` | One of the PRD's predefined radius options (FR-07.1). |
| `gender_preference` | Enum (Male / Female / Any) | Yes | `NULL` | FR-08.1. |
| `approx_latitude` / `approx_longitude` | Decimal (reduced precision) | Yes | `NULL` | Deliberately **rounded/reduced** location, never the raw device GPS reading (Architecture Section 8); cleared whenever `status` moves to `Offline` or `Busy` (FR-09.3). |
| `last_location_updated_at` | Timestamp (UTC) | Yes | `NULL` | Supports periodic refresh (FR-06.3). |
| `updated_at` | Timestamp (UTC) | No | current time, on update | |

**Primary Key:** `user_id`
**Foreign Keys:** `user_id` → `Users.id`; `selected_activity_id` → `Activities.id`
**Business Rules:**
- Because this is a single row per user, a user can only ever have **one** status value at a time — this is how "one active search at a time" (Section 12) is naturally enforced by the schema shape itself, with no additional constraint needed.
- `selected_activity_id`, `search_radius_km`, and `gender_preference` are required (application-layer, not database-layer, `NOT NULL`) only when `status = Searching`; keeping them nullable at the schema level avoids awkward placeholder values for Offline/Busy users.
- `approx_latitude`/`approx_longitude` must never be written with full device precision — precision reduction happens before this row is written (Architecture Section 8).
**Validation Rules:** `search_radius_km` must be one of the platform's predefined values (FR-07.1); `Verification.email_verified_at` must be non-null before `status` may become `Searching`.

---

### 4.4 Activities (Sports Catalog)

**Purpose:** The extensible catalog of sports (FR-05), designed explicitly so new sports are a **data change**, not a code deployment (NFR-12).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | Catalog data is not personally sensitive, so the internal numeric ID doubles as the stable public reference — no separate UUID is needed here. |
| `name` | String (short) | No | — | Display name, e.g., "Football", "Badminton". |
| `slug` | String (short) | No | — | Stable, URL/reference-friendly key, e.g., `football`. |
| `icon_key` | String (short) | Yes | `NULL` | Reference to a frontend icon/asset. |
| `is_active` | Boolean | No | `true` | Used to retire a sport from selection without deleting it, preserving historical `Sessions.activity_id` references. |
| `sort_order` | Integer | No | `0` | Controls catalog display order. |
| `created_at` | Timestamp (UTC) | No | current time | |

**Primary Key:** `id`
**Foreign Keys:** None.
**Unique Constraints:** `name`, `slug`.
**Business Rules:** Sports are never hard-deleted once any `Sessions` or `UserStatus` row references them — only deactivated (`is_active = false`) — to preserve referential integrity and historical accuracy (Section 12).
**Seed Data:** Initial rows correspond exactly to the 19 sports listed in PRD FR-05.2: Football, Cricket, Badminton, Padel, Basketball, Volleyball, Table Tennis, Tennis, Chess, Running, Cycling, Swimming, Frisbee, Skating, Hiking, Yoga, Jogging, Boxing (training partner), and Martial Arts Practice.

---

### 4.5 Sessions

**Purpose:** The Session domain object (Architecture Section 8.0) — created only once a match is mutually confirmed (FR-11.3), and tracked through to a terminal outcome.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `public_id` | UUID | No | generated at creation | External reference for this Session (e.g., used by the Frontend/Chat/Maps flows). |
| `activity_id` | Integer (FK → Activities.id) | No | — | The Session's sport (FR-05.4). |
| `status` | Enum (Active / Completed / Expired / Cancelled) | No | `Active` | A row is only ever inserted once both participants accept (FR-11.3); pre-acceptance candidate matching is a transient Matching Module concern (Architecture Section 6) and is never persisted here. |
| `compatibility_score` | Integer | Yes | `NULL` | The informational Match Compatibility Score shown at "Match Found" (FR-23.1); retained for post-hoc analysis of the scoring rules' usefulness. |
| `meeting_point_name` | String (short) | Yes | `NULL` | Populated at Session creation from the top-ranked Meeting Point Recommendation (FR-12.5). |
| `meeting_point_latitude` / `meeting_point_longitude` | Decimal | Yes | `NULL` | The **venue's** public location — not a user's location; safe to store precisely since it is a public place (FR-12.4). |
| `meeting_point_place_ref` | String (short) | Yes | `NULL` | An external Maps place identifier, allowing the venue to be redisplayed without a redundant re-query. |
| `expires_at` | Timestamp (UTC) | Yes | `NULL` | Set at creation to `created_at + 10 minutes` by default (FR-18.1); the authoritative, server-side expiration timer (Architecture Section 13). |
| `created_at` | Timestamp (UTC) | No | current time | The moment the Session became Active. |
| `ended_at` | Timestamp (UTC) | Yes | `NULL` | Populated when the Session reaches a terminal state. |
| `ended_reason` | Enum (Completed / Expired / CancelledByParticipant) | Yes | `NULL` | Populated alongside `ended_at`. |

**Primary Key:** `id`
**Foreign Keys:** `activity_id` → `Activities.id`
**Business Rules:**
- Only the single top-ranked venue is persisted (FR-12.5); the full ranked candidate list computed by the Maps Module is a transient, in-request computation and is never written to the database (Section 15, Performance).
- `status` transitions strictly follow Architecture Section 6's state machine: `Active → {Completed, Expired, Cancelled}` only; no other transitions are valid.
- `expires_at` is authoritative and server-set; it is never derived from or overridden by client-supplied timestamps.
**Validation Rules:** A Session cannot be created referencing an `activity_id` where `Activities.is_active = false`.

---

### 4.6 SessionParticipants

**Purpose:** The bridge between `Users` and `Sessions`, modeled as a proper many-to-many association — not two fixed, named columns — specifically so that a future Group Sessions feature (PRD Section 18, Architecture Section 21.1–21.2) is additive rather than a redesign.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | A surrogate key is used (rather than a composite `session_id + user_id` PK) to leave room for future per-participant metadata without complicating the key structure. |
| `session_id` | Integer (FK → Sessions.id) | No | — | |
| `user_id` | Integer (FK → Users.id) | No | — | |
| `accepted_at` | Timestamp (UTC) | No | current time | When this participant accepted the match (FR-11.1). |
| `left_at` | Timestamp (UTC) | Yes | `NULL` | Populated if this participant manually cancels/leaves before the Session's natural end (FR-18.4). |
| `no_show` | Boolean | No | `false` | Set to `true` if the Session expires without this participant confirming the meetup began; feeds `ReputationMetrics.no_shows`. |
| `created_at` | Timestamp (UTC) | No | current time | |

**Primary Key:** `id`
**Foreign Keys:** `session_id` → `Sessions.id`; `user_id` → `Users.id`
**Unique Constraints:** (`session_id`, `user_id`) — a user cannot appear twice in the same Session.
**Business Rules:** The **MVP enforces exactly two rows per `session_id`** (PRD Section 5, Scope: strictly one-to-one matching) — this is an **application-layer** invariant, deliberately not a hard schema limit, so that a future Group Sessions capability can be introduced by relaxing that application check alone (Architecture Section 21.1–21.2), with no table redesign.

---

### 4.7 Messages

**Purpose:** The Anonymous Temporary Match Chat's message content, scoped strictly to one Session (FR-16), and designed from the outset for short retention rather than as a permanent chat history store.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `session_id` | Integer (FK → Sessions.id) | No | — | Every message belongs to exactly one Session; delivery is scoped accordingly (FR-16.2). |
| `sender_user_id` | Integer (FK → Users.id) | No | — | Used only for delivery routing and moderation attribution — **never** surfaced as an identity to the other participant (FR-16.2). |
| `body` | Text | No | — | Message content. Filtering for identity-revealing patterns (FR-16.2) and moderation flags (FR-17.1) is application logic, not a schema-level constraint. |
| `created_at` | Timestamp (UTC) | No | current time | |

**Primary Key:** `id`
**Foreign Keys:** `session_id` → `Sessions.id`; `sender_user_id` → `Users.id`
**Business Rules:**
- Rows in this table are expected to be **hard-deleted** shortly after their owning Session ends (FR-16.3, FR-16.4) — this table is not intended to accumulate indefinitely (see Section 6, Data Lifecycle, and Section 18, Risks).
- There is deliberately **no foreign key from `Reports` to `Messages`** — a reported message is captured as a standalone excerpt in `Reports.message_excerpt` (Section 4.8) precisely so that `Messages` rows can be purged on their own schedule without being blocked by, or accidentally cascading into, moderation records.

---

### 4.8 Reports

**Purpose:** A user-submitted report of another user's behavior (FR-16.5, FR-17), optionally tied to the Session/chat context it arose from.

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `public_id` | UUID | No | generated at creation | A reference number that could be surfaced to the reporting user. |
| `reporter_user_id` | Integer (FK → Users.id) | No | — | |
| `reported_user_id` | Integer (FK → Users.id) | No | — | |
| `session_id` | Integer (FK → Sessions.id) | Yes | `NULL` | Populated when the report originates from an active Session/chat. |
| `category` | Enum (Harassment / SexualHarassment / Grooming / HateSpeech / Threats / AbusiveLanguage / ContactInfoSharing / OffPlatformAttempt / Other) | No | — | Mirrors the unacceptable-behavior categories in FR-17.1. |
| `message_excerpt` | Text | Yes | `NULL` | A minimal, time-boxed excerpt retained strictly for manual review (FR-17.5) — never a full transcript. |
| `status` | Enum (Pending / UnderReview / Confirmed / Dismissed) | No | `Pending` | |
| `created_at` | Timestamp (UTC) | No | current time | |
| `reviewed_at` | Timestamp (UTC) | Yes | `NULL` | |

**Primary Key:** `id`
**Foreign Keys:** `reporter_user_id` → `Users.id`; `reported_user_id` → `Users.id`; `session_id` → `Sessions.id`
**Unique Constraints:** `public_id`.
**Business Rules:**
- `reporter_user_id` must never equal `reported_user_id` (application-layer validation).
- `message_excerpt`, when present, is purged once the report reaches a terminal `status` (`Confirmed`/`Dismissed`) plus a short grace period, even though the `Reports` row itself is retained for historical moderation statistics (Section 7, Privacy Strategy).
- Automated detection may create a `Pending` Report automatically; final `Confirmed`/`Dismissed` status always requires human review (FR-17.2) — this is an application-workflow rule, not a schema constraint.

---

### 4.9 ModerationActions

**Purpose:** Enforcement actions taken against a user, implementing the PRD's two-strike policy, and reserving structure for a future Appeal Logging capability (FR-17.3, FR-17.4).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `user_id` | Integer (FK → Users.id) | No | — | The user the action applies to. |
| `report_id` | Integer (FK → Reports.id) | Yes | `NULL` | The confirmed report that triggered this action, where applicable. |
| `action_type` | Enum (TemporarySuspension / PermanentBan) | No | — | |
| `reason` | String (medium) | No | — | A brief, non-verbatim summary — not a copy of chat content. |
| `starts_at` | Timestamp (UTC) | No | current time | |
| `ends_at` | Timestamp (UTC) | Yes | `NULL` | `starts_at + 7 days` for `TemporarySuspension`; always `NULL` for `PermanentBan`. |
| `appeal_status` | Enum (NotApplicable / Pending / Reviewed) | No | `NotApplicable` | Reserved for a **future** Appeal Logging capability noted in FR-17; not part of any active MVP workflow. |
| `created_at` | Timestamp (UTC) | No | current time | |

**Primary Key:** `id`
**Foreign Keys:** `user_id` → `Users.id`; `report_id` → `Reports.id`
**Business Rules:**
- A user's **second** confirmed `ModerationActions` row results in `action_type = PermanentBan` (FR-17.4); this is enforced at the application layer by counting prior rows for that `user_id`, not by a database trigger, to keep the schema simple.
- Every insert into this table must be mirrored by an update to `Users.account_status` and an `AuditLogs` entry (`UserSuspended` / `UserBanned`).

---

### 4.10 ReputationMetrics

**Purpose:** Objective, system-derived statistics shown to a matched user (FR-19) — deliberately excludes any subjective, peer-submitted rating, since PlayNow does not implement peer-to-peer ratings (FR-19.5).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `user_id` | Integer (PK & FK → Users.id) | No | — | One row per user (1:1). |
| `matches_completed` | Integer | No | `0` | Incremented when a Session this user participated in reaches `Completed`. |
| `cancelled_matches` | Integer | No | `0` | Incremented when a Session this user participated in reaches `Cancelled`. |
| `no_shows` | Integer | No | `0` | Incremented when this user's `SessionParticipants.no_show = true` on an `Expired` Session. |
| `updated_at` | Timestamp (UTC) | No | current time, on update | |

**Primary Key:** `user_id`
**Foreign Keys:** `user_id` → `Users.id`
**Business Rules:** All three counters are written **exclusively** by system logic reacting to `Sessions`/`SessionParticipants` terminal state transitions (Section 6, Data Lifecycle). There is intentionally **no application code path that allows a user to directly write to another user's `ReputationMetrics` row** — this is the schema-level guarantee behind "no peer-to-peer ratings" (FR-19.5).

---

### 4.11 AuditLogs

**Purpose:** An append-only record of significant system events for debugging, auditing, and monitoring (Architecture Section 1.9, Logging Module).

| Column | Type | Nullable | Default | Notes |
|---|---|---|---|---|
| `id` | Integer (internal PK) | No | auto-increment | |
| `event_type` | String (short) | No | — | E.g., `UserLogin`, `SessionCreated`, `SessionCompleted`, `SessionExpired`, `ReportSubmitted`, `UserSuspended`, `UserBanned`. |
| `user_id` | Integer (FK → Users.id) | Yes | `NULL` | The primary user associated with the event, where applicable. |
| `session_id` | Integer (FK → Sessions.id) | Yes | `NULL` | Populated for Session-lifecycle events. |
| `metadata` | Text (JSON-structured) | Yes | `NULL` | Minimal structured context (e.g., previous status → new status); explicitly excludes PII and raw message content. |
| `created_at` | Timestamp (UTC) | No | current time | |

**Primary Key:** `id`
**Foreign Keys:** `user_id` → `Users.id`; `session_id` → `Sessions.id`
**Business Rules:** This table is **append-only** in normal operation — no application workflow updates or deletes existing rows. Retention/archival is an operational policy decision (Section 15, Section 18), not a schema constraint.

---

## 5. Relationship Design

**One-to-one:**
- `Users` ↔ `Verification`, `Users` ↔ `UserStatus`, `Users` ↔ `ReputationMetrics` — each exists as its own table (rather than columns on `Users`) specifically to isolate a distinct write pattern (highly volatile for `UserStatus`; append-mostly for `ReputationMetrics`; rarely-written for `Verification`) from the durable core identity row, following the normalization philosophy in Section 1.

**One-to-many:**
- `Activities → Sessions` — one sport can be the subject of many Sessions over time.
- `Activities → UserStatus` — one sport can currently be selected by many Searching users.
- `Sessions → Messages` — one Session's chat can contain many messages.
- `Users → Reports` (**two distinct relationships**: `reporter_user_id` and `reported_user_id`) — one user can file many reports, and one user can be the subject of many reports; modeled as two separate foreign keys on the same `Reports` table rather than two separate tables, since a report is a single, atomic event with two user-role perspectives.
- `Sessions → Reports` — one Session can be the context for more than one report (e.g., both participants report each other).
- `Users → ModerationActions` — one user can accumulate multiple actions over time (this is exactly how the two-strike policy is evaluated).
- `Reports → ModerationActions` — in practice usually zero-or-one, modeled as one-to-many for schema simplicity (a report could theoretically inform more than one action record, e.g., an initial action plus a later reversal).
- `Users → AuditLogs`, `Sessions → AuditLogs` — one user or Session can generate many log events over its lifetime.

**Many-to-many:**
- `Users ↔ Sessions` via `SessionParticipants` — the only many-to-many relationship in the schema. It exists because a Session fundamentally involves multiple users and a user can be involved in multiple Sessions over time (never simultaneously in the MVP, but the schema does not encode that as a hard limit — see Section 4.6). This is also the specific design choice that keeps the schema Group-Session-ready without a redesign.

---

## 6. Data Lifecycle

```
User Registration (Users row created)
        ↓
Profile Creation (Users updated; Verification row created with Level 1 pending)
        ↓
Searching (UserStatus row updated: status=Searching, availability=AvailableNow)
        ↓
Session Created (Sessions row inserted with status=Active;
                  SessionParticipants rows inserted for both participants;
                  UserStatus.status updated to Matched for both)
        ↓
Chat Messages (Messages rows inserted, scoped to session_id)
        ↓
Session Completed / Expired / Cancelled (Sessions.status updated, ended_at/ended_reason set;
                  SessionParticipants.no_show set where applicable;
                  ReputationMetrics updated for both participants;
                  UserStatus.status returned to Searching or left as-is per user action)
        ↓
Reports (if any) — Reports row inserted, optionally referencing the Session;
                  ModerationActions row inserted if the report is confirmed
```

**Expired Sessions:** `Sessions.status` is set to `Expired`, `ended_at`/`ended_reason` populated, and the associated `Messages` rows for that Session become eligible for purge (see below). `SessionParticipants.no_show` is set to `true` for any participant who did not confirm the meetup began, which is then reflected in `ReputationMetrics.no_shows`.

**Cancelled Sessions:** Same as Expired, except `ended_reason = CancelledByParticipant` and `SessionParticipants.left_at` is populated for the cancelling participant; `ReputationMetrics.cancelled_matches` is incremented for the relevant participant(s) rather than `no_shows`.

**Temporary Chat Messages:** `Messages` rows are hard-deleted on a short, scheduled cadence after their owning Session reaches a terminal state (`Completed`/`Expired`/`Cancelled`) — except where a specific message has already been copied into a `Reports.message_excerpt` before deletion, per the moderation reporting flow (FR-16.5, FR-17.5). The `Messages` table is not intended to hold data indefinitely under any circumstance.

**Reports:** Retained after resolution for historical moderation statistics and audit purposes, but `message_excerpt` is purged after a short grace period following a terminal `status` (Section 4.8). Reports are never anonymized away from their `reporter_user_id`/`reported_user_id` references, since moderation accountability requires knowing who was involved — this is handled instead by strict access control (Section 14), not by data deletion.

**Inactive Accounts:** *(Not explicitly specified as a feature in PRD v1.2 — the following is a conservative, privacy-respecting default consistent with the PRD's stated principles, flagged here as a design assumption; see Section 17.)* Accounts with no login for an extended period may have their `UserStatus` approximate-location fields cleared as a hygiene measure, but `Users`, `Verification`, and `ReputationMetrics` rows are retained unless the account is explicitly deleted.

**Deleted Accounts:** *(Also a design assumption, not an explicit PRD feature.)* On account deletion, `Users.account_status` is set to `Deactivated` and PII-bearing columns (`email`, `display_name`, `photo_url`) are either removed or irreversibly anonymized, while non-identifying historical records needed for other users' `ReputationMetrics` (e.g., a completed Session count that involved the deleted user) are preserved in aggregate form rather than being deleted, so that a remaining participant's own statistics remain accurate. `Messages` and `Reports.message_excerpt` referencing a deleted account follow the same short-retention rules as any other user's data (Section 7).

---

## 7. Privacy Strategy

- **No precise GPS coordinates shared between users:** `UserStatus.approx_latitude`/`approx_longitude` store only reduced-precision values (Architecture Section 8); nothing in the schema carries raw device GPS precision to another user. `Sessions.meeting_point_*` coordinates are the **venue's** public location, not a user's — safe to store precisely.
- **No permanent chat history:** `Messages` rows are designed for short, scheduled deletion after a Session ends (Section 6), with the sole, narrow exception of a minimal excerpt copied into `Reports.message_excerpt` strictly for moderation review (FR-17.5).
- **Sensitive information stored separately:** Verification proof (`Verification`) and volatile location/matching state (`UserStatus`) are isolated from the core `Users` identity row, limiting the blast radius of any single table's exposure and making access-control policy easier to scope per table (Section 14).
- **PII never exposed unnecessarily:** `Messages.sender_user_id` and `SessionParticipants.user_id` exist for delivery/attribution only and are never surfaced as an identity to the other participant at the application layer (FR-16.2); `ReputationMetrics` and `Verification.verification_level` are designed to be displayable to a matched user without revealing any underlying identifier (FR-19.2, FR-21.3).
- **Hashing used where applicable:** `Users.password_hash` (salted password hash) and `Verification.mobile_number_hash` (to detect duplicate verification attempts without retaining the raw number) are the two hashed fields in this schema.
- **Data minimization in Verification:** Only a mobile number *hash* and a college email *domain* (not the full address) are retained — the minimum needed to prove verification occurred, nothing more.

---

## 8. Moderation Data

Moderation data is intentionally split across two tables with distinct responsibilities, mirroring the Moderation Module's role in Architecture.md Section 4:

- **Reports** — the raw record of what was reported, by whom, about whom, and (optionally) in what Session context, plus a minimal, time-limited excerpt for review (Section 4.8). This is the "evidence" table.
- **ModerationActions** — the record of what was *done* as a result — `TemporarySuspension` (7 days, first confirmed violation) or `PermanentBan` (second confirmed violation) — optionally linked back to the triggering `Reports` row (Section 4.9). This is the "consequences" table.

**Ban History / Suspensions:** Fully represented by querying `ModerationActions` for a given `user_id`, ordered by `starts_at` — no separate "history" table is needed, since `ModerationActions` is itself an append-only history.

**Warnings:** Not a distinct entity in this design, since the PRD's moderation policy (FR-17.3, FR-17.4) defines exactly two consequence tiers (Temporary Suspension, then Permanent Ban) with no separate "warning" step; introducing a Warnings table would be scope not present in the frozen PRD.

**Appeals (future):** Represented today only as the reserved `ModerationActions.appeal_status` column (default `NotApplicable`), so that a future appeal workflow can populate `Pending`/`Reviewed` values and, if a fuller appeal record is later needed, add a new `Appeals` table referencing `ModerationActions.id` — additive, not a redesign.

**No implementation, design only:** As with the rest of this document, the above describes structure and intent; the actual detection logic, review workflow, and UI are backend/frontend implementation concerns outside this document's scope.

---

## 9. Session Model

The `Sessions` table (Section 4.5), together with `SessionParticipants` (4.6), `Messages` (4.7), and the venue fields embedded in `Sessions` itself, together constitute the Session domain object described in Architecture.md Section 8.0:

- **Status:** `Active → {Completed, Expired, Cancelled}` (Section 4.5); no "PendingAcceptance" state is persisted, since a Session row is only ever created once both users have accepted (FR-11.3) — pre-acceptance candidate matching is a transient Matching Module concern, not database state.
- **Participants:** Represented via `SessionParticipants`, a proper many-to-many bridge rather than two fixed columns (`user_a`/`user_b`) — the specific design choice that keeps this model **Group-Session-ready** (Architecture Section 21.1–21.2) while the MVP enforces exactly two rows per Session at the application layer.
- **Meeting Point:** Stored directly on `Sessions` (`meeting_point_name`, `meeting_point_latitude`, `meeting_point_longitude`, `meeting_point_place_ref`) as the single top-ranked recommendation (FR-12.5) — the underlying ranked-candidate computation is transient and not persisted (Section 15).
- **Chat:** Represented by `Messages`, scoped via `session_id`, with a short-retention lifecycle independent of the Session's own retention (Section 6).
- **Activity:** A required foreign key to `Activities` (FR-05.4).
- **Timer / Expiration:** `Sessions.expires_at`, set at creation to the default 10-minute window (FR-18.1) and treated as authoritative server-side state (Architecture Section 13).
- **Completion:** `status = Completed`, `ended_reason = Completed`, `ended_at` populated; feeds `ReputationMetrics.matches_completed` for both participants.
- **Cancellation:** `status = Cancelled`, `ended_reason = CancelledByParticipant`, the cancelling participant's `SessionParticipants.left_at` populated (FR-18.4).
- **Future group support:** Already accommodated structurally by `SessionParticipants` being a bridge table rather than fixed columns; extending to more than two participants per Session requires no schema change, only an application-layer relaxation of the "exactly two rows" rule (Section 4.6, Section 16).

---

## 10. Activity Model

The `Activities` table (Section 4.4) is intentionally minimal: a `name`, a `slug`, an optional `icon_key`, an `is_active` flag, and a `sort_order`. This is sufficient to represent every sport named in the PRD (FR-05.2) — Football, Cricket, Badminton, Padel, Basketball, Volleyball, Tennis, Table Tennis, Chess, Running, Cycling, Swimming, Hiking, Frisbee, Skating, Yoga, Boxing (training partner), and Martial Arts Practice — as **rows**, not as columns or application-level constants.

**Why this design supports future activities without redesign:** Adding a new sport is a single `INSERT` into `Activities`; no other table's structure references sport names or types directly — every other table that cares about "which sport" does so via `activity_id`, a stable foreign key. Retiring a sport is a single `UPDATE` to `is_active`, never a `DELETE`, which would otherwise risk breaking referential integrity with historical `Sessions` rows.

---

## 11. User Model

Rather than a single, wide `Users` table, the "user" concept in this schema is deliberately composed of **four related tables**, each with a distinct responsibility and write pattern — directly following the normalization philosophy in Section 1:

| Concern | Table | Why Separated |
|---|---|---|
| Authentication & core identity | `Users` | Rarely written after registration; the durable "root" record. |
| Verification | `Verification` | Written only when a user completes an optional step; isolates PII-adjacent proof data. |
| Availability Status & current matching state | `UserStatus` | Written extremely frequently (every status/location refresh); isolating it prevents write contention on the durable `Users` row. |
| Reputation Metrics | `ReputationMetrics` | Written only by system logic reacting to Session outcomes; conceptually a *fact about* a user's history, not an intrinsic identity attribute. |

**Trust Indicators / Privacy Settings:** "Trust Indicators" (the PRD's earlier term) are fully represented by the combination of `Verification.verification_level`, `Users.created_at` ("Member Since"), and `ReputationMetrics` — no separate "Trust Indicators" table exists, since it would only duplicate data already modeled elsewhere. "Privacy Settings" as a distinct, user-configurable table is **not present**, because the PRD does not define any user-configurable privacy toggles beyond what is already structurally guaranteed (approximate-location-only, ephemeral chat, no PII display) — these are architectural guarantees (Section 7), not user preferences, so no such table is invented here.

**What is deliberately NOT stored:** No phone number in plaintext, no full college/university email address, no exact GPS history, no free-text "bio," and no social-media handles — consistent with FR-16.2/FR-19.2 and the instruction to store only what is necessary.

---

## 12. Constraints

- **Unique email:** `Users.email` is unique — enforced at the database level, not just the application layer, to prevent race-condition duplicate registrations.
- **Unique public identifiers:** `Users.public_id`, `Sessions.public_id`, `Reports.public_id` are each unique.
- **Valid Session states:** `Sessions.status` is constrained to the four-value enum (`Active`, `Completed`, `Expired`, `Cancelled`); no other value is representable.
- **Valid Availability/User Status states:** `UserStatus.status` is constrained to (`Offline`, `Searching`, `Matched`, `Busy`); `UserStatus.availability` is constrained to a single value (`AvailableNow`) for the MVP (FR-22.1), deliberately leaving room for a future enum expansion rather than a new column.
- **One active Session at a time (per user):** Not enforced by a single simple column constraint (a user could technically appear in more than one `SessionParticipants` row over time), but enforced by **application-layer logic** that refuses to create a new Session for a user whose `UserStatus.status` is already `Matched`. This is intentionally an application invariant rather than a rigid schema constraint, since a strict database-level version would complicate the future Group Sessions extension.
- **One active search at a time (per user):** Naturally guaranteed by `UserStatus` being a single row per `user_id` (1:1) — a user cannot represent two simultaneous search states in one row.
- **No self-reporting:** `Reports.reporter_user_id ≠ Reports.reported_user_id`, enforced at the application layer.
- **Referential integrity for historical accuracy:** `Activities` rows are never deleted while referenced by `Sessions` or `UserStatus` — only deactivated (`is_active = false`), preserving the accuracy of historical records.
- **SessionParticipants uniqueness:** (`session_id`, `user_id`) is unique — a user cannot be counted twice in the same Session.

---

## 13. Indexing Strategy

| Table.Column(s) | Purpose |
|---|---|
| `Users.email` | Login lookups and uniqueness enforcement (FR-02) — the single most frequent lookup in the system. |
| `Users.public_id` | External reference resolution without exposing internal IDs. |
| `UserStatus.status`, `UserStatus.selected_activity_id`, `UserStatus.search_radius_km`, `UserStatus.gender_preference` (composite) | Supports the Matching Module's core query: "find other `Searching` users with the same activity, compatible radius, and compatible gender preference" (FR-10.1) — the most performance-sensitive query in the system. |
| `UserStatus.approx_latitude`, `UserStatus.approx_longitude` | Supports nearby-candidate filtering; a simple range-based index is sufficient at MVP scale (see Section 15 on avoiding premature geo-optimization). |
| `Sessions.public_id` | External Session lookups from the Frontend/Chat/Maps flows. |
| `Sessions.status` | Supports background jobs scanning for `Active` Sessions nearing `expires_at` (FR-18.2). |
| `Sessions.expires_at` | Supports the expiration sweep (Section 15). |
| `SessionParticipants.session_id`, `SessionParticipants.user_id` | Both directions are queried constantly — "who is in this Session" and "what Sessions is this user part of." |
| `Messages.session_id` | The chat's primary access pattern is always "all messages for this Session," in order. |
| `Reports.status` | Supports the Moderation Module's queue of `Pending`/`UnderReview` reports. |
| `Reports.reported_user_id` | Supports counting prior confirmed reports against a user for the two-strike policy (FR-17.4). |
| `ModerationActions.user_id`, `ModerationActions.ends_at` | Supports checking a user's current suspension/ban status quickly, and scanning for suspensions about to expire. |
| `AuditLogs.event_type`, `AuditLogs.created_at` (composite) | Supports monitoring/auditing queries filtered by event type over a time range. |

No index is proposed for low-cardinality or rarely-filtered columns (e.g., `Users.gender`, `Users.theme_preference`) to avoid unnecessary write overhead, consistent with "avoid premature optimization" (Section 15).

---

## 14. Security Considerations

- **Password hashing:** `Users.password_hash` stores a salted hash only (e.g., a strong, purpose-built password hashing algorithm); plaintext passwords are never written to any table or log (NFR-04).
- **JWT relationship:** Per Architecture.md Section 2/9, authentication uses signed JWTs. The database does **not** store active tokens — JWTs are stateless and self-validating (signature + expiry), so no `Sessions`-style token table is needed for authentication (not to be confused with the `Sessions` table in this schema, which represents a match, not a login session). Logout is handled client-side/at the token-validation layer, not via a database row.
- **Data encryption:** Encryption in transit (TLS/HTTPS) is handled at the application/hosting layer (Architecture Section 12). Encryption at rest is expected to be provided by the managed MySQL hosting platform (Railway); no column-level application encryption is introduced for MVP scope, keeping with "avoid unnecessary complexity."
- **PII protection:** The tables most likely to contain PII-adjacent data (`Users`, `Verification`, `Messages`) are the explicit focus of the access-control and retention rules in Section 7; `ReputationMetrics`, `Activities`, and `AuditLogs.metadata` are designed to never require PII to function.
- **Moderation records:** `Reports` and `ModerationActions` are treated as sensitive operational data — access should be limited to the Moderation Module's backend logic, not general application queries (least-privilege, below).
- **Audit logging:** `AuditLogs` provides an independent, append-only trail of security-relevant events (login, suspension, ban) that is not writable by the modules whose actions it records being able to alter after the fact (Architecture Section 12).
- **Database backups:** Regular automated backups (via the managed MySQL hosting provider) are assumed as an operational baseline; backup retention should be no longer than necessary given the ephemeral-by-design nature of `Messages` and `Reports.message_excerpt` — i.e., backup retention policy should not become a backdoor around the application-level purge rules in Section 6.
- **Least privilege access:** The application's database credentials should be scoped to only the operations each module needs (e.g., a reporting/analytics credential could be read-only and excluded from `Messages` entirely); this is a deployment/configuration concern building on the schema design here, not a schema feature itself.

---

## 15. Performance Considerations

- **Search performance:** The Matching Module's core query (find compatible `Searching` users) is the system's hottest read path; the composite index in Section 13 on `UserStatus` is designed specifically to serve it directly, avoiding a full table scan even as user count grows.
- **Session lookup:** `Sessions.public_id` and `SessionParticipants` indexes ensure that "load this Session and its participants" — needed on nearly every Session-related screen — resolves via indexed lookups, not scans.
- **Nearby activity lookup:** For MVP scale (a single pilot city), a straightforward range filter on `approx_latitude`/`approx_longitude` is sufficient; a specialized spatial index or geohashing scheme is explicitly **not** introduced now (Section 16 discusses this as a future evolution once multiple cities are supported), consistent with "avoid premature optimization."
- **Pagination:** Any list views (e.g., a user's own report history, if ever surfaced) should paginate by `created_at` + `id` (a stable, indexed ordering) rather than offset-based pagination, to remain performant as tables grow.
- **Archiving:** `AuditLogs` and terminal `Sessions` rows are the tables most likely to grow unbounded over time; both are good candidates for periodic archival to cold storage once volume warrants it (an operational decision, not a schema constraint) — see Section 18, Risks.
- **Future caching:** Architecture.md notes Redis as a possible future addition for the hottest `Searching`-pool data; this document does not assume or require a cache for the MVP, since the underlying MySQL indexing strategy above is expected to be sufficient at pilot scale.

---

## 16. Scalability

- **Additional activities:** Purely additive rows in `Activities` (Section 10) — no schema change.
- **Group sessions:** `SessionParticipants` is already a many-to-many bridge (Section 4.6, Section 9); supporting more than two participants per Session is an application-layer change (relaxing the "exactly two rows" rule) plus, if needed later, a "needed participant count" column on `Sessions` or `Activities` for the Waiting Lobby concept (PRD Section 18) — additive, not a redesign.
- **Multiple cities:** `UserStatus.approx_latitude`/`approx_longitude` and `Sessions.meeting_point_*` are already generic coordinate fields with no city-specific assumptions baked in; operating in a new city requires no schema change, only sufficient user density (an operational, not architectural, concern, per Architecture Section 14).
- **International expansion:** `utf8mb4` encoding (already specified, Section 1) supports the full range of Unicode characters needed for names, messages, and sport names in any language; all timestamps are stored in UTC and are timezone-agnostic at the storage layer, with display-timezone conversion handled by the Frontend.
- **Mobile applications:** The schema has no web-specific assumptions (e.g., no session-cookie-style tables); a native mobile client consuming the same backend API requires zero database changes.
- **Future integrations (weather, push notifications, etc.):** Discussed in detail in Section 19; the general pattern is new, small, clearly-scoped tables referencing existing entities (`Users`, `Sessions`) via foreign key, never a modification of existing table structure.

---

## 17. Assumptions

*Architectural/database assumptions, clearly distinct from PRD requirements and Architecture Document decisions.*

- Account deletion and inactivity handling (Section 6) are **not** explicitly specified in PRD v1.2; the behavior described here is a conservative, privacy-consistent default and should be confirmed with product stakeholders before implementation, not treated as a frozen requirement.
- MySQL 8.x's native JSON-capable text column is sufficient for `AuditLogs.metadata` at MVP scale; no dedicated structured-logging store is assumed necessary yet.
- A single managed MySQL instance (per Architecture.md Section 2, Railway MySQL) is sufficient for the pilot's expected data volume and concurrency; no read-replica or sharding strategy is assumed necessary for the MVP.
- The exact retention window for `Messages` and `Reports.message_excerpt` purge jobs (referred to generally as "shortly after" and "a short grace period") is left as an operational/product decision to be finalized during implementation, not fixed numerically in this document, since the PRD does not specify an exact number of days.
- Background/scheduled jobs (Session expiration sweep, Messages purge, excerpt purge) are assumed to run on the FastAPI backend (Architecture.md Section 2), not as MySQL-native scheduled events, keeping scheduling logic in one place.

---

## 18. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| `Messages` table grows unbounded if the scheduled purge job fails silently | Privacy risk (retained chat content beyond intended lifetime) and performance risk (table bloat) | Monitor purge-job execution via `AuditLogs`/external monitoring; treat purge-job failure as a paging-level incident, not a background warning. |
| Location privacy misconfiguration (e.g., a future code change accidentally writes full-precision coordinates to `UserStatus`) | High — undermines the product's core privacy guarantee | Enforce precision reduction at a single, well-tested boundary in the Location Module (Architecture Section 8), and treat `UserStatus.approx_latitude`/`approx_longitude` precision as something covered by automated tests, not just code review. |
| Abandoned Sessions (e.g., a client disconnects and never triggers expiration client-side) | Medium — stale `Matched` status could block a user from re-entering `Searching` | The server-side `expires_at` timer (Section 4.5) and a scheduled sweep are authoritative regardless of client behavior (Architecture Section 13), so this is mitigated by design; the sweep job itself should be monitored. |
| Spam or bad-faith reports | Medium — could overwhelm the Moderation Module's review queue | `Reports.status` indexing (Section 13) supports queue triage; rate-limiting report submission is an application-layer control (Architecture Section 12), not a schema feature. |
| Moderation data growth (`Reports`, `ModerationActions`) over time | Low–Medium — long-term table size and query performance | Both tables are narrow and append-only; standard archival of old, resolved records (Section 15) is sufficient without a redesign. |
| Denormalized `Verification.verification_level` drifting out of sync with the underlying `*_verified_at` timestamps | Low — display inconsistency | Recompute and rewrite `verification_level` in the same transaction as any `*_verified_at` update, since it is the one intentionally denormalized field in this schema (Section 1, Section 4.2). |

---

## 19. Future Database Evolution

The schema is intentionally shaped so the following PRD Future Enhancements (PRD Section 18) can be added later **without a full redesign**:

- **Leaderboards:** Could be derived directly from existing `ReputationMetrics` counters (e.g., ranking by `matches_completed`) without any new table, or, if richer criteria are needed later, a small `LeaderboardSnapshots` table referencing `Users` — additive only.
- **Events (Community Events):** A new `Events` table (name, activity_id, location, time window) plus an `EventParticipants` bridge table, deliberately modeled the same way `Sessions`/`SessionParticipants` already are — reusing a proven pattern rather than inventing a new one.
- **Teams:** A new `Teams` table plus a `TeamMembers` bridge table, following the same many-to-many bridge pattern already established by `SessionParticipants` — no changes to existing tables required.
- **Weather:** A new nullable reference or a small `SessionWeatherSnapshot` table keyed by `session_id`, capturing conditions at the time of a Session's meeting point recommendation — purely additive to `Sessions`.
- **Wearables:** Would most likely contribute additional `UserStatus`-adjacent input (e.g., an activity-detection signal) via a new table referencing `Users`, without altering the core matching or Session tables.
- **Notifications:** A new `Notifications` table (user_id, event_type, delivered_at, read_at) that subscribes to the same event types already flowing into `AuditLogs` (Section 4.11) — the Logging Module's event stream is a natural foundation for a future notification feed.

In every case, the guiding principle already established throughout this document — narrow, single-responsibility tables connected by foreign keys, with `Activities` and `SessionParticipants` in particular designed as extensible from day one — is what allows these Future Enhancements to be layered on top rather than requiring existing tables to be restructured.

---

*End of Document*

