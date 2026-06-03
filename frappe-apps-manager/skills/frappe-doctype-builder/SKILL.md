---
name: frappe-doctype-builder
description: Build Frappe DocTypes with fields, permissions, and naming configurations. Use this skill when creating or modifying DocType structures.
---

# Frappe DocType Builder Skill

Build complete DocType definitions with proper field types, permissions, and configurations.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

> Upstream data-model planning (which DocTypes to create, how they relate, reuse vs extend) is done by the `frappe-doctype-architect` skill; this skill emits the JSON from that plan.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to create a new DocType
- User needs to add fields to an existing DocType
- User asks about DocType structure or design
- User wants to modify DocType properties
- User needs help with DocType JSON schema

## Capabilities

### 1. DocType JSON Generation
Create complete DocType JSON files with:
- Metadata (name, module, naming, permissions)
- Fields with proper types and options
- Permissions for different roles
- Form layout and sections
- Naming series configuration

### 2. Field Type Expertise
Support all Frappe field types:
- **Data**: Short text fields
- **Text**: Long text with editor options
- **Int**: Integer numbers
- **Float**: Decimal numbers
- **Currency**: Money values
- **Date**: Date picker
- **Datetime**: Date and time
- **Time**: Time picker
- **Link**: Reference to another DocType
- **Select**: Dropdown with options
- **Check**: Boolean checkbox
- **Table**: Child table
- **Attach**: File upload
- **Attach Image**: Image upload with preview
- **Signature**: Signature capture
- **HTML**: Custom HTML content
- **Markdown Editor**: Markdown content
- **Code**: Code editor with syntax highlighting
- **Dynamic Link**: Polymorphic references
- **Rating**: Star rating
- **Color**: Color picker
- **Geolocation**: GPS coordinates

### 3. DocType Patterns

**Master DocType:**
```json
{
  "name": "Customer",
  "module": "CRM",
  "autoname": "naming_series:",
  "naming_rule": "By naming series",
  "track_changes": 1,
  "is_submittable": 0
}
```

**Transaction DocType:**
```json
{
  "name": "Sales Order",
  "module": "Selling",
  "is_submittable": 1,
  "autoname": "naming_series:",
  "track_changes": 1
}
```
A submittable DocType (`is_submittable: 1`) automatically gets a `docstatus` field — `0` Draft, `1` Submitted, `2` Cancelled. Do not declare `docstatus` as a field.

