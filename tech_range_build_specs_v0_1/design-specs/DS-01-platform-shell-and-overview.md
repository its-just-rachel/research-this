# DS-01 — Platform Shell, Navigation, and Overview Tab

**Spec version:** 0.1  
**Status:** Draft  
**Last updated:** 2026-07-07  
**Related specs:** DS-08 (Governance Queues), DS-11 (User Management), DS-12 (Profile & Preferences)  
**Related decisions:** ADR-009 (AI Action Tiers), ADR-010 (Neighborhood Quadrant Layout), TS-07 (Alerts & Watchlist), TS-08 (Roles & Permissions), TS-10 (KPIs)

---

## 1. Purpose

This spec covers the platform shell, primary and utility navigation, the conditional display framework, and the Overview tab. It also establishes design principles inherited by all other DS specs.

**In scope:**
- Application shell (authenticated and unauthenticated states)
- Primary nav (six-tab model)
- Utility nav (top-right controls)
- Conditional display framework: role-driven visibility and preference-driven highlights
- Overview tab: snapshot modal, topics of interest strip, animated temporal radar, and KPI dashboard
- Empty and loading states for the above

**Out of scope:**
- Individual tab content beyond Overview (covered in their own DS specs)
- Authentication flow implementation (covered separately)
- Data pipeline and ingestion logic (covered in TS specs)

---

## 2. Navigation Model

### 2.1 Primary Nav

Six tabs displayed in fixed order across the top of the application:

| Position | Tab | Purpose |
|---|---|---|
| 1 | Overview | Program-level ambient view, KPIs, snapshot on login |
| 2 | Surface | Signal ingestion, triage, and early classification |
| 3 | Research | Interactive radar, Technology Profiles, cluster detail |
| 4 | Solution | Analysis artifacts, briefs, frameworks |
| 5 | Prototype | Active prototype tracking and evaluation |
| 6 | Advise | Advisory authoring, review, and publication |

The primary nav is always visible to all authenticated users. Tab labels and order are constant. Content within each tab is role-conditional (see Section 3).

### 2.2 Utility Nav

Displayed in the top-right corner of the application shell. Visible whenever a user is authenticated.

| Control | Visibility | Destination |
|---|---|---|
| Profile | All authenticated users | DS-12 (Profile & Preferences) |
| User Management | `admin` role only | DS-11 (User Management) |
| Notification bell | All authenticated users | TS-07 alerts and watchlist view |
| Queues | `analyst+` | Notification bell "View all" link, or `/queues` direct route for `analyst+` |
| Sign out | All authenticated users | Sign-out action → unauthenticated state |

> `/archaeology` is a direct route accessible to `senior_analyst+`. It may be surfaced as a utility nav item for those roles. See DS-09.

The User Management link is hidden entirely for non-`admin` roles. It does not render as disabled — it is not present in the DOM for those users.

### 2.3 Unauthenticated State

The platform requires authentication. Unauthenticated users see only the sign-in page. There is no public-facing content, no guest mode, and no partial view of platform data. All routes redirect to sign-in if the user is not authenticated.

---

## 3. Conditional Display Framework

### 3.1 Principle

The information architecture is fixed. Every user navigates the same six-tab structure with the same page layouts. Role and personal preferences are applied at render time to determine what is visible and what actions are available.

This is the **single IA, role-differentiated content** principle. Rachel's framing: *"a stage set with mannequins — we're changing the outfit, not how the stage is set."* The stage (structure, layout, navigation) never changes. The costume (visible sections, available actions, surfaced data) is fitted to the role.

Four layers of conditional display:

- **Sections** — entire page sections may be hidden for roles that have no access to that content category
- **Actions** — buttons, controls, and edit affordances render only for roles with the relevant permission
- **Data fields** — sensitive fields (e.g. `sensitive_ref_id` fields) render only for `senior_analyst` and above
- **Highlights** — personalized content surfaced based on topics of interest (preference-driven, not role-driven; see Section 3.3)

### 3.2 Role Capability Matrix

The canonical role set is defined in TS-08. The six human-facing roles are:

| Role | Description |
|---|---|
| `viewer` | Read-only access to published content |
| `analyst` | Full signal triage, classification, and evidence work |
| `senior_analyst` | All analyst actions plus approval authority |
| `architect` | All senior_analyst actions plus schema and governance decisions |
| `admin` | Full platform access including user management |
| `service_account` | Machine-to-machine only; not a human-facing role |

Key action permissions (full matrix in TS-08):

| Action | viewer | analyst | senior_analyst | architect | admin |
|---|---|---|---|---|---|
| View published Advisories | ✓ | ✓ | ✓ | ✓ | ✓ |
| Triage signals | — | ✓ | ✓ | ✓ | ✓ |
| Dispute a classification | — | ✓ | ✓ | ✓ | ✓ |
| Approve a classification | — | — | ✓ | ✓ | ✓ |
| Publish an Advisory | — | — | ✓ | ✓ | ✓ |
| Create a Technology Profile | — | — | ✓ | ✓ | ✓ |
| Request a record change | — | ✓ | ✓ | ✓ | ✓ |
| Execute a record change | — | — | — | ✓ | ✓ |
| Manage users | — | — | — | — | ✓ |

