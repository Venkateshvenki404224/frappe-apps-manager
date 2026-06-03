---
name: frappe-api-handler
description: Create custom API endpoints and whitelisted methods for Frappe applications. Use when building REST APIs or custom endpoints.
---

# Frappe API Handler Skill

Create secure, efficient custom API endpoints for Frappe applications.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to create custom API endpoints
- User needs to whitelist Python methods for API access
- User asks about REST API implementation
- User wants to integrate external systems with Frappe
- User needs help with API authentication or permissions

## Capabilities

### 1. Whitelisted Methods

Create Python methods accessible via API. Always type-hint parameters (Frappe validates and casts args by type hint, preventing type-confusion; without hints all args arrive as untrusted strings) and pin the HTTP verb with `methods=[...]`:

```python
import frappe
from frappe import _

@frappe.whitelist(methods=["GET"])
def get_customer_details(customer_name: str):
    """Get customer details with validation"""
    # Permission check (mirror the controller; keep checks on every call path)
    if not frappe.has_permission("Customer", "read"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)

    customer = frappe.get_doc("Customer", customer_name)

    return {
        "name": customer.name,
        "customer_name": customer.customer_name,
        "email": customer.email_id,
        "phone": customer.mobile_no,
        "outstanding_amount": customer.get_outstanding()
    }
```

### 2. API Method Patterns

**Public Methods (No Authentication):**
```python
@frappe.whitelist(allow_guest=True, methods=["GET"])
def public_api_method():
    """Accessible without login. Return only fields guests need — never leak
    user emails, internal IDs, or permission-sensitive data."""
    return {"message": "Public data"}
```

**Authenticated Methods:**
```python
@frappe.whitelist(methods=["GET"])
def authenticated_method():
    """Requires valid session or API key"""
    user = frappe.session.user
    return {"user": user}
```

**Permission-based Methods:**
```python
@frappe.whitelist(methods=["POST"])
def delete_customer(customer_name: str):
    """Check permissions before action"""
    if not frappe.has_permission("Customer", "delete"):
        frappe.throw(_("Not permitted"))

    frappe.delete_doc("Customer", customer_name)
    return {"message": "Customer deleted"}
```

### 3. REST API Endpoints

**GET Request Handler:** (use `get_list` for user-facing reads so DocType permissions are enforced)
```python
@frappe.whitelist(methods=["GET"])
def get_items(filters: dict | None = None, fields: list | None = None, limit: int = 20):
    """Get list of items with filters"""
    items = frappe.get_list(
        "Item",
        filters=filters or {},
        fields=fields or ["name"],
        limit=limit,
        order_by="creation desc"
    )

    return {"items": items}
```

**POST Request Handler:**
```python
@frappe.whitelist(methods=["POST"])
def create_sales_order(customer: str, items: list, delivery_date: str | None = None):
    """Create sales order from API. No frappe.db.commit() — Frappe commits POST on success."""
    doc = frappe.get_doc({
        "doctype": "Sales Order",
        "customer": customer,
        "delivery_date": delivery_date or frappe.utils.today(),
        "items": items
    })

    doc.insert()
    doc.submit()

    return {"name": doc.name, "grand_total": doc.grand_total}
```

**PUT/UPDATE Handler:**
```python
@frappe.whitelist(methods=["PUT"])
def update_customer(customer_name: str, data: dict):
    """Update customer details"""
    doc = frappe.get_doc("Customer", customer_name)
    doc.update(data)
    doc.save()

    return {"name": doc.name, "message": "Updated successfully"}
```

**DELETE Handler:**
```python
@frappe.whitelist(methods=["DELETE"])
def delete_document(doctype: str, name: str):
    """Delete a document"""
    if not frappe.has_permission(doctype, "delete"):
        frappe.throw(_("Not permitted"))

    frappe.delete_doc(doctype, name)
    return {"message": f"{doctype} {name} deleted"}
```

### 4. Error Handling

