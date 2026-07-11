# Product Requirements Document (PRD)

**Project Name:** PlayNow
**Tagline:** Connecting people through sports, one match at a time.
**Document Type:** Product Requirements Document
**Status:** Draft v1.2 (Revised after Internal Architecture Review)
**Prepared For:** Engineering, UI/UX, QA, and Project Management Teams

---

## 1. Executive Summary

PlayNow is a location-aware web application that helps people find nearby, like-minded players for recreational sports in real time. Rather than messaging multiple group chats or friends hoping someone is free, a user simply selects a sport, sets their preferences, and goes "online." PlayNow searches for compatible nearby users, facilitates a match, and suggests a safe, public venue for both parties to meet.

Internally, each confirmed match is represented as a **Session** — a lightweight domain object bundling the activity, participants, anonymous chat, meeting point, timer, and status — while the user-facing interface continues to use familiar language like "Match Found." Version 1.2 also introduces optional identity verification levels, an informational match compatibility estimate, and a rule-based (non-AI) approach to recommending fair, suitable meeting points.

This document defines the full product scope for the Minimum Viable Product (MVP), including functional and non-functional requirements, user flows, use cases, acceptance criteria, risks, and future roadmap items. It is intended to serve as the single source of truth for all teams involved in designing, building, and testing PlayNow.

While this project originates as a college software engineering project, it is scoped and written using real-world startup product practices so that it can realistically evolve into a production-grade application.

---

## 2. Vision Statement

To make spontaneous, real-world sports participation as easy as sending a message — reducing screen time, strengthening local communities, and helping people build healthier, more active lifestyles by removing the friction of finding someone to play with, right now.

---

## 3. Problem Statement

Recreational sports participation is declining, not because people don't want to play, but because coordinating a game has become inconvenient. Existing solutions — WhatsApp groups, personal contacts, informal meetups — rely on pre-existing social networks and manual coordination, which frequently fail due to:

- Friends being unavailable at the same time
- No visibility into other nearby people who also want to play
- Time wasted messaging multiple people/groups with no response
- No simple way to discover strangers with the same sporting interest nearby

PlayNow addresses this gap by allowing any user, regardless of their existing social circle, to instantly discover other nearby users who want to play the same sport at the same time, and to arrange a safe, public meeting point to do so.

---

## 4. Objectives

| # | Objective | Description |
|---|-----------|-------------|
| O1 | Reduce coordination friction | Allow users to find a playing partner/group in minutes instead of hours |
| O2 | Encourage outdoor activity | Provide a simple digital nudge that leads to real-world physical activity |
| O3 | Ensure user safety and privacy | Never expose exact location; always recommend public meeting venues |
| O4 | Deliver a lightweight, focused MVP | Ship only the essential features required to validate the core matching concept |
| O5 | Build a scalable foundation | Architect the system so future features (see Section 18) can be added without a rewrite |
| O6 | Maximize pre-meetup trust and safety | Provide anonymous, temporary communication and clear safety guidance so users feel secure before ever meeting in person |
| O7 | Build trust with objective, non-invasive signals | Offer optional identity verification and system-tracked reputation metrics, without subjective peer ratings or exposing personal identity |
| O8 | Design for architectural longevity | Structure core domain concepts (e.g., the Session model) so that future needs — group activities, scheduling, additional cities — do not require rebuilding the MVP |

---

## 5. Scope

The MVP scope is strictly limited to the following features:

1. User Registration
2. User Login
3. Profile Creation
4. Gender Selection
5. Sport Selection (from an expandable Sports Catalog)
6. Location Permission
7. Search Radius Selection
8. Gender Preference Filter
9. User Status System (Offline / Searching / Matched / Busy) — *replaces the original binary Go Online / Offline toggle*
10. Real-time Nearby Matching
11. Match Confirmation
12. Suggested Public Meeting Location (via Google Maps)
13. Responsive Website (desktop + mobile browser)
14. Logout
15. Theme Preference (Dark / Light / System Default)
16. Anonymous Temporary Match Chat (with basic moderation, reporting, and blocking)
17. Match Expiration (automatic countdown after confirmation, with manual cancellation)
18. Reputation Metrics (Matches Completed, Cancelled Matches, No-Shows, Member Since, Verification Level) — *supersedes the original "Public Profile Trust Indicators"*
19. Safety Center (in-app safety guidance)
20. Verification Levels (Level 1: Verified Email — required; Level 2: Verified Mobile — optional; Level 3: Verified College/University Identity — optional)
21. Availability Status — "Available Now" only for MVP (no scheduling)
22. Match Compatibility Score (informational estimate shown alongside "Match Found")
23. Meeting Point Recommendation (rule-based ranking of nearby public venues; supersedes the simple "suggest one venue" behavior of v1.0/v1.1)

All functional requirements in Section 8 map directly to these 23 items. No additional feature should be implemented without a formal scope change request and update to this document.

> **Revision Note (v1.1):** Items 9, 15–19 were added or revised following stakeholder review. Item 9 (User Status System) supersedes the original binary "Go Online / Offline" toggle described in v1.0 but preserves the underlying offline/online distinction as a subset of the new state model.

> **Revision Note (v1.2):** Following internal architecture review, item 18 was renamed from "Public Profile Trust Indicators" to "Reputation Metrics" and expanded to objective, system-tracked statistics only (no peer-to-peer ratings). Items 20–23 were added: Verification Levels, Availability Status, Match Compatibility Score, and rule-based Meeting Point Recommendation. Internally, a confirmed match is now represented as a **Session** domain object (see Section 8 introduction); this is an internal/architectural concept and does not change any user-facing scope item above.

---

## 6. Out of Scope

The following are explicitly **excluded** from this version of the product. Engineering and design teams should not build supporting infrastructure for these items unless a future version of this PRD authorizes it.

- Voice Chat
- AI Chatbot
- Machine Learning-based matching or recommendations
- Skill Rating / Skill Level system
- Payments or in-app purchases
- Advertisements
- Tournaments or leagues
- Friends System / social graph
- Push or in-app Notifications
- Admin Dashboard
- Social Feed
- Native Mobile Application (iOS/Android)
- Gender-based UI theming (superseded by the user-controlled Theme Preference in Section 8.15 — theme has no relationship to gender)
- Persistent/permanent chat history or general messaging beyond the temporary anonymous match chat
- Advanced automated moderation (e.g., ML-based content classification) — v1.1 introduces only basic rule-based detection combined with user reporting and manual review
- Scheduled/future availability ("Available Later"); MVP supports only "Available Now" (see Section 8, Availability Status)
- Group Sessions / multi-participant matching (more than two users per session); MVP remains strictly one-to-one, though the architecture is designed to accommodate this later (see Section 21, Architecture & Scalability Considerations)
- Peer-to-peer user ratings or reviews of any kind — this is a deliberate product decision, not merely deferred; trust is built only through objective, system-tracked Reputation Metrics (see FR-19)
- Mandatory mobile or college/university verification — both remain strictly optional (Verified Email is the only required verification level)
- Artificial Intelligence, Machine Learning, or predictive modeling of any kind in the MVP — matching, venue ranking, and compatibility scoring are all deterministic and rule-based (see Sections 8.12 and 8.23)

---

## 7. User Personas

### Persona 1: Aditya — The College Student
- **Age:** 20
- **Occupation:** Undergraduate student
- **Goal:** Wants to play badminton in the evening but his usual friend group is busy.
- **Pain Point:** Doesn't know other students nearby who play badminton at his skill/interest level.
- **How PlayNow Helps:** Goes online, selects badminton and a 2 km radius, gets matched with another nearby student within minutes.

### Persona 2: Priya — The Working Professional
- **Age:** 27
- **Occupation:** Software engineer
- **Goal:** Wants a quick, low-commitment way to fit a run or cycling session into a busy schedule.
- **Pain Point:** Doesn't have a fixed group; work schedule changes daily.
- **How PlayNow Helps:** Uses the gender preference filter for comfort and safety, finds a compatible partner nearby, meets at a suggested public park.

