---
name: frappe-external-api-connector
description: Generate code to integrate Frappe with external REST APIs. Use when connecting to third-party services, payment gateways, or external data sources.
---

# Frappe External API Connector

Generate robust API client code for integrating Frappe with external REST APIs, handling authentication, error recovery, and data transformation.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to integrate external REST APIs
- User needs to call third-party services
- User mentions API integration, external system connection
- User wants to integrate payment gateways, shipping APIs, etc.
- User needs OAuth or API key authentication

## Capabilities

### 1. API Client Class

**REST API Client Template:**
```python
import requests
import frappe
from frappe import _

class ExternalAPIClient:
    def __init__(self):
        self.base_url = frappe.conf.get('external_api_url')
        self.api_key = frappe.conf.get('external_api_key')
        self.timeout = 30

    def get_headers(self):
        return {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json',
            'User-Agent': 'Frappe/1.0'
        }

    def get(self, endpoint, params=None):
        """GET request with error handling"""
        try:
            response = requests.get(
                f'{self.base_url}/{endpoint}',
                params=params,
                headers=self.get_headers(),
                timeout=self.timeout
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            frappe.throw(_('Request timeout'))
        except requests.exceptions.HTTPError as e:
            self._handle_http_error(e)
        except Exception as e:
            frappe.log_error(frappe.get_traceback(),
                'External API Error')
            frappe.throw(_('API request failed'))

    def post(self, endpoint, data):
        """POST request"""
        response = requests.post(
            f'{self.base_url}/{endpoint}',
            json=data,
            headers=self.get_headers(),
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()

    def _handle_http_error(self, error):
        """Handle HTTP errors"""
        status_code = error.response.status_code
        if status_code == 401:
            frappe.throw(_('API authentication failed'))
        elif status_code == 404:
            frappe.throw(_('Resource not found'))
        elif status_code == 429:
            frappe.throw(_('Rate limit exceeded'))
        else:
            frappe.throw(_(f'API error: {status_code}'))
```

### 2. OAuth Integration

**OAuth 2.0 Flow:**
```python
def get_oauth_token():
    """Get OAuth access token with refresh"""
    # Check cache first. Note: frappe.cache is an attribute, NOT frappe.cache().
    # Pass a logical key — the wrapper auto-prefixes the site name; don't prefix manually.
    token = frappe.cache.get_value('oauth_token:provider')
    if token:
        return token

    # Get from settings
    settings = frappe.get_single('OAuth Settings')

    response = requests.post(
        settings.token_url,
        data={
            'grant_type': 'client_credentials',
            'client_id': settings.client_id,
            'client_secret': settings.get_password('client_secret')
        },
        timeout=30
    )

    if response.status_code == 200:
        token_data = response.json()
        access_token = token_data['access_token']

        # Cache token, refreshing well before expiry (60s buffer)
        frappe.cache.set_value(
            'oauth_token:provider',
            access_token,
            expires_in_sec=token_data.get('expires_in', 3600) - 60
        )

        return access_token

    frappe.throw(_('OAuth authentication failed'))
```

### 3. Run Outbound Calls in the Background

Never block the web request on a slow or unreliable external call — enqueue it. Use the `long` queue with an explicit `timeout` for heavy work, and `enqueue_after_commit=True` when the job depends on data written in the current request:

```python
# apps/myapp/myapp/integrations/tasks.py
import frappe
import requests

def push_to_external(invoice_name: str):
    """Runs in a worker process — fetch data via args, not request state."""
    doc = frappe.get_doc("Sales Invoice", invoice_name)
    try:
        resp = requests.post(
            frappe.conf.get("external_api_url"),
            json=doc.as_dict(),
            timeout=30
        )
        resp.raise_for_status()
    except Exception:
        # Log full traceback with a context label; do NOT frappe.db.commit() here
        frappe.log_error(frappe.get_traceback(), "External push failed")
        raise
```

Enqueue from a controller hook:

```python
class SalesInvoice(Document):
    def on_submit(self):
        frappe.enqueue(
            "myapp.integrations.tasks.push_to_external",
            queue="long",
            timeout=600,
            enqueue_after_commit=True,   # job sees this request's writes
            invoice_name=self.name
        )
```

### 4. Caching, Logging, and Retries

- **Cache tokens/responses** with `frappe.cache.set_value(key, value, expires_in_sec=...)` and read with `frappe.cache.get_value(key)`. `frappe.cache` is an attribute (not `frappe.cache()`); it auto-prefixes the site name, so pass plain logical keys. Refresh tokens before they expire.
- **Log every failure** with `frappe.log_error(frappe.get_traceback(), "<context>")` so the full traceback is captured.
- Always set explicit request `timeout`s and add a **bounded** retry (a fixed small number of attempts), not an unbounded loop.
- **Never call `frappe.db.commit()`** after API calls in request context — Frappe auto-commits on success.

## References

**Frappe Integration Patterns:**
- Integrations: https://github.com/frappe/frappe/tree/develop/frappe/integrations
- ERPNext Integrations: https://github.com/frappe/erpnext/tree/develop/erpnext/erpnext_integrations
