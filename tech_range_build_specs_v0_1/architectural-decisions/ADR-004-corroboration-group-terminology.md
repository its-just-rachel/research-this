# ADR-004: Corroboration Group Terminology

## Status: Accepted

---

## Context

Early in the Tech Range data model, the concept of "a collection of signals from multiple independent sources that confirm a pattern" had no agreed-upon name. Different parts of the codebase and documentation used different terms as the concept evolved:

- `corroboration_set` — the original schema name, used in early TS-02 drafts
- `signal_group` — used in some pipeline code and API sketches
- `evidence_group` — used in some research notes and ERD diagrams
- `Group` (unqualified) — used as shorthand in ADR-003 before it was corrected, and in some UI copy

This naming inconsistency is not just cosmetic. The entity sits at the center of the Tech Range evidence graph: signals belong to corroboration groups, analyst findings reference them, and the confidence scoring pipeline aggregates across them. Every layer of the stack — schema table names, foreign key columns, API resource paths, UI labels, and inter-service contracts — must agree on a single canonical name.

When multiple terms coexist in a codebase, the following failure modes emerge:

1. **Schema divergence**: a new migration creates `corroboration_groups` while old code still queries `corroboration_sets`, causing silent query failures or missing-table errors.
2. **Broken foreign keys**: a `signal` table row with a `corroboration_set_id` column cannot be joined to a `corroboration_groups` table without an alias layer — one that is easy to forget and hard to audit.
3. **Incomplete evidence graph traversal**: queries that walk the graph using the canonical name will skip any signals or findings that were written using the legacy name, producing confidence scores calculated on partial data.
4. **API contract drift**: a client built against `/corroboration-sets` and a client built against `/corroboration-groups` cannot share response parsing code, doubling maintenance surface.

ADR-003 was recently corrected to remove abbreviated "Group" references and use "Corroboration Group" throughout. This ADR formalizes the canonical name across all layers and provides migration guidance for any existing usage of legacy terms.

---

## Decision

The canonical name for this entity is **Corroboration Group**.

| Layer | Canonical form |
|---|---|
| Database table | `corroboration_groups` |
| Foreign key column | `corroboration_group_id` |
| API resource path | `/corroboration-groups` |
| API response key | `corroboration_group` (singular), `corroboration_groups` (plural) |
| UI label | "Corroboration Group" |
| Code identifiers (class, type, variable) | `CorroborationGroup` (PascalCase), `corroboration_group` (snake_case), `corroborationGroup` (camelCase) |

The abbreviated form "Group" is acceptable **only** in deeply nested UI contexts where a clear parent heading already establishes the entity type (e.g., a detail panel titled "Corroboration Group" may use "Group members" in a sub-section label). It is not acceptable in any schema identifier, API path, API response key, or top-level UI label.

All code, schemas, migrations, API documentation, and UI copy that uses any of the following legacy terms must be updated:

- `corroboration_set` / `CorroborationSet` / `corroborationSet`
- `signal_group` / `SignalGroup` / `signalGroup`
- `evidence_group` / `EvidenceGroup` / `evidenceGroup`
- Unqualified `group` / `Group` when referring to this entity (in any schema identifier, API path, or top-level UI label)

---

## Rationale

**Why not keep `corroboration_set`?**
"Set" carries a specific meaning in mathematics and computer science: an unordered collection with a uniqueness constraint on members. A Corroboration Group is neither. Members (signals) can be weighted, can be ordered by recency or confidence, and the same signal can theoretically contribute to multiple groups at different confidence weights. Using "set" would imply constraints the data model does not enforce and would mislead implementers.

**Why not just "Group"?**
"Group" is one of the most overloaded words in any engineering context. In Tech Range alone, there are analyst groups, user groups, tag groups, and notification groups. An unqualified `group` in a grep, a schema search, or a UI label returns noise from all of them. It also breaks discoverability: a new engineer reading `group_id` on a `signals` table has no idea what kind of group is being referenced without reading surrounding code. Unqualified "Group" fails the self-documentation test.

**Why "Corroboration Group"?**
The name is compositional and self-documenting:

- *Corroboration* names the function: multiple independent sources confirming a pattern. Any engineer who reads the word knows what the entity is for without consulting a glossary.
- *Group* names the structural form: a collection of members with a shared identity.

Together, "Corroboration Group" is specific enough to grep without false positives, descriptive enough to require no annotation, and short enough to use as a UI label without truncation in standard viewport widths.

