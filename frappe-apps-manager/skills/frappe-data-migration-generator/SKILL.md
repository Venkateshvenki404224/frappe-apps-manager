---
name: frappe-data-migration-generator
description: Generate data migration scripts for Frappe. Use when migrating data from legacy systems, transforming data structures, or importing large datasets.
---

# Frappe Data Migration Generator

Generate robust data migration scripts with validation, error handling, and progress tracking for importing data into Frappe.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to migrate data from legacy systems
- User needs to import large CSV/Excel files
- User mentions data migration, ETL, or data import
- User wants to transform data structures
- User needs bulk data operations

## Capabilities

### 1. CSV Import Script

Migrate through `doc.save()` so controller validation and lifecycle hooks run. Make the importer **idempotent** — guard each row with `frappe.db.exists(...)` (or a flag) so re-running doesn't create duplicates. Do **not** call `frappe.db.commit()` inside the loop: when this runs as a patch, a POST request, or a background job, Frappe auto-commits on success and auto-rolls-back on an uncaught error. Reserve `frappe.db.set_value()` for derived/cached/counter fields with no validation.

**Production-Ready CSV Importer:**
```python
import csv
import frappe
from frappe.utils import flt, cint, getdate

def import_customers_from_csv(file_path):
    """Import customers — validated, idempotent, no manual commit"""
    success = []
    errors = []

    with open(file_path, 'r', encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)

        for idx, row in enumerate(reader, start=2):
            try:
                # Validate required fields
                if not row.get('Customer Name'):
                    raise ValueError('Customer name required')

                # Transform data
                customer = {
                    'doctype': 'Customer',
                    'customer_name': row['Customer Name'].strip(),
                    'customer_group': row.get('Customer Group', 'Commercial'),
                    'territory': row.get('Territory', 'All Territories'),
                    'email_id': row.get('Email', '').strip(),
                    'mobile_no': row.get('Phone', '').strip(),
                    'credit_limit': flt(row.get('Credit Limit', 0))
                }

                # Idempotency guard — skip/update existing instead of duplicating
                existing = frappe.db.exists('Customer',
                    {'customer_name': customer['customer_name']})

                if existing:
                    doc = frappe.get_doc('Customer', existing)
                    doc.update(customer)
                    doc.save()  # runs validate() + lifecycle hooks
                else:
                    doc = frappe.get_doc(customer)
                    doc.insert()  # runs validate() + lifecycle hooks

                success.append(row['Customer Name'])

            except Exception as e:
                # Row failure is recorded; the surrounding transaction
                # auto-rolls-back only on an uncaught error.
                errors.append({'row': idx, 'data': row, 'error': str(e)})

    # No frappe.db.commit() — Frappe auto-commits on successful completion.
    return {'success': success, 'errors': errors}
```

### 2. Patch Migrations

Patches run inside a managed transaction: Frappe **auto-commits after a successful `execute()`** and auto-rolls-back on error. Don't sprinkle `frappe.db.commit()`. Prefer the ORM / query builder over raw SQL, and batch-fetch lookups to avoid N+1.

```python
import frappe

def execute():
    """Backfill customer_group — idempotent, ORM-based, no manual commit."""
    rows = frappe.db.get_all(
        "Customer",
        filters={"customer_group": ["in", ["", None]]},
        fields=["name", "territory"],
    )
    if not rows:
        return  # already migrated — safe to re-run

    # Batch-fetch the territory -> default group map (avoid querying in the loop)
    territory_ids = {r.territory for r in rows if r.territory}
    group_map = {
        t.name: t.default_customer_group
        for t in frappe.db.get_all(
            "Territory",
            filters={"name": ["in", list(territory_ids)]},
            fields=["name", "default_customer_group"],
        )
    }

    for row in rows:
        group = group_map.get(row.territory) or "Commercial"
        # Has validation/lifecycle → go through the doc so hooks run
        doc = frappe.get_doc("Customer", row.name)
        doc.customer_group = group
        doc.save()
    # auto-commit on successful execute(); no frappe.db.commit() here
```

For a derived/cached field with no validation, a single bulk UPDATE via the query builder is preferable to looping:

```python
# Bulk UPDATE via frappe.qb (preferred over raw frappe.db.sql("UPDATE ..."))
Customer = frappe.qb.DocType("Customer")
(
    frappe.qb.update(Customer)
    .set(Customer.customer_group, "Commercial")
    .where(Customer.customer_group.isin(["", None]))
).run()
```

> If you keep ANY manual commit, restrict it to a **deliberate checkpoint in a long standalone backfill** (run outside a request, where a mid-way crash would otherwise lose hours of work) and say why — the default is no manual commit.

### 3. Very Large Datasets

Stream rows instead of loading them all into memory: combine `frappe.db.unbuffered_cursor()` with `query.run(as_iterator=True)`.

```python
def backfill_large_table():
    query = frappe.qb.get_query(
        "Sales Invoice",
        fields=["name", "customer"],
        filters={"customer_group": ["in", ["", None]]},
    )
    with frappe.db.unbuffered_cursor():
        for row in query.run(as_iterator=True, as_dict=True):
            process(row)
```

## References

**Frappe Data Import:**
- Data Import: https://github.com/frappe/frappe/blob/develop/frappe/core/doctype/data_import/data_import.py
- CSV Utils: https://github.com/frappe/frappe/blob/develop/frappe/utils/csvutils.py