**Child Table:**
```json
{
  "name": "Sales Order Item",
  "module": "Selling",
  "istable": 1,
  "editable_grid": 1
}
```
A child DocType needs `"istable": 1` and an empty `__init__.py` in its folder; it carries no `permissions` block (it inherits the parent's).

**Settings DocType:**
```json
{
  "name": "System Settings",
  "module": "Core",
  "issingle": 1
}
```

### 4. Common Field Patterns

**Naming Series field** (pairs with `autoname: "naming_series:"`):
```json
{
  "fieldname": "naming_series",
  "fieldtype": "Select",
  "label": "Naming Series",
  "options": "CUST-.YYYY.-\nCUST-",
  "reqd": 1
}
```

### Naming strategies

Pick exactly one. Do not conflate naming series with an expression — they are distinct mechanisms.

- **Naming series** — `autoname: "naming_series:"` plus a `naming_series` Select field whose options use the dotted series syntax (`CUST-.YYYY.-`). User-configurable per document.
- **Field (natural key)** — `autoname: "field:fieldname"`. The document name is taken from that field's value (e.g. `field:email`).
- **Hash (random)** — `autoname: "hash"`. Use for child/join rows that are never referenced by a readable name.
- **Expression** — `autoname: "format:EXP-{####}"` with `naming_rule: "Expression"`. A fixed format string (no Select field involved).
- **Autoincrement** — `autoname: "autoincrement"`. Integer primary key.
- **Prompt** — `autoname: "prompt"`. User types the name manually.

**Status Field:**
```json
{
  "fieldname": "status",
  "fieldtype": "Select",
  "label": "Status",
  "options": "Draft\nSubmitted\nCancelled",
  "default": "Draft"
}
```

**Link Field:**
```json
{
  "fieldname": "customer",
  "fieldtype": "Link",
  "label": "Customer",
  "options": "Customer",
  "reqd": 1
}
```

**Child Table:**
```json
{
  "fieldname": "items",
  "fieldtype": "Table",
  "label": "Items",
  "options": "Sales Order Item",
  "reqd": 1
}
```

**Computed Field:**
```json
{
  "fieldname": "total",
  "fieldtype": "Currency",
  "label": "Total Amount",
  "read_only": 1
}
```

**Fetch From (denormalized copy):**
```json
{
  "fieldname": "customer_name",
  "fieldtype": "Data",
  "label": "Customer Name",
  "fetch_from": "customer.customer_name",
  "read_only": 1
}
```

### Link vs Dynamic Link

- **Link** targets one known DocType (`"options": "Customer"`). Use when the related DocType is fixed.
- **Dynamic Link** is polymorphic: a pair of fields — a `reference_doctype` (`fieldtype: Link`, `options: DocType`) that names the target DocType, plus a `reference_name` (`fieldtype: Dynamic Link`, `options: reference_doctype`) that holds the record name. Use when a row can point at different DocTypes.

### Child table vs separate DocType

- **Child table** (`"istable": 1`, no `permissions` block — it inherits the parent's) when rows are owned by exactly one parent and never queried on their own. The parent holds a `Table` field pointing to it.
- **Separate DocType** with a `Link` field back to the owner when rows have their own lifecycle, permissions, or need to be queried independently.

### `fetch_from`

`fetch_from` copies a value from a linked record into a read-only field. Single hop only: `<link_field_on_this_doctype>.<field_on_linked_doctype>` (e.g. `customer.customer_name`). Never a two-dot grandparent expression. It is a denormalized cached copy, not a live join.

### 5. Permission Configuration

```json
{
  "permissions": [
    {
      "role": "Sales User",
      "read": 1,
      "write": 1,
      "create": 1,
      "delete": 0,
      "submit": 0,
      "cancel": 0
    },
    {
      "role": "Sales Manager",
      "read": 1,
      "write": 1,
      "create": 1,
      "delete": 1,
      "submit": 1,
      "cancel": 1
    }
  ]
}
```

### 6. Advanced Features

**Dependent Fields:**
```json
{
  "fieldname": "customer_group",
  "fieldtype": "Link",
  "options": "Customer Group",
  "depends_on": "eval:doc.customer"
}
```

**Mandatory Depends On:**
```json
{
  "fieldname": "tax_id",
  "fieldtype": "Data",
  "label": "Tax ID",
  "mandatory_depends_on": "eval:doc.country=='United States'"
}
```

**Read Only Depends On:**
```json
{
  "fieldname": "posted_date",
  "fieldtype": "Date",
  "read_only_depends_on": "eval:doc.docstatus==1"
}
```

## Output Format

When building a DocType, provide:
1. Complete JSON structure
2. Explanation of key fields
3. Permission rationale
4. Controller method suggestions (if needed)
5. Migration instructions

## Best Practices

1. **Naming**: Use clear, descriptive field names in snake_case
2. **Required Fields**: Mark essential fields as required
3. **Defaults**: Provide sensible default values
4. **Permissions**: Start restrictive, expand as needed
5. **Indexing**: Add database indexes for frequently queried fields
6. **Validation**: Use field properties for basic validation
7. **Organization**: Group related fields with sections and column breaks

## Integration with Controllers

After creating DocType JSON, suggest controller methods:
- `validate()` - Pre-save validation
- `before_save()` - Modify values before saving
- `on_submit()` - Actions when document is submitted
- `on_cancel()` - Actions when document is cancelled
- `on_trash()` - Actions before deletion

## Example Usage Flow

1. **User asks**: "Create a Customer DocType with name, email, and phone"
2. **Skill generates**:
   - Complete DocType JSON
   - Appropriate field types
   - Basic permissions
   - Naming configuration
3. **Output includes**:
   - JSON file content
   - Where to save it (`apps/<app>/<app>/<module>/doctype/customer/customer.json`)
   - Migration command (`bench --site <site> migrate` — never `mkdir` the doctype folder)
   - Next steps for customization

## File Structure

Generated files should follow (the app name appears twice — directory + Python package):
```
apps/
└── <app>/
    └── <app>/
        └── <module>/
            └── doctype/
                └── <doctype_name>/
                    ├── __init__.py
                    ├── <doctype_name>.json
                    ├── <doctype_name>.py
                    └── <doctype_name>.js
```

Never `mkdir` the doctype folder by hand — write the JSON and run `bench --site <site> migrate`, which creates the structure.

Remember: This skill is model-invoked. Claude will use it autonomously when detecting DocType-related tasks.
