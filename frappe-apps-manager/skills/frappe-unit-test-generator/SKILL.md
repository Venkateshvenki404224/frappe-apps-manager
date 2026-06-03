---
name: frappe-unit-test-generator
description: Generate comprehensive unit tests for Frappe DocTypes, controllers, and API methods. Use when creating test files, writing test cases, or setting up test infrastructure for Frappe/ERPNext applications.
---

# Frappe Unit Test Generator

Generate production-ready unit tests for Frappe applications following patterns from ERPNext and Frappe core.

## Global Rules

These Frappe conventions apply to everything this skill generates, and override any conflicting example below.

- **Bench commands:** use bare `bench` (never `./env/bin/bench` or a full path). Always pass `--site <site>` explicitly — never run a bare `bench migrate` / `bench run-tests`. Run `bench start` in the background and only if it isn't already running. Don't run discovery commands (`which bench`, `bench --version`).
- **DocType files** live at `apps/<app>/<app>/<module>/doctype/<name>/<name>.json` — the app name appears twice (directory + Python package) — with an empty `__init__.py` alongside. Never `mkdir` the folder; write the JSON and run `bench --site <site> migrate` to create the structure. Don't add `creation`, `modified`, `owner`, `modified_by`, or `docstatus` as fields — Frappe manages them.
- **Database & ORM:** prefer `frappe.qb.get_query()` over raw `frappe.db.sql()`. Use `frappe.db.get_all()` for server logic (ignores permissions) and `frappe.db.get_list()` for user-facing APIs (enforces them). Never use `frappe.db.set_value()` on a field with validation or lifecycle logic — load the doc and `doc.save()` so controller hooks run. Batch-fetch related records; never query inside a loop (N+1).
- **Never call `frappe.db.commit()`** in controllers, request handlers, background jobs, or patches — Frappe auto-commits on success and rolls back on uncaught errors. Flush manually only to make a write visible to a subsequent `frappe.enqueue()` (or pass `enqueue_after_commit=True`).
- **Permissions & APIs:** put permission checks inside controller methods (enforced on every call path), not in API wrappers. Type-hint every `@frappe.whitelist()` parameter so Frappe validates and casts it, and pass `methods=[...]` to pin the HTTP verb.

## When to Use This Skill

Claude should invoke this skill when:
- User wants to write unit tests for DocTypes
- User needs to test controller methods
- User requests API endpoint tests
- User wants to test business logic or validations
- User mentions testing, test cases, or test files
- User wants to set up test fixtures or test data
- User needs to test permissions or workflows

## Test Site

Run tests on a **separate site** from the one the user is actively developing on. Tests create, modify, and delete data — running them on the dev site will pollute it.

Convention: if the dev site is `app.localhost`, create `app-test.localhost` for tests and install the app there:
```bash
bench new-site app-test.localhost --admin-password admin
bench --site app-test.localhost install-app <app>
```

Always run tests against the test site, and always pass `--site` — never run a bare `bench run-tests`:
```bash
bench --site app-test.localhost run-tests --app <app>
```

If tests fail with "DocType not found", run `bench --site app-test.localhost migrate` first.

## Choosing a Base Class

- **DB-backed tests** (anything that creates/reads documents) inherit from `frappe.tests.IntegrationTestCase` — **not** `unittest.TestCase`. Tests run inside a transaction that automatically rolls back, so no manual cleanup or `frappe.db.rollback()` is needed.
- **Pure-logic tests** (utility functions, calculations, parsing — no DB or Frappe context) inherit from `frappe.tests.UnitTestCase`. It skips DB setup/teardown, so it is faster.
- Test files are named `test_<doctype>.py`; test classes are `Test<DocTypeName>`; each test method starts with `test_`.
- Test expected exceptions with `self.assertRaises(frappe.ValidationError, doc.insert)`.

## Capabilities

### 1. DocType Test File Structure

Generate complete test files following Frappe's testing framework.

**Basic Test Structure** (DB-backed, uses `IntegrationTestCase`):
```python
import frappe
from frappe.tests import IntegrationTestCase

class TestItem(IntegrationTestCase):
    def setUp(self):
        """Set up test fixtures before each test (transaction auto-rolls-back)"""
        frappe.set_user("Administrator")
        self.test_item = self._create_test_item()

    def test_item_creation(self):
        """Test basic item creation"""
        item = frappe.get_doc({
            "doctype": "Item",
            "item_code": "_Test Item",
            "item_name": "Test Item",
            "item_group": "Products",
            "stock_uom": "Nos"
        })
        item.insert()

        self.assertEqual(item.item_code, "_Test Item")
        self.assertEqual(item.item_group, "Products")

        # Verify item was created
        self.assertTrue(frappe.db.exists("Item", "_Test Item"))

    def _create_test_item(self):
        """Helper method to create test item"""
        if frappe.db.exists("Item", "_Test Item"):
            return frappe.get_doc("Item", "_Test Item")

        item = frappe.get_doc({
            "doctype": "Item",
            "item_code": "_Test Item",
            "item_name": "Test Item",
            "item_group": "Products",
            "stock_uom": "Nos",
            "is_stock_item": 1
        })
        item.insert()
        return item
```

