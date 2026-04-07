# Bynry-Case-Study
StockFlow – Backend Engineer Intern Case Study

## Part 1: Code Review & Debugging
 
### The Buggy Code
 
```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
 
    # Create new product
    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )
 
    db.session.add(product)
    db.session.commit()
 
    # Update inventory count
    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )
 
    db.session.add(inventory)
    db.session.commit()
 
    return {"message": "Product created", "product_id": product.id}
```
 
---
### Issues Identified
 
#### Issue 1 — No Input Validation
 
**Problem:** The code directly accesses `data['name']`, `data['sku']`, etc. without checking if these keys exist or if the values are valid.
 
**Impact in Production:**
- If a client sends an incomplete request body, the server throws an unhandled `KeyError` and returns a `500 Internal Server Error` instead of a meaningful `400 Bad Request`.
- Corrupted or null data can silently enter the database.
- Clients get no helpful feedback about what field was missing.
 
---
 
#### Issue 2 — No SKU Uniqueness Check
 
**Problem:** There is no check to enforce that the `sku` is unique before inserting. The business requirement explicitly states SKUs must be unique across the platform.
 
**Impact in Production:**
- If the database has a unique constraint on `sku`, this will raise an unhandled `IntegrityError`, returning a raw `500` to the client.
- If there is no DB constraint (even worse), duplicate SKUs silently enter the database, breaking any lookup, reporting, or reconciliation logic that relies on SKU uniqueness.
 
---
 
#### Issue 3 — Two Separate `db.session.commit()` Calls (Non-Atomic Transaction)
 
**Problem:** The `Product` and `Inventory` records are committed in two separate transactions. If the server crashes, or the second commit fails (e.g., due to a DB constraint on `Inventory`), you end up with a `Product` row in the database with no corresponding `Inventory` record.
 
**Impact in Production:**
- Data inconsistency — a product exists in the system but has no inventory record, causing silent failures in stock queries, dashboards, and reporting.
- Very hard to detect and fix after the fact in a production database.
- Violates atomicity: either both records should be saved, or neither.
 
---
 
#### Issue 4 — `warehouse_id` Stored on Both `Product` and `Inventory` (Incorrect Schema Design)
 
**Problem:** Since products can exist in multiple warehouses (per the business requirement), storing `warehouse_id` directly on the `Product` model is semantically wrong. The `Product` model should be warehouse-agnostic; the warehouse relationship lives on the `Inventory` record.
 
**Impact in Production:**
- A product can only be associated with one warehouse at creation time, making it impossible to expand to multiple warehouses without a schema migration.
- Misleads future developers into thinking a product belongs to a single warehouse.
 
---
 
#### Issue 5 — No Error Handling or HTTP Status Codes
 
**Problem:** The function has no `try/except` block, and always returns a `200 OK` even if something goes wrong. Successful creation should return `201 Created`.
 
**Impact in Production:**
- Clients cannot distinguish between success and failure from the HTTP status code alone.
- Any unexpected exception (DB down, network issue, validation error) surfaces as an unhandled `500`, possibly leaking internal stack traces.
 
---
 
#### Issue 6 — Price Has No Type Validation
 
**Problem:** `price` is passed directly from request JSON into the model. There is no check that it is a valid positive decimal number.
 
**Impact in Production:**
- A client could send `price: -50` or `price: "free"`, which either causes a DB error or stores invalid data.
- Financial data integrity is compromised.
 
---

### Fixed Version
 
```python
from flask import request, jsonify
from decimal import Decimal, InvalidOperation
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    # 1. Basic input validation — ensure all required fields are present
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [f for f in required_fields if f not in data or data[f] is None]
    if missing:
        return jsonify({"error": f"Missing required fields: {', '.join(missing)}"}), 400

    # Validate price — convert safely to Decimal and ensure it's not negative
    try:
        price = Decimal(str(data['price']))  # use str to avoid float precision issues
        if price < 0:
            raise ValueError
    except (InvalidOperation, ValueError):
        return jsonify({"error": "Price must be a valid non-negative number"}), 400

    # Validate initial quantity — must be a non-negative integer
    if not isinstance(data['initial_quantity'], int) or data['initial_quantity'] < 0:
        return jsonify({"error": "initial_quantity must be a non-negative integer"}), 400

    # 2. Check if SKU already exists to prevent duplicates (business constraint)
    existing = Product.query.filter_by(sku=data['sku']).first()
    if existing:
        return jsonify({"error": f"SKU '{data['sku']}' already exists"}), 409  # conflict

    try:
        # 3. Wrap product + inventory creation in a single transaction
        # so we don't end up with partial data
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price,
            description=data.get('description')  # optional field
        )
        db.session.add(product)

        # Flush sends data to DB without committing, so we can access product.id
        db.session.flush()

        # Inventory is tied to product and warehouse
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)

        # Commit once — ensures atomicity (both succeed or both fail)
        db.session.commit()

        # 4. Return success response with created resource info
        return jsonify({
            "message": "Product created successfully",
            "product_id": product.id
        }), 201

    except IntegrityError:
        # Handles DB-level issues like unique constraints or FK violations
        db.session.rollback()
        return jsonify({"error": "Database integrity error. Possibly a duplicate SKU."}), 409

    except Exception as e:
        # Catch-all for unexpected failures — rollback to keep DB clean
        db.session.rollback()

        # Log actual error internally for debugging (don't expose to client)
        app.logger.error(f"Failed to create product: {e}")

        return jsonify({"error": "An unexpected error occurred. Please try again."}), 500
```
 
---
 
### Summary of Fixes
 
| # | Issue | Fix |
|---|-------|-----|
| 1 | No input validation | Added required field checks and type validation before any DB operation |
| 2 | No SKU uniqueness check | Query DB for existing SKU before insert; return `409 Conflict` if found |
| 3 | Two separate commits (non-atomic) | Used `db.session.flush()` + single `db.session.commit()` inside a `try/except` |
| 4 | `warehouse_id` on `Product` model | Removed from `Product`; kept only on `Inventory` to support multi-warehouse |
| 5 | No error handling / wrong status code | Wrapped in `try/except`; return `201` on success, appropriate 4xx/5xx on failure |
| 6 | Price not validated | Parsed with `Decimal`, checked for non-negative value |
 
---
 
### Key Takeaways
 
- **Atomicity matters.** Any operation that writes to multiple tables must be a single transaction. Partial writes cause silent, hard-to-debug inconsistencies.
- **Validate at the boundary.** Never trust incoming data. Validate shape, type, and business rules before touching the database.
- **HTTP semantics are part of the API contract.** Use `201 Created`, `400 Bad Request`, `409 Conflict`, and `500 Internal Server Error` correctly so clients can handle responses programmatically.
- **Schema should reflect domain logic.** Since a product can exist in multiple warehouses, the warehouse relationship belongs on `Inventory`, not `Product`.
- **Never leak internal errors to clients.** Log them server-side and return a generic message externally.
 
---
