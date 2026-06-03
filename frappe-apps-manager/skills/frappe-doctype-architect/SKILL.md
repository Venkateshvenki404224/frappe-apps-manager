---
name: frappe-doctype-architect
description: Plan and design a DocType architecture / data model for a new Frappe feature like a seasoned lead developer — interview the user, map entities, propose DocType names and links, and draw a flowchart of how the doctypes connect. Use this skill when the user says any of "help me plan a doctype architecture", "design a data model for", "what doctypes do I need for", "model a feature", "I want to build <feature>", "I need to implement an evaluation system", "issuing certificates", "design the schema for", "how should I structure my doctypes", "map the entities for", "plan the doctypes for a <X> system" (CRM, helpdesk, ticketing, LMS, booking, approval, subscription, inventory, etc.). It GRILLS the user with staged questions, proposes sensible defaults, keeps a running model, scans the existing app codebase to REUSE or EXTEND existing doctypes (adding fields / Custom Fields) before creating new ones, then produces a Mermaid diagram + per-DocType spec tables and offers to hand off to frappe-doctype-builder to generate the JSON. This skill PLANS only — it does not write files. Also use it when the user asks "can I reuse/extend an existing doctype for this?" or "is there already a doctype for X?".
---

# Frappe DocType Architect Skill

Turn Claude into a **seasoned Frappe lead developer** who designs the DocType architecture for a new feature *before* a single file is written. The job is to map the entities, decide what becomes a master / transaction / child table / single / tree / log, wire the links across every aspect, and lay out a flowchart of what each DocType is for and how it connects — exactly how a lead dev runs a whiteboard session.

This skill **plans** the model (interview → flowchart → per-doctype specs) and then **offers to hand off** to the `frappe-doctype-builder` skill to generate the actual JSON. It does **not** write files itself.

## When to Use This Skill

Claude should invoke this skill when:
- The user wants to **plan or design a DocType architecture / data model** for a feature ("I need an evaluation system for issuing certificates", "design a ticketing system", "model a subscription billing feature").
- The user asks **"what doctypes do I need for X?"** or "how should I structure this?"
- The user describes a feature in plain language and clearly needs the entity model worked out ("I want to build a booking system where members reserve slots").
- The user wants a **flowchart / diagram** of how their doctypes relate.
- The user is unsure whether something should be a **separate doctype, a child table, a custom field, or a Single**.
- The conversation is upstream of `frappe-doctype-builder` — i.e., the *what and why* must be settled before the *JSON*.

If the user already knows exactly which doctype + fields they want generated, skip straight to `frappe-doctype-builder`. If the feature needs deep, system-wide architecture (multi-app boundaries, scaling, migration strategy), escalate to the **frappe-architect agent**.

---

## The Interview Engine (the heart of this skill)

A lead dev does not guess. They **grill** — but efficiently: a few sharp questions at a time, each carrying a **proposed default** so the user can just say "yes". Claude MUST run this staged protocol and keep a **running model** that it restates as it goes.

### Operating rules (follow these the whole way through)

0. **ALWAYS open with the interview — never the deliverable.** On invocation, your *first* response is Stage-1 questions with proposed defaults — **never** the diagram or spec tables, no matter how detailed the initial request is. Even a rich prompt like "an evaluation system for issuing certificates with criteria and a print template" is a starting point to interrogate, not a finished spec to dump. The model is only "decidable" *after* the user has answered Stages 1–4.

1. **Ask 2–4 sharp questions per turn — never a wall of 20.** Batch related questions; let the user answer in one breath.

2. **A stage is NOT a single turn.** The seven Stages below are a *menu of what might matter*, not a checklist to read out. Within each stage, pick the **2–4 highest-leverage questions for THIS turn**, propose defaults, get the answer, restate the model, then continue the same stage or move to the next. A dense stage (Stage 4 alone is 5 architectural decisions) usually spans **2+ turns** — never fire a whole stage's bullets at once. Each stage is annotated below with its likely turn span.

3. **Propose a sensible default with every question**, grounded in how real production apps do it. Phrase as: *"I'd default to X (that's how Frappe Helpdesk models statuses). Sound right, or do you need Y?"* The user should be able to reply "yes to all".

4. **Fill the obvious gaps yourself.** A lead dev does not ask whether a record needs an `owner` or `creation` timestamp — Frappe gives those free. Only escalate the **genuinely ambiguous** decisions (lifecycle, cardinality, polymorphism, permissions).

5. **Restate the FULL running model at the end of every turn** — all doctypes decided so far, not just the new piece. Lead with "Running model now:" and a short bullet list. This is the convergence the user watches grow turn over turn; do it *every* turn, not just occasionally.

6. **Name things as you go** using the prefix convention (see Naming below). Real names make the model concrete.

7. **Never emit the deliverable until the model is user-confirmed.** Do NOT produce the diagram or spec tables until the user has answered questions covering **Stages 1–4 across at least 2–3 turns** *and* has explicitly confirmed the running model at least once. "Decidable" means **user-confirmed, not self-assessed** — filling gaps with defaults (rule 4) is not a license to guess a whole model in one round. Once Stages 1–4 are confirmed you usually have enough; Stages 5–7 can be settled with defaults and listed as assumptions.

8. **Push back on weak answers.** You are a lead dev, not a survey. If the user gives a sloppy, vague, or non-scaling answer (e.g. "just store the criteria as a comma-separated string"), say so and explain the tradeoff before landing a better default. Grilling means challenging, not only proposing.

9. **Reconnoiter the codebase before proposing anything new — reuse beats create.** Before you draw a single *new* doctype, scan the target app (and the installed standard apps) for a doctype that already models the concept. Default order is **reuse > extend > create new**. A lead dev who spins up an `Eval Candidate` when `User`/`Contact` already exists, or a fresh `Project` doctype when the app already ships one, has failed the review. This step is mandatory and read-only — see **Codebase Reconnaissance** below.

### Codebase Reconnaissance — reuse before you create *(read-only; resolve before the deliverable)*

A lead dev never designs against a blank page. Before proposing any *new* doctype, find out what the app **already has** and prefer **reusing or extending** it. This is read-only analysis — it never writes files.

**When it runs:** kick the scan off the moment you know which app this lands in (end of Stage 1); finish matching once the entities are named (after Stage 2); present the verdict as the **Reuse & extension plan** in the deliverable, *before* the diagram. Never draw a new node for a concept the app already models.