**Core governance pattern:** analysts can *request* changes to records; architects and admins can *execute* them. This two-step pattern is the foundation of record governance across the platform. It applies wherever data integrity requires a human checkpoint above the analyst tier.

### 3.3 Preference-Driven Highlights

Topics of interest and specialties (configured in DS-12 Profile) drive a **highlight strip** that surfaces personalized content on relevant pages. Highlights are additive — they surface relevant items at the top of a section without hiding or deprioritizing anything else.

A user with no preferences set sees the default unfiltered view with a prompt to configure topics of interest. Highlights never replace content; they layer on top of it.

---

## 4. Overview Tab

The Overview tab is the program-level ambient view. It is the first screen a user sees after sign-in. Its purpose is orientation: what has happened since my last visit, what requires my attention, and where is the portfolio right now.

### 4.1 Snapshot Modal

On first load after login — or after a qualifying absence period — a **modal overlay** appears before the main page content becomes interactive. The modal presents up to three snapshot cards summarizing activity since the user's last visit. It is dismissible by clicking outside it or pressing Escape.

> **Resolved:** Snapshot modal display threshold is configurable per user in DS-12 §6.3 (Profile and Preferences → Snapshot modal preference).

Preference to auto-dismiss after a set duration or always require manual dismissal is set in DS-12.

---

**Card 1 — Signal Volume**

What it shows:
- Total new signals collected since the user's last visit
- Number of active sources contributing during that period
- A visual indicator (sparkline or delta indicator) showing whether volume is up, down, or flat versus the equivalent prior period

Purpose: immediate situational awareness on pipeline health. If signal volume is anomalously low, the user knows to check sources before acting on content.

---

**Card 2 — Watchlist Movement**

What it shows:
- Changes on objects the user is currently watching: state transitions, new classifications, burst events, classification disputes
- Format: a compact list of the top N changed objects, each with the object name, type, and a one-line description of what changed
- Link at bottom: "View all watchlist activity" → TS-07 alerts view

If the user has no watchlist objects, this card shows: *"Nothing on your watchlist yet"* with a link to browse the Research tab and start watching objects.

---

**Card 3 — Your Queue**

What it shows:
- Items currently requiring the user's action: review requests, pending approvals, disputes on their own classifications, advisory sign-offs
- Format: a count badge plus the top 3 queue items with action links (e.g. "Review," "Approve," "Respond")
- Link at bottom: "Go to queue" → DS-08 governance queues

Role-conditional: `viewer` does not see this card. Nothing in the platform requires a viewer's action, so the queue concept does not apply to that role. The modal for a `viewer` contains only Card 1 and Card 2.

---

### 4.2 Topics of Interest Highlight Strip

Location: below the snapshot modal (or below the modal dismiss point if the modal has already been dismissed).

Layout: a horizontal strip of 3–5 highlight cards. Each card corresponds to one of the user's configured topics of interest and shows:
- Topic name
- Recent signal count (last 7 days)
- Burst indicator if the topic's associated neighborhood or cluster is currently in a burst state (per TS-07)
- Link to the relevant cluster or neighborhood in the Research tab

Users with no topics of interest configured see a single prompt card: *"Set your topics of interest to see personalized highlights"* with a link to DS-12 Profile.

The strip is preference-driven, not role-driven. All roles with access to the Overview tab see it (except `viewer` seeing only published content — their highlight cards link only to published entities).

### 4.3 Animated Temporal Radar

The primary program-level visualization. Occupies the upper half of the Overview page below the highlight strip.

> **DECISION NEEDED —** Should the Overview radar use ADR-010's Neighborhood-based quadrant layout, or a different organizational scheme suited to the ambient view? *Recommendation: match ADR-010 exactly so that the ambient radar and the Research tab radar are visually consistent — users should be able to orient to one and immediately navigate the other.*

**What it shows:**

The full Technology Portfolio plotted across four quadrants (per ADR-010 Neighborhood-based layout). Entities are represented as blips:

- Small dots at the outer ring: early signals and clusters (low maturity, low confidence)
- Progressively larger blips toward the center: established Technology Profiles with higher maturity and evidence density
- Blip size, color, and position encode maturity stage and Neighborhood assignment (full encoding key defined in DS-03 Research Tab)

**Animation:**

Movement is animated over a rolling 7-day window. On page load, blips begin at their positions from 7 days ago and animate to their current positions. The animation runs once, then holds at the current state. Blips that are newly appeared (entered the portfolio in the last 7 days) fade in during the animation. Blips that have been archived or merged fade out.

**Controls:**

