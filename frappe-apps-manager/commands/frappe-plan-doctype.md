---
description: Plan a Frappe DocType architecture for a feature via a guided lead-dev interview
argument-hint: <feature to design, e.g. "evaluation system for issuing certificates">
---

# Frappe Plan DocType Command

Plan the DocType architecture for a feature exactly like a seasoned Frappe lead developer — interview the user, map the entities, propose DocType names and links, and lay out a flowchart of how everything connects. This command is a thin wrapper that defers to the **frappe-doctype-architect** skill.

The feature to design is provided in `$ARGUMENTS`.

Steps:

1. **Get the feature.** If `$ARGUMENTS` is non-empty, treat it as the feature description (e.g., "an evaluation system for issuing certificates"). If it is empty, ask the user one question: *"What feature do you want to design the DocType architecture for?"* and wait for the answer.

2. **ALWAYS open with the interview — never the deliverable.** No matter how detailed `$ARGUMENTS` is, your first response is **Stage-1 questions with proposed defaults**, not the diagram or spec tables. A rich prompt is a starting point to interrogate, not a finished spec. Then run the **frappe-doctype-architect** staged Interview Engine end to end:
   - **Reconnaissance (read-only, before drawing anything):** scan the target app (and installed apps) for existing doctypes and decide **reuse > extend > create** for each entity — reuse `User`/`Contact`/`Print Format` rather than re-modeling; extend an existing doctype via own-app fields or Custom Field fixtures (other-app) before creating a new one. Locate doctypes at `apps/<app>/**/doctype/*/*.json`; check `hooks.py` fixtures for existing custom fields. Surface the verdict as deliverable item (b).
   - Stage 1 — Domain & actors (roles, the core "thing", the driving lifecycle).
   - Stage 2 — Entities & cardinality (nouns → master vs transaction; 1:N vs M:N; persist vs compute).
   - Stage 3 — Lifecycle & status (docstatus vs status field vs Workflow; states + legal transitions + triggers; audit trail).
   - Stage 4 — Relationships (Link vs Dynamic Link; child table vs separate doctype; M:N → join doctype; trees; single-hop `fetch_from`).
   - Stage 5 — Fields & data (required/computed, `fetch_from`, naming strategy, uniqueness, conditional fields).
   - Stage 6 — Permissions & visibility (roles, owner-based, User Permissions, sharing).
   - Stage 7 — Integrations & automation (Log doctypes, notifications, scheduled jobs, Single settings).
   - Obey the operating rules: a stage is NOT one turn — pick only the **2–4 highest-leverage questions per turn**, propose a sensible production-grade default with each (cite the real app it comes from), push back on weak/non-scaling answers, restate the FULL running model after every turn, and fill obvious gaps yourself. Do **not** emit the deliverable until the user has answered questions covering Stages 1–4 across at least 2–3 turns and has explicitly confirmed the running model at least once ("decidable" = user-confirmed, not self-assessed).

3. **Produce output items (a)–(f) from the skill's Output Format**, in that exact order: (a) restated understanding, (b) the **reuse & extension plan** (per entity: reuse `X` / extend `X` (add …) / create new, with the reason, and the mechanism for each *extend* — own-app field vs Custom Field/Property Setter fixture), (c) a Mermaid `erDiagram` flowchart of every doctype and its links (mark reused vs new nodes; every edge backed by a real field; Link targets must match the spec rows; a master-driven status is a `Link`, a hardcoded set is a `Select`), (d) a per-DocType spec table framed as input to the builder (kind, naming strategy, key fields with fieldname | fieldtype | options/target | reqd | why; plus one table per *extended* doctype listing only the added fields + mechanism), (e) the relationship map in prose, (f) open questions / assumptions made.

4. **Then deliver item (g), the handoff offer — and do NOT write any files.** Ask whether to:
   - generate the actual DocType JSON via the **frappe-doctype-builder** skill (doctype by doctype, masters → child tables → transactions);
   - generate status-transition controller code via the **frappe-state-machine-helper** skill (`validate_state_transition` / `on_submit`; `on_cancel` only if the design is made submittable);
   - escalate system-wide architecture to the **frappe-architect** agent.

This command only plans and diagrams. File generation happens only after the user accepts the builder handoff.