**1. Locate the app and its doctypes.** Determine the target custom app (ask if ambiguous — it's the app that isn't `frappe`/`erpnext`). Then enumerate what exists:
- DocType JSON lives at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json`. List with Glob `apps/<app>/**/doctype/*/*.json` (or `find apps -path '*/doctype/*' -name '*.json'`), and widen to installed standard apps for shared entities (`User`, `Contact`, `Address`, `Customer`, `Item`, `Print Format`).
- Confirm what's installed via `apps/apps.txt`, `sites/apps.json`, or `bench --site <site> list-apps`.
- Check what custom fields the app **already** ships: its `hooks.py` `fixtures` list, any `**/custom/*.json` fixture exports, and `create_custom_fields(...)` calls.

**2. Read each candidate's `.json`.** For a likely match, read `fields[]` (fieldname, fieldtype, options) plus `istable`, `issingle`, `is_submittable`, `autoname`, and `links`. Now you know what it already stores and how it's wired.

**3. Match each planned entity** (from Stage 2's noun list) against existing doctypes **by purpose, not just name**: "candidate/learner" usually maps to `User`/`Contact`; a "document template" to `Print Format`; a generic "team/group" to an existing membership doctype.

**4. Decide per entity — Reuse / Extend / Create new** (in that priority), and tell the user the verdict + reason for each:
- **Reuse as-is** — the existing doctype already holds what you need; just `Link` to it. (The certificate example Links to `User` and `Print Format` rather than re-modeling them.)
- **Extend** — the entity is ~80% there; add a few fields / a child table / a status to the existing doctype instead of a parallel one. Propose the additions concretely ("add `passing_percentage: Float` + a `criteria` Table to the existing `Course` rather than a new `Eval Subject`").
- **Create new** — only when no existing doctype fits, OR when extending would force unrelated concerns into a cohesive record. A lead dev knows extension has limits: don't bloat `User` with 30 evaluation fields — create a linked profile doctype instead.

**5. Pick the right extension mechanism** (this tells the builder *how* to apply it):
| You own the doctype? | How to extend | Notes |
|---|---|---|
| **Yes — it's in this app** | Add fields directly to its `.json` (+ controller logic). | Cleanest; you own the schema. |
| **No — it's in `frappe`/`erpnext`/another app** | **Custom Field** (+ **Property Setter** to tweak existing field props). Primary mechanism: `create_custom_fields({"Sales Invoice": [{...}], ...})` (dict keyed by doctype) in an `after_install` / migrate patch. Optionally export as **fixtures** — but **filter to your own fields** (`{"dt": "Custom Field", "filters": [["fieldname", "in", [...]]]}` in `hooks.py`); a bare `fixtures = ["Custom Field", "Property Setter"]` exports *every* customization on the site, including other apps'. | Never edit another app's files (they get overwritten on its next migrate). This is exactly how India Compliance adds GST fields to ERPNext `Sales Invoice`/`Item`. The UI equivalent is **Customize Form**. |
| **Behavior only (no new field)** | `doc_events` hooks in `hooks.py`, a Server Script, or a Client Script; **Property Setter** to relabel/hide an existing field. | Add `validate`/`on_update` logic or adjust a field without forking. |

Carry the chosen verdict + mechanism into the spec so the builder knows whether to emit a standard doctype JSON, a **Custom Field fixture**, or a **Property Setter**.

### Stage 1 — Domain & actors *(usually 1 turn)*
Establish what we are even building.
- Who are the **actors / roles**? (e.g., student, evaluator, admin — like LMS Student / Course Evaluator / Moderator; or Sales User / Sales Manager in CRM.)
- What is the **core "thing"** being managed — the noun the whole feature orbits? (the ticket in Helpdesk = `HD Ticket`; the lead in CRM = `CRM Lead`; the certificate in LMS = `LMS Certificate`.)
- What **event or lifecycle** drives it? (a customer raises an issue; a candidate sits an evaluation; a deal moves through a pipeline.)
- Is this its **own app/module**, or does it bolt onto an existing one (ERPNext, an existing custom app)?

### Stage 2 — Entities & cardinality *(usually 1–2 turns)*
Turn the description into nouns and count the lines between them.
- List the **nouns**. For each, decide: **master** (reference data that exists independently — `Customer`, `LMS Course`, `Course Evaluator`) vs **event/transaction** (something that *happens*, has a timestamp and a lifecycle — `Sales Order`, `LMS Certificate Evaluation`, `HD Ticket`).
- For each relationship: **one-to-many or many-to-many?** (One course has many lessons = 1:N. A user belongs to many channels and a channel has many users = N:M.)
- What must **persist** (stored) vs what can be **computed/derived** (read-only, recalculated — like CRM `net_total`, GP `comments_count`, LMS `progress`)?
- Is there **reusable, configurable list data** (statuses, categories, sources)? Those become lightweight metadata-config masters (`CRM Lead Status`, `HD Ticket Priority`).

### Stage 3 — Lifecycle & status *(usually 1 turn)*
This decides docstatus vs status-field vs Workflow — get it explicit.
- Does the core record get **submitted** (financial/legal immutability, GL impact, stock movement)? If yes → `is_submittable=1`, lifecycle is `docstatus 0 → 1 → 2` (draft → submitted → cancelled). (ERPNext `Sales Invoice`, India Compliance `Bill of Entry`.)
- Or is it just **status-tracked** with no submit semantics? Most collaboration/content apps choose this (LMS, Helpdesk, CRM, Gameplan all avoid `is_submittable` and use a status field or boolean flags instead).
- What are the **states** and the **legal transitions** between them? What **triggers** each transition (user action, agent reply, deadline, payment)?
- Should statuses be a **hardcoded Select** (GP Task: Backlog/Todo/In Progress/Done/Canceled) or a **configurable status master** so admins can add their own (`HD Ticket Status` with a `category` of Open/Paused/Resolved)?
- Do you need an **audit trail** of transitions? (CRM `CRM Status Change Log` visible child table; or Frappe `track_changes=1`.)

### Stage 4 — Relationships (the wiring) *(usually 2+ turns — pick 2–4 decisions per turn)*
This is where a lead dev earns their keep. These are **five distinct decisions**; do NOT fire them all in one turn. Apply the relevant ones out loud as they come up:
- **Fixed link vs polymorphic link.** If a field always points at *one* known doctype → **Link** (`Sales Order.customer → Customer`). If it must attach to *several* unrelated doctypes → **Dynamic Link** pair: `reference_doctype` (Link → DocType) + `reference_name` (Dynamic Link). This is the single most important pattern across the studied apps (CRM `FCRM Note`/`CRM Task`, Gameplan `GP Comment`/`GP Activity`, ERPNext `Payment Entry.party`, e-Invoice/e-Waybill logs, Raven document-linked notifications).
- **Child table vs separate doctype.** Decision rule: **tightly-coupled lifecycle + always fetched with the parent + no independent permissions/listing/queries → child table** (`Sales Order Item`, `LMS Quiz Question`, `Evaluator Schedule`, `CRM Service Level Priority`). **Otherwise → separate doctype with a Link back** (`Address`, `Contact`, `LMS Question`, `HD Ticket Comment` are independent and reusable, so they are NOT child tables).
- **Many-to-many → a join doctype.** Never an array field (Frappe has no native array type). Make a standalone doctype carrying both Links plus rich metadata (`Raven Channel Member` with `is_admin`/`last_visit`; `GP Member`; ERPNext `Item Supplier`). Use a child table join only when one side fully owns the relationship and it carries no independent queries (`HD Team Member`).
- **Hierarchy → tree (Nested Set).** Self-referential parent + `is_tree=1` (adds `lft`/`rgt`/`parent_*` Nested Set columns) for cascading rollups (`CRM Territory`, `Item Group`, `Customer Group`, `HD Article Category`).
- **Denormalization → `fetch_from` (single-hop only).** When a child/dependent record needs a value from a doctype it Links to, read-only, use `fetch_from` instead of duplicating data. **`fetch_from` is single-hop:** it copies a field from the doctype targeted by a Link field **on this same doctype**, written as `fetch_from: <link_field_on_this_doctype>.<field_on_linked_doctype>` (CRM Contacts pulls `full_name` from a linked record; LMS fetches `member_name` from the linked `member`). It does **not** chain through a grandparent in one expression. To get a **grandparent's** value you must either (a) add a Link to the grandparent on this doctype and fetch from *that*, or (b) chain fetches one level at a time — the parent fetches from the grandparent, then the child fetches from the parent's already-fetched field. See Pattern 8.

### Stage 5 — Fields & data *(usually 1 turn, defaults-heavy)*
- For each doctype, which fields are **required** vs optional? Which are **computed/read-only** (recalc server-side)?
- Where does **`fetch_from`** apply (single-hop read-only denormalized names/titles)?
- **Naming strategy** per doctype (pick the strategy + why; the exact `autoname` JSON is generated by `frappe-doctype-builder`):
  - **Naming series** — dotted `.YYYY.-` style tied to a `naming_series` Select field (`CRM-LEAD-.YYYY.-`, `SAL-ORD-.YYYY.-`).
  - **`field:fieldname`** — natural-key naming (Course Evaluator named by its `evaluator`).
  - **`hash` / Random** — for child/join rows that never need a human-facing ID.
  - **`autoname` expression** — hash-digit / format placeholders like `ASG-{#####}` or composite `{course}/{slug}` (`naming_rule: Expression`; this is *not* a naming series — naming series use the dotted `.####`/`.YYYY.` syntax).
  - **`autoincrement`** — DB auto-increment PK for high-volume internal records where the ID is never user-facing (several Gameplan doctypes use this).
  - Tree masters are **commonly (not necessarily)** named by `field:fieldname` — that is convention, not a rule; `is_tree` only governs the `lft`/`rgt`/`parent` hierarchy and imposes no naming constraint. A tree can equally be named by Prompt, naming series, or expression (e.g. ERPNext `Account`/`Cost Center` are trees named `field:` only by convention).
- **Uniqueness / dedup** constraints (Raven enforces unique `(message, owner, reaction)` at the DB level; `Course Evaluator.evaluator` is unique).
- **Conditional fields**: `depends_on`, `mandatory_depends_on`, `read_only_depends_on` (LMS: `rating` mandatory only when `status == 'Pass'`; CRM call log shows caller/receiver by call type).

### Stage 6 — Permissions & visibility *(usually 1 turn, defaults-heavy)*
- Which **roles** exist and what can each do (CRUD/submit/cancel)? (CRM: System Manager / Sales Manager / Sales User. LMS: Moderator / Course Creator / Batch Evaluator / LMS Student.)
- **Owner-based** access (`if_owner` write so students edit only their own `LMS Certificate` / submission)?
- **User Permissions** to scope a user to specific records (e.g., a user only sees their territory)?
- **Document sharing** for ad-hoc cross-team access (Helpdesk tickets use `share`)?
- **Field-level** read-only / permlevel for sensitive fields (India Compliance hides credentials; SLA timestamps are read-only).

### Stage 7 — Integrations & automation *(usually 1 turn, defaults-heavy)*
- External systems? → model an **immutable Log doctype** with a Dynamic Link back to the source and read-only response fields (India Compliance `e-Invoice Log` / `e-Waybill Log`; Raven `Raven Document Notification`). Track async request lifecycle in a child table (`GSTR Action`).
- **Notifications**? → a notification doctype with `from_user`/`to_user` + a reference link (`CRM Notification`, `HD Notification`, `GP Notification`).
- **Scheduled jobs / deadlines** (auto-close tickets, evaluation windows, SLA clocks)?
- **Global configuration** → a **Single** settings doctype (`issingle=1`) aggregating child tables for config (`GST Settings`, `HD Settings`, ERPNext `Selling Settings`).

---

## Decision Frameworks (crisp, reusable)

### DocType kind: master vs transaction vs child vs single vs tree vs log
| Kind | Use when | `is_submittable` / flags | Real example |
|---|---|---|---|
| **Master** | Reference data with independent lifecycle, linked from many places | normal | `Customer`, `LMS Course`, `Course Evaluator`, `HD Team` |
| **Transaction** | Something that *happens*, has a status/lifecycle, references masters | status field, sometimes `is_submittable=1` | `Sales Order`, `LMS Certificate Evaluation`, `HD Ticket` |
| **Child table** | Rows owned by a parent, fetched with it, no independent listing | `istable=1` | `Sales Order Item`, `LMS Quiz Question`, `Evaluator Schedule` |
| **Single** | One global config record | `issingle=1` | `GST Settings`, `HD Settings`, `Stock Settings` |
| **Tree** | Self-referential hierarchy with cascading rollups | `is_tree=1` (`lft`/`rgt`) | `Item Group`, `CRM Territory`, `HD Article Category` |
| **Log** | Immutable audit / external-sync record | normal, read-only fields | `e-Invoice Log`, `GP Activity`, `HD Ticket Activity` |

### Link vs Dynamic Link
- **Link** → field always targets one known doctype. Cheap, validated, filterable. Default choice.
- **Dynamic Link** (`reference_doctype` + `reference_name`) → field must target *many* unrelated doctypes. Use for activities/comments/notes/logs/payments that attach to anything. Avoids one nullable FK per possible parent. Validate the pairing in code. (CRM `FCRM Note`, GP `GP Comment`, ERPNext `Payment Entry.party`, `e-Invoice Log`, Raven document notifications.)

### Child table vs separate doctype
- **Child table** if: owned by exactly one parent, lifecycle dies with parent, always loaded inline, never independently queried/permissioned. (`Sales Order Item`, `CRM Products`, `LMS Quiz Result`.)
- **Separate doctype** if: reusable across parents, independently queried/listed, has its own permissions or status, or is referenced polymorphically. (`Address`, `Contact`, `LMS Question`, `HD Ticket Comment`, `Raven Channel Member`.)

### status field vs Workflow doctype vs docstatus
- **docstatus (0/1/2)**: only for genuine submit semantics — financial/legal immutability, GL/stock impact. (`Sales Invoice`, `Bill of Entry`.) Most apps studied **do not** use it.
- **status field (Select or Link)**: business-readable lifecycle. Hardcoded `Select` (an inline newline-delimited option list) for fixed small sets (GP Task); **Link to a status master** when admins must extend it and control behavior per status (`HD Ticket Status.category`). Note: a status that comes from a configurable master MUST be a `Link` (with `options` = the master doctype), never a `Select` — a `Select` can only hold a hardcoded inline list and cannot reference another doctype.
- **Workflow doctype**: only when you need approval gates, role-gated transitions, and SLA on transitions. Heavier; add as a separate layer. For pure transition *code* (`validate_state_transition`, `on_submit`), hand off to **frappe-state-machine-helper**.

### Reuse vs Extend vs Create new (run reconnaissance first)
Resolve this for **every** entity before drawing it. Priority: **reuse > extend > create**.
- **Reuse as-is** — an existing doctype (in this app or an installed one) already models the concept; just `Link` to it. (`User`, `Contact`, `Address`, `Customer`, `Item`, `Print Format` are almost always reused, never re-modeled.)
- **Extend an existing doctype** — the concept is *mostly* there; add a few fields / a child table / a status rather than a parallel doctype. (Helpdesk adds fields to core `Contact`; India Compliance adds GST fields to `Item`/`Sales Invoice`.) **How** depends on ownership: own-app → edit its `.json`; other-app → **Custom Field**/**Property Setter** via `create_custom_fields` + `fixtures` (never fork the other app). See the Reconnaissance step.
- **Create new** when the concept has its own identity, lifecycle, list view, or many fields, OR when extending would force unrelated concerns into a cohesive record (don't bloat `User` with 30 evaluation fields — create a linked profile doctype).

### Naming strategies (pick which + why; the builder emits the JSON)
- **Naming series** for user-facing sequential IDs via the dotted syntax + a `naming_series` Select field: `CRM-LEAD-.YYYY.-`, `SAL-ORD-.YYYY.-`.
- **`field:fieldname`** when one field is the natural key (Course Evaluator named by `evaluator`).
- **`hash` / Random** for child tables and join rows that never need a human-facing ID (`Evaluator Schedule`, reactions).
- **`autoincrement`** for high-volume internal records where the ID is never shown to users (several Gameplan doctypes).
- **`autoname` expression** (`naming_rule: Expression`) for hash-digit / composite keys like LMS `ASG-{#####}` or Cohort `{course}/{slug}` — distinct from naming series.
- Trees impose no naming requirement (`is_tree` only adds `lft`/`rgt`/`parent`); `field:fieldname` for trees is convention, not a rule.

---

## Pattern Library (distilled from 7 production apps)

Named, transferable patterns. Reach for these instead of reinventing.

1. **Activity / Comment / Note via Dynamic Link** — one doctype attaches to any parent via `reference_doctype` + `reference_name`. *Apps:* CRM (`FCRM Note`, `CRM Task`, `CRM Call Log`), Gameplan (`GP Comment`, `GP Activity`). *Use when* notes/tasks/comments/logs must hang off multiple entity types.
2. **Configurable Status Master** — status is a **Link** to a master with a `category`/`type` and a color, not a hardcoded `Select` enum. *Apps:* Helpdesk (`HD Ticket Status` → Open/Paused/Resolved), CRM (`CRM Lead Status`). *Use when* admins must add statuses or status must drive behavior (pause SLA, portal label).
3. **Visible Audit-History Child Table** — record transitions as a child table on the record itself, not a hidden log. *App:* CRM (`CRM Status Change Log` with from/to/duration/owner). *Use when* users should see "who moved this when" in context.
4. **SLA / Targets via per-priority child table** — one agreement holds a child table of targets keyed by priority, plus working-hours and holiday children. *Apps:* Helpdesk (`HD Service Level Agreement` + `HD Service Level Priority` + `HD Service Day` + `HD Service Holiday List`), CRM (mirror). *Use when* response/resolution targets vary by priority and must respect business hours.
5. **Membership via Join Doctype** — many-to-many becomes a standalone doctype carrying both links + metadata. *Apps:* Raven (`Raven Channel Member` with `is_admin`, `last_visit`), Gameplan (`GP Member`), ERPNext (`Item Supplier`). *Use when* the relationship itself has attributes or must be queried.
6. **Settings via Single** — global config in `issingle=1` aggregating config child tables. *Apps:* `GST Settings`, `HD Settings`, `Selling Settings`. *Use when* there's exactly one config record per site.
7. **External Sync via Log doctype** — immutable, non-submittable record with read-only API-response fields + Dynamic Link to the source + async child table for request polling. *App:* India Compliance (`e-Invoice Log`, `e-Waybill Log`, `GSTR Action`). *Use when* you call an external API and need an audit trail decoupled from submit status.
8. **Single-hop `fetch_from`, chained level by level** — keep a denormalized value read-only via `fetch_from` instead of duplicating data, remembering it is **one hop only** (`<link_field_on_this_doctype>.<field_on_linked>`). For deeper scope, either add a direct Link to the higher-level doctype on this record and fetch from it, or chain: the parent fetches a field from the grandparent, then the child fetches that already-populated field from the parent. *Apps:* Gameplan (each level carries its own `team` Link/fetched field rather than reaching two hops up in one expression), LMS (a chapter fetches `course`, then a lesson fetches `course` from its chapter — one hop each). *Use when* a deeper record needs a high-level scope field for filtering/permissions; never write a two-dot grandparent expression — it silently never populates.
9. **Role Config + Booking Transaction split** — separate *who/when available* (master with a recurring-schedule child table) from *the actual booked session* (transaction). *App:* LMS (`Course Evaluator` + `Evaluator Schedule` vs `LMS Certificate Evaluation` + `LMS Certificate Request`). *Use when* you schedule appointments against a resource's availability.
10. **Issued Credential pattern** — a transaction record representing an issued certificate with a unique number, issue/expiry dates, the issuing evaluator, and a Link to a render template (Print Format). *App:* LMS (`LMS Certificate` → `template` is a `Print Format`). *Use when* you issue certificates, licenses, badges.
11. **Reusable Question / Quiz composition** — a reusable question master, joined into a quiz via a child table that carries per-quiz marks, with submissions captured as a transaction + per-question result child table. *App:* LMS (`LMS Question` ← `LMS Quiz Question` → `LMS Quiz`; `LMS Quiz Submission` → `LMS Quiz Result`). *Use when* assessment items are reused across assessments.
12. **Feature flags via Check fields** — optional features toggled by booleans rather than status (`enable_certification`, `paid_certificate`, `is_ai_thread`, `is_archived`). *Apps:* LMS, Raven. *Use when* features are orthogonal add-ons, not lifecycle states.

---

## Output Format

**Precondition:** Do NOT produce items a–g until the user has answered questions covering Stages 1–4 across at least 2–3 turns **and has explicitly confirmed the running model at least once.** "Decidable" means user-confirmed, not self-assessed. Run the interview — and the reconnaissance — first.

When that precondition is met, deliver these in this exact order (a–g):

**a. Restated understanding** — 2–4 sentences confirming the feature, the core entity, the actors, and the driving lifecycle.

**b. Reuse & extension plan** — the reconnaissance verdict (see the **Codebase Reconnaissance** step). A short table: for each planned entity, one of **Reuse `X`** / **Extend `X` (add …)** / **Create new**, plus the reason, and for every *Extend* the mechanism (own-app field vs Custom Field/Property Setter fixture). Name the standard doctypes you're reusing (`User`, `Print Format`, …) so the diagram's reused nodes are explained. This comes **before** the diagram — never propose a new doctype the app already has.

**c. Mermaid diagram** — every proposed doctype and its links. This is the flowchart the user asked for. Prefer `erDiagram` for entity/link clarity; use `flowchart` if the user wants the lifecycle flow. The diagram MUST agree with the spec tables: every drawn relationship must be backed by a real field in some spec table, and every Link target in the diagram must match the `options/target` in the corresponding spec row. Template:

```mermaid
erDiagram
    MASTER_A ||--o{ TRANSACTION_B : "has many"
    TRANSACTION_B ||--|{ CHILD_C : "contains (child table)"
    MASTER_D ||--o{ TRANSACTION_B : "referenced by"
    STATUS_MASTER ||--o{ TRANSACTION_B : "status (Link)"
    TRANSACTION_B ||..o{ LOG_E : "Dynamic Link"
    MASTER_A {
        Link owner_field "User"
        Link status "Link to STATUS_MASTER"
        Select kind "inline hardcoded options"
    }
```
Notation guide to include for the user: `||--o{` one-to-many, `}o--o{` many-to-many (via join doctype), `||--|{` child-table containment, `..` dotted = polymorphic Dynamic Link. A master-driven status is a **`Link`** to the status master; a `Select` is only for a hardcoded inline option list.

**Mark each node's origin** so reused vs new is obvious — this is the whole point of reconnaissance showing up in the diagram. In the spec table append `(reuse)`, `(extend: +fields)`, or `(new)` to each entity's purpose; if you draw a `flowchart` instead of `erDiagram`, add a `classDef` (e.g. green = reuse, amber = extend, blue = new) and class each node. In an `erDiagram`, reused standard doctypes like `USER` and `PRINT_FORMAT` are entities you **Link to**, not new ones — call that out in item **b**.

**d. Per-DocType spec table** — one block per doctype (and one per *extended* existing doctype, listing only the added fields + the mechanism). Treat each table as a **spec to hand to the builder**, not final JSON.
> **`DocType Name`** — *kind:* master/transaction/child/single/tree/log · *naming:* `<strategy>` · *purpose:* one line.

| fieldname | fieldtype | options / target | reqd | why |
|---|---|---|---|---|
| `member` | Link | `User` | yes | who is being evaluated |
| `status` | Link | `Eval Status` | yes | configurable status master (admins extend it) |
| `rating` | Rating | — | cond. | `mandatory_depends_on eval:doc.status=='Pass'` |

(If a status is a fixed inline set instead of a master, use `Select` with a `Pending\nIn Progress\nPass\nFail` option list — never `Select` pointing at another doctype.)

**e. Relationship map in prose** — "X links to Y via `<field>` because …; Z is a child table of X because its lifecycle is owned by X; W uses a Dynamic Link so it can attach to both A and B." Every relationship here must name the backing field.

**f. Open questions / assumptions made** — list the defaults you chose on the user's behalf and the few genuinely ambiguous decisions still needing a yes/no.

**g. Next step / handoff offer** — explicitly offer:
- "Want me to generate the actual DocType JSON files? I'll hand this spec to the **frappe-doctype-builder** skill, doctype by doctype."
- If there are non-trivial state transitions: "I can also hand the lifecycle to **frappe-state-machine-helper** to generate the `validate_state_transition` / `on_submit` controller code."
- For system-wide concerns: "For deeper architecture (app boundaries, scaling, data migration), the **frappe-architect** agent can take it from here."

Do **not** write files in this skill. Stop at the plan and the offer.

---

## Full Worked Example: "An evaluation system for issuing certificates"

This shows how a lead dev runs it, grounded in the real Frappe LMS model. Note the cadence: 2–4 questions per turn, a full running-model restatement every turn, and one exchange where Claude **pushes back** on a weak answer.

### The interview (abridged, lead-dev voice)

**Claude (Stage 1 — domain & actors):**
> Let's map this. I'm reading three actors: the **candidate** being evaluated, the **evaluator** who scores them, and an **admin/moderator** who configures everything. The core "thing" is the **evaluation event**, and it culminates in an **issued certificate**. That mirrors how Frappe LMS splits it: `Course Evaluator` (who evaluates), `LMS Certificate Evaluation` (the session), and `LMS Certificate` (the credential). Two quick questions:
> 1. Are evaluations tied to a **course/subject**, or is it a standalone exam? I'd default to a `Subject` master so certificates say *what* was certified — yes?
> 2. Is a certificate issued **automatically on pass**, or only after an evaluator signs off? I'd default to evaluator sign-off (Pass → issue), like LMS — okay?

**User:** Yes to a subject. Evaluator signs off, and only a Pass issues a certificate.

**Claude (Stage 2 — entities & cardinality, turn 1):**
> Good. **Running model now:**
> - `Eval Subject` *(master)* — what's being certified.
> - `Eval Evaluator` *(master)* — wraps a `User`, will hold availability.
> - `Certificate Evaluation` *(transaction)* — one candidate's scored session.
> - `Issued Certificate` *(transaction)* — the credential, unique number + validity.
>
> Now the scoring rubric. How do you want to capture *what* the candidate is graded on?

**User:** Just store the criteria as a comma-separated string on the subject — simplest.

**Claude (push-back):**
> I'd push back on that. A comma-separated string blocks per-criterion scoring, per-criterion weighting, and reuse of a criterion across subjects — and you can't query "average score on Communication" later. I'd insist on a reusable `Eval Criteria` *master* joined into each subject via a `Subject Criteria` *child table* that carries the weight (this is exactly the LMS `LMS Question` ← `LMS Quiz Question` → `LMS Quiz` shape). The cost is one extra master + one child table; the payoff is per-criterion scoring and reuse. The open call: are criteria **shared across subjects** (reusable master, M:N via the join child) or **owned per-subject** (1:N, simpler)? I'd default to reusable. Which way?

**User:** You're right — reusable criteria. Different subjects share the "communication" criterion.

**Claude (Stage 2 — turn 2, restate):**
> **Running model now:**
> - `Eval Subject` *(master)* — what's certified; owns a `criteria` child table.
> - `Eval Criteria` *(master)* — reusable rubric line item.
> - `Subject Criteria` *(child table of Eval Subject)* — joins a criterion in with a `max_marks` weight (M:N join).
> - `Eval Evaluator` *(master)*.
> - `Certificate Evaluation` *(transaction)*.
> - `Issued Certificate` *(transaction)*.
>
> Cardinality is now: one subject → many evaluations (1:N); one evaluation → many per-criterion scores (1:N, owned → child table). Good so far?

**User:** Yes.

**Claude (reconnaissance — reuse check, now that the entities are named):**
> Before I wire anything, I scanned the app for doctypes that already model these. Findings:
> - **Candidate & evaluator → reuse `User`.** No new "person" doctype; I'll `Link` to `User` and `fetch_from` the name. I won't bloat `User` with evaluation fields.
> - **Certificate render → reuse `Print Format`.** The credential's `template` Links to a standard `Print Format`, exactly like LMS — no custom template doctype.
> - **`Eval Subject` / `Eval Criteria` / `Certificate Evaluation` / `Issued Certificate` → create new.** Nothing in the app models a certification rubric, a scored session, or a credential, so these are genuinely new.
> - Note: *if* you already had an ERPNext `Course`, I'd propose **extending** it — add `passing_percentage` + a `criteria` child table via **Custom Field fixtures** (it lives in another app, so we don't fork it) — instead of a new `Eval Subject`. You don't, so we build it.
>
> Reuse > extend > create, applied. Proceeding to lifecycle.