**Pure-Logic Test** (no DB, uses `UnitTestCase`):
```python
from frappe.tests import UnitTestCase
from my_app.utils import calculate_tax

class TestTaxUtils(UnitTestCase):
    def test_calculate_tax(self):
        """Pure calculation — no Frappe context or database needed"""
        self.assertEqual(calculate_tax(100, 0.1), 10)
```

### 2. Validation Testing

**Test Controller Validations** (from Sales Invoice):
```python
# Pattern from: erpnext/accounts/doctype/sales_invoice/test_sales_invoice.py
class TestSalesInvoice(IntegrationTestCase):
    def test_posting_date_validation(self):
        """Test posting date cannot be future date"""
        si = self._get_test_sales_invoice()
        si.posting_date = frappe.utils.add_days(frappe.utils.today(), 1)

        self.assertRaises(frappe.ValidationError, si.insert)

    def test_items_required(self):
        """Test that items are required"""
        si = frappe.get_doc({
            "doctype": "Sales Invoice",
            "customer": "_Test Customer",
            "items": []
        })

        self.assertRaises(frappe.ValidationError, si.insert)

    def test_negative_quantity(self):
        """Test negative quantities are not allowed"""
        si = self._get_test_sales_invoice()
        si.items[0].qty = -1

        with self.assertRaises(frappe.ValidationError) as context:
            si.insert()

        self.assertIn("Quantity cannot be negative", str(context.exception))

    def test_duplicate_items(self):
        """Test duplicate items with same item code"""
        si = self._get_test_sales_invoice()
        si.append("items", {
            "item_code": si.items[0].item_code,
            "qty": 5,
            "rate": 100
        })

        # Depending on requirements, this might succeed or fail
        # Document the expected behavior
        si.insert()
        self.assertEqual(len(si.items), 2)
```

### 3. Calculation Testing

**Test Amount Calculations** (from Sales Invoice):
```python
# Pattern from: erpnext/accounts/doctype/sales_invoice/test_sales_invoice.py
class TestSalesInvoice(IntegrationTestCase):
    def test_total_calculation(self):
        """Test total amount calculation"""
        si = frappe.get_doc({
            "doctype": "Sales Invoice",
            "customer": "_Test Customer",
            "items": [{
                "item_code": "_Test Item",
                "qty": 10,
                "rate": 100
            }, {
                "item_code": "_Test Item 2",
                "qty": 5,
                "rate": 50
            }]
        })
        si.insert()

        self.assertEqual(si.total, 1250)  # (10*100) + (5*50)

    def test_discount_calculation(self):
        """Test discount application"""
        si = self._get_test_sales_invoice()
        si.discount_amount = 100
        si.save()

        expected_total = si.total - 100
        self.assertEqual(si.grand_total, expected_total)

    def test_tax_calculation(self):
        """Test tax calculation with tax template"""
        si = self._get_test_sales_invoice()
        si.taxes_and_charges = "_Test Tax Template"
        si.save()

        # Tax amount should be calculated
        self.assertGreater(si.total_taxes_and_charges, 0)
        self.assertEqual(
            si.grand_total,
            si.total + si.total_taxes_and_charges
        )
```

### 4. Workflow and State Testing

**Test Document States** (from Stock Entry):
```python
# Pattern from: erpnext/stock/doctype/stock_entry/test_stock_entry.py
class TestStockEntry(IntegrationTestCase):
    def test_submit_workflow(self):
        """Test document submission"""
        se = self._get_test_stock_entry()
        se.insert()

        # Verify draft state
        self.assertEqual(se.docstatus, 0)

        # Submit and verify
        se.submit()
        self.assertEqual(se.docstatus, 1)

        # Verify cannot edit submitted doc
        se.purpose = "Different Purpose"
        self.assertRaises(frappe.ValidationError, se.save)

    def test_cancel_workflow(self):
        """Test document cancellation"""
        se = self._get_test_stock_entry()
        se.insert()
        se.submit()

        # Cancel and verify
        se.cancel()
        self.assertEqual(se.docstatus, 2)

        # Verify cancelled doc cannot be submitted again
        self.assertRaises(frappe.ValidationError, se.submit)

    def test_amendment(self):
        """Test document amendment after cancellation"""
        se = self._get_test_stock_entry()
        se.insert()
        se.submit()
        se.cancel()

        # Create amended document
        amended_se = frappe.copy_doc(se)
        amended_se.amended_from = se.name
        amended_se.docstatus = 0
        amended_se.insert()
        amended_se.submit()

        self.assertEqual(amended_se.amended_from, se.name)
        self.assertEqual(amended_se.docstatus, 1)
```

