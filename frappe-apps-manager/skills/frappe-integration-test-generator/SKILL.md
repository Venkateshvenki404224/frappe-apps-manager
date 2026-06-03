---
name: frappe-integration-test-generator
description: Generate integration tests for multi-DocType workflows in Frappe. Use when testing end-to-end workflows, state transitions, or complex business processes.
---

# Frappe Integration Test Generator

Generate comprehensive integration tests for multi-DocType workflows and end-to-end business processes in Frappe.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to test complete workflows
- User needs end-to-end scenario testing
- User mentions integration tests or workflow testing
- User wants to test multi-DocType interactions
- User needs to verify business process integrity

## Test Site

Run tests on a **separate site** from the dev site — integration tests create, modify, and delete data across many DocTypes. If the dev site is `app.localhost`, create `app-test.localhost`, install the app there, and run tests against it:
```bash
bench new-site app-test.localhost --admin-password admin
bench --site app-test.localhost install-app <app>
bench --site app-test.localhost run-tests --app <app>
```
Always pass `--site` — never run a bare `bench run-tests`. If a test fails with "DocType not found", run `bench --site app-test.localhost migrate` first.

## Conventions

- Inherit from `frappe.tests.IntegrationTestCase` (multi-DocType workflows always touch the DB).
- Each test runs inside a transaction that **auto-rolls-back** — do not write manual teardown/cleanup or call `frappe.db.rollback()`.
- For permission-context tests, switch the active user with `frappe.set_user("user@example.com")` and restore it (e.g. `frappe.set_user("Administrator")`) at the end of the test.
- **Mock or avoid real external API calls** in tests — use `unittest.mock.patch` so tests stay deterministic and offline.
- Feature-wise tests spanning multiple DocTypes live at `apps/<app>/<app>/tests/test_<feature>.py`; a workflow test centred on one DocType can live alongside it at `apps/<app>/<app>/<module>/doctype/<doctype>/test_<doctype>.py`.

## Capabilities

### 1. Workflow Integration Test

**Complete Sales Workflow Test:**
```python
import frappe
from frappe.tests import IntegrationTestCase

class TestSalesWorkflow(IntegrationTestCase):
    def test_complete_sales_cycle(self):
        """Test end-to-end sales process"""
        # 1. Create Customer
        customer = self._create_test_customer()

        # 2. Create Sales Order
        so = self._create_sales_order(customer.name)
        so.submit()

        # 3. Create Sales Invoice from SO
        si = self._make_sales_invoice_from_order(so.name)
        si.insert()
        si.submit()

        # 4. Create Payment Entry
        pe = self._create_payment_entry(si)
        pe.insert()
        pe.submit()

        # Verify workflow completed
        si.reload()
        self.assertEqual(si.status, 'Paid')
        self.assertEqual(si.outstanding_amount, 0)

    def _create_test_customer(self):
        return frappe.get_doc({
            'doctype': 'Customer',
            'customer_name': '_Test Customer',
            'customer_group': 'Commercial'
        }).insert()

    def _create_sales_order(self, customer):
        return frappe.get_doc({
            'doctype': 'Sales Order',
            'customer': customer,
            'delivery_date': frappe.utils.add_days(frappe.utils.today(), 7),
            'items': [{
                'item_code': '_Test Item',
                'qty': 10,
                'rate': 100
            }]
        })
```

### 2. State Transition Test

**Test Document States:**
```python
class TestInvoiceStates(IntegrationTestCase):
    def test_invoice_state_transitions(self):
        """Test all possible state transitions"""
        si = self._get_test_invoice()

        # Draft state
        si.insert()
        self.assertEqual(si.docstatus, 0)
        self.assertEqual(si.status, 'Draft')

        # Submit transition
        si.submit()
        self.assertEqual(si.docstatus, 1)
        self.assertEqual(si.status, 'Submitted')

        # Cannot edit submitted
        si.customer = 'Different Customer'
        with self.assertRaises(frappe.ValidationError):
            si.save()

        # Cancel transition
        si.cancel()
        self.assertEqual(si.docstatus, 2)
        self.assertEqual(si.status, 'Cancelled')
```

### 3. Permission-Context Test

Switch the active user to assert a workflow behaves correctly per role, then restore the user. The auto-rollback transaction cleans up any documents created.
```python
class TestApprovalPermissions(IntegrationTestCase):
    def test_only_manager_can_approve(self):
        """A regular user cannot approve; a manager can"""
        expense = self._create_test_expense()

        # Regular user is denied
        frappe.set_user("user@example.com")
        with self.assertRaises(frappe.PermissionError):
            expense.approve()

        # Manager succeeds
        frappe.set_user("manager@example.com")
        expense.reload()
        expense.approve()
        self.assertEqual(expense.status, "Approved")

        # Restore the default test user
        frappe.set_user("Administrator")
```

### 4. Mocking External Calls

Never hit a real external service from a test. Patch the call so the test stays deterministic and offline.
```python
from unittest.mock import patch

class TestPaymentSync(IntegrationTestCase):
    @patch("my_app.gateway.charge_card")
    def test_payment_marks_invoice_paid(self, mock_charge):
        mock_charge.return_value = {"status": "succeeded"}
        si = self._create_test_invoice()
        si.collect_payment()
        self.assertEqual(si.status, "Paid")
        mock_charge.assert_called_once()
```

## References

**Integration Test Examples:**
- Sales Invoice Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/accounts/doctype/sales_invoice/test_sales_invoice.py
- Stock Entry Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/stock/doctype/stock_entry/test_stock_entry.py