**User:** Good — none of that exists yet.

**Claude (Stage 3 — lifecycle & status):**
> Lifecycle of `Certificate Evaluation`:
> - Does it need **submit** semantics (immutable, docstatus)? I'd say **no** — LMS deliberately avoids `is_submittable` here; a status field is cleaner and lets evaluators edit until final. We'll use `status: Pending → In Progress → Pass | Fail`, exactly like `LMS Certificate Evaluation`.
> - I'll make `rating` and `summary` **conditionally mandatory** — `rating` required when `status == 'Pass'`, `summary` required once it leaves Pending/In Progress (the real LMS rule). OK?
>
> **Running model now:** the six doctypes above, with `Certificate Evaluation` driven by a `status` Select (no docstatus) and conditionally-mandatory `rating`/`summary`. On **Pass**, a controller issues the `Issued Certificate` — that transition is a candidate for **frappe-state-machine-helper** later.

**User:** Perfect.

**Claude (Stage 4 — turn 1: the role-config-vs-booking split only):**
> Now the evaluator side. Evaluators have **recurring availability**, and each candidate **books a slot** against it. I'd apply the LMS role-config-vs-booking split:
> - `Eval Evaluator` (master) holds an `Evaluator Schedule` child table (day-of-week + start/end) plus `unavailable_from`/`unavailable_to`.
> - The actual booked appointment is a **separate** `Certificate Request` transaction (`Upcoming → Completed | Cancelled`) with date/time + a meet link.
>
> **Running model now:** the six doctypes, plus `Evaluator Schedule` (child of `Eval Evaluator`) and a new `Certificate Request` (transaction). Take the split, or fold booking into the evaluation record?