---

## Migration Guidance

### Finding legacy terms in the codebase

Run the following to locate all files containing legacy terminology:

```bash
grep -r "corroboration_set" --include="*.sql" --include="*.py" --include="*.ts" --include="*.tsx" -l
grep -r "signal_group" --include="*.sql" --include="*.py" --include="*.ts" --include="*.tsx" -l
grep -r "evidence_group" --include="*.sql" --include="*.py" --include="*.ts" --include="*.tsx" -l
```

For unqualified `group` references in schema files, review results carefully — not all matches will be this entity:

```bash
grep -rn "group_id\|group_name\|GroupId\|GroupName" --include="*.sql" --include="*.py" --include="*.ts" -l
```

### Schema migration pattern

For any database that has a `corroboration_sets` table:

```sql
-- Step 1: Rename the table
ALTER TABLE corroboration_sets RENAME TO corroboration_groups;

-- Step 2: Rename the primary key sequence if applicable (PostgreSQL example)
ALTER SEQUENCE corroboration_sets_id_seq RENAME TO corroboration_groups_id_seq;

-- Step 3: Rename any foreign key columns in referencing tables
ALTER TABLE signals RENAME COLUMN corroboration_set_id TO corroboration_group_id;

-- Step 4: Rename indexes
ALTER INDEX idx_signals_corroboration_set_id RENAME TO idx_signals_corroboration_group_id;

-- Step 5: Drop and recreate foreign key constraints with updated names
ALTER TABLE signals DROP CONSTRAINT fk_signals_corroboration_set;
ALTER TABLE signals ADD CONSTRAINT fk_signals_corroboration_group
  FOREIGN KEY (corroboration_group_id) REFERENCES corroboration_groups(id);
```

All migration steps should be wrapped in a single transaction and tested against a staging database before applying to production.

### API versioning

If any deployed API version currently exposes `/corroboration-sets`, the old endpoint must not be removed without a deprecation window:

1. In the first release after this ADR is accepted, add `/corroboration-groups` as the canonical endpoint.
2. Configure `/corroboration-sets` to return HTTP 301 (Moved Permanently) redirecting to `/corroboration-groups`.
3. Log a deprecation warning for any client that hits the old endpoint.
4. Remove `/corroboration-sets` after one full release cycle (or per the project's API deprecation policy, whichever is longer).

### Documentation and diagrams

- Update all ERD diagrams to show the `corroboration_groups` table name.
- Search all architecture docs, spec files, and ADRs for legacy terms using the grep commands above.
- Update any OpenAPI / AsyncAPI schema files to use `corroboration_group` as the resource name and response key.

---

## Consequences

**Positive**
- Single unambiguous term across all layers of the stack.
- Self-documenting: any new engineer can read `corroboration_group_id` and understand what it references.
- Grepping for `corroboration_group` returns only this entity — no false positives from generic "group" matches.
- The evidence graph is unambiguous: all foreign keys point to one table with one name.

**Negative**
- One-time migration cost for any existing code, schema, or documentation using legacy names.
- If deployed database tables already exist under legacy names, a coordinated migration window is required.

**Risk if not done**
Terminology drift will cause evidence graph queries to return incomplete results. As new code writes to `corroboration_groups` and old code reads from `corroboration_sets`, confidence scores will be calculated on partial signal sets. This is a silent correctness failure — queries will succeed and return data, but the data will be wrong.

---

## DECISION NEEDED

> **DECISION NEEDED —** Is there any existing deployed code or database using `corroboration_set` (or any other legacy term) that requires a formal migration plan with a cutover date, rollback procedure, and stakeholder sign-off? *Recommendation: Audit all existing repos and any deployed database instances before the first implementation sprint begins. If legacy tables exist in any deployed environment, schedule the rename migration as a dedicated task in Phase 1 — before any new code writes to the table. Do not allow new code and old code to coexist in production against the same database.*

> **DECISION NEEDED —** Should the abbreviated form "Group" be completely banned in all code identifiers (variable names, function names, API response keys), or only in table names and column names? *Recommendation: Ban "Group" as a standalone identifier in all schema identifiers (table names, column names, index names), all API paths, and all API response keys. Allow it as a local variable name only within a function scope that is unambiguously about Corroboration Groups — for example, `const group = await fetchCorroborationGroup(id)` is acceptable; `const group_id` as a parameter name in a public function signature is not.*