### Persona 3: Rahul — New to the City
- **Age:** 24
- **Occupation:** Recently relocated for work
- **Goal:** Wants to make local connections through a shared interest — football.
- **Pain Point:** Has no local social network yet.
- **How PlayNow Helps:** Discovers nearby football enthusiasts without needing pre-existing contacts.

### Persona 4: Sana — The Safety-Conscious User
- **Age:** 23
- **Occupation:** Graduate student
- **Goal:** Wants to play sports with strangers but is cautious about safety.
- **Pain Point:** Concerned about sharing exact location or meeting in private/unknown places.
- **How PlayNow Helps:** Approximate distance only (no exact location shared), gender preference filter, mandatory public-venue meeting points, an anonymous temporary chat to coordinate without sharing personal contact details, and access to the in-app Safety Center before her first match.

---

## 8. Functional Requirements

Each requirement is labeled with a unique ID (FR-XX) for traceability into test cases and future sprints.

### 8.0 Core Domain Concept: Session — *New in v1.2*

Following internal architecture review, PlayNow introduces **Session** as the internal domain object created whenever two (and, in future versions, potentially more) users mutually confirm a match. A Session bundles together:

- **Activity** — the selected sport, from the Sports Catalog (FR-05).
- **Participants** — the user(s) who are part of the Session.
- **Anonymous Chat** — the temporary, identity-free chat channel (FR-16) scoped to that Session.
- **Meeting Point** — the recommended public venue (FR-12) associated with that Session.
- **Timer** — the expiration countdown (FR-18) governing how long the Session remains active before it must be confirmed as started or is automatically expired.
- **Session Status** — the lifecycle state of the Session (e.g., Pending Acceptance, Active, Expired, Cancelled, Completed).

This is an **internal/architectural concept only**. The user-facing interface continues to use familiar, user-friendly language such as "Match Found" and "Match Confirmed" — the term "Session" is not required to appear in the UI. Wherever this document refers to "the Session" (capitalized) going forward, it means the object created upon match confirmation as described above; "Match" continues to describe the user-facing event of two people being paired. Lowercase "session" elsewhere in this document (e.g., "login session," "chat session") retains its ordinary technical meaning and is unrelated to this domain object unless stated otherwise.

### 8.1 User Registration (FR-01)
- FR-01.1: The system shall allow a new user to register using name, email, password, and date of birth (for minimum age validation).
- FR-01.2: The system shall validate email format and password strength (minimum length, at least one number/special character).
- FR-01.3: The system shall prevent duplicate registrations using the same email address.
- FR-01.4: The system shall display clear validation error messages for invalid input.

### 8.2 User Login (FR-02)
- FR-02.1: The system shall allow registered users to log in using email and password.
- FR-02.2: The system shall display an appropriate error message for incorrect credentials without revealing whether the email or password was wrong (security best practice).
- FR-02.3: The system shall support a "Forgot Password" flow (basic email-based reset).
- FR-02.4: The system shall maintain a secure authenticated session until logout or expiry.

### 8.3 Profile Creation (FR-03)
- FR-03.1: The system shall require users to complete a basic profile (display name, age, gender, profile photo optional) before using matching features.
- FR-03.2: The system shall allow users to edit their profile information post-creation.
- FR-03.3: The system shall not display sensitive information (email, exact location, phone number) on any public-facing profile.
- FR-03.4: The system shall allow the user to set a Theme Preference (see FR-15) and view their Reputation Metrics and Verification Level (see FR-19, FR-21) from within their profile/settings.

### 8.4 Gender Selection (FR-04)
- FR-04.1: The system shall require users to select their gender from a defined set of options (e.g., Male, Female, Other/Prefer not to say) during profile setup.
- FR-04.2: Gender selection shall be used only for the Gender Preference Filter (FR-08) and shall not be publicly broadcast beyond what is needed for matching.
- FR-04.3: Gender selection shall have no effect on application appearance, theme, or styling of any kind. Visual theming is governed exclusively by the user-controlled Theme Preference (see FR-15).

### 8.5 Sport Selection (FR-05)
- FR-05.1: The system shall allow the user to select one sport at a time from a **Sports Catalog** — a database-backed, extensible list of sports, rather than a hardcoded set of values. This allows new sports to be added by an administrator/data update without requiring a code deployment.
- FR-05.2: The initial Sports Catalog shall include, at minimum: Football, Cricket, Badminton, Padel, Basketball, Volleyball, Table Tennis, Tennis, Chess, Running, Cycling, Swimming, Frisbee, Skating, Hiking, Yoga, Jogging, Boxing (training partner), and Martial Arts Practice.
- FR-05.3: The system shall allow the user to change their selected sport at any time while their status is Offline (see FR-09).
- FR-05.4: Matching (FR-10) shall only occur between users who have selected the same sport from the catalog.
- FR-05.5: Each sport entry in the catalog shall support a display name and icon/label, allowing future sports to be added without impacting existing functionality.

### 8.6 Location Permission (FR-06)
- FR-06.1: The system shall request browser-based geolocation permission before enabling the "Go Online" feature.
- FR-06.2: If location permission is denied, the system shall display a message explaining that location access is required to use matching features, and shall prevent the user from going online.
- FR-06.3: The system shall periodically refresh the user's location while online to maintain matching accuracy.

### 8.7 Search Radius Selection (FR-07)
- FR-07.1: The system shall allow users to select a search radius from a predefined set of options (e.g., 1 km, 2 km, 5 km, 10 km).
- FR-07.2: The selected radius shall be used to filter potential matches shown to the user.
- FR-07.3: The system shall allow the user to adjust the radius while online without requiring a full page reload.

### 8.8 Gender Preference Filter (FR-08)
- FR-08.1: The system shall allow users to filter potential matches by preferred gender (e.g., Male, Female, Any).
- FR-08.2: This filter shall be applied in combination with sport and radius filters during the matching process (FR-10).

### 8.9 User Status System (FR-09)

*Revised in v1.1: the original binary Go Online / Offline toggle is replaced by a four-state model. This provides more accurate discoverability and prevents users from being matched while already in a match or otherwise unavailable.*

- FR-09.1: The system shall maintain exactly one of the following statuses for each user at any time:
  - **Offline** — Invisible and not discoverable by any other user.
  - **Searching** — Actively discoverable and eligible to be matched with other Searching users.
  - **Matched** — Temporarily unavailable; paired with another user in an active, unexpired match. Not eligible for new matches.
  - **Busy** — Manually or automatically set as unavailable; visible as inactive but cannot receive new match invitations.
- FR-09.2: The system shall provide a clearly visible control allowing the user to switch between Offline, Searching, and Busy at will. Matched status is set automatically by the system upon mutual match confirmation (see FR-11) and cleared automatically on match expiration (FR-18) or manual cancellation.
- FR-09.3: When Offline or Busy, the user's profile and approximate location shall not be visible or discoverable to any other user, and the user shall not be eligible for new matches.
- FR-09.4: The system shall allow the user to set their status to Offline at any time, including mid-search, while Matched, or while Busy.
- FR-09.5: Setting status to Offline shall immediately cancel any pending, unconfirmed match search and, if in an active Matched state, shall end any associated anonymous chat (see FR-16).
- FR-09.6: For the MVP, the system shall support only one Availability Status value: **Available Now** (see FR-22). A user must be Searching (i.e., Available Now) to participate in matching; scheduled/future availability is explicitly out of scope for the MVP (see Section 6, Out of Scope).

### 8.10 Real-time Nearby Matching (FR-10)
- FR-10.1: While a user's status is **Searching**, the system shall continuously search for other **Searching** users who match on: same sport (from the Sports Catalog, FR-05), overlapping search radius, and compatible gender preference (bidirectional — both users' preferences must be satisfied).
- FR-10.2: Users with status **Matched**, **Busy**, or **Offline** shall be excluded from the matching pool and shall not receive new match invitations.
- FR-10.3: The system shall display only approximate distance (e.g., "~1.2 km away") to the user before a match is confirmed — never exact coordinates or an address.
- FR-10.4: If one or more compatible users are found, the system shall display a "Match Found" notification/screen to both parties, including the Match Compatibility Score (see FR-23).
- FR-10.5: If no compatible user is found, the system shall continue searching in the background and inform the user they remain in "Searching" state until they cancel or change status.
- FR-10.6: The system shall support matching being re-triggered automatically if a previously found match is not confirmed by one or both parties within a reasonable time window.