**User:** Take the split.

**Claude (Stage 4 — turn 2: Link targets + fetch_from only):**
> Two wiring decisions left:
> 1. **Evaluator link target.** On `Certificate Evaluation` and `Certificate Request`, do I Link the evaluator to plain **`User`**, or to **`Eval Evaluator`**? I'd default to `Link → User` (lightest; LMS does this) and only switch to `Eval Evaluator` if you need profile fields *on the session*. I'll draw the diagram to match whichever you pick.
> 2. **Backing the schedule edge.** I'll add a `request` Link on `Certificate Evaluation` → `Certificate Request` so the evaluation knows which booking scheduled it (that's the field behind the "schedules" arrow).
> 3. **`fetch_from` (single-hop).** `member_name` ← `member.full_name`, `subject_title` ← `subject.subject_name`, `evaluator_name` ← `evaluator.full_name` — each one hop off a Link on the same doctype. Good?

**User:** Link to User is fine. Yes to the rest.

**Claude (Stage 5/6 — turn 1: naming only):**
> **Naming:** `Certificate Evaluation` → naming series `EVAL-.YYYY.-`; `Certificate Request` → `REQ-.YYYY.-`; `Issued Certificate` → `CERT-.YYYY.-` with a **unique** `certificate_number`; `Eval Evaluator` → `field:evaluator`; `Eval Subject`/`Eval Criteria` → `field:`-name; child tables → `hash`. The certificate links a `Print Format` as its render `template` (LMS does exactly this). Sound right?

