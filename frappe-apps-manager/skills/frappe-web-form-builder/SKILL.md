---
name: frappe-web-form-builder
description: Generate Frappe Web Forms for public-facing forms. Use when creating customer portals, registration forms, surveys, or public data collection forms.
---

# Frappe Web Form Builder

Generate public-facing web forms with validation, file uploads, and integration with Frappe DocTypes.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to create public forms
- User needs customer-facing forms
- User mentions web forms, surveys, or registration
- User wants portal functionality
- User needs forms without login

## Access & Security

- **Guest vs authenticated:** set `"login_required": 1` to require a logged-in user (the submission is tied to `frappe.session.user`); set `"login_required": 0` to allow anonymous/Guest submissions. Use anonymous access only for genuinely public intake (registration, surveys, contact).
- **DocType permissions still apply.** A Web Form is a thin layer over its `doc_type`. The underlying DocType's permission rules govern who can create/read/edit records — exposing a form does not bypass them. For Guest submissions, the Guest role needs create permission on the DocType (commonly via `has_web_view`/web permissions), so review those before publishing.
- **Never expose sensitive fields** (internal status, approval flags, pricing, owner/role fields) in `web_form_fields` on a public form. Only list the fields the submitter should fill.
- **Validate on the server.** Web Form client scripts (and `reqd`/`depends_on` in the JSON) are **UX only** and can be bypassed. Authoritative validation must live in the target DocType's controller `validate()` method.
- **Success handling:** `success_message` shows inline confirmation text after submit; `success_url` redirects to another route instead (e.g. `/thank-you`). Set one or the other to control the post-submit experience.

## Capabilities

### 1. Web Form JSON

**Customer Registration Form:**
```json
{
  "name": "Customer Registration",
  "route": "customer-registration",
  "title": "Register as Customer",
  "doc_type": "Customer",
  "is_standard": 0,
  "login_required": 0,
  "allow_multiple": 1,
  "show_sidebar": 0,
  "success_url": "/thank-you",
  "success_message": "Registration successful! We'll contact you soon.",
  "introduction_text": "Please fill in your details to register",
  "web_form_fields": [
    {
      "fieldname": "customer_name",
      "label": "Full Name",
      "fieldtype": "Data",
      "reqd": 1
    },
    {
      "fieldname": "email_id",
      "label": "Email",
      "fieldtype": "Data",
      "options": "Email",
      "reqd": 1
    },
    {
      "fieldname": "mobile_no",
      "label": "Phone",
      "fieldtype": "Data",
      "reqd": 1
    },
    {
      "fieldname": "customer_group",
      "label": "Type",
      "fieldtype": "Select",
      "options": "Individual\nCompany",
      "reqd": 1
    },
    {
      "fieldname": "company_name",
      "label": "Company Name",
      "fieldtype": "Data",
      "depends_on": "eval:doc.customer_group=='Company'"
    }
  ]
}
```

### 2. Multi-Step Form

**Multi-Page Web Form:**
```json
{
  "name": "Job Application",
  "is_multi_step": 1,
  "web_form_fields": [
    {
      "fieldname": "step_1",
      "fieldtype": "Section Break",
      "label": "Personal Information"
    },
    {"fieldname": "full_name", "fieldtype": "Data", "reqd": 1},
    {"fieldname": "email", "fieldtype": "Data", "options": "Email", "reqd": 1},

    {
      "fieldname": "step_2",
      "fieldtype": "Section Break",
      "label": "Experience"
    },
    {"fieldname": "years_experience", "fieldtype": "Int"},
    {"fieldname": "current_company", "fieldtype": "Data"},

    {
      "fieldname": "step_3",
      "fieldtype": "Section Break",
      "label": "Documents"
    },
    {"fieldname": "resume", "fieldtype": "Attach", "reqd": 1}
  ]
}
```

### 3. Routing

A Web Form is served at its own `route` (e.g. `/customer-registration`). If you need a custom public URL that maps to a portal page or DocType instead of the form's default route, add a rule to `website_route_rules` in the app's `hooks.py`:

```python
# apps/<app>/<app>/hooks.py
website_route_rules = [
    {"from_route": "/register", "to_route": "customer-registration"},
]
```

Keep the Web Form `route` and any `website_route_rules` entries consistent so a single public path resolves predictably.

## References

**Web Form Implementation:**
- Web Form DocType: https://github.com/frappe/frappe/tree/develop/frappe/website/doctype/web_form
- Web Form Controller: https://github.com/frappe/frappe/blob/develop/frappe/website/doctype/web_form/web_form.py