| Control | Behavior |
|---|---|
| Pause / Play button | Always visible; pauses animation at the current frame; pressing Play resumes from current frame |
| Progress / scrub bar | A weather-radar-style timeline bar below the visualization representing the 7-day window. Click any point to jump blips to that moment. Drag to scrub. |
| Hover on a blip | Animation pauses; a mini card appears (see below) |
| Click on a blip or mini card link | Navigates to Research tab, opening that entity's detail view |

**Mini card (on hover):**

The hover mini card shows:
- Entity name
- Entity type: cluster or Technology Profile
- Current state (e.g. Emerging, Active, Boundary Case)
- Neighborhood assignment
- Link: "Open in Research →"

**What the Overview radar does not do:**

The animated radar is a read-only ambient visualization. It does not:
- Allow editing, reclassification, or state changes
- Replace the authoritative interactive radar (which lives on the Research tab, DS-03)
- Surface the full signal evidence or record detail

Any action requiring edit or classification authority requires navigating to the Research tab.

### 4.4 KPI Dashboard

Below the animated radar: a row of KPI tiles followed by secondary dashboard panels.

**KPI tiles:**

Displayed in a single horizontal row (wrapping on narrow viewports). Actual KPI definitions, thresholds, and calculation logic are specified in TS-10. Example tiles:

- Active Technology Profiles
- Signals collected (last 30 days)
- Advisories published (last 90 days)
- Prototypes underway
- Boundary cases pending review

Each tile shows the current value, a trend indicator (up/down/flat vs. prior period), and is tappable to navigate to the relevant section.

**Secondary panels:**

Two panels below the KPI row:

*Recent Advisories* — the last 3 published Advisories, showing title, publication date, and a link to the full document. `viewer` sees published Advisories only. `analyst` and above see pipeline status on Advisories in review (e.g. "2 pending approval").

*Active Bursts* — Neighborhoods or clusters currently in a burst state per TS-07, showing entity name, burst start time, and a link to the entity in Research. Role-conditional: `viewer` sees burst indicators only on entities with published content; `analyst`+ sees all active bursts regardless of publication status.

> **DECISION NEEDED —** Should the agentic weekly summary (an AI-generated pattern and trend digest surfaced on the Overview tab) require `senior_analyst` approval before it displays to all users, or can it auto-publish? *Recommendation: require `senior_analyst` approval, consistent with ADR-009 Tier 2 AI actions — AI suggests, human approves before the digest surfaces to the full user base.*

---

## 5. Empty and Loading States

### 5.1 New User (No Watchlist, No Preferences)

When a user logs in for the first time or has not yet configured any preferences or watchlist objects:

- Snapshot modal appears with Card 1 (Signal Volume) populated if data exists
- Card 2 (Watchlist Movement) shows: *"Nothing on your watchlist yet — visit Research to start watching signals and profiles"*
- Card 3 (Your Queue) shows: *"No pending items"*
- Topics of interest strip shows the single prompt card linking to DS-12
- The animated radar renders normally (it reflects the portfolio, not user preferences)
- KPI tiles render with current platform-wide values

### 5.2 Loading States

- **Snapshot modal**: appears immediately on login; modal data is a lightweight query and should resolve before the user dismisses it. Cards show a skeleton loader if data is still in flight
- **Animated radar**: displays a skeleton placeholder (quadrant outlines, empty blip field) while entity data loads; animation does not begin until data is fully resolved
- **KPI tiles**: individual skeleton loaders per tile; tiles populate as data resolves rather than waiting for all tiles

### 5.3 No Data States

For cases where the platform has been freshly deployed and no data has been ingested:

- Animated radar: renders quadrant outlines with an empty state message — *"No portfolio data yet"* — and a link to Surface for `analyst`+ users to begin signal ingestion; `viewer` sees the empty state with no action link
- KPI tiles: render with zero values and no trend indicator
- Recent Advisories panel: shows *"No Advisories published yet"*
- Active Bursts panel: shows *"No active bursts"*

---

## 6. Design Principles Inherited by All DS Specs

These principles apply platform-wide and are established here as the ground truth for all other design specs.

**1. Single IA, role-differentiated content.** The page structure never changes based on role. Visibility of sections, actions, and fields changes. Designers and engineers implementing any DS spec must preserve layout stability across roles.

**2. Additive personalization.** Preference-driven features (highlight strips, topics of interest) surface content above the default view. They never replace, hide, or reorder the default content below them.

**3. No orphaned affordances.** If a user cannot take an action, the affordance for that action is not rendered. Disabled buttons are used only when an action is temporarily unavailable due to state (e.g. "Approve" grayed out because required fields are incomplete), not due to role. Role-restricted actions are absent, not disabled.

**4. Governance actions require explicit confirmation.** Any action that changes a record's state, classification, or publication status requires a confirmation step. This applies to all roles with those permissions.

**5. AI-assisted actions follow ADR-009 tiers.** AI suggestions surface as suggestions. Tier 2 actions (digest publication, advisory drafts, classification recommendations) require human approval before they are visible to all users.

**6. Consistent radar orientation.** The Overview animated radar and the Research tab interactive radar use the same quadrant layout (ADR-010). Users orient to one and can immediately navigate the other.