**User:** Yes.

**Claude (Stage 5/6 — turn 2: permissions only):**
> **Roles:** Moderator (full), Evaluator (create/edit evaluations + requests), Candidate (read own certs via `if_owner`). I'll list these as assumptions in the output unless you object. Confirm and I'll draw the full model.

**User:** Confirmed.

### Reuse & extension plan (item b — comes before the diagram)
| Planned entity | Verdict | Mechanism / reason |
|---|---|---|
| Candidate, Evaluator (person) | **Reuse `User`** | Standard doctype; `Link` + single-hop `fetch_from` for the name. Don't extend `User` with eval fields. |
| Certificate template | **Reuse `Print Format`** | Standard render doctype; `Issued Certificate.template` Links to it (LMS pattern). |
| `Eval Subject`, `Eval Criteria`, `Subject Criteria` | **Create new** | No existing doctype models a certification rubric. |
| `Eval Evaluator` | **Create new** | Linked-profile master wrapping `User` (holds availability + the `Evaluator Schedule` child); keeps evaluation fields *off* `User`. |
| `Certificate Evaluation`, `Evaluation Score` | **Create new** | No existing scored-session concept. |
| `Issued Certificate`, `Certificate Request`, `Evaluator Schedule` | **Create new** | No existing credential / booking / availability model. |

