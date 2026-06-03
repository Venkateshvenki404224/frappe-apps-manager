---
name: frappe-fixture-creator
description: Generate fixture files for Frappe test data and master data. Use when creating test fixtures, setup data, or master data for new sites.
---

# Frappe Fixture Creator

Generate fixture JSON files for test data, master data, and initial site configuration in Frappe applications.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to create test fixtures
- User needs master data setup
- User mentions fixtures, test data, or setup data
- User wants repeatable site setup
- User needs demo or sample data

## Capabilities

### 1. Test Fixture Generation

**Item Fixtures:**
```json
[
  {
    "doctype": "Item",
    "item_code": "_Test Item",
    "item_name": "Test Item",
    "item_group": "Products",
    "stock_uom": "Nos",
    "is_stock_item": 1,
    "is_purchase_item": 1,
    "is_sales_item": 1,
    "opening_stock": 100,
    "valuation_rate": 100,
    "standard_rate": 150
  },
  {
    "doctype": "Item",
    "item_code": "_Test Service Item",
    "item_name": "Test Service",
    "item_group": "Services",
    "stock_uom": "Nos",
    "is_stock_item": 0,
    "is_sales_item": 1,
    "standard_rate": 500
  }
]
```

### 2. Hierarchical Fixtures

**Customer Group Tree:**
```json
[
  {
    "doctype": "Customer Group",
    "customer_group_name": "All Customer Groups",
    "is_group": 1
  },
  {
    "doctype": "Customer Group",
    "customer_group_name": "Commercial",
    "parent_customer_group": "All Customer Groups",
    "is_group": 0
  },
  {
    "doctype": "Customer Group",
    "customer_group_name": "Individual",
    "parent_customer_group": "All Customer Groups",
    "is_group": 0
  }
]
```

### 3. Import Fixture

**Load Fixture in App:**
```python
# In app setup
def before_install():
    """Install fixtures before site setup"""
    from frappe.core.page.data_import_tool.data_import_tool import import_doc

    import_doc('my_app/fixtures/item_groups.json')
    import_doc('my_app/fixtures/territories.json')
```

### 4. Hooks-based fixtures (preferred)

Declare a `fixtures` list in the app's `hooks.py` (`apps/<app>/<app>/hooks.py`). On `bench --site <site> export-fixtures --app <app-name>`, Frappe writes each entry's records to `apps/<app>/<app>/fixtures/`, and reimports them on `bench --site <site> migrate` / install.

```python
# hooks.py
fixtures = [
    "Item Group",
    {"dt": "Role", "filters": [["name", "in", ["Expense Manager", "Expense User"]]]},
]
```

Export:
```bash
bench --site <site> export-fixtures --app <app-name>
```

**CRITICAL — always filter `Custom Field` and `Property Setter` to your own records.** A bare `"Custom Field"` in `fixtures` exports EVERY customization on the site, including those owned by other apps and by the user. Filter to the fieldnames your app introduces:

```python
# hooks.py
fixtures = [
    {"dt": "Custom Field", "filters": [["fieldname", "in", ["my_field_1", "my_field_2"]]]},
    {"dt": "Property Setter", "filters": [["name", "in", ["Sales Invoice-my_field_1-hidden"]]]},
]
```

### 5. Extend a DocType you don't own

To add fields to a DocType owned by another app (e.g. ERPNext's `Sales Invoice`), **never fork its JSON.** Add a **Custom Field** (and a **Property Setter** to tweak existing properties) from an `after_install` hook:

```python
# hooks.py
after_install = "my_app.setup.after_install"
```

```python
# my_app/setup.py
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields

def after_install():
    create_custom_fields({
        "Sales Invoice": [
            {
                "fieldname": "my_field_1",
                "label": "My Field",
                "fieldtype": "Data",
                "insert_after": "customer",
            },
        ],
    })
```

Then export those Custom Fields as filtered fixtures (see section 4) so they ship with the app.

## Related Skills

- `frappe-doctype-architect` decides — in its reuse/extend plan — when a DocType should be extended with a Custom Field fixture (the section 5 pattern) versus created fresh. Implement fixtures from that plan.

## References

**Frappe Fixture Patterns:**
- Install Fixtures: https://github.com/frappe/frappe/blob/develop/frappe/core/page/data_import_tool/data_import_tool.py
- ERPNext Fixtures: https://github.com/frappe/erpnext/tree/develop/erpnext/setup/fixtures