```python
@frappe.whitelist(methods=["POST"])
def safe_api_method(param: str):
    """API method with proper error handling"""
    try:
        # Validate input
        if not param:
            frappe.throw(_("Parameter is required"))

        # Process request
        result = process_data(param)

        return {"success": True, "data": result}

    except frappe.ValidationError as e:
        frappe.log_error(frappe.get_traceback(), "API Validation Error")
        return {"success": False, "message": str(e)}

    except Exception as e:
        frappe.log_error(frappe.get_traceback(), "API Error")
        return {"success": False, "message": "Internal server error"}
```

### 5. Input Validation

```python
@frappe.whitelist(methods=["POST"])
def validated_method(email: str, phone: str, amount: float):
    """Validate all inputs"""
    # Email validation
    if not frappe.utils.validate_email_address(email):
        frappe.throw(_("Invalid email address"))

    # Phone validation
    if not phone or len(phone) < 10:
        frappe.throw(_("Invalid phone number"))

    # Amount validation
    amount = frappe.utils.flt(amount)
    if amount <= 0:
        frappe.throw(_("Amount must be greater than zero"))

    return {"valid": True}
```

### 6. Pagination

```python
@frappe.whitelist(methods=["GET"])
def paginated_list(doctype: str, page: int = 1, page_size: int = 20, filters: dict | None = None):
    """Get paginated results"""
    filters = filters or {}

    # Get total count
    total = frappe.db.count(doctype, filters=filters)

    # Get data (get_list enforces DocType permissions for the calling user)
    data = frappe.get_list(
        doctype,
        filters=filters,
        fields=["name"],
        start=(page - 1) * page_size,
        page_length=page_size,
        order_by="creation desc"
    )

    return {
        "data": data,
        "total": total,
        "page": page,
        "page_size": page_size,
        "total_pages": (total + page_size - 1) // page_size
    }
```

### 7. File Upload Handling

```python
@frappe.whitelist(methods=["POST"])
def upload_file():
    """Handle file upload"""
    from frappe.utils.file_manager import save_file

    if not frappe.request.files:
        frappe.throw(_("No file uploaded"))

    file = frappe.request.files['file']

    # Save file
    file_doc = save_file(
        fname=file.filename,
        content=file.stream.read(),
        dt="Customer",  # DocType
        dn="CUST-001",  # Document name
        is_private=1
    )

    return {
        "file_url": file_doc.file_url,
        "file_name": file_doc.file_name
    }
```

### 8. Bulk Operations

```python
@frappe.whitelist(methods=["POST"])
def bulk_create(doctype: str, records: list):
    """Create multiple documents. For plain CRUD, prefer the built-in
    /api/v2/document/<DocType>/bulk_update endpoint instead of a custom method."""
    created = []
    errors = []

    for record in records:
        try:
            doc = frappe.get_doc(record)
            doc.insert()
            created.append(doc.name)
        except Exception as e:
            errors.append({
                "record": record,
                "error": str(e)
            })

    return {
        "created": created,
        "errors": errors,
        "success_count": len(created),
        "error_count": len(errors)
    }
```

### 9. API Response Formats

**Success Response:**
```python
return {
    "success": True,
    "data": result,
    "message": "Operation completed successfully"
}
```

**Error Response:**
```python
return {
    "success": False,
    "message": "Error message",
    "errors": validation_errors
}
```

**List Response:**
```python
return {
    "success": True,
    "data": items,
    "total": total_count,
    "page": current_page
}
```

### 10. Authentication Patterns

**API Key/Secret:**
```python
@frappe.whitelist(allow_guest=True, methods=["POST"])
def api_key_method():
    """Authenticate using API key"""
    api_key = frappe.get_request_header("Authorization")

    if not api_key:
        frappe.throw(_("API key required"))

    # Validate API key
    user = frappe.db.get_value("User", {"api_key": api_key}, "name")
    if not user:
        frappe.throw(_("Invalid API key"))

    frappe.set_user(user)

    # Process request
    return {"authenticated": True}
```

**Token-based:**
```python
@frappe.whitelist(allow_guest=True, methods=["POST"])
def token_auth():
    """JWT or custom token authentication"""
    token = frappe.get_request_header("Authorization", "").replace("Bearer ", "")

    if not token:
        frappe.throw(_("Token required"))

    # Validate token
    user_data = validate_token(token)
    frappe.set_user(user_data["email"])

    return {"authenticated": True}
```