*(Counterfactual: had the app already shipped a `Course`/`Subject`, the verdict would be **Extend** — add `passing_percentage` + a `criteria` Table; via **Custom Field fixtures** if it lived in another app, or a direct `.json` edit if we owned it. Reuse > extend > create.)*

### Resulting Mermaid diagram

```mermaid
erDiagram
    EVAL_SUBJECT ||--o{ SUBJECT_CRITERIA : "rubric (child table)"
    EVAL_CRITERIA ||--o{ SUBJECT_CRITERIA : "referenced by"
    EVAL_SUBJECT ||--o{ CERTIFICATE_EVALUATION : "evaluated for"
    EVAL_EVALUATOR ||--o{ EVALUATOR_SCHEDULE : "availability (child table)"
    USER ||--o{ CERTIFICATE_EVALUATION : "scores (evaluator)"
    USER ||--o{ CERTIFICATE_REQUEST : "booked for (evaluator)"
    CERTIFICATE_REQUEST ||--o| CERTIFICATE_EVALUATION : "schedules (via request Link)"
    CERTIFICATE_EVALUATION ||--|{ EVALUATION_SCORE : "per-criterion (child table)"
    EVAL_CRITERIA ||--o{ EVALUATION_SCORE : "scored against"
    CERTIFICATE_EVALUATION ||--o| ISSUED_CERTIFICATE : "issues on Pass"
    PRINT_FORMAT ||--o{ ISSUED_CERTIFICATE : "render template (Link)"
    USER ||--o{ ISSUED_CERTIFICATE : "member"
    CERTIFICATE_EVALUATION {
        Link member "User"
        Link subject "Eval Subject"
        Link evaluator "User"
        Link request "Certificate Request"
        Select status "inline: Pending|In Progress|Pass|Fail"
        Rating rating "reqd if Pass"
    }
    ISSUED_CERTIFICATE {
        Data certificate_number "unique"
        Date issue_date
        Date expiry_date
        Link template "Print Format"
    }
```
(Status is a hardcoded `Select` here because the four states are fixed and admin-extension was not requested; if admins needed to add states it would become a `Link` to an `Eval Status` master. The evaluator edges originate from `USER` because the spec Links the evaluator to `User`. The `CERTIFICATE_REQUEST → CERTIFICATE_EVALUATION` edge is backed by the `request` Link field on `Certificate Evaluation`.)

