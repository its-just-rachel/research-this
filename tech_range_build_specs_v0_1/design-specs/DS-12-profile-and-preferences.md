# DS-12 — Profile & Preferences

**Status:** Draft  
**Last updated:** 2026-07-07  
**Relates to:** DS-01 (Overview), DS-11 (Admin & Role Management), TS-07 (Alerts & Watchlists), TS-08 (Roles & Permissions)

---

## 1. Purpose

Profile & Preferences is the personal configuration page available to every authenticated user. It controls two categories of settings:

- **Personalization** — topics of interest and specialties, which shape how content is surfaced to the user (highlighted, not filtered)
- **Notification preferences** — in-platform alerts, email digest frequency, and snapshot modal behavior

This page does not control role assignment or permissions. Those are admin-only functions defined in DS-11. The role field is shown here for the user's reference only.

---

## 2. Page Layout

- Single-page layout with clearly delineated sections. Not a multi-step wizard.
- Sections appear in order: Personal Info → Topics of Interest → Specialties → Notifications → Email Subscriptions
- Each section has its own **Save** button. Changes are saved per section, not globally. This reduces frustration when one section fails validation or hits an error — other sections remain saved.
- If the user navigates away with unsaved changes in any section, display an **unsaved changes warning** ("You have unsaved changes in [Section Name]. Leave anyway?") with options to stay or discard.
- Page is accessible from the utility nav via the Profile link, visible to all authenticated users.

---

## 3. Personal Info

| Field | Behavior |
|---|---|
| Display name | Editable. Used throughout the platform wherever the user's name appears. |
| Email | Read-only. Tied to SSO identity. Cannot be changed here. |
| Role | Read-only. Set by admin in DS-11. Shown for the user's reference. Canonical roles: `viewer`, `analyst`, `senior_analyst`, `architect`, `admin`. |
| Avatar / initials | If no avatar is uploaded, display auto-generated initials from the display name. Avatar upload optional. |
| Timezone | Dropdown selector. Affects all timestamp display throughout the platform (signals, advisories, activity feeds, digests). Default to browser-detected timezone on first login. |

---

## 4. Topics of Interest

Topics of interest drive a personalized spotlight — a highlight strip that surfaces relevant content on the DS-01 Overview page and on contextual pages (Surface, Research, etc.). They do not filter content or remove anything from view.

### 4.1 Technology Neighborhoods

Multi-select from the six canonical neighborhoods:

- Agentic / Vertical AI
- Edge / AI-RAN
- Physical AI
- Quantum
- SDx
- Cloud Platforms + Emerging

### 4.2 Domain Areas

Multi-select from the six canonical domain areas:

- National Security & Cyber
- Defense Tech
- Global Defense
- Civil
- Space Systems
- Vertical AI / Data Platforms

### 4.3 Interaction and limits

- Selections across both Neighborhoods and Domain Areas count toward a combined soft limit of **5 topics**.
- If the user selects more than 5, display a note: "We recommend choosing up to 5 topics so your highlight strip stays focused. You can still save more." Do not hard-block selection beyond 5.
- Help text displayed below the section heading:

> "Your selections surface highlighted content in your Overview and on relevant pages. They don't filter what you see — just add a spotlight."

### 4.4 Effect

Selections drive the topics-of-interest highlight strip defined in DS-01 §[Overview highlight strip]. The strip appears on the Overview page and as contextual highlight zones on Surface, Research, and other relevant stage pages.

> **DECISION NEEDED —** Should topics of interest drive algorithmic ranking of content (subtle reordering of signal lists, search results) in addition to the highlight strip? *Recommendation: no for MVP — highlight strip only; algorithmic ranking risks creating filter bubbles in an intelligence context and should be evaluated carefully before enabling.*

---

## 5. Specialties

Specialties capture the user's areas of functional expertise within the platform. They are used to contextualize the user — visible on contributions and useful for routing review requests. Specialties are not shown publicly; they are visible in admin view (DS-11) and to internal routing logic.

Example values (internal reference): signals analysis, threat modeling, supply chain, RF systems. *[Internal examples — full taxonomy TBD.]*

> **DECISION NEEDED —** Should Specialties be freeform tags (flexible but harder to aggregate) or a structured list (constrains vocabulary but enables routing and analytics)? *Recommendation: start with a curated list of [Internal specialty taxonomy] with a freeform "Other" option; prevents vocabulary explosion while remaining practical.*

---

## 6. Notification Preferences

### 6.1 In-Platform Notifications

Toggle on/off per notification type. All default to on for new users.

| Notification type | Default |
|---|---|
| New signal matches my watchlist | On |
| Burst detected in a neighborhood I follow | On |
| Boundary case created on an object I'm watching | On |
| State change on a watched object | On |
| Advisory published in my topics | On |
| Item assigned to me for review | On |

Toggling a type off suppresses it from the in-platform notification panel only. It does not affect email digest unless the user also adjusts email settings.

### 6.2 Email Digest

Controls whether and how often the user receives email notifications.

| Setting | Behavior |
|---|---|
| Immediate | One email sent per qualifying event. Intended for high-urgency users. |
| Daily digest | All qualifying events batched into a single daily email, sent at a user-configured time. |
| Weekly digest | All qualifying events batched into a single weekly email, sent on a user-configured day and time. |
| Off | No email notifications sent. |

Digest content mirrors the in-platform notification types in §6.1. Notification types toggled off in §6.1 are excluded from the digest.

For Daily and Weekly modes, the user can set their preferred send time (hour + timezone, pulled from Personal Info §3).

### 6.3 Snapshot Modal Preference

Controls whether the "since your last visit" snapshot modal (defined in DS-01 §4.1) appears on login.

| Setting | Behavior |
|---|---|
| Always show | Modal appears on every login session. |
| Show after absence | Modal appears only if the user has been absent for longer than a configured threshold. |
| Never show | Modal is suppressed. User can access snapshot via a manual trigger in the UI. |

Default: **Show after absence**, threshold = **4 hours**.

For the "show after absence" option, expose a threshold input (hours) so the user can adjust the default.

---

## 7. Watchlist Summary

A read-only panel providing a quick view of the user's current watchlist. This is not the primary place to manage watchlists — it is a convenience view only.

- Displays objects currently on the user's watchlist, grouped by type: Signals, Tech Profiles, Clusters, Advisories.
- Each item shows: object name, type, and date added.
- **Quick-remove** action per item (removes from watchlist without navigating away).
- Link to full watchlist management: "Manage watchlist →" routes to the alerts and watchlist UI defined in TS-07.
- If the watchlist is empty, display an empty state with a prompt to explore signals or profiles and add items to watch.

---

## 8. Open Decisions

> **DECISION NEEDED —** Should Specialties be freeform tags (flexible but harder to aggregate) or a structured list (constrains vocabulary but enables routing and analytics)? *Recommendation: start with a curated list of [Internal specialty taxonomy] with a freeform "Other" option; prevents vocabulary explosion while remaining practical.*

> **DECISION NEEDED —** Should topics of interest drive algorithmic ranking of content (subtle reordering of signal lists, search results) in addition to the highlight strip? *Recommendation: no for MVP — highlight strip only; algorithmic ranking risks creating filter bubbles in an intelligence context and should be evaluated carefully before enabling.*
