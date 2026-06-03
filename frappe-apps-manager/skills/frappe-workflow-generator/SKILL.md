---
name: frappe-workflow-generator
description: Generate Frappe Workflows for document state management and approvals. Use when creating approval workflows, state transitions, or multi-step document processes.
---

# Frappe Workflow Generator

Generate production-ready Frappe Workflows for document state management, approvals, and business process automation.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to create approval workflows
- User needs document state transitions
- User requests multi-step approval processes
- User mentions workflows, approvals, or document states
- User wants role-based approval routing
- User needs to implement document lifecycle management

## When a Frappe Workflow Is Warranted

Use a Frappe Workflow only when transitions are **role-gated approvals** — each transition is granted to specific roles (multi-role sign-off, approve/reject routing). When you only need simple status tracking with no approval gating, a plain status field is lighter and easier to maintain — hand off to `frappe-state-machine-helper`.

For **submittable** documents, Workflow states map onto `docstatus`: `0` Draft, `1` Submitted, `2` Cancelled (set per state via `doc_status`). Each transition is gated by an `allowed` role. Do not declare `docstatus` as a field — Frappe manages it on submittable DocTypes.

## Capabilities

### 1. Workflow JSON Generation

**Basic Approval Workflow:**
```json
{
  "name": "Sales Invoice Approval",
  "document_type": "Sales Invoice",
  "is_active": 1,
  "workflow_state_field": "workflow_state",
  "states": [
    {
      "state": "Draft",
      "doc_status": "0",
      "allow_edit": "Sales User",
      "update_field": "status",
      "update_value": "Draft"
    },
    {
      "state": "Pending Approval",
      "doc_status": "0",
      "allow_edit": "Sales Manager",
      "update_field": "status",
      "update_value": "Pending"
    },
    {
      "state": "Approved",
      "doc_status": "1",
      "update_field": "status",
      "update_value": "Approved"
    },
    {
      "state": "Rejected",
      "doc_status": "2",
      "update_field": "status",
      "update_value": "Rejected"
    }
  ],
  "transitions": [
    {
      "state": "Draft",
      "action": "Submit for Approval",
      "next_state": "Pending Approval",
      "allowed": "Sales User",
      "allow_self_approval": 0
    },
    {
      "state": "Pending Approval",
      "action": "Approve",
      "next_state": "Approved",
      "allowed": "Sales Manager"
    },
    {
      "state": "Pending Approval",
      "action": "Reject",
      "next_state": "Rejected",
      "allowed": "Sales Manager"
    },
    {
      "state": "Rejected",
      "action": "Revise",
      "next_state": "Draft",
      "allowed": "Sales User"
    }
  ]
}
```

### 2. Conditional Transitions

**Workflow with Amount-Based Routing:**
```json
{
  "transitions": [
    {
      "state": "Draft",
      "action": "Submit",
      "next_state": "Pending L1 Approval",
      "allowed": "Sales User",
      "condition": "doc.grand_total <= 10000"
    },
    {
      "state": "Draft",
      "action": "Submit",
      "next_state": "Pending L2 Approval",
      "allowed": "Sales User",
      "condition": "doc.grand_total > 10000 && doc.grand_total <= 50000"
    },
    {
      "state": "Draft",
      "action": "Submit",
      "next_state": "Pending Director Approval",
      "allowed": "Sales User",
      "condition": "doc.grand_total > 50000"
    }
  ]
}
```

## References

**Frappe Workflow Core:**
- Workflow: https://github.com/frappe/frappe/blob/develop/frappe/workflow/doctype/workflow/workflow.py
- Workflow State: https://github.com/frappe/frappe/blob/develop/frappe/workflow/doctype/workflow_state/workflow_state.py

**Official Documentation:**
- Workflows: https://frappeframework.com/docs/user/en/desk/workflows