### Per-DocType specs
(Each table is a spec to hand to `frappe-doctype-builder`, not final JSON.)

> **`Eval Subject`** — *master* · *naming:* `field:subject_name` · what is being certified.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `subject_name` | Data | — | yes | natural key / title |
| `passing_percentage` | Float | — | yes | pass threshold |
| `criteria` | Table | `Subject Criteria` | yes | the rubric for this subject |
| `enable_certification` | Check | — | no | feature flag (LMS pattern) |

> **`Eval Criteria`** — *master* · *naming:* `field:criteria_name` · reusable rubric line item.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `criteria_name` | Data | — | yes | reusable across subjects |
| `description` | Text | — | no | guidance for evaluators |

> **`Subject Criteria`** — *child table* (`istable=1`) · *naming:* `hash` · joins a criterion into a subject with weight.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `criteria` | Link | `Eval Criteria` | yes | which rubric item |
| `criteria_name` | Data | — | no | `fetch_from: criteria.criteria_name` (single hop, read-only) |
| `max_marks` | Float | — | yes | per-subject weight |

> **`Eval Evaluator`** — *master* · *naming:* `field:evaluator` · evaluator profile + availability.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `evaluator` | Link | `User` | yes | unique; the evaluator |
| `full_name` | Data | — | no | `fetch_from: evaluator.full_name` (single hop) |
| `schedule` | Table | `Evaluator Schedule` | no | recurring weekly availability |
| `unavailable_from` | Date | — | no | vacation window start |
| `unavailable_to` | Date | — | no | vacation window end |

> **`Evaluator Schedule`** — *child table* · *naming:* `hash` · recurring weekly slot.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `day` | Select | `Sun\nMon\nTue\nWed\nThu\nFri\nSat` | yes | day of week (hardcoded inline list) |
| `start_time` | Time | — | yes | slot start |
| `end_time` | Time | — | yes | slot end |

> **`Certificate Request`** — *transaction* · *naming:* `REQ-.YYYY.-` (naming series) · booked appointment.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `member` | Link | `User` | yes | candidate |
| `subject` | Link | `Eval Subject` | yes | what's being evaluated |
| `evaluator` | Link | `User` | no | assigned evaluator |
| `date` | Date | — | yes | appointment date |
| `start_time` | Time | — | yes | slot start |
| `end_time` | Time | — | yes | slot end |
| `meet_link` | Data | — | no | video link |
| `status` | Select | `Upcoming\nCompleted\nCancelled` | yes | appointment lifecycle (inline) |

> **`Certificate Evaluation`** — *transaction* · *naming:* `EVAL-.YYYY.-` (naming series) · the scored session.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `member` | Link | `User` | yes | candidate |
| `subject` | Link | `Eval Subject` | yes | what's evaluated |
| `evaluator` | Link | `User` | no | who scored |
| `request` | Link | `Certificate Request` | no | the booking that scheduled this session (backs the "schedules" edge) |
| `scores` | Table | `Evaluation Score` | no | per-criterion marks |
| `total_score` | Float | — | no | computed, read-only |
| `status` | Select | `Pending\nIn Progress\nPass\nFail` | yes | lifecycle (no docstatus; hardcoded inline) |
| `rating` | Rating | — | cond. | `mandatory_depends_on eval:doc.status=='Pass'` |
| `summary` | Text | — | cond. | `mandatory_depends_on eval:doc.status not in ('Pending','In Progress')` |

> **`Evaluation Score`** — *child table* · *naming:* `hash` · marks per criterion.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `criteria` | Link | `Eval Criteria` | yes | which rubric item |
| `max_marks` | Float | — | no | `fetch_from: criteria.max_marks`-style single hop (or set from the subject rubric in code) |
| `marks_obtained` | Float | — | yes | evaluator's score |

