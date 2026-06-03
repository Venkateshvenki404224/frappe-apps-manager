---
name: frappe-performance-optimizer
description: Generate optimized queries, caching, and indexes for Frappe performance. Use when optimizing slow queries, implementing caching, or improving performance.
---

# Frappe Performance Optimizer

Generate performance-optimized code including efficient queries, caching strategies, and database indexes for Frappe applications.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User reports slow queries or performance issues
- User wants to add caching
- User needs database indexing
- User mentions performance, optimization, or slow queries
- User wants to eliminate N+1 queries

## Capabilities

### 1. Query Optimization

Prefer `frappe.qb.get_query()` over raw `frappe.db.sql()` for joins, aggregations, OR conditions, subqueries, and locking. It auto-joins linked/child fields via dot notation and stays composable. Reach for `frappe.db.sql()` only for things the builder can't express (CTEs, etc.).

**Optimized Report Query (query builder):**
```python
def get_sales_summary(from_date, to_date):
    query = frappe.qb.get_query(
        "Sales Invoice",
        fields=[
            "customer",
            "customer.customer_name as customer_name",  # auto-joins Customer
            "customer.customer_group as customer_group",
            {"COUNT": "name", "as": "invoice_count"},
            {"SUM": "grand_total", "as": "total_amount"},
        ],
        filters={
            "posting_date": ["between", [from_date, to_date]],
            "docstatus": 1,
        },
        group_by="customer",
        order_by="total_amount desc",
        limit=100,
    )
    return query.run(as_dict=True)

# Add a covering index for the hot filter columns
frappe.db.add_index('Sales Invoice', ['customer', 'posting_date', 'docstatus'])
```

Use `get_all` for server-side logic (ignores permissions) and `get_list` for user-facing data (enforces them). Under concurrency, lock the rows you're about to update with `for_update=True`:

```python
# Row locking to avoid lost updates when two jobs touch the same doc
query = frappe.qb.get_query("Stock Entry", filters={"name": name}, for_update=True)
row = query.run(as_dict=True)
```

### 2. Caching Implementation

`frappe.cache` is an attribute, NOT a callable — use `frappe.cache.get_value(...)` / `frappe.cache.set_value(...)`, never `frappe.cache()`. Keys are logical strings (`price:ITEM:LIST`); the wrapper auto-prefixes them with the site name, so never prefix the database/site name yourself.

**Cache Expensive Calculations:**
```python
def get_item_price(item_code, price_list, customer=None):
    """Get price with caching"""
    cache_key = f"price:{item_code}:{price_list}:{customer or 'default'}"

    # Try cache
    cached_price = frappe.cache.get_value(cache_key)
    if cached_price is not None:
        return cached_price

    # Calculate price (expensive)
    price = frappe.db.get_value('Item Price',
        filters={'item_code': item_code, 'price_list': price_list},
        fieldname='price_list_rate'
    )

    # Cache for 1 hour with an explicit TTL
    if price:
        frappe.cache.set_value(cache_key, price, expires_in_sec=3600)

    return price
```

**CRITICAL — pair every cached write with invalidation.** Clear the key from the relevant doc lifecycle hook so the cache can't go stale:

```python
class ItemPrice(Document):
    def on_update(self):
        self._clear_price_cache()

    def on_trash(self):
        self._clear_price_cache()

    def _clear_price_cache(self):
        # delete_value accepts a list to clear several keys at once
        frappe.cache.delete_value(f"price:{self.item_code}:{self.price_list}:default")
```

**What NOT to cache:** persistent source-of-truth data (keep it in the DB), and large blobs (Redis is not a blob store). Cache short-lived/ephemeral computed values with a TTL.

### 3. Batch Operations

**Bulk Update Pattern (query builder, no manual commit):**
```python
def bulk_update_items(updates):
    """Update multiple items in a single query"""
    # updates = [{'item_code': 'ITEM-001', 'is_active': 1}, ...]
    item_codes = [u['item_code'] for u in updates]

    Item = frappe.qb.DocType("Item")
    (
        frappe.qb.update(Item)
        .set(Item.is_active, 1)
        .where(Item.name.isin(item_codes))
    ).run()
    # No frappe.db.commit() — Frappe auto-commits on success.
    # `modified`/`modified_by` are managed by Frappe; don't set them here.
```

### 4. Eliminating N+1 Queries

Never query inside a loop. Collect the ids, run one `in` query, then map results back:

```python
# BAD — one query per row (N+1)
for exp in expenses:
    exp.category_label = frappe.db.get_value("Expense Category", exp.category, "label")

# GOOD — batch-fetch once, build a dict, map back
cat_ids = {e.category for e in expenses if e.category}
cat_map = {
    c.name: c.label
    for c in frappe.db.get_all(
        "Expense Category",
        filters={"name": ["in", list(cat_ids)]},
        fields=["name", "label"],
    )
}
for exp in expenses:
    exp.category_label = cat_map.get(exp.category)
```

### 5. Bulk Deletion

Use `frappe.db.delete` for a single DELETE query when the DocType has no `on_trash`/`after_delete` hooks. Only loop `frappe.delete_doc` when those controller trash hooks must fire.

```python
# Fast path — no trash hooks: one query
frappe.db.delete("Activity Log", {"creation": ["<", cutoff]})

# Slow path — only when on_trash/after_delete must run for each doc
for name in frappe.db.get_all("Expense", filters={"status": "Stale"}, pluck="name"):
    frappe.delete_doc("Expense", name)
```

## References

**Performance Examples:**
- Stock Ledger: https://github.com/frappe/erpnext/blob/develop/erpnext/stock/stock_ledger.py
- Get Item Details: https://github.com/frappe/erpnext/blob/develop/erpnext/stock/get_item_details.py
