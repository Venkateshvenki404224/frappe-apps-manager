---
name: frappe-webhook-manager
description: Create webhook handlers for Frappe integrations. Use when implementing webhooks, event-driven integrations, or external system notifications.
---

# Frappe Webhook Manager

Generate secure webhook receivers and senders for Frappe integrations with external systems.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to receive webhooks from external services
- User needs to send webhooks to external systems
- User mentions webhook, event-driven integration, or external notifications
- User wants to integrate payment gateways, APIs, or third-party services
- User needs to handle real-time events from external systems

## Capabilities

### 1. Webhook Receiver

**Secure Webhook Endpoint:** verify the signature BEFORE trusting any of the payload, and pin the verb to POST.
```python
import frappe
from frappe import _
import hmac
import hashlib

@frappe.whitelist(allow_guest=True, methods=["POST"])
def webhook_receiver():
    """Receive webhook from external service"""
    # Get signature
    signature = frappe.get_request_header('X-Webhook-Signature') or ''

    # Verify signature against the raw body FIRST — never trust the payload before this
    secret = frappe.conf.get('webhook_secret')
    expected = hmac.new(
        secret.encode(),
        frappe.request.data,
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        frappe.throw(_('Invalid signature'), frappe.AuthenticationError)

    # Only now is it safe to parse the payload
    payload = frappe.parse_json(frappe.request.data)

    # Process webhook (offload anything slow to a background job)
    event = payload.get('event')
    if event == 'payment.success':
        handle_payment_success(payload)
    elif event == 'customer.updated':
        handle_customer_update(payload)

    # Return only a status — never echo sensitive fields back
    return {'status': 'success'}
```

### 2. Outgoing Webhook

Trigger outbound webhooks from `doc_events` in `hooks.py` rather than inside the DocType controller, and **enqueue** delivery — never send inline inside the document event (a slow/failed receiver must not block or roll back the save).

**Wire the trigger in `hooks.py`** (handlers are dotted paths to importable functions):
```python
# apps/myapp/myapp/hooks.py
doc_events = {
    "Sales Invoice": {
        "on_submit": "myapp.webhooks.queue_invoice_webhook",
    }
}
```

**Enqueue delivery** (the handler receives `doc` and the event `method`):
```python
# apps/myapp/myapp/webhooks.py
import frappe

def queue_invoice_webhook(doc, method):
    """doc_events handler: enqueue, don't send inline."""
    frappe.enqueue(
        "myapp.webhooks.send_webhook",
        queue="short",
        timeout=120,
        job_id=f"webhook::{doc.name}::invoice.submitted",  # stable id
        deduplicate=True,                  # skip if an identical job is already queued
        enqueue_after_commit=True,         # only fire after the save commits
        event="invoice.submitted",
        data=doc.as_dict(),
    )
```

**Send with HMAC signature and a bounded retry:**
```python
# apps/myapp/myapp/webhooks.py
import frappe
import hmac
import hashlib
import json
import requests

def send_webhook(event, data):
    """Runs in a worker. No frappe.db.commit() — Frappe auto-commits on success."""
    webhook_url = frappe.conf.get("external_webhook_url")
    secret = frappe.conf.get("webhook_secret")

    payload = {"event": event, "data": data, "timestamp": frappe.utils.now()}
    body = json.dumps(payload)
    signature = hmac.new(secret.encode(), body.encode(), hashlib.sha256).hexdigest()

    headers = {"Content-Type": "application/json", "X-Webhook-Signature": signature}

    last_error = None
    for attempt in range(3):  # bounded retry
        try:
            resp = requests.post(webhook_url, data=body, headers=headers, timeout=10)
            if resp.status_code == 200:
                return True
            last_error = f"HTTP {resp.status_code}"
        except Exception:
            last_error = frappe.get_traceback()

    frappe.log_error(last_error, f"Webhook failed: {event} -> {webhook_url}")
    return False
```

Refer to the background-jobs reference for the full set of `frappe.enqueue` options (`job_id`, `deduplicate`, `enqueue_after_commit`, queues, timeouts).

## References

**Frappe Webhook Implementation:**
- Webhook DocType: https://github.com/frappe/frappe/tree/develop/frappe/integrations/doctype/webhook
- Integration Request: https://github.com/frappe/frappe/tree/develop/frappe/integrations/doctype/integration_request
