# DS-11 — User Management

**Status:** Draft  
**Last updated:** 2026-07-07  
**Related specs:** TS-08 (Roles & Permissions), TS-08 (Audit Log)

---

## 1. Purpose

User Management is an admin-only page for creating, editing, and deactivating platform users. It is accessible via the utility nav and is not visible to non-admin roles.

Admins use this page to:

- Invite new users and assign their initial role
- Update user names and roles
- Deactivate users who should lose platform access
- Reactivate previously deactivated users
- Manage service accounts and API keys
- Review per-user audit history

All changes made on this page are logged to the platform audit log per TS-08. The platform does not manage passwords; authentication is handled exclusively via SSO/SAML. Analysts may request changes through other channels, but only architects and admins may execute them.

---

## 2. Page Layout

### Header

- Page title: **User Management**
- Primary action button: **Invite User** (admin only; disabled or hidden for non-admin roles)

### Search

- Single search bar spanning the top of the user list
- Searches across: name, email, role
- Search is live-filtered (no submit required)

### User List Table

| Column | Notes |
|---|---|
| Name | Full name; sortable |
| Email | SSO identity email; not editable |
| Role | Current TS-08 role |
| Status | `Active`, `Invited`, or `Deactivated` |
| Last Login | Timestamp of most recent login; blank if never logged in |
| Date Added | Date the user record was created |
| Actions | Contextual row actions (see below) |

**Row actions:**

- **Edit** — opens edit panel (see Section 5)
- **Deactivate** — triggers confirmation dialog (see Section 6); shown only for active and invited users
- **Reactivate** — shown only for deactivated users
- **Expand** — expands inline audit trail for the user (see Section 7)

There is no Delete action. Hard deletes are prohibited per TS-08; users are deactivated only.

### Pagination

Paginated list with configurable page size (default: 50 rows). Pagination controls appear at the bottom of the list.

---

## 3. User List Behavior

### Default Sort

Alphabetical by last name, then first name.

### Filter Controls

Displayed above the table, alongside the search bar:

- **Role** — multi-select dropdown; options are all TS-08 human roles: `viewer`, `analyst`, `senior_analyst`, `architect`, `admin`
- **Status** — single-select: `All` (default), `Active`, `Invited`, `Deactivated`

### Deactivated Users

Deactivated users are shown by default (Status filter defaults to `All`). They are visually distinguished:

- Row text is muted (reduced opacity)
- Name rendered with strikethrough

This ensures admins have full visibility without needing to toggle a separate view.

### Bulk Actions

- Checkbox column at the far left of each row
- Selecting one or more rows reveals a bulk action bar above the table
- Available bulk action: **Deactivate Selected** (with confirmation dialog before execution)
- Bulk reactivation is not supported in bulk; must be done row by row to avoid accidental mass-reactivation

---

## 4. Invite / Create User Flow

Triggered by the **Invite User** button. Opens as a side panel (not a modal, not a separate page), so the admin retains context of the user list behind it.

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| First Name | Text | Yes | |
| Last Name | Text | Yes | |
| Email | Email | Yes | Must be unique; validated against existing user records |
| Role | Dropdown | Yes | All TS-08 human roles (see role picker below) |
| Send invite email | Checkbox | — | Default: checked; unchecking creates the user record without sending an SSO invitation email |

### Role Picker

The role dropdown displays each option with a brief inline description so the admin can make an informed choice without consulting external documentation:

- **viewer** — Read-only access to signals and advisories; no triage or classification
- **analyst** — Triage and classify signals; cannot approve or publish
- **senior_analyst** — All analyst capabilities plus approval authority for signal classifications
- **architect** — All senior analyst capabilities plus platform configuration and rule management
- **admin** — Full platform access including user management and audit log

### Validation

- Email must be unique across all user records (active, invited, and deactivated)
- Role must be selected before submitting
- First and last name must not be blank
- Inline validation errors appear below the relevant field

### On Submit

1. User record is created with status `Invited`
2. If "Send invite email" is checked: SSO invitation email is dispatched
3. User appears immediately in the user list with status `Invited`
4. Side panel closes; success toast confirms creation
5. Creation is logged to audit_log (actor: the admin who submitted, timestamp, fields set)

---

## 5. Edit User

Triggered by the **Edit** row action. Opens as a side panel.

### Editable Fields

| Field | Editable | Notes |
|---|---|---|
| First Name | Yes | |
| Last Name | Yes | |
| Email | No | Tied to SSO identity; shown read-only for reference |
| Role | Yes | Requires confirmation step (see below) |