### 5. Permission Testing

**Test Role Permissions** (from Frappe Core):
```python
# Pattern from: frappe/tests/test_permissions.py
class TestCustomerPermissions(IntegrationTestCase):
    def setUp(self):
        self.test_user = "test@example.com"
        self._setup_test_user()

    def test_read_permission(self):
        """Test user can read allowed documents"""
        frappe.set_user(self.test_user)

        # Should succeed
        customer = frappe.get_doc("Customer", "_Test Customer")
        self.assertEqual(customer.name, "_Test Customer")

    def test_write_permission(self):
        """Test user can edit allowed documents"""
        frappe.set_user(self.test_user)

        customer = frappe.get_doc("Customer", "_Test Customer")
        customer.customer_name = "Updated Name"
        customer.save()

        # Verify change persisted
        customer.reload()
        self.assertEqual(customer.customer_name, "Updated Name")

    def test_create_permission(self):
        """Test user can create new documents"""
        frappe.set_user(self.test_user)

        customer = frappe.get_doc({
            "doctype": "Customer",
            "customer_name": "New Customer"
        })
        customer.insert()

        self.assertTrue(frappe.db.exists("Customer", customer.name))

    def test_denied_access(self):
        """Test user cannot access restricted documents"""
        frappe.set_user(self.test_user)

        # Should raise PermissionError
        self.assertRaises(
            frappe.PermissionError,
            frappe.get_doc,
            "Customer",
            "_Restricted Customer"
        )

    def _setup_test_user(self):
        """Create test user with specific roles"""
        if not frappe.db.exists("User", self.test_user):
            user = frappe.get_doc({
                "doctype": "User",
                "email": self.test_user,
                "first_name": "Test",
                "roles": [{"role": "Sales User"}]
            })
            user.insert(ignore_permissions=True)
```

### 6. API Method Testing

**Test Whitelisted Methods** (from Frappe Core):
```python
# Pattern from: frappe/tests/test_api.py
class TestCustomerAPI(IntegrationTestCase):
    def test_get_customer_details(self):
        """Test API method returns correct data"""
        from my_app.api import get_customer_details

        frappe.set_user("Administrator")
        result = get_customer_details("_Test Customer")

        self.assertIsNotNone(result)
        self.assertEqual(result["name"], "_Test Customer")
        self.assertIn("customer_group", result)

    def test_api_authentication(self):
        """Test API requires authentication"""
        frappe.set_user("Guest")

        from my_app.api import get_customer_details

        self.assertRaises(
            frappe.PermissionError,
            get_customer_details,
            "_Test Customer"
        )

    def test_api_validation(self):
        """Test API validates input parameters"""
        from my_app.api import get_customer_details

        frappe.set_user("Administrator")

        # Test with invalid customer
        self.assertRaises(
            frappe.DoesNotExistError,
            get_customer_details,
            "Invalid Customer"
        )

    def test_api_with_filters(self):
        """Test API method with filter parameters"""
        from my_app.api import get_customers

        frappe.set_user("Administrator")
        result = get_customers(filters={
            "customer_group": "Commercial"
        })

        self.assertIsInstance(result, list)
        for customer in result:
            self.assertEqual(customer["customer_group"], "Commercial")
```

### 7. Database Query Testing