> **`Issued Certificate`** — *transaction* · *naming:* `CERT-.YYYY.-` (naming series) · the credential.

| fieldname | fieldtype | options/target | reqd | why |
|---|---|---|---|---|
| `certificate_number` | Data | — | yes | **unique** public ID |
| `member` | Link | `User` | yes | who earned it |
| `subject` | Link | `Eval Subject` | yes | what was certified |
| `evaluation` | Link | `Certificate Evaluation` | yes | source of issuance |
| `issue_date` | Date | — | yes | issued on |
| `expiry_date` | Date | — | no | validity end |
| `template` | Link | `Print Format` | yes | render template (LMS pattern) |

### Relationship map (prose)
- `Certificate Evaluation` **references** `Eval Subject` (via `subject` Link) and `User` (via `member` + `evaluator` Links) — fixed targets, so plain Links.
- `Certificate Evaluation.request` is a Link → `Certificate Request`; this field backs the "schedules" edge, so the evaluation knows which booking produced it.
- `Evaluation Score` is a **child table** of `Certificate Evaluation` (no life outside its session); it Links to `Eval Criteria` (reusable) and single-hop `fetch_from`s `max_marks`.
- `Subject Criteria` is a **child table** of `Eval Subject` joining the reusable `Eval Criteria` master — the M:N "subjects share criteria" relationship handled via a join child table carrying `max_marks` (LMS `LMS Quiz Question` pattern).
- `Eval Evaluator` owns `Evaluator Schedule` (child table) for recurring availability; the actual booking is the separate `Certificate Request` transaction (the role-config-vs-booking split).
- `Issued Certificate` Links back to its `Certificate Evaluation` (via `evaluation`) and to a `Print Format` (via `template`); issued only when the evaluation hits `Pass`.

### Open questions / assumptions
- **Assumed** no `is_submittable` (status-field lifecycle). Flip to submittable only if certificates need legal immutability / cancellation reversal.
- **Assumed** the evaluator is referenced by `Link → User` (matches the spec and the diagram). Switch *both* spec rows and diagram edges to `Link → Eval Evaluator` if you need profile fields on the session.
- **Assumed** four hardcoded statuses (`Select`). If admins must add states, promote `status` to a `Link → Eval Status` master.
- **Open:** Should `certificate_number` be a naming series (auto) or a manually-entered registry ID? Should expired certificates auto-flag via a scheduled job?

### Next step
> Want me to generate these as real DocType JSON? I'll hand each spec to **frappe-doctype-builder** in dependency order (masters → child tables → transactions). Since this design is **not** submittable, the issuance lifecycle is status-based: I can pass the `Certificate Evaluation` Pending→Pass→issue transition to **frappe-state-machine-helper** for `validate_state_transition` / `on_submit`-style controller code (no `on_cancel` unless you decide to make it submittable). For broader app/module structure, the **frappe-architect** agent can take over.

---

## Best Practices

1. **Interview before you architect.** Never emit a model from a one-line request without grilling at least Stages 1–4 across multiple turns. A lead dev surfaces the lifecycle and cardinality first. (This is a hard gate, encoded in Operating rules 0 and 7 — not just advice.)
2. **Reconnoiter before you model.** Scan the app (and installed apps) for an existing doctype before proposing a new one; default **reuse > extend > create**. Reuse `User`/`Contact`/`Print Format` rather than re-modeling them; extend via own-app fields or **Custom Field fixtures** (other-app) before forking. (Operating rule 9; surfaces as deliverable item **b**.)
3. **Always carry a default.** Every question proposes the production-grade answer so the user can rubber-stamp it. Cite the app the default comes from.
4. **Restate the FULL running model every turn** so the user sees it converge.
5. **Push back on weak answers.** Challenge non-scaling choices with the tradeoff before landing a better default — that is the grilling.
6. **Prefer the studied patterns over invention.** Dynamic Link for polymorphism, join doctype for M:N, status master (a `Link`) for extensible statuses, Single for config, Log for external sync.
7. **Reserve docstatus for true submit semantics.** Most features want a status field, not `is_submittable`.
8. **Child table vs separate doctype is the most common mistake** — apply the lifecycle/independence rule explicitly and say why.
9. **Use single-hop `fetch_from` for read-only denormalized names/scopes**, never copy-paste data and never write two-dot grandparent expressions (they silently never populate).
10. **A master-driven status is a `Link`, never a `Select`** — `Select` is only for hardcoded inline option lists.
11. **Name everything with the app prefix** and pick a deliberate naming strategy per doctype.
12. **Stop at the plan.** This skill never writes files — it produces the reuse plan + diagram + specs and hands off.
13. **Escalate genuinely hard, system-wide questions** to the frappe-architect agent rather than guessing.

## References

**Production Frappe apps studied (mine these for real names + patterns):**
- Frappe CRM — https://github.com/frappe/crm (Dynamic Link activities, status masters, SLA child tables, Territory tree)
- Frappe Helpdesk — https://github.com/frappe/helpdesk (status-as-master, per-priority SLA, team rosters, escalation rules)
- Frappe Gameplan — https://github.com/frappe/gameplan (team→project→discussion containment, per-level fetched scope, orthogonal boolean states, autoincrement on several doctypes)
- Frappe Raven — https://github.com/The-Commit-Company/raven (membership join doctypes like `Raven Channel Member`, document-linked notifications via Dynamic Link, dedup constraints)
- Frappe LMS — https://github.com/frappe/lms (**backbone of the certificate example**: evaluator + schedule + evaluation + certificate + reusable questions)
- India Compliance — https://github.com/resilient-tech/india-compliance (immutable Log doctypes, async child tables, Single settings with config children)
- ERPNext — https://github.com/frappe/erpnext (master/transaction/child/tree taxonomy, docstatus lifecycle, naming series, Dynamic Link party)

**Sibling components to hand off to:**
- `frappe-doctype-builder` skill — generates the actual DocType JSON from this plan. Exact `autoname`/`options`/field-type JSON lives there; this skill picks the *strategy* and frames each spec table as input to the builder, not as final JSON.
- `frappe-state-machine-helper` skill — generates status-transition controller code (`validate_state_transition`, `on_submit`; `on_cancel` only if the design is flipped to `is_submittable`).
- `frappe-architect` agent — deep, system-wide architecture (app boundaries, scaling, migration strategy).

Remember: this skill is model-invoked and **plans only**. It interviews, diagrams, and specs — then offers the build handoff.
