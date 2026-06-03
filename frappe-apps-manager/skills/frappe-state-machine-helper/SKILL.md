---
name: frappe-state-machine-helper
description: Generate state machine logic for Frappe DocTypes. Use when implementing complex status workflows, state transitions, or document lifecycle management.
---

# Frappe State Machine Helper

Generate state machine logic for managing complex document states and transitions in Frappe DocTypes.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to implement document status management
- User needs state transition logic
- User mentions state machine, status workflow, or document lifecycle
- User wants to validate state transitions
- User needs state-dependent behavior

## Choosing a state mechanism

Pick the lightest mechanism that fits — most apps need only a status field, not `docstatus`.

- **Status field (Select or Link)** — for ordinary, non-submittable documents. Use a hardcoded `Select` (inline `\n` list) for a small fixed set of states; use a `Link` to a status master DocType when admins must extend the states, or when a status drives behavior and needs its own attributes.
- **`docstatus` (0 Draft / 1 Submitted / 2 Cancelled)** — only for genuine submit semantics, i.e. financial/legal immutability where a submitted document must not be edited (it can only be cancelled and amended). Set `is_submittable: 1`; do not declare `docstatus` as a field.
- **Frappe Workflow doctype** — for role-gated approval transitions / multi-role sign-off. Hand off to `frappe-workflow-generator`.

### Where transition logic goes

- Validate the transition in `validate` / `before_save` (reject illegal source→target moves).
- Run side effects (notifications, GL entries, etc.) in `on_update` / `on_submit`.
- **Never use `frappe.db.set_value()` to change a status** — it bypasses transition validation and lifecycle hooks. Mutate the doc and `doc.save()` so `validate`/`on_update` run.

## Capabilities

### 1. State Transition Logic

**Order Status State Machine:**
```python
class SalesOrder(Document):
    def validate(self):
        self.validate_state_transition()

    def validate_state_transition(self):
        """Validate allowed state transitions"""
        if not self.is_new():
            old_status = frappe.db.get_value('Sales Order', self.name, 'status')

            # Define allowed transitions
            allowed_transitions = {
                'Draft': ['Pending', 'Cancelled'],
                'Pending': ['Confirmed', 'Cancelled'],
                'Confirmed': ['In Progress', 'Cancelled'],
                'In Progress': ['Completed', 'On Hold'],
                'On Hold': ['In Progress', 'Cancelled'],
                'Completed': [],  # Terminal state
                'Cancelled': []   # Terminal state
            }

            if old_status != self.status:
                allowed = allowed_transitions.get(old_status, [])
                if self.status not in allowed:
                    frappe.throw(
                        _(f'Cannot transition from {old_status} to {self.status}')
                    )

    def on_submit(self):
        self.status = 'Confirmed'

    def on_cancel(self):
        self.status = 'Cancelled'
```

### 2. State-Dependent Actions

**Actions Based on State:**
```python
class PaymentEntry(Document):
    def validate(self):
        if self.status == 'Draft':
            self.validate_draft_entry()
        elif self.status == 'Submitted':
            self.validate_submitted_entry()

    def before_submit(self):
        # Actions before state change
        if self.payment_type == 'Pay':
            self.validate_sufficient_balance()

        self.status = 'Submitted'

    def on_cancel(self):
        # Reverse actions
        if self.status == 'Submitted':
            self.reverse_gl_entries()

        self.status = 'Cancelled'
```

## Related Skills

- For full role-gated approval workflows (multi-role sign-off), hand off to `frappe-workflow-generator`.
- The status-master and audit-log patterns are designed by `frappe-doctype-architect`; this skill implements the transition logic from that plan.

## References

**State Management Examples:**
- Sales Invoice: https://github.com/frappe/erpnext/blob/develop/erpnext/accounts/doctype/sales_invoice/sales_invoice.py
- Purchase Order: https://github.com/frappe/erpnext/blob/develop/erpnext/buying/doctype/purchase_order/purchase_order.py