**Test Database Operations**:
```python
# Pattern from: frappe/tests/test_db.py
class TestCustomerQueries(IntegrationTestCase):
    def test_get_all_with_filters(self):
        """Test frappe.get_all with filters"""
        customers = frappe.get_all(
            "Customer",
            filters={"customer_group": "Commercial"},
            fields=["name", "customer_name"]
        )

        self.assertIsInstance(customers, list)
        self.assertGreater(len(customers), 0)

        # Verify all results match filter
        for customer in customers:
            doc = frappe.get_doc("Customer", customer.name)
            self.assertEqual(doc.customer_group, "Commercial")

    def test_get_value(self):
        """Test frappe.db.get_value"""
        customer_group = frappe.db.get_value(
            "Customer",
            "_Test Customer",
            "customer_group"
        )

        self.assertIsNotNone(customer_group)
        self.assertIsInstance(customer_group, str)

    def test_exists(self):
        """Test frappe.db.exists"""
        self.assertTrue(
            frappe.db.exists("Customer", "_Test Customer")
        )
        self.assertFalse(
            frappe.db.exists("Customer", "Non Existent Customer")
        )

    def test_sql_query(self):
        """Test raw SQL queries"""
        result = frappe.db.sql("""
            SELECT name, customer_name
            FROM `tabCustomer`
            WHERE customer_group = %s
            LIMIT 10
        """, ("Commercial",), as_dict=True)

        self.assertIsInstance(result, list)
        for row in result:
            self.assertIn("name", row)
            self.assertIn("customer_name", row)
```

### 8. Child Table Testing

**Test Child Table Operations** (from Sales Invoice):
```python
# Pattern from: erpnext/accounts/doctype/sales_invoice/test_sales_invoice.py
class TestSalesInvoiceItems(IntegrationTestCase):
    def test_add_items(self):
        """Test adding items to child table"""
        si = self._get_test_sales_invoice()
        initial_count = len(si.items)

        si.append("items", {
            "item_code": "_Test Item 2",
            "qty": 5,
            "rate": 150
        })
        si.save()

        self.assertEqual(len(si.items), initial_count + 1)

    def test_remove_items(self):
        """Test removing items from child table"""
        si = self._get_test_sales_invoice()
        initial_count = len(si.items)

        si.items.pop()
        si.save()

        self.assertEqual(len(si.items), initial_count - 1)

    def test_update_item_qty(self):
        """Test updating item quantity"""
        si = self._get_test_sales_invoice()
        original_total = si.total

        si.items[0].qty = si.items[0].qty * 2
        si.save()

        # Total should be recalculated
        self.assertNotEqual(si.total, original_total)
        self.assertGreater(si.total, original_total)
```

### 9. Test Fixtures and Data

**Create Reusable Test Data**:
```python
# Pattern from: erpnext/setup/doctype/company/test_company.py
class TestCompany(IntegrationTestCase):
    @classmethod
    def setUpClass(cls):
        """Set up class-level fixtures"""
        cls._create_test_data()

    @classmethod
    def tearDownClass(cls):
        """Clean up class-level fixtures"""
        cls._cleanup_test_data()

    @classmethod
    def _create_test_data(cls):
        """Create test data used by multiple tests"""
        # Create test company
        if not frappe.db.exists("Company", "_Test Company"):
            company = frappe.get_doc({
                "doctype": "Company",
                "company_name": "_Test Company",
                "abbr": "_TC",
                "default_currency": "USD",
                "country": "United States"
            })
            company.insert()

        # Create test customer
        if not frappe.db.exists("Customer", "_Test Customer"):
            customer = frappe.get_doc({
                "doctype": "Customer",
                "customer_name": "_Test Customer",
                "customer_group": "Commercial",
                "territory": "All Territories"
            })
            customer.insert()

    @classmethod
    def _cleanup_test_data(cls):
        """Clean up test data"""
        for doctype, name in [
            ("Company", "_Test Company"),
            ("Customer", "_Test Customer")
        ]:
            if frappe.db.exists(doctype, name):
                frappe.delete_doc(doctype, name, force=True)
```

### 10. Mock and Patch Testing

**Mock External Dependencies**:
```python
# Pattern from: frappe/tests/test_email.py
from unittest.mock import patch, MagicMock

class TestEmailNotification(IntegrationTestCase):
    @patch('frappe.sendmail')
    def test_send_notification(self, mock_sendmail):
        """Test email notification is sent"""
        from my_app.notifications import send_welcome_email

        send_welcome_email("test@example.com")

        # Verify sendmail was called
        mock_sendmail.assert_called_once()
        args, kwargs = mock_sendmail.call_args
        self.assertIn("test@example.com", kwargs["recipients"])

    @patch('requests.post')
    def test_external_api_call(self, mock_post):
        """Test external API integration"""
        mock_post.return_value = MagicMock(
            status_code=200,
            json=lambda: {"success": True}
        )

        from my_app.integrations import sync_with_external_system

        result = sync_with_external_system("_Test Customer")

        self.assertTrue(result["success"])
        mock_post.assert_called_once()
```

## Test Organization Patterns

### File Structure