### 8.11 Match Confirmation (FR-11)
- FR-11.1: Upon a match being found, the system shall require both users to explicitly accept the match before any further information (e.g., suggested venue, anonymous chat) is revealed.
- FR-11.2: If either user declines or the match request times out, the system shall discard the match and return both users to **Searching** status (if they have not changed status manually).
- FR-11.3: Once both users accept, the system shall set both users' status to **Matched**, notify both parties that the match is confirmed, and create a **Session** (see Section 8.0) that starts the Match Expiration countdown (see FR-18) and enables the Anonymous Temporary Match Chat (see FR-16).
- FR-11.4: The Session created in FR-11.3 shall track its own lifecycle status (e.g., Active, Expired, Cancelled, Completed) independently of the two participating users' individual status values.

### 8.12 Meeting Point Recommendation (FR-12) — *Revised in v1.2*

*v1.2 replaces the simple "suggest one venue" behavior with a rule-based recommendation approach. This is a deterministic ranking system — it does not use Artificial Intelligence, Machine Learning, or predictive modeling of any kind.*

- FR-12.1: After match confirmation, the system shall obtain the approximate locations of the Session's participants (see FR-10.3) and calculate an approximate midpoint between them.
- FR-12.2: The system shall use the Google Maps API to search for nearby public activity venues suitable for the Session's selected sport around that approximate midpoint (e.g., a football ground for Football, an indoor court for Badminton, a public park or community space for Chess, a public cycling route for Cycling, a public track for Running, a public pool for Swimming).
- FR-12.3: The system shall rank candidate venues using the following rule-based factors, without claiming or requiring any AI/ML technique:
  - **Distance fairness** — how evenly the venue's distance is split between both participants.
  - **Public accessibility** — whether the location is genuinely open to the public.
  - **Open status** — whether the venue is currently open, based on available operating-hours data.
  - **Venue ratings** — publicly available ratings sourced from the mapping provider (this is a rating of the *place*, not of any user — see FR-19.5 for the prohibition on peer-to-peer user ratings).
  - **Activity compatibility** — whether the venue type matches the selected sport (e.g., a court for Badminton, a ground for Football).
  - **Safety** — preference for well-known, publicly monitored, or well-lit locations where such data is available.
- FR-12.4: The system shall only suggest publicly accessible locations — never private addresses or residential locations.
- FR-12.5: The system shall display the top-ranked suggested venue(s) on an embedded map view with name and approximate distance for both participants.
- FR-12.6: This section describes the recommendation behavior conceptually; specific ranking weights, scoring formulas, and implementation details are left to the engineering team's design and are outside the scope of this PRD.

### 8.13 Responsive Website (FR-13)
- FR-13.1: The system shall render correctly and remain fully functional across common desktop and mobile browser viewport sizes.
- FR-13.2: All core flows (registration, login, sport selection, matching, confirmation) shall be usable on a mobile browser without loss of functionality.

### 8.14 Logout (FR-14)
- FR-14.1: The system shall provide a clearly accessible logout option from any authenticated page.
- FR-14.2: Logging out shall automatically set the user's status to "Offline" and terminate the active session.
- FR-14.3: After logout, the system shall redirect the user to the login page.