## Built-in v2 REST API

Frappe v15+ auto-generates CRUD — don't write custom endpoints for plain create/read/update/delete:

```
GET    /api/v2/document/<DocType>                          # list (fields, filters, order_by, start, limit)
POST   /api/v2/document/<DocType>                          # create
GET    /api/v2/document/<DocType>/<name>/                  # read
PUT    /api/v2/document/<DocType>/<name>/                  # update
DELETE /api/v2/document/<DocType>/<name>/                  # delete
POST   /api/v2/document/<DocType>/<name>/method/<method>   # call a doc-level method
POST   /api/v2/method/<DocType>/<method>                   # call a doctype-level function
POST   /api/v2/document/<DocType>/bulk_update              # body: {"docs": [{"name": "...", ...}]}
POST   /api/v2/document/<DocType>/bulk_delete              # body: {"names": [...]}
```

List responses include a `has_next_page` boolean for pagination. Large bulk operations are auto-enqueued as background jobs. Only write a custom `@frappe.whitelist()` endpoint for logic that goes beyond CRUD.

## Doc-level methods vs DocType-level functions

- **Doc-level methods** are `@frappe.whitelist()` methods on the `Document` subclass (operate on `self`). Call them via `frm.call("approve")` in client JS, or `POST /api/v2/document/<DocType>/<name>/method/approve`.
- **DocType-level functions** are module-level `@frappe.whitelist()` functions in the controller file. Call them via the full dotted path (`method: "myapp.mymodule.doctype.expense.expense.get_summary"`) or `POST /api/v2/method/<DocType>/get_summary`. The legacy `POST /api/method/<dotted.path>` also works for module-level functions.

## API Endpoint URLs

Module-level whitelisted functions are accessible at:
```
/api/method/{app_name}.{module}.{file}.{method_name}
```

Example:
```
POST /api/method/my_app.api.customer.get_customer_details
Content-Type: application/json

{
  "customer_name": "CUST-001"
}
```

## Best Practices

1. **Always validate inputs** - Type-hint params; never trust user data
2. **Check permissions** - Inside controller methods (every call path), not API wrappers
3. **Handle errors gracefully** - Return user-friendly messages
4. **Log errors** - Use `frappe.log_error()` for debugging
5. **Never call `frappe.db.commit()` yourself** - Frappe auto-commits POST/PUT requests on success and rolls back on uncaught errors
6. **Rate limiting** - Consider implementing for public APIs
7. **Version your APIs** - Include version in URL or headers
8. **Document your APIs** - Provide clear documentation
9. **Pin the HTTP verb** - Pass `methods=[...]` on every endpoint
10. **Sanitize output** - Don't expose sensitive data

## Anti-patterns

- **Don't wrap a doc's whitelisted method in a standalone `api.py` function.** If the controller method is whitelisted, clients call it directly via `frm.call("approve")` or `POST /api/v2/document/<DocType>/<name>/method/approve` — don't add a separate function that just fetches the doc and calls it.
- **Don't put doc-scoped logic in standalone APIs.** A function that fetches one doc, validates the caller, and acts on it belongs as a doc-level method. Reserve `api/` files for cross-document operations, aggregations, and endpoints with no document context.
- **Don't leak sensitive fields in `allow_guest=True` endpoints.** Return only what guests need — never user emails, internal IDs, or permission-sensitive data.

## File Location

API methods should be placed in:
```
apps/<app_name>/api.py
```
or
```
apps/<app_name>/<module>/api.py
```

## Testing APIs

Use curl or Postman:
```bash
# With session
curl -X POST \
  http://localhost:8000/api/method/my_app.api.get_items \
  -H "Content-Type: application/json" \
  -d '{"filters": {"item_group": "Products"}}'

# With API key
curl -X POST \
  http://localhost:8000/api/method/my_app.api.get_items \
  -H "Authorization: token xxx:yyy" \
  -d '{"filters": {"item_group": "Products"}}'
```

Remember: This skill is model-invoked. Claude will use it autonomously when detecting API development tasks.