**Standard Test File Location** (the app name appears twice — directory + Python package):
```
apps/my_app/my_app/
└── my_module/
    └── doctype/
        └── my_doctype/
            ├── my_doctype.py
            ├── my_doctype.json
            ├── my_doctype.js
            └── test_my_doctype.py  ← Test file here
```

For feature-wise tests that span multiple DocTypes, place them in the app's tests directory: `apps/<app>/<app>/tests/test_<feature>.py`.

### Test Naming Conventions

- Test files: `test_<doctype_name>.py`
- Test classes: `Test<DocTypeName>`
- Test methods: `test_<description>`
- Helper methods: `_<method_name>`

### Test Method Organization

```python
class TestMyDocType(IntegrationTestCase):
    # 1. Setup and teardown
    def setUp(self):
        pass

    def tearDown(self):
        pass

    # 2. Creation tests
    def test_creation(self):
        pass

    # 3. Validation tests
    def test_validation_required_fields(self):
        pass

    # 4. Calculation tests
    def test_total_calculation(self):
        pass

    # 5. Workflow tests
    def test_submit_workflow(self):
        pass

    # 6. Permission tests
    def test_user_permissions(self):
        pass

    # 7. Helper methods
    def _create_test_doc(self):
        pass
```

## References

### Frappe Core Test Examples (Primary Reference)

**Frappe Framework Tests:**
- Test Document: https://github.com/frappe/frappe/blob/develop/frappe/tests/test_document.py
- Test DB: https://github.com/frappe/frappe/blob/develop/frappe/tests/test_db.py
- Test Permissions: https://github.com/frappe/frappe/blob/develop/frappe/tests/test_permissions.py
- Test API: https://github.com/frappe/frappe/blob/develop/frappe/tests/test_api.py
- Test Utils: https://github.com/frappe/frappe/blob/develop/frappe/tests/utils.py

**ERPNext Test Examples:**
- Sales Invoice Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/accounts/doctype/sales_invoice/test_sales_invoice.py
- Stock Entry Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/stock/doctype/stock_entry/test_stock_entry.py
- Payment Entry Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/accounts/doctype/payment_entry/test_payment_entry.py
- Item Tests: https://github.com/frappe/erpnext/blob/develop/erpnext/stock/doctype/item/test_item.py

### Official Documentation (Secondary Reference)

- Testing Guide: https://frappeframework.com/docs/user/en/testing
- Unit Testing: https://frappeframework.com/docs/user/en/testing/unit-testing
- Python unittest: https://docs.python.org/3/library/unittest.html

## Best Practices

1. **Test Independence**: Each test should run independently
2. **Use the right base class**: `IntegrationTestCase` for DB tests, `UnitTestCase` for pure logic
3. **Rely on auto-rollback**: `IntegrationTestCase` runs each test in a transaction that rolls back — no manual `frappe.db.rollback()` cleanup needed
4. **Test One Thing**: Each test should verify one behavior
5. **Descriptive Names**: Test names should describe what they test
6. **Use Assertions**: Use specific assertions (`assertEqual`, not just `assertTrue`)
7. **Test Edge Cases**: Test boundary conditions and error cases
8. **Mock External Calls**: Don't rely on external services in tests
9. **Use Test Data**: Prefix test data with `_Test` for easy identification
10. **Document Expected Behavior**: Add docstrings explaining what's tested

## Common Assertions

```python
# Equality checks
self.assertEqual(a, b)
self.assertNotEqual(a, b)

# Boolean checks
self.assertTrue(condition)
self.assertFalse(condition)

# Existence checks
self.assertIsNone(value)
self.assertIsNotNone(value)

# Collection checks
self.assertIn(item, collection)
self.assertNotIn(item, collection)

# Numeric comparisons
self.assertGreater(a, b)
self.assertLess(a, b)
self.assertGreaterEqual(a, b)

# Exception checks
self.assertRaises(Exception, callable, *args)
with self.assertRaises(Exception):
    risky_operation()

# Type checks
self.assertIsInstance(obj, MyClass)
```

## Running Tests

Always pass `--site` (point it at the dedicated test site). Never run a bare `bench run-tests`.

```bash
# Run all tests for an app
bench --site app-test.localhost run-tests --app my_app

# Run tests for a specific doctype
bench --site app-test.localhost run-tests --doctype "My DocType"

# Run a specific test module (file)
bench --site app-test.localhost run-tests --module my_app.my_module.doctype.my_doctype.test_my_doctype

# Run a single test method
bench --site app-test.localhost run-tests --module my_app.my_module.doctype.my_doctype.test_my_doctype --test test_item_creation

# Verbose output
bench --site app-test.localhost run-tests --app my_app -v
```