### Role Change Confirmation

When an admin changes a user's role, a confirmation step is required before saving. The confirmation message is role-aware:

> Changing **[Name]**'s role from `[current_role]` to `[new_role]` will **[plain-language description of capability change]**. Confirm?

Examples:
- Promoting from `analyst` to `senior_analyst`: "…will grant approval authority for signal classifications."
- Promoting from `senior_analyst` to `architect`: "…will grant platform configuration access and rule management."
- Demoting from `architect` to `analyst`: "…will remove platform configuration access."

> **DECISION NEEDED —** Should role assignment require a second admin to confirm changes above a certain privilege level (e.g. promoting to `architect` or `admin`)? *Recommendation: yes — any promotion to `architect` or `admin` should require confirmation from a second `admin` account to prevent privilege escalation by a single compromised admin.*

### On Save

1. User record is updated
2. Side panel closes; success toast confirms update
3. Change is logged to audit_log (actor, timestamp, field changed, old value, new value)

---

## 6. Deactivate User

Triggered by the **Deactivate** row action or the bulk deactivate action.

### Confirmation Dialog

Single-user deactivation:

> **Deactivate [Name]?**
>
> They will lose access immediately. Their records and contributions are preserved.
>
> [Cancel] [Deactivate]

Bulk deactivation (N users selected):

> **Deactivate [N] users?**
>
> All selected users will lose access immediately. Their records and contributions are preserved.
>
> [Cancel] [Deactivate [N] Users]

### On Confirm

1. User status set to `Deactivated`
2. All active sessions for the user are revoked immediately
3. User cannot log in via SSO
4. User's data — signals triaged, classifications made, advisories contributed — is preserved and remains attributed to them
5. User row remains visible in the list with deactivated visual styling
6. Deactivation logged to audit_log (actor, timestamp, reason if captured)

### Reactivate

Admins can reactivate a deactivated user via the **Reactivate** row action.

- No side panel required; confirmation dialog only:

> **Reactivate [Name]?**
>
> They will regain access with their previous role: `[role]`.
>
> [Cancel] [Reactivate]

- On confirm: user status returns to `Active`; they can log in via SSO immediately
- Reactivation logged to audit_log

---

## 7. Audit Trail Panel

Each user row in the list can be expanded inline to reveal that user's change history.

### Contents

Displayed as a chronological log within the expanded row:

| Timestamp | Event | Actor |
|---|---|---|
| 2026-06-15 09:42 UTC | Role changed from `analyst` to `senior_analyst` | admin@techrange.io |
| 2026-04-02 14:18 UTC | User invited | admin@techrange.io |

Event types surfaced here:
- User created / invited
- Role changed (shows old → new role)
- User deactivated
- User reactivated

### Link to Full Audit Log

A **View full audit log →** link at the bottom of the expanded panel links to the TS-08 audit_log filtered to that user's record. The full audit log includes additional system-level events beyond what is summarized here.

---

## 8. Service Accounts

Service accounts (`service_account` role) are displayed in a separate section below the human user list, under the heading **Service Accounts**.

### Service Account List Columns

| Column | Notes |
|---|---|
| Name | Service account display name |
| API Key Status | `Active`, `Expired`, or `Revoked` |
| Last Used | Timestamp of most recent authenticated API request |
| Date Added | Date the service account was created |
| Actions | Create Key, Rotate Key, Deactivate |

### Create Service Account

Same **Invite User** side panel, but role is locked to `service_account`. No SSO invitation email is sent. On creation, the API key is generated and displayed once in the panel:

> **API Key**
> `tr_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
>
> Copy this key now. It will not be shown again.

After the panel is closed, the key value is never shown again. Only the key status and last-used date are visible in the list.

### Rotate API Key

Triggering **Rotate Key** opens a confirmation dialog:

> **Rotate API key for [Name]?**
>
> The existing key will be revoked immediately. A new key will be generated and shown once.
>
> [Cancel] [Rotate Key]

On confirm: old key is revoked; new key is generated and displayed once using the same one-time display pattern.

### Deactivate Service Account

Same confirmation pattern as human user deactivation. On deactivation, the API key is revoked immediately.

---

## Appendix: Status Definitions

| Status | Meaning |
|---|---|
| `Active` | User has accepted their invitation and can log in |
| `Invited` | User record created; SSO invitation sent; user has not yet logged in |
| `Deactivated` | User access revoked; sessions terminated; record preserved |
