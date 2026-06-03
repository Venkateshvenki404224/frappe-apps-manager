---
name: frappe-documentation-generator
description: Generate API documentation, user guides, and technical documentation for Frappe apps. Use when documenting APIs, creating user guides, or generating OpenAPI specs.
---

# Frappe Documentation Generator

Generate comprehensive documentation for Frappe applications including API documentation, user guides, and OpenAPI specifications.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to document APIs
- User needs user documentation
- User mentions documentation, API docs, or guides
- User wants OpenAPI/Swagger specs
- User needs to document DocTypes or workflows

## Capabilities

### 1. API Documentation

**Whitelisted Method Documentation:** document the type-hinted signature, the declared HTTP method(s), and the v2 endpoint URL with token auth.
```python
@frappe.whitelist(methods=["GET"])
def get_customer_details(customer: str):
    """
    Get detailed customer information

    Args:
        customer (str): Customer ID or name

    Returns:
        dict: Customer details including:
            - name: Customer ID
            - customer_name: Full name
            - email_id: Email address
            - mobile_no: Phone number
            - credit_limit: Credit limit amount
            - outstanding_amount: Current outstanding

    Raises:
        frappe.PermissionError: If user lacks read permission
        frappe.DoesNotExistError: If customer not found

    Example:
        >>> get_customer_details("CUST-001")
        {
            "name": "CUST-001",
            "customer_name": "John Doe",
            "email_id": "john@example.com",
            ...
        }

    Endpoint (module-level function):
        POST /api/v2/method/Customer/get_customer_details
        Authorization: token <api_key>:<api_secret>
        Content-Type: application/json
        { "customer": "CUST-001" }
    """
    if not frappe.has_permission('Customer', 'read'):
        frappe.throw(_('Not permitted'), frappe.PermissionError)

    customer_doc = frappe.get_doc('Customer', customer)

    return {
        'name': customer_doc.name,
        'customer_name': customer_doc.customer_name,
        'email_id': customer_doc.email_id,
        'mobile_no': customer_doc.mobile_no,
        'credit_limit': customer_doc.credit_limit,
        'outstanding_amount': customer_doc.get_outstanding()
    }
```

### 2. OpenAPI Specification

**Generate OpenAPI/Swagger:**
```yaml
openapi: 3.0.0
info:
  title: My Frappe App API
  version: 1.0.0
  description: API documentation for My Frappe App

servers:
  - url: https://example.com/api/v2
    description: Production server

components:
  securitySchemes:
    tokenAuth:
      type: apiKey
      in: header
      name: Authorization
      description: 'Use: token <api_key>:<api_secret>'

security:
  - tokenAuth: []

paths:
  /method/Customer/get_customer_details:
    get:
      summary: Get customer details
      description: Retrieve detailed information for a customer
      tags:
        - Customers
      parameters:
        - in: query
          name: customer
          required: true
          schema:
            type: string
          description: Customer ID
      responses:
        '200':
          description: Customer details
          content:
            application/json:
              schema:
                type: object
                properties:
                  name:
                    type: string
                  customer_name:
                    type: string
                  email_id:
                    type: string
        '403':
          description: Permission denied
        '404':
          description: Customer not found
```

### 3. User Guide Generation

**DocType User Guide:**
```markdown
# Customer Management Guide

## Overview
The Customer DocType stores information about your customers including contact details, credit limits, and transaction history.

## Creating a Customer

1. Go to **Selling > Customer**
2. Click **New Customer**
3. Fill in required fields:
   - Customer Name: Full name of the customer
   - Customer Group: Classification (Individual/Company)
   - Territory: Geographic location
4. Optional fields:
   - Email, Phone, Address
   - Credit Limit and Payment Terms
5. Click **Save**

## Key Features

### Credit Management
- Set credit limits to control customer purchases
- Monitor outstanding amounts
- Get alerts on credit limit breach

### Transaction History
View all customer transactions:
- Sales Invoices
- Payment Entries
- Delivery Notes

## Workflows

### Standard Flow
1. Create Customer
2. Create Sales Order
3. Create Sales Invoice
4. Receive Payment
5. Deliver Goods

## Tips
- Use customer groups for bulk operations
- Set default price lists per customer
- Configure payment terms for auto-fill
```

## API Documentation Conventions

When documenting Frappe APIs:

- Show **type-hinted** `@frappe.whitelist()` signatures with the declared `methods=[...]`.
- Use **v2 endpoint URLs**: doc-level methods at `POST /api/v2/document/<DocType>/<name>/method/<method>`, doctype-level functions at `POST /api/v2/method/<DocType>/<func>`, and built-in CRUD at `/api/v2/document/<DocType>` (GET list / POST create / GET-PUT-DELETE `/<name>/`).
- Document **token auth** via the header `Authorization: token <api_key>:<api_secret>`.

**Cross-references:**
- API surfaces to document come from the `frappe-api-handler` skill.
- The data model (DocTypes, fields, links) comes from the `frappe-doctype-architect` skill.

## References

**Frappe Documentation Patterns:**
- Frappe Docs: https://github.com/frappe/frappe/tree/develop/frappe/docs
- ERPNext Docs: https://github.com/frappe/erpnext/tree/develop/erpnext/docs