### 8.15 Theme Preference (FR-15) — *New in v1.1*
- FR-15.1: The system shall allow users to select a Theme Preference: **Dark Theme**, **Light Theme**, or **System Default** (follows the device/browser's OS-level theme setting).
- FR-15.2: Theme Preference shall be stored as a user preference tied to the account, independent of and unrelated to gender or any other profile attribute.
- FR-15.3: The system shall apply the selected theme consistently across all pages of the application.
- FR-15.4: The Theme Preference control shall be accessible from both Profile and Settings.

### 8.16 Anonymous Temporary Match Chat (FR-16) — *New in v1.1*
- FR-16.1: The system shall enable a text chat channel between two users **only** after both users have mutually accepted a match (see FR-11.3).
- FR-16.2: The chat shall be fully anonymous: no usernames, real names, phone numbers, email addresses, profile links, social media handles, exact locations, or other personal identifiers shall be displayed within the chat interface.
- FR-16.3: The chat shall automatically close and its message history shall be discarded when any of the following occurs:
  - The match expires (see FR-18)
  - Either user manually leaves/ends the chat
  - 10 minutes have elapsed since the chat became active
  - The match is cancelled by either user
- FR-16.4: The system shall not persist chat message content beyond the active chat session.
- FR-16.5: The chat interface shall provide **Report User**, **Block User**, and **End Chat** controls at all times during an active chat.

### 8.17 Chat Moderation (FR-17) — *New in v1.1*
- FR-17.1: The system shall apply basic automated detection (e.g., keyword/pattern-based filtering) to flag potentially unacceptable behavior, including but not limited to: harassment, sexual harassment, grooming, pedophilic behavior, hate speech, threats, repeated abusive language, sharing of personal contact information, and attempts to move the conversation to external platforms.
- FR-17.2: Automated detection shall be combined with user-submitted reports and manual human review before any moderation action is finalized. The system shall not claim or rely upon perfect automated detection.
- FR-17.3: Upon a **first confirmed violation**, the system shall apply a temporary account suspension of 7 days.
- FR-17.4: Upon a **second confirmed violation**, the system shall apply a permanent account ban.
- FR-17.5: The system shall log reported incidents with sufficient context (e.g., anonymized chat excerpt, timestamp, reporting user) to support manual review, while continuing to respect the privacy principles in FR-16.2 for all non-implicated parties.
- FR-17.6: The system shall notify a user when a report they submitted has been reviewed, without revealing personally identifying information about the reported party beyond what that party already disclosed.

### 8.18 Match Expiration (FR-18) — *New in v1.1*
- FR-18.1: Upon mutual match confirmation (FR-11.3), the system shall start a countdown timer, defaulting to **10 minutes**.
- FR-18.2: If both users do not indicate that the meetup has begun (or an equivalent confirmation signal) before the countdown reaches zero, the system shall automatically expire the match.
- FR-18.3: Upon expiration, the system shall close the anonymous chat (FR-16.3), return both users to **Searching** status if they remain online, and notify both users that the match has expired.
- FR-18.4: The system shall allow either user to manually cancel a confirmed match at any time before the timer expires, which shall immediately trigger the same expiration behavior described in FR-18.3.

### 8.19 Reputation Metrics (FR-19) — *Renamed and expanded in v1.2 (previously "Public Profile Trust Indicators")*

*Following internal architecture review, this requirement was renamed from "Public Profile Trust Indicators" to "Reputation Metrics" to make explicit that all signals are objective, system-tracked statistics rather than subjective opinions.*

- FR-19.1: The system shall display simple, non-identifying reputation signals to a matched user, including: **Verification Level** (see FR-21), **Member Since** (month/year of account creation), **Matches Completed** (total count), **Cancelled Matches** (total count), and **No-Shows** (total count of Sessions that expired without the user indicating the meetup began).
- FR-19.2: Reputation Metrics shall never include or imply real name, exact location, contact information, or any other personally identifying detail.
- FR-19.3: Reputation Metrics shall be visible only after a match is confirmed, consistent with the anonymity principles in FR-16.2.
- FR-19.4: The system shall maintain these statistics automatically based on system-observed events (e.g., a Session reaching Completed vs. Expired vs. Cancelled status); no manual data entry by other users is involved in calculating them.
- FR-19.5: The system shall **not** implement any peer-to-peer user rating, review, or scoring system. Users may never rate, review, or score one another. This is a deliberate product decision to reduce popularity bias and encourage accountability through objective statistics only.

### 8.20 Safety Center (FR-20) — *New in v1.1*
- FR-20.1: The system shall provide a dedicated Safety Center accessible from Settings at any time.
- FR-20.2: The system shall present the Safety Center to a user automatically before their first confirmed match.
- FR-20.3: The Safety Center shall include, at minimum, guidance to: meet only in public places; inform a trusted friend or family member before meeting; never share passwords or financial information; never pressure others to share personal information; report suspicious behavior immediately; and respect all participants.

### 8.21 Verification Levels (FR-21) — *New in v1.2*

*This expands the single "Verified Email" signal from v1.1 into a tiered, optional verification system.*

- FR-21.1: The system shall support three verification levels for each user account:
  - **Level 1 — Verified Email:** Confirmed via the standard email verification flow during registration (FR-01/FR-02). This is the only **mandatory** verification level.
  - **Level 2 — Verified Mobile Number (Optional):** Users may optionally verify a mobile number (e.g., via a one-time code) to reach Level 2.
  - **Level 3 — Verified College/University Identity (Optional):** Users may optionally verify enrollment or affiliation with a college/university (e.g., via an institutional email domain or equivalent) to reach Level 3.
- FR-21.2: Verification beyond Level 1 (Email) shall always remain optional; the system shall not require Level 2 or Level 3 verification to use any core matching feature.
- FR-21.3: The user's current Verification Level shall be displayed as part of their Reputation Metrics (FR-19) as a simple badge (e.g., "Level 2 Verified"), without revealing the underlying phone number, institution name, or any other personal identifier.
- FR-21.4: The system shall record each user's Verification Level so that a future version may allow filtering or preferring matches by verification level (see Future Enhancements); this filtering capability itself is out of scope for the MVP.

### 8.22 Availability Status (FR-22) — *New in v1.2*
- FR-22.1: The system shall support exactly one Availability Status value in the MVP: **Available Now**.
- FR-22.2: Only users who are both Available Now and in **Searching** status (see FR-09) shall participate in matching.
- FR-22.3: The system shall not implement scheduling, future availability windows, or planned/advance sessions in the MVP; this is explicitly deferred (see Section 6, Out of Scope, and Section 18, Future Enhancements).

### 8.23 Match Compatibility Score (FR-23) — *New in v1.2*

*This is a rule-based, weighted estimate — not a prediction generated by Artificial Intelligence or Machine Learning.*

- FR-23.1: When a match is found, the system shall calculate and display a **Match Compatibility Score** alongside the "Match Found" screen (see FR-10.4).
- FR-23.2: The score shall be derived from simple, transparent, rule-based factors, including but not limited to: same activity/sport, distance between participants, availability alignment, and search radius compatibility.
- FR-23.3: The system shall present the score as an **informational estimate only** — the UI copy and any accompanying explanation shall make clear that it is not a guarantee of a successful or enjoyable match.
- FR-23.4: The specific weighting or formula used to combine these factors is an implementation detail left to the engineering team and is outside the scope of this PRD.

---

## 9. Non-Functional Requirements

| ID | Requirement | Description |
|----|-------------|--------------|
| NFR-01 | Responsive Design | UI must adapt cleanly across screen sizes (mobile, tablet, desktop) using a responsive layout framework. |
| NFR-02 | Fast Loading | Core pages should load within 2–3 seconds under normal network conditions. |
| NFR-03 | Simple UI | Interface should prioritize clarity and minimal steps to go from login to "online" status. |
| NFR-04 | Secure Authentication | Passwords must be hashed (never stored in plain text); sessions must use secure tokens (e.g., JWT) with appropriate expiry. |
| NFR-05 | Scalable Architecture | Backend should be designed so the matching service can handle a growing number of concurrent online users without major rearchitecture. |
| NFR-06 | Maintainable Codebase | Code should follow consistent style guidelines, modular structure, and be documented for handoff between team members. |
| NFR-07 | Cross-browser Compatibility | Application must function correctly on the latest versions of Chrome, Firefox, Safari, and Edge. |
| NFR-08 | Data Privacy | Location data must never be persisted beyond what is operationally necessary, and exact coordinates must never be exposed to other users. |
| NFR-09 | Availability | The core matching flow should degrade gracefully (e.g., clear error messaging) in case of geolocation or Maps API failures. |
| NFR-10 | Chat Ephemerality & Security | Anonymous match chat content must be transmitted securely (e.g., encrypted in transit) and must not be persisted beyond the active chat session; no personal identifiers may pass through the chat UI layer by design. |
| NFR-11 | Moderation Reliability | Automated moderation detection should minimize false negatives/positives where reasonably possible, but is not guaranteed to catch all violations; user reporting and manual review must remain available as a complementary safeguard at all times. |
| NFR-12 | Extensible Data Model | The Sports Catalog must be stored as data (e.g., in a database table) rather than hardcoded in application code, so new sports can be added without a code deployment. |
| NFR-13 | Future Scalability (Design Consideration) | The architecture should be designed with future expansion in mind — additional activities, a larger user base, native mobile applications, group sessions, multiple cities, and internationalization — without requiring an MVP rewrite. This is a design consideration for the MVP architecture, not an MVP feature (see Section 21, Architecture & Scalability Considerations). |

### 9.1 Security & Privacy Principles — *Expanded in v1.2*

The following principles consolidate and reinforce privacy and security commitments already established throughout this PRD (see FR-10.3, FR-12.4, FR-16, NFR-08, NFR-10):

- Only **approximate location** is used for matching and is shown to users; exact coordinates are never shared between users.
- Meeting points recommended by the system are always **public venues** (FR-12.4).
- The anonymous match chat **automatically expires** and is scoped to the lifetime of its associated Session (FR-16.3, FR-18).
- Chat history is **not permanently stored** after a Session ends, except where temporarily retained for moderation purposes strictly in accordance with the moderation policy (FR-17.5).
- **Personally identifiable information (PII)** — real name, phone number, email address, exact location, profile/social links — is never displayed to a matched user through the chat, Reputation Metrics, or Match Compatibility Score features.

---

## 10. User Flow

### 10.1 High-Level Flow (Text Description)

1. **Landing Page** → User selects "Sign Up" or "Log In."
2. **Registration** → User provides name, email, password, DOB → Email/password validated → Account created.
3. **Profile Setup** → User adds display name, gender, optional photo, and Theme Preference. Email verification (Level 1) is required; the user may optionally verify a mobile number (Level 2) or college/university identity (Level 3).
4. **Safety Center (First-Time Prompt)** → Shown automatically before the user's first match; can be revisited any time from Settings.
5. **Login** → Returning users authenticate with email/password.
6. **Home / Dashboard** → User selects:
   - Sport (from Sports Catalog)
   - Search Radius
   - Gender Preference
7. **Location Permission Prompt** → User grants browser location access.
8. **Set Status to Searching (Available Now)** → User status changes from Offline to "Searching…"; for the MVP, this always implies "Available Now" — there is no scheduling of future availability.
9. **Matching Engine** runs in the background:
   - **If match found:** Both users see "Match Found" screen, including a Match Compatibility Score → Each user accepts/declines.
   - **If no match found:** User remains in "Searching…" state until cancelled or matched.
10. **Match Confirmed** → Both users' status changes to "Matched" and a Session is created internally → A rule-based recommended public venue is displayed via Google Maps → Anonymous Temporary Chat becomes available → Match Expiration countdown (default 10 min) begins.
11. **During Chat** → Users may Report, Block, or End Chat at any time. Chat auto-closes on expiration, cancellation, either user leaving, or 10 minutes elapsing.
12. **Match Outcome:**
    - Meetup begins → Session completes; Matches Completed count increases for both users.
    - Timer expires without meetup, or either user manually cancels → Session expires → No-Show or Cancelled Matches count increases as applicable → both users return to "Searching" status (if still active).
13. **Post-Match** → User can set status to Offline/Busy, search again, or log out.
14. **Logout** → The user's authenticated login session ends (distinct from a matching Session, see Section 8.0), status set to Offline, redirected to login page.

### 10.2 Flow Diagram (Conceptual)

```
[Landing Page]
      |
      v
[Register] --------> [Login] <---------------------------------------------+
      |                  |                                                  |
      v                  v                                                  |
[Profile Setup] --> [Safety Center (first match only)] --> [Home / Dashboard]|
                                                                  |          |
                                                                  v          |
                                          [Select Sport, Radius, Gender Pref]|
                                                                  |          |
                                                                  v          |
                                                    [Location Permission]    |
                                                                  |          |
                                                                  v          |
                                            [Status: Offline -> Searching]   |
                                                                  |          |
                                                                  v          |
                                          [Searching for Match] <----+       |
                                                  |                 |       |
                                     +------------+------------+    |       |
                                     |                         |    |       |
                                     v                         v    |       |
                       [Match Found + Compatibility Score]  [No Match Yet]-+ |
                                     |                                      |
                                     v                                      |
                       [Both Users Accept?] --No--> [Discard Match, Resume Search]
                                     |                                      |
                                    Yes                                     |
                                     |                                      |
                                     v                                      |
   [Status: Matched -> Session Created: Venue Recommendation + Anon Chat]   |
                                     |                                      |
                                     v                                      |
                       [Session Timer (Match Expiration): 10 min, cancellable] |
                                     |                                      |
                         +-----------+-----------+                         |
                         |                       |                         |
                         v                       v                         |
      [Meetup Begins -> Session Completed]  [Timer Expires / Cancelled]    |
                         |                       |                         |
                         |                       v                         |
                         |   [Session Ends: Chat Closed, Status -> Searching,
                         |    Cancelled/No-Show count updated]--------------+
                         v
      [Reputation Metrics Updated: Matches Completed++]
                         |
                         v
              [Set Status: Offline / Busy / Logout] -------------------------->+
```

---

## 11. Use Cases

### UC-01: Register a New Account
- **Actor:** New user
- **Precondition:** User does not have an existing account.
- **Main Flow:** User provides registration details → System validates → Account created → User redirected to profile setup.
- **Alternate Flow:** Email already exists → System displays error and prompts login instead.

### UC-02: Log In to Existing Account
- **Actor:** Registered user
- **Precondition:** User has a verified account.
- **Main Flow:** User enters credentials → System authenticates → User redirected to dashboard.
- **Alternate Flow:** Invalid credentials → System displays generic error message.

### UC-03: Set Up Matching Preferences and Go Searching (Available Now)
- **Actor:** Logged-in user
- **Precondition:** Profile is complete; location permission available.
- **Main Flow:** User selects sport, radius, and gender preference → Grants location access → Sets status to "Searching" (Available Now) → System begins searching.

### UC-04: Get Matched with a Nearby Player
- **Actor:** Two Searching users with compatible preferences
- **Precondition:** Both users are Searching with overlapping sport, radius, and gender preferences.
- **Main Flow:** System detects compatibility → Displays "Match Found" with Match Compatibility Score to both → Both accept → Match confirmed → Session created → Recommended public venue suggested.
- **Alternate Flow:** One user declines or times out → Match discarded → Both return to Searching (if status unchanged).

### UC-05: View Recommended Meeting Venue
- **Actor:** Matched users
- **Precondition:** Match has been mutually confirmed; a Session exists.
- **Main Flow:** System calculates an approximate midpoint between both users, queries the Google Maps API for nearby public activity venues suited to the Session's sport, ranks them using rule-based factors (distance fairness, public accessibility, open status, venue rating, activity compatibility, safety) → Displays the top result(s) with map and distance to both users.

### UC-06: Change Status to Offline or Busy
- **Actor:** Any user with an active status (Searching, Matched, or Busy)
- **Precondition:** User is logged in.
- **Main Flow:** User selects "Offline" or "Busy" → System removes user from the discoverable/matching pool → Any pending unconfirmed match is cancelled → If previously Matched, the associated anonymous chat is closed.

### UC-07: Log Out
- **Actor:** Authenticated user
- **Precondition:** User is logged in.
- **Main Flow:** User selects "Logout" → System ends session, sets status to Offline → User redirected to login page.

### UC-08: Use Anonymous Temporary Match Chat
- **Actor:** Two mutually matched users
- **Precondition:** Match has been confirmed (status: Matched).
- **Main Flow:** Chat becomes available to both users → Users exchange anonymous messages to coordinate the meetup → Either user may Report, Block, or End Chat at any time.
- **Alternate Flow:** Match expires, is cancelled, either user leaves, or 10 minutes elapse → Chat automatically closes and message history is discarded.

### UC-09: Report or Block a User During Chat
- **Actor:** User in an active anonymous chat
- **Precondition:** Chat is active.
- **Main Flow:** User selects "Report User" → System logs the incident with relevant context for manual review → Automated detection and human review determine if a violation occurred → Moderation action applied per policy (7-day suspension on first confirmed violation, permanent ban on second).
- **Alternate Flow:** User selects "Block User" → Future matching between the two users is prevented; current chat ends immediately.

### UC-10: Match Expiration and Manual Cancellation
- **Actor:** Two mutually matched users
- **Precondition:** Match is confirmed and the expiration countdown is running.
- **Main Flow:** Countdown reaches zero without meetup confirmation → System expires the match, closes the chat, and returns both users to Searching status (if still active).
- **Alternate Flow:** Either user manually cancels the match before the timer expires → Same expiration behavior is triggered immediately.

### UC-11: Set Theme Preference
- **Actor:** Any logged-in user
- **Precondition:** User has access to Profile or Settings.
- **Main Flow:** User selects Dark Theme, Light Theme, or System Default → Preference is saved to the account → Theme is applied consistently across the application.

### UC-12: View Safety Center
- **Actor:** Any logged-in user
- **Precondition:** None (accessible at any time from Settings); shown automatically before first match.
- **Main Flow:** User views safety guidance (public meetups, informing a trusted contact, not sharing sensitive information, reporting suspicious behavior) → User proceeds to dashboard or matching flow.

### UC-13: Verify Mobile Number or College/University Identity — *New in v1.2*
- **Actor:** Any logged-in user (Level 1 Verified Email already completed)
- **Precondition:** User wishes to increase their Verification Level.
- **Main Flow:** User opts to verify a mobile number (e.g., via one-time code) or a college/university identity (e.g., via institutional email) → Verification succeeds → System updates the user's Verification Level and reflects it in their Reputation Metrics badge.
- **Alternate Flow:** User declines to verify beyond Email → No impact on ability to use core matching features (verification remains optional).

### UC-14: View Match Compatibility Score — *New in v1.2*
- **Actor:** Two users shown a potential match
- **Precondition:** A compatible match has been found.
- **Main Flow:** System calculates a rule-based Match Compatibility Score from same-activity, distance, availability, and radius-compatibility factors → Displays the score alongside "Match Found," with copy clarifying it is an informational estimate, not a guarantee.

---

## 12. Acceptance Criteria

| Feature | Acceptance Criteria |
|---------|---------------------|
| User Registration | Given valid details, a new account is created and the user is redirected to profile setup. Duplicate emails are rejected with a clear error. |
| User Login | Given correct credentials, the user reaches the dashboard. Given incorrect credentials, a generic error is shown without revealing which field was wrong. |
| Profile Creation | User cannot access matching features until required profile fields (name, gender, age) are completed. |
| Gender Selection | Gender must be selected before profile is considered complete; value is used only for filtering, not shown publicly beyond what's needed. |
| Sport Selection | User can select exactly one sport at a time from the Sports Catalog before setting status to Searching; new sports can be added to the catalog without a code change. |
| Location Permission | "Searching" status cannot be enabled until location permission is granted; a clear message explains why if denied. |
| Search Radius Selection | User can choose from predefined radius options; matches shown must fall within the selected radius. |
| Gender Preference Filter | Matches shown must satisfy both users' gender preferences simultaneously. |
| User Status System | User can switch between Offline, Searching, and Busy; Matched is set automatically on match confirmation and cleared automatically on expiration/cancellation; Offline/Busy users are excluded from the discoverable pool. |
| Real-time Nearby Matching | Only approximate distance is displayed pre-match; exact location is never exposed; only Searching users are eligible to match. |
| Match Confirmation | A match only becomes final after both users explicitly accept; declining by either party discards the match; acceptance triggers Matched status, expiration timer, and chat availability. |
| Suggested Public Meeting Location | Only publicly accessible venues are suggested; the recommended venue reflects a rule-based ranking (distance fairness, accessibility, open status, venue rating, activity compatibility, safety) around the approximate midpoint of both users; each suggestion displays on a map with name and distance. |
| Theme Preference | User can select Dark, Light, or System Default theme; selection is unrelated to gender and applies consistently across the app. |
| Anonymous Temporary Match Chat | Chat is available only after match confirmation (i.e., after a Session is created); displays no personal identifiers; closes automatically on expiration, cancellation, 10 minutes elapsed, or either user leaving. |
| Chat Moderation | Report User, Block User, and End Chat controls are always available during an active chat; first confirmed violation results in a 7-day suspension, second results in a permanent ban; automated detection is combined with user reports and manual review. |
| Match Expiration | A 10-minute countdown starts upon match confirmation (Session creation); expiration (or manual cancellation by either user) closes the chat and returns both users to Searching status if still active; a Cancelled Match or No-Show is recorded as applicable. |
| Reputation Metrics | Verification Level, Member Since, Matches Completed, Cancelled Matches, and No-Shows are shown post-match without revealing personally identifying information; no peer-to-peer rating of any kind exists anywhere in the product. |
| Verification Levels | Level 1 (Email) is mandatory and gates no additional features beyond registration; Levels 2 (Mobile) and 3 (College/University) are always optional and never block core matching functionality. |
| Availability Status | Only "Available Now" exists as a value in the MVP; only users who are Available Now and Searching participate in matching; no scheduling UI or logic exists in the MVP. |
| Match Compatibility Score | A score is shown alongside every "Match Found" screen with copy clarifying it is an informational estimate; the score is computed from disclosed rule-based factors only, with no AI/ML claimed anywhere in the product. |
| Safety Center | Accessible from Settings at any time and shown automatically before a user's first confirmed match. |
| Responsive Website | All core flows are fully usable on both a standard desktop viewport and a common mobile viewport (e.g., 375px width). |
| Logout | Logging out ends the session, sets status to Offline, and redirects to the login screen. |

---

## 13. Safety Center — *New in v1.1*

The Safety Center is an in-app section dedicated to helping users make safe decisions before and during a meetup arranged through PlayNow. It is accessible from **Settings** at any time, and is presented automatically to every user before their **first confirmed match**.

### 13.1 Purpose
To reduce real-world risk when two previously unacquainted users meet in person, without requiring PlayNow to verify identity or supervise meetups directly.

### 13.2 Core Guidance Displayed

- Meet only in public places.
- Inform a trusted friend or family member before meeting.
- Never share passwords or financial information.
- Do not pressure others to share personal information.
- Report suspicious behavior immediately.
- Respect all participants.

### 13.3 Placement and Access

| Trigger | Behavior |
|---------|----------|
| First-ever match confirmation for a user | Safety Center is shown automatically and must be acknowledged before proceeding to the suggested venue/chat. |
| Settings menu | Safety Center is available on demand at any time, for any user. |
| Pre-chat reminder (optional lightweight banner) | A short reminder of core safety points may be shown at the top of the anonymous chat screen. |

### 13.4 Relationship to Other Features
The Safety Center complements, but does not replace, the structural safety measures already built into the product: approximate-location-only display (FR-10.3), public-venue-only suggestions (FR-12.4), anonymous chat (FR-16), Report/Block controls (FR-16.5), and chat moderation (FR-17).

---

## 14. Failure Scenarios — *New in v1.1*

This section documents expected system behavior and user experience when common failure conditions occur, so that engineering and QA can design consistent, graceful handling rather than undefined behavior.

| Scenario | Expected User Experience | Recovery Behavior |
|----------|---------------------------|--------------------|
| Location permission denied | User sees a clear message explaining that location access is required to use matching features; "Searching" status remains unavailable. | User can grant permission later via browser/device settings and retry; no crash or silent failure. |
| Internet disconnected | User sees a persistent, non-intrusive "connection lost" indicator; active searches pause. | System attempts automatic reconnection; on reconnect, status and any active match/chat state is restored where possible. |
| Google Maps unavailable | Suggested venue section shows a friendly error message instead of a blank/broken map. | Match confirmation and chat remain functional independent of the Maps feature; venue suggestion retries or can be manually retried by the user. |
| No nearby players found | User remains in "Searching" state with a clear "still looking…" indicator rather than an error. | Search continues in the background; user can adjust radius, sport, or gender preference at any time while Offline. |
| User logs out during an active match | Session ends immediately. | The other matched user is notified the match ended; their status returns to Searching if still active; chat is closed per FR-16.3. |
| Browser closed unexpectedly | Session eventually times out server-side. | Status is not left in a "phantom Matched/Searching" state indefinitely — an inactivity timeout reverts status to Offline after a defined period. |
| GPS/location unavailable (e.g., indoors, hardware issue) | User sees a message indicating location could not be determined and matching cannot proceed. | User is prompted to retry once location becomes available; no false approximate distance is shown. |
| Match timeout / expiration | Both users are notified the match has expired. | Chat closes, both users return to Searching status if still active, per FR-18.3. |

---

## 15. UI Screen Reference — *New in v1.1*

No visual design is prescribed here; this section defines the **purpose** of each core screen so that UI/UX design can proceed with a shared understanding of intent. Visual design, layout, and styling are left to the design team, subject to the Theme Preference requirement (FR-15).

| Screen | Purpose |
|--------|---------|
| Landing Page | First entry point for unauthenticated visitors; routes to Sign Up or Log In. |
| Login | Authenticates a returning user via email and password. |
| Register | Collects registration details for a new account (name, email, password, DOB). |
| Dashboard | Central hub after login; entry point to sport/radius/gender preference selection and status control. |
| Search Screen | Where the user configures sport, search radius, and gender preference before searching. |
| Searching State | Shown while status is "Searching"; communicates that the system is actively looking for a match. |
| Match Found | Displays a potential match's approximate distance and Match Compatibility Score, with copy clarifying the score is an informational estimate; prompts accept/decline from both users. |
| Anonymous Chat | Temporary, anonymous messaging interface available only after match confirmation (Session creation); includes Report, Block, End Chat controls and expiration countdown. |
| Map | Displays the recommended public venue(s) for a confirmed match, ranked using rule-based factors (distance fairness, accessibility, open status, venue rating, activity compatibility, safety), with name and approximate distance. |
| Profile | Displays and allows editing of the user's own profile details, including Theme Preference, Verification Level, and Reputation Metrics. |
| Settings | Central location for account-level preferences: Theme Preference, verification options (mobile/college ID), notifications (future), and access to the Safety Center. |
| Safety Center | Presents core safety guidance; shown automatically before first match and accessible anytime from Settings. |
| Error Pages | Generic and scenario-specific error messaging (see Section 14 Failure Scenarios) for conditions like lost connectivity or unavailable services. |
| Loading Screens | Lightweight indicators shown during data fetches (e.g., searching for matches, loading map data) to avoid a perceived "frozen" UI. |

---

## 16. Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Low user density in early launch (not enough nearby users online at the same time) | High — core value proposition fails without critical mass | High (early stage) | Start with a limited campus/city launch to concentrate user density; consider seeding initial test users during pilot. |
| Browser location permission denial | Medium — blocks core feature for affected users | Medium | Provide clear explanation of why location is required; graceful fallback messaging. |
| Google Maps API rate limits or downtime | Medium — suggested venue feature degraded | Low–Medium | Implement error handling/fallback messaging if the Maps API is temporarily unavailable. |
| User safety concerns meeting strangers | High — reputational and user trust risk | Medium | Enforce public-venue-only meeting suggestions, approximate-location-only display, anonymous chat, and the Safety Center. |
| Fake or inactive profiles | Medium — reduces trust and match quality | Medium | Basic email verification during registration (MVP-level control); optional Level 2/3 verification and Reputation Metrics (FR-19, FR-21) provide additional, still-optional signal. |
| Scope creep (features from "Do Not Include" list) | Medium — delays MVP delivery | Medium | Enforce strict adherence to Section 5/6 scope; changes require formal PRD revision. |
| Chat misuse (harassment, grooming, attempts to move off-platform) | High — user safety and trust risk | Medium | Basic rule-based moderation combined with mandatory Report/Block controls and manual review (FR-17); note detection is not guaranteed to be perfect. |
| Sports Catalog data quality/scalability as list grows | Low–Medium — inconsistent sport naming or duplicate entries | Low | Store catalog in a structured database table with validation on entry (FR-05.1, NFR-12). |
| Match Compatibility Score misinterpreted as a guarantee | Low–Medium — could set unrealistic user expectations | Medium | Clear UI copy stating the score is an informational estimate only (FR-23.3); no claim of AI/ML accuracy. |
| Low adoption of optional verification levels | Low — limits usefulness of Verification Level as a trust signal | Medium | Keep verification genuinely optional per product decision; consider light incentives in a future version without making it mandatory. |
| Meeting Point Recommendation quality depends on third-party venue/ratings data completeness | Low–Medium — may occasionally surface a suboptimal venue | Low–Medium | Fall back gracefully to a simpler nearest-public-venue suggestion when ranking data is incomplete (see Section 14, Failure Scenarios). |

---

## 17. Assumptions

- Users have devices with browser-based geolocation capability and grant permission when prompted.
- The MVP will be piloted within a limited geographic area (e.g., a college campus or single city) to ensure sufficient user density for matching to function meaningfully.
- Users have basic internet connectivity sufficient for real-time location updates.
- Google Maps API access (or an equivalent mapping service) will be available and within free/low-tier usage limits for the MVP's expected user volume.
- Only one active sport selection per user session is required for MVP; multi-sport simultaneous matching is not needed.
- Basic rule-based chat moderation combined with user reporting and manual review is sufficient for the pilot phase; advanced ML-based moderation is deferred to a future version.
- A default 10-minute match expiration window is a reasonable starting point for the pilot and may be tuned based on real usage data.
- Users will find it acceptable that only "Available Now" is supported in the MVP (no advance scheduling); this is assumed reasonable given the product's spontaneous, real-time positioning.
- A meaningful proportion of users will complete at least Level 1 (Email) verification without friction; uptake of optional Level 2/3 verification is expected to be lower and is not required for core functionality to work.
- Google Maps venue/place data (ratings, open-status, category) will be sufficiently complete and available for the rule-based Meeting Point Recommendation to function in the pilot area.

---

## 18. Future Enhancements (Version 2+)

The following features are explicitly deferred beyond the MVP and should be considered for future roadmap planning. *Updated in v1.2 following internal architecture review; note that peer-to-peer ratings/reviews are intentionally excluded from this list — see Section 6, Out of Scope, and FR-19.5.*

- **Group Sessions** — multi-participant activities (e.g., Football, Cricket, Volleyball, Basketball) requiring more than two participants, including a **Waiting Lobby** concept (e.g., "Football — Needed: 10 Players — Currently: 6/10 — Waiting…"). See Section 21 for architecture/data considerations that keep the MVP ready for this.
- Weather Integration (e.g., avoid suggesting outdoor venues during poor weather)
- Smart Venue Recommendation using richer historical/aggregate data (an evolution of the rule-based system in FR-12; would need explicit AI/ML scoping and review if pursued)
- Skill Level Matching (beginner/intermediate/advanced)
- Voice Chat
- Friend Requests / Social Connections
- Push Notifications (match alerts, reminders)
- Event Scheduling (planning matches in advance rather than only real-time; extension of the "Available Now"-only Availability Status in FR-22)
- Community Events (organized, larger-scale meetups beyond one-to-one or small-group matching)
- Leaderboards (would need careful design to avoid reintroducing subjective, popularity-driven dynamics)
- Tournament Mode
- Native Mobile Applications (iOS and Android)
- Apple Health / Google Fit Integration
- Wearable Device Support
- Admin Dashboard for platform monitoring and analytics
- Advanced ML-based chat moderation and content classification
- Configurable/admin-adjustable match expiration duration (beyond the fixed 10-minute default)
- Expanded, still-objective Reputation Metrics (e.g., average response time, on-time arrival rate) — no peer-to-peer rating
- Filtering matches by Verification Level (the data is recorded in the MVP per FR-21.4; the filtering UI itself is deferred)
- Group anonymous chat (for future Group Sessions)

---

## 19. Success Metrics

*Revised in v1.1: fixed numerical targets have been replaced with measurable pilot metrics. Actual targets should be established once baseline data is collected during pilot testing, rather than assumed in advance.*

| Metric | Description |
|--------|--------------|
| Average Match Time | Average time elapsed from a user entering "Searching" status to a confirmed match. |
| Average App Session Duration | Average length of time a user remains active (Searching, Matched, or Busy) per app usage session (distinct from the matching Session domain object, see Section 8.0). |
| Number of Successful Matches | Total count of matches that reach mutual confirmation during the pilot period. |
| Match Completion Rate | % of confirmed matches (Sessions) that reach Completed status versus those that Expire or are Cancelled. |
| No-Show / Cancellation Rate | % of confirmed matches ending in a No-Show or Cancelled Match, used to sanity-check the 10-minute expiration default and overall trust in the matching flow. |
| User Feedback Score | Aggregated rating or qualitative feedback collected from users after a completed match (e.g., a simple post-match survey). |
| Repeat Usage | Number/percentage of users who return to use PlayNow again after their first completed or attempted match. |
| Chat Moderation Activity | Number of reports submitted, violations confirmed, and moderation actions taken during the pilot, used to calibrate detection thresholds. |
| Verification Adoption Rate | % of users who complete optional Level 2 (Mobile) or Level 3 (College/University) verification, used to gauge interest without making it mandatory. |

These metrics should be collected throughout pilot testing and used to inform realistic, data-backed targets for a subsequent release, rather than assuming fixed percentages up front.

---

## 20. Development Roadmap — *New in v1.1*

*This section outlines the anticipated phases of work for building PlayNow. It is a planning aid, not a fixed schedule — actual durations and sequencing should be refined by the project/engineering lead.*

| Phase | Description |
|-------|-------------|
| 1. Planning | Define project goals, confirm MVP scope, align stakeholders on this PRD. |
| 2. Requirements Approval | Formal sign-off on this PRD (v1.2) by all stakeholders before design/development begins. |
| 3. Architecture Design | Define overall system architecture, including the Session domain model (Section 8.0), real-time matching approach, chat infrastructure, and data flow — designed with the future scalability considerations in Section 21 in mind. |
| 4. Database Design | Design schema for users, profiles, sports catalog, Sessions (matches), chat sessions, reports, verification levels, and reputation metrics. |
| 5. API Design | Define API contracts for authentication, profile management, matching, compatibility scoring, meeting point recommendation, chat, moderation, and Maps integration. |
| 6. UI Design | Translate the UI Screen Reference (Section 15) into wireframes and visual designs, incorporating Theme Preference. |
| 7. Frontend Development | Build the responsive web client for all core screens and flows. |
| 8. Backend Development | Implement authentication, matching engine, status/availability system, meeting point ranking, compatibility scoring, chat service, moderation logic, verification, and match expiration logic. |
| 9. Maps Integration | Integrate the Google Maps API for rule-based venue recommendation and embedded map views. |
| 10. Testing | Functional, non-functional, and safety-focused QA testing against the Acceptance Criteria (Section 12) and Failure Scenarios (Section 14). |
| 11. Deployment | Release the application to a controlled pilot environment. |
| 12. Pilot Testing | Run the pilot within a limited geographic area; collect Success Metrics (Section 19) and user feedback. |
| 13. Future Versions | Plan and prioritize Version 2+ features (Section 18) based on pilot learnings. |

---

## 21. Architecture & Scalability Considerations — *New in v1.2*

*This section is a design consideration for the engineering/architecture team. It intentionally avoids database schemas, API contracts, or other implementation detail — consistent with this PRD's role as a requirements document rather than a technical design document.*

### 21.1 Architecture Considerations — Group Session Readiness

The MVP implements strictly one-to-one matching (Section 5, Scope). However, the **Session** domain object introduced in Section 8.0 already models **Participants** as a concept distinct from a hardcoded pair of two users. Architecturally, this means:

- The matching engine, chat, meeting point recommendation, and expiration timer should all be designed to operate against "the Session's participants" rather than assuming exactly two individuals, even though the MVP will only ever populate two.
- Activities that naturally involve more people (e.g., Football, Cricket, Volleyball, Basketball) are the most likely candidates for a future **Group Sessions** feature, which would introduce a **Waiting Lobby** concept — for example, "Football — Needed: 10 Players — Currently: 6/10 — Waiting…" (see Section 18, Future Enhancements).
- None of this is to be built in the MVP; it is a forward-looking constraint on how the one-to-one MVP architecture is structured, so that group support can be added later without a full rewrite.

### 21.2 Database Considerations — Group Session Readiness

At a conceptual level only (no schema is specified here):

- Data structures representing a Session's participants should be designed as a **collection/list** rather than two fixed, named fields (e.g., "user_a" / "user_b"), so that a future version can support more than two participants without restructuring existing data.
- Similarly, the Sports Catalog (FR-05) and future Group Session "needed participant count" per activity should be modeled as data, not hardcoded logic, consistent with the extensibility principle already established for the Sports Catalog (NFR-12).

### 21.3 General Scalability Considerations

Per NFR-13, the architecture should be designed — as a forward-looking consideration, not an MVP deliverable — to accommodate:

- **Additional activities** beyond the initial Sports Catalog list (FR-05.2).
- **A larger user base** than the initial pilot, without requiring a rearchitecture of the matching engine (see NFR-05).
- **Native mobile applications** (iOS/Android) consuming the same backend/APIs as the responsive web client.
- **Group Sessions**, as discussed in Sections 21.1–21.2.
- **Multiple cities**, meaning location-based matching logic should not assume a single fixed geographic area.
- **Internationalization**, meaning user-facing text should not be hardcoded in a way that prevents future localization.

---

## 22. Glossary

| Term | Definition |
|------|------------|
| MVP | Minimum Viable Product — the smallest functional version of the product needed to validate the core concept. |
| Match | A mutually accepted pairing between two users who share sport, compatible radius, and compatible gender preference (user-facing term). |
| Session | The internal domain object created upon match confirmation, bundling Activity, Participants, Anonymous Chat, Meeting Point, Timer, and Session Status (see Section 8.0). Distinct from an ordinary login/app "session." |
| Offline (Status) | A user state in which they are not discoverable and not participating in matching. |
| Searching (Status) | A user state in which they are actively discoverable and eligible to be matched; implies Availability Status "Available Now" in the MVP. |
| Matched (Status) | A user state indicating an active, unexpired confirmed match (i.e., an active Session); user is not eligible for new matches. |
| Busy (Status) | A user state indicating manual unavailability; user is visible as inactive and cannot receive new match invitations. |
| Availability Status | A user-set indication of when they wish to play; the MVP supports only one value, "Available Now" — scheduling future availability is deferred. |
| Search Radius | The maximum distance within which a user is willing to be matched with another player. |
| Gender Preference Filter | A user-set filter determining which gender(s) of other users they are willing to be matched with. |
| Approximate Distance | A rounded, non-precise distance value (e.g., "~1.2 km") shown instead of exact coordinates. |
| Public Venue | A publicly accessible location (park, sports complex, community court) suitable and safe for two strangers to meet. |
| Sports Catalog | A database-backed, extensible list of sports available for selection, as opposed to a hardcoded list. |
| Theme Preference | A user-controlled setting (Dark, Light, or System Default) governing the app's visual appearance; unrelated to gender. |
| Anonymous Temporary Match Chat | A time-limited, identity-free text chat scoped to a Session, available only between mutually confirmed matched users. |
| Match Expiration | The automatic (or manually triggered) end of a confirmed match/Session after a countdown period (default 10 minutes) if no meetup begins. |
| Verification Level | A tiered, optional identity-confidence signal: Level 1 (Verified Email, mandatory), Level 2 (Verified Mobile, optional), Level 3 (Verified College/University Identity, optional). |
| Reputation Metrics | Objective, system-tracked statistics (Verification Level, Member Since, Matches Completed, Cancelled Matches, No-Shows) shown to a matched user to build confidence without revealing identity or relying on peer-to-peer ratings. |
| Match Compatibility Score | A rule-based, informational estimate of match fit, based on factors such as same activity, distance, availability, and radius compatibility; not a guarantee and not AI/ML-derived. |
| Meeting Point Recommendation | The rule-based process of calculating an approximate midpoint between matched users and ranking nearby public venues by distance fairness, accessibility, open status, venue rating, activity compatibility, and safety. |
| Safety Center | An in-app section providing safety guidance for meeting other users in person. |
| PRD | Product Requirements Document — this document. |

---

## 23. Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-07-11 | Product Management | Initial draft of PlayNow PRD covering MVP scope, functional/non-functional requirements, user flows, use cases, acceptance criteria, risks, and future roadmap. |
| 1.1 | 2026-07-11 | Product Management | Revised following stakeholder review: replaced gender-based theming with a user-controlled Theme Preference; added Anonymous Temporary Match Chat with moderation, reporting, and blocking; expanded Sport Selection into a scalable Sports Catalog; introduced automatic Match Expiration with manual cancellation; replaced the binary Online/Offline model with a four-state User Status System (Offline/Searching/Matched/Busy); added Public Profile Trust Indicators; added a new Safety Center section; added Failure Scenarios documentation; added a UI Screen Reference section; replaced fixed-percentage Success Metrics with measurable pilot metrics; added a Development Roadmap section. Updated Objectives, Scope, Out of Scope, Functional/Non-Functional Requirements, User Flow, Use Cases, Acceptance Criteria, Risks, Assumptions, Future Enhancements, and Glossary accordingly. |
| 1.2 | 2026-07-11 | Product Management / Chief Software Architect | Revised following internal architecture review: introduced **Session** as the internal domain object underlying a confirmed match (UI continues to say "Match Found"); renamed and expanded "Public Profile Trust Indicators" to **Reputation Metrics** (Matches Completed, Cancelled Matches, No-Shows, Member Since, Verification Level) and explicitly prohibited peer-to-peer ratings; introduced tiered **Verification Levels** (Level 1 Email — mandatory; Level 2 Mobile and Level 3 College/University — optional); replaced the simple suggested-venue behavior with a rule-based **Meeting Point Recommendation** (approximate midpoint + ranking by distance fairness, accessibility, open status, venue rating, activity compatibility, safety); added an informational, rule-based **Match Compatibility Score**; clarified **Availability Status** as "Available Now"-only for the MVP, with scheduling deferred; added Architecture and Database Considerations for future Group Sessions (with Waiting Lobby concept) without adding Group Sessions to MVP scope; added a new NFR and section on **Scalability Considerations**; expanded the Security & Privacy Principles; removed or properly caveated all references suggesting AI/ML is used in the MVP, retaining AI-related ideas only under Future Enhancements. Updated Objectives, Scope, Out of Scope, Functional Requirements (FR-03, FR-04 unchanged; FR-09, FR-10, FR-11, FR-12, FR-19 revised; FR-21–FR-23 added), Non-Functional Requirements (NFR-13 added), User Flow, Use Cases, Acceptance Criteria, Risks, Assumptions, Future Enhancements, Success Metrics, Development Roadmap, Glossary, and added Section 21 (Architecture & Scalability Considerations) accordingly. |

---

*End of Document*

