
Part 1: Code Review & Debugging
•  Framework: Python / Flask / SQLAlchemy

Original Code Under Review
The following endpoint was written by a previous intern. It compiles but has multiple bugs affecting correctness, security, and reliability in production.


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


1. Issues Identified — Overview
Eight distinct problems were found spanning input validation, data integrity, security, and API design.

#	Issue	Category	Severity
1	No input validation / missing KeyError handling	Technical	Critical
2	No SKU uniqueness check	Business Logic	Critical
3	Two separate commits — non-atomic transaction	Technical	Critical
4	No error handling or rollback	Technical	High
5	No authentication / authorisation guard	Security	Critical
6	Price not type-coerced (float precision bug)	Business Logic	High
7	warehouse_id not validated against DB or tenant	Business Logic	High
8	Wrong HTTP status code (200 instead of 201)	API Design	Medium

2. Detailed Issues, Impact & Fixes

Issue 1 — No Input Validation
Problem: The code uses data['name'], data['sku'] etc. with bare key access. If any field is absent or if the request body is not JSON, Python raises an unhandled KeyError or TypeError.
Production Impact: The server returns a raw HTTP 500 stack trace instead of a descriptive 400 Bad Request. Clients receive no actionable feedback. Also, data.get('description') being absent would silently skip optional fields rather than crashing — but bare key access offers no such safety.
Fix: Use request.get_json() (returns None on parse failure), check for None, then validate all required fields explicitly before touching the DB. Use data.get(key) for optional fields.
Assumption: Fields description and product_type are treated as optional per the additional context; all others (name, sku, price, warehouse_id, initial_quantity) are required.

Issue 2 — No SKU Uniqueness Check
Problem: No query is made to verify if the SKU already exists before inserting. The additional context states: "SKUs must be unique across the platform."
Production Impact: If there is no UNIQUE constraint at DB level, duplicate SKUs are silently inserted — corrupting the product catalogue and breaking all SKU-based lookups. If the DB does enforce uniqueness, the uncaught IntegrityError returns a raw 500 with no guidance to the caller.
Fix: Query Product.query.filter_by(sku=...) before inserting. Normalise the SKU to uppercase for consistent comparison. Return HTTP 409 Conflict if a duplicate is found.

Issue 3 — Two Separate Commits (Non-Atomic Transaction)
Problem: The code calls db.session.commit() twice — once after creating Product and once after creating Inventory. These are two separate database transactions.
Production Impact: If the server crashes, a network timeout occurs, or any error is raised between the two commits, a Product row is persisted with NO corresponding Inventory row. The database is left in a permanently inconsistent state that is hard to detect and repair.
Fix: Use db.session.flush() after adding Product (to obtain product.id without committing), then add Inventory and call db.session.commit() exactly once. Both records are saved atomically — either both persist or neither does.

Issue 4 — No Error Handling / No Rollback
Problem: There is no try/except block around any database operation, and no call to db.session.rollback() on failure.
Production Impact: Any DB exception (connection drop, constraint violation, deadlock) will propagate as an unhandled 500, and the SQLAlchemy session may be left in a broken/dirty state, causing subsequent requests in the same session to also fail.
Fix: Wrap the entire DB block in try/except. Call db.session.rollback() in the except clause. Log the error server-side and return a clean 500 JSON to the client.

Issue 5 — No Authentication / Authorisation
Problem: The endpoint has no auth guard. Any unauthenticated request can reach the DB. There is also no check that the requesting user owns the warehouse_id they are submitting.
Production Impact: This is a critical security hole in a multi-tenant B2B system. Any actor can create products in any warehouse, including those belonging to other companies, completely violating tenant data isolation.
Fix: Add a @require_auth decorator (or equivalent middleware) that validates the JWT/session token and populates g.current_user. Use g.current_user.company_id to scope all queries.
Assumption: A @require_auth decorator or similar auth middleware exists in the codebase. The authenticated user object exposes a company_id attribute.

Issue 6 — Price Not Type-Coerced (Float Precision Bug)
Problem: The additional context states "Price can be decimal values." The raw value from JSON is stored directly. Python's float cannot precisely represent many decimals (e.g. 19.99 becomes 19.989999999999998...).
Production Impact: Financial calculations (totals, invoices, reports) will accumulate rounding errors. There is also no validation preventing negative prices.
Fix: Coerce price to Decimal(str(data['price'])) — converting via string avoids float imprecision. Validate it is non-negative. The DB column should be NUMERIC(12, 2) not FLOAT.

Issue 7 — warehouse_id Not Validated
Problem: The additional context states "Products can exist in multiple warehouses." But the code never checks whether the provided warehouse_id actually exists in the database, or belongs to the authenticated company.
Production Impact: Products can be assigned to ghost warehouses. If FK constraints are enforced at the DB level the insert fails with an unhandled IntegrityError (returning 500). If not enforced, orphaned products accumulate silently.
Fix: Query for Warehouse.query.filter_by(id=..., company_id=g.current_user.company_id).first() and return 404 if not found. This also enforces tenant isolation for warehouses.

Issue 8 — Wrong HTTP Status Code (200 instead of 201)
Problem: The endpoint returns HTTP 200 OK for a successful resource creation. REST convention requires HTTP 201 Created.
Production Impact: API clients, SDKs, and monitoring tools that check status codes correctly cannot distinguish a successful creation from a generic successful response. It also breaks OpenAPI/Swagger contract expectations.
Fix: Return jsonify({...}), 201.

3. Corrected Version — Full Implementation
The fixed code below addresses all eight issues. Each fix is annotated inline with a FIX N comment.


from flask import request, jsonify, g
from decimal import Decimal, InvalidOperation
from sqlalchemy.exc import IntegrityError
 
@app.route('/api/products', methods=['POST'])
@require_auth                          # FIX 8: Authentication guard
def create_product():
    data = request.get_json()          # FIX 1: get_json() is safer than .json
    if not data:
        return jsonify({'error': 'Request body must be valid JSON'}), 400
 
    # FIX 2: Validate all required fields upfront
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [f for f in required_fields if f not in data]
    if missing:
        return jsonify({'error': f'Missing required fields: {missing}'}), 400
 
    # FIX 3: Handle optional fields gracefully
    description = data.get('description')   # optional
    product_type = data.get('product_type', 'standard')  # optional with default
 
    # FIX 4: Type-coerce and validate price (supports decimals, rejects negatives)
    try:
        price = Decimal(str(data['price']))
        if price < 0:
            raise ValueError('Price cannot be negative')
    except (InvalidOperation, ValueError) as e:
        return jsonify({'error': f'Invalid price: {e}'}), 400
 
    # FIX 5: Validate initial_quantity
    if not isinstance(data['initial_quantity'], int) or data['initial_quantity'] < 0:
        return jsonify({'error': 'initial_quantity must be a non-negative integer'}), 400
 
    # FIX 6: Validate warehouse exists AND belongs to authenticated company
    warehouse = Warehouse.query.filter_by(
        id=data['warehouse_id'],
        company_id=g.current_user.company_id   # scoped to tenant
    ).first()
    if not warehouse:
        return jsonify({'error': 'Warehouse not found or access denied'}), 404
 
    # FIX 7: Check SKU uniqueness platform-wide before insert
    if Product.query.filter_by(sku=data['sku'].strip().upper()).first():
        return jsonify({'error': f"SKU '{data['sku']}' already exists"}), 409
 
    # FIX 9 & 10: Single atomic transaction with rollback on any failure
    try:
        product = Product(
            name=data['name'].strip(),
            sku=data['sku'].strip().upper(),   # normalise SKU
            price=price,                       # Decimal, not raw float
            warehouse_id=data['warehouse_id'],
            description=description,
            product_type=product_type,
            company_id=g.current_user.company_id
        )
        db.session.add(product)
        db.session.flush()    # get product.id WITHOUT committing yet
 
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)
        db.session.commit()   # ONE commit — both records saved atomically
 
        # FIX 11: Return correct HTTP 201 Created
        return jsonify({
            'message': 'Product created successfully',
            'product_id': product.id
        }), 201
 
    except IntegrityError:
        db.session.rollback()
        return jsonify({'error': 'DB constraint violation (possible duplicate SKU)'}), 409
    except Exception as e:
        db.session.rollback()
        app.logger.error(f'Product creation failed: {e}')
        return jsonify({'error': 'Internal server error'}), 500


4. Assumptions Made
Due to incomplete context, the following assumptions were made. These would be confirmed with the team before merging to production.

Assumption	Reasoning / Where It Applies
Flask + SQLAlchemy + PostgreSQL is the target stack.	Consistent with the original code structure; NUMERIC(12,2) and flush() are SQLAlchemy-specific.
A @require_auth decorator exists and populates g.current_user with a company_id attribute.	Standard Flask pattern for JWT/session auth in multi-tenant apps.
SKU uniqueness is platform-wide (not per-company).	The additional context states 'SKUs must be unique across the platform'.
description and product_type are optional fields; all other fields are required.	The additional context says 'some fields might be optional' without specifying which — chose the two most likely candidates.
The Warehouse model has a company_id foreign key for tenant scoping.	Essential for tenant isolation; standard in any B2B SaaS schema.
initial_quantity must be a non-negative integer (0 is a valid new product stock).	Business logic — a product can be created with zero stock to be restocked later.

5. Key Reasoning Summary
The most important design decisions and why they matter:

•	flush() before commit() is the correct SQLAlchemy pattern for multi-object atomic writes — it generates the primary key in memory without starting a new transaction.
•	Using Decimal(str(price)) rather than float(price) is essential for any financial application — float arithmetic is non-deterministic for decimal fractions.
•	Validating all inputs before touching the database reduces unnecessary DB round trips and makes error messages precise and actionable for the API caller.
•	Scoping the warehouse lookup to g.current_user.company_id means even a valid warehouse_id from another tenant will correctly return 404 — the principle of least privilege in a multi-tenant system.
•	Normalising SKUs to uppercase prevents logical duplicates like 'WID-001' and 'wid-001' from both passing the uniqueness check.

 
Part 2: Database Design
•  Target DB: PostgreSQL  •  Notation: SQL DDL + annotated schema tables

Approach & Design Philosophy
The schema is designed for a multi-tenant B2B SaaS platform where strict tenant (company) isolation is paramount. Every table that stores business data is scoped to a company. The design follows four core principles:

•	Atomicity — related records are written in single transactions; audit logs are append-only.
•	Normalisation — no data duplication; each fact lives in exactly one table.
•	Extensibility — junction tables (inventory, product_suppliers, bundle_items) allow many-to-many relationships to grow without schema changes.
•	Query Performance — indexes are placed on every foreign key and every column used in WHERE or ORDER BY clauses in anticipated high-frequency queries.


1. Schema Design — Annotated Tables
Nine tables cover all stated requirements plus the gaps identified in Section 3. Each table is presented with column-level reasoning.

Table 1: companies
The root tenant table. Every piece of data ultimately traces back to a company.

companies
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Auto-incrementing surrogate key — avoids exposing business logic in IDs.
name	VARCHAR(255)	NOT NULL	Company's display name.
plan_tier	VARCHAR(50)	DEFAULT 'starter'	Assumption: SaaS billing tier (starter/pro/enterprise). Supports feature gating.
is_active	BOOLEAN	DEFAULT TRUE	Soft-delete support — deactivate without losing data.
created_at	TIMESTAMP	DEFAULT NOW()	Audit trail — when the tenant was onboarded.

Table 2: warehouses
A company can own many warehouses. Products are stocked per warehouse.

warehouses
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
company_id	INT	NOT NULL REFERENCES companies(id) ON DELETE CASCADE	FK — CASCADE ensures warehouses are removed if the company is deleted.
name	VARCHAR(255)	NOT NULL	Human-readable label e.g. 'North Mumbai Hub'.
location	VARCHAR(500)		Optional address/GPS string — may be structured later.
is_active	BOOLEAN	DEFAULT TRUE	Soft-delete — warehouses can be closed without orphaning history.
created_at	TIMESTAMP	DEFAULT NOW()	Audit trail.

Table 3: products
Platform-wide product catalogue scoped to a company. SKU is globally unique per platform requirement.

products
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
company_id	INT	NOT NULL REFERENCES companies(id)	Tenant scope — products belong to a company.
name	VARCHAR(255)	NOT NULL	Display name of the product.
sku	VARCHAR(100)	NOT NULL UNIQUE	UNIQUE globally per platform requirement. Normalised to UPPER on write.
description	TEXT		Optional — long-form product description.
price	NUMERIC(12,2)	NOT NULL CHECK (price >= 0)	NUMERIC avoids float precision errors for financial data.
product_type	VARCHAR(50)	DEFAULT 'standard'	Values: standard | bundle. Drives bundle_items join logic.
low_stock_threshold	INT	DEFAULT 10	Per-product alert threshold — required by Part 3 business rules.
is_active	BOOLEAN	DEFAULT TRUE	Soft-delete — deactivate without losing sales history.
created_at	TIMESTAMP	DEFAULT NOW()	Audit trail.
updated_at	TIMESTAMP	DEFAULT NOW()	Updated via DB trigger or application on every write.

Table 4: inventory
The central junction table representing current stock levels. A product can have a row per warehouse it is stocked in, satisfying the requirement that products can exist in multiple warehouses with different quantities.

inventory
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key for the inventory line.
product_id	INT	NOT NULL REFERENCES products(id)	FK to product.
warehouse_id	INT	NOT NULL REFERENCES warehouses(id)	FK to warehouse — the stock location.
quantity	INT	NOT NULL DEFAULT 0 CHECK (quantity >= 0)	CHECK prevents negative stock at DB level — second line of defence after app validation.
updated_at	TIMESTAMP	DEFAULT NOW()	Timestamp of last stock change for freshness checks.
		UNIQUE (product_id, warehouse_id)	Compound unique — prevents duplicate rows for the same product+warehouse combo.

Table 5: inventory_transactions
Append-only audit log for every inventory change. Never updated — only inserted. Satisfies the requirement to track when inventory levels change and enables sales velocity calculations.

inventory_transactions
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
product_id	INT	NOT NULL REFERENCES products(id)	Which product changed.
warehouse_id	INT	NOT NULL REFERENCES warehouses(id)	Which warehouse location.
change_qty	INT	NOT NULL	Positive = restock / receipt. Negative = sale / write-off.
transaction_type	VARCHAR(50)	NOT NULL	Values: sale | restock | adjustment | transfer_in | transfer_out | write_off.
reference_id	INT		Optional FK to orders(id) or purchase_orders(id) for traceability.
notes	TEXT		Free-text reason — useful for adjustments and write-offs.
created_by	INT	REFERENCES users(id)	Who triggered the change — accountability in a multi-user system.
created_at	TIMESTAMP	DEFAULT NOW()	Immutable timestamp — never update this column.

Table 6: suppliers
Supplier master data. Assumption: suppliers are shared across the platform (not per-company) as a supplier may serve multiple StockFlow tenants. A join table links suppliers to companies.

suppliers
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
name	VARCHAR(255)	NOT NULL	Supplier's trading name.
contact_email	VARCHAR(255)		Primary reorder contact email.
contact_phone	VARCHAR(50)		Optional phone for urgent reorders.
address	TEXT		Optional — shipping origin address.
created_at	TIMESTAMP	DEFAULT NOW()	Audit trail.

Table 7: product_suppliers
Junction table linking products to their suppliers with commercial terms. A product can have multiple suppliers (redundancy, cost comparison). is_preferred flags the default reorder contact.

product_suppliers
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key for the relationship.
product_id	INT	NOT NULL REFERENCES products(id)	FK to product.
supplier_id	INT	NOT NULL REFERENCES suppliers(id)	FK to supplier.
unit_cost	NUMERIC(12,2)		Cost price from this supplier — NUMERIC for precision.
lead_time_days	INT		How many days from PO to delivery — drives reorder timing.
is_preferred	BOOLEAN	DEFAULT FALSE	TRUE = this supplier is the primary reorder contact for this product.
		UNIQUE (product_id, supplier_id)	Prevents duplicate supplier relationships per product.

Table 8: bundle_items
Self-referential junction table for product bundles. A bundle product contains component products. The CHECK constraint prevents a product from containing itself.

bundle_items
Column	Data Type	Constraints	Notes / Reasoning
bundle_product_id	INT	NOT NULL REFERENCES products(id)	The parent bundle product.
component_product_id	INT	NOT NULL REFERENCES products(id)	A component product inside the bundle.
quantity	INT	NOT NULL DEFAULT 1	How many units of the component are in one bundle.
		PRIMARY KEY (bundle_product_id, component_product_id)	Composite PK — no duplicates, no surrogate key needed.
		CHECK (bundle_product_id != component_product_id)	Prevents a product bundling itself — infinite loop guard.

Table 9: orders + order_items
Sales orders are required to compute recent sales activity and days_until_stockout for Part 3. Two tables follow the standard header-line pattern.

orders
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
company_id	INT	NOT NULL REFERENCES companies(id)	Tenant scope.
status	VARCHAR(50)	DEFAULT 'pending'	Values: pending | confirmed | shipped | delivered | cancelled.
created_at	TIMESTAMP	DEFAULT NOW()	Order date — used to filter recent sales in Part 3.

order_items
Column	Data Type	Constraints	Notes / Reasoning
id	SERIAL	PRIMARY KEY	Surrogate key.
order_id	INT	NOT NULL REFERENCES orders(id)	FK to parent order.
product_id	INT	NOT NULL REFERENCES products(id)	Which product was sold.
warehouse_id	INT	NOT NULL REFERENCES warehouses(id)	Which warehouse fulfilled the order line.
quantity	INT	NOT NULL CHECK (quantity > 0)	Units sold — used to compute avg_daily_sales.
unit_price	NUMERIC(12,2)	NOT NULL	Price at time of sale — snapshot to protect against product price changes.
created_at	TIMESTAMP	DEFAULT NOW()	Line timestamp.


2. Full SQL DDL
The complete schema in PostgreSQL DDL format, including all indexes and constraints.


-- ============================================================
-- StockFlow Database Schema — PostgreSQL
-- Author: Siddharth Kumar | April 2026
-- ============================================================
 
CREATE TABLE companies (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    plan_tier   VARCHAR(50)  DEFAULT 'starter',
    is_active   BOOLEAN      DEFAULT TRUE,
    created_at  TIMESTAMP    DEFAULT NOW()
);
 
CREATE TABLE warehouses (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    location    VARCHAR(500),
    is_active   BOOLEAN   DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW()
);
 
CREATE TABLE products (
    id                   SERIAL PRIMARY KEY,
    company_id           INT NOT NULL REFERENCES companies(id),
    name                 VARCHAR(255)   NOT NULL,
    sku                  VARCHAR(100)   NOT NULL UNIQUE,
    description          TEXT,
    price                NUMERIC(12,2)  NOT NULL CHECK (price >= 0),
    product_type         VARCHAR(50)    DEFAULT 'standard',
    low_stock_threshold  INT            DEFAULT 10,
    is_active            BOOLEAN        DEFAULT TRUE,
    created_at           TIMESTAMP      DEFAULT NOW(),
    updated_at           TIMESTAMP      DEFAULT NOW()
);
 
CREATE TABLE inventory (
    id            SERIAL PRIMARY KEY,
    product_id    INT NOT NULL REFERENCES products(id),
    warehouse_id  INT NOT NULL REFERENCES warehouses(id),
    quantity      INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    updated_at    TIMESTAMP DEFAULT NOW(),
    UNIQUE (product_id, warehouse_id)
);
 
CREATE TABLE inventory_transactions (
    id                SERIAL PRIMARY KEY,
    product_id        INT NOT NULL REFERENCES products(id),
    warehouse_id      INT NOT NULL REFERENCES warehouses(id),
    change_qty        INT         NOT NULL,
    transaction_type  VARCHAR(50) NOT NULL,
    reference_id      INT,
    notes             TEXT,
    created_by        INT REFERENCES users(id),
    created_at        TIMESTAMP DEFAULT NOW()
);
 
CREATE TABLE suppliers (
    id             SERIAL PRIMARY KEY,
    name           VARCHAR(255) NOT NULL,
    contact_email  VARCHAR(255),
    contact_phone  VARCHAR(50),
    address        TEXT,
    created_at     TIMESTAMP DEFAULT NOW()
);
 
CREATE TABLE product_suppliers (
    id              SERIAL PRIMARY KEY,
    product_id      INT NOT NULL REFERENCES products(id),
    supplier_id     INT NOT NULL REFERENCES suppliers(id),
    unit_cost       NUMERIC(12,2),
    lead_time_days  INT,
    is_preferred    BOOLEAN DEFAULT FALSE,
    UNIQUE (product_id, supplier_id)
);
 
CREATE TABLE bundle_items (
    bundle_product_id     INT NOT NULL REFERENCES products(id),
    component_product_id  INT NOT NULL REFERENCES products(id),
    quantity              INT NOT NULL DEFAULT 1,
    PRIMARY KEY (bundle_product_id, component_product_id),
    CHECK (bundle_product_id != component_product_id)
);
 
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id),
    status      VARCHAR(50) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT NOW()
);
 
CREATE TABLE order_items (
    id            SERIAL PRIMARY KEY,
    order_id      INT NOT NULL REFERENCES orders(id),
    product_id    INT NOT NULL REFERENCES products(id),
    warehouse_id  INT NOT NULL REFERENCES warehouses(id),
    quantity      INT           NOT NULL CHECK (quantity > 0),
    unit_price    NUMERIC(12,2) NOT NULL,
    created_at    TIMESTAMP DEFAULT NOW()
);
 
-- ── Indexes ─────────────────────────────────────────────────
CREATE INDEX idx_warehouses_company     ON warehouses(company_id);
CREATE INDEX idx_products_company       ON products(company_id);
CREATE INDEX idx_products_sku           ON products(sku);
CREATE INDEX idx_inventory_product      ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse    ON inventory(warehouse_id);
CREATE INDEX idx_txn_product_date       ON inventory_transactions(product_id, created_at);
CREATE INDEX idx_txn_warehouse_date     ON inventory_transactions(warehouse_id, created_at);
CREATE INDEX idx_orders_company_date    ON orders(company_id, created_at);
CREATE INDEX idx_orderitems_product_dt  ON order_items(product_id, created_at);
CREATE INDEX idx_orderitems_warehouse   ON order_items(warehouse_id);
CREATE INDEX idx_prodsup_supplier       ON product_suppliers(supplier_id);



3. Gaps Identified — Questions for the Product Team
The requirements were intentionally incomplete. The following questions must be answered before finalising the schema for production.

#	Question	Why It Matters
1	Are SKUs unique per company or globally across all StockFlow tenants?	Determines whether the UNIQUE constraint on products.sku should be platform-wide or a compound UNIQUE (sku, company_id).
2	What qualifies as 'recent sales activity' — last 7 days, 30 days, or configurable?	Directly affects the WHERE clause and index design for low-stock alert queries in Part 3.
3	Is low_stock_threshold set per product, per product-warehouse combination, or per product category?	If per warehouse-product, the column moves from products to inventory, changing the schema significantly.
4	Can a product be transferred between warehouses, and should that be audited?	If yes, we need a transfer workflow and inventory_transactions entries for transfer_out and transfer_in.
5	Are bundle products tracked as physical inventory themselves, or is stock computed dynamically from components?	If computed, stock checks for bundles need a recursive query; if tracked, bundles need their own inventory rows.
6	Do suppliers belong to a specific company (private) or are they shared across the platform (global)?	If private, suppliers needs a company_id FK. Currently assumed global/shared.
7	Is there a purchase order (PO) workflow for reordering from suppliers?	If yes, we need purchase_orders and purchase_order_items tables, and reference_id in inventory_transactions becomes typed.
8	Soft-delete or hard-delete for products, warehouses, and companies?	Currently using is_active boolean for soft-delete. Hard delete risks FK violations and data loss.
9	Is multi-currency support required for product prices or supplier costs?	If yes, we need a currency_code column and a currencies table or external FX rate service.
10	Is there a users/roles model — who can create products, approve POs, view reports?	The created_by FK in inventory_transactions assumes a users table. RBAC scope is unclear.
11	Can a single order span multiple warehouses?	Currently order_items has a warehouse_id per line, allowing mixed-warehouse fulfilment. Confirm this is intended.
12	Are product categories / tags needed?	Categories could drive threshold rules, supplier groupings, and reporting — common in inventory systems.


4. Design Decisions — Reasoning
Every significant choice in the schema is justified below.

Decision	Reasoning
NUMERIC(12,2) for all monetary columns	Float types (FLOAT, DOUBLE) cannot exactly represent most decimal fractions. 19.99 becomes 19.989999999998... Financial applications must use NUMERIC to avoid rounding errors accumulating across calculations.
inventory_transactions is append-only	Never UPDATE or DELETE rows — only INSERT. This creates a complete, tamper-evident audit trail. Current stock in inventory is the authoritative fast-read value; the transaction log lets you reconstruct any historical state or compute velocity.
Two separate tables: inventory (current) vs inventory_transactions (log)	Separating current state from history is a standard CQRS pattern. inventory gives O(1) stock lookups. inventory_transactions enables time-series analytics without scanning all rows every time.
UNIQUE (product_id, warehouse_id) on inventory	Without this, multiple rows could represent the same product in the same warehouse — queries would need SUM(quantity) everywhere and updates would be ambiguous. The compound unique key enforces one canonical row per slot.
CHECK (quantity >= 0) on inventory	A negative stock level is physically meaningless. The CHECK constraint is a last-resort guard at the DB level even if application code somehow allows it.
bundle_items uses composite PK, no surrogate key	The relationship itself (bundle A contains component B) is the identity — a surrogate id would add no value. Composite PK also implicitly indexes both columns.
CHECK (bundle_product_id != component_product_id)	Prevents a product from being its own component — which would create an infinite recursion when computing bundle stock.
ON DELETE CASCADE for warehouses -> companies	If a company is deleted (GDPR, churn), all their warehouses should go too. Products are not cascaded — they are soft-deleted via is_active to preserve order history.
unit_price on order_items (price snapshot)	Storing the price at time of sale protects historical revenue data from being corrupted if the product's current price changes later. This is a standard e-commerce pattern.
Indexes on (product_id, created_at) for transactions and order_items	The most common analytics queries filter by product over a date range. A composite index on these two columns allows PostgreSQL to use an Index Range Scan rather than a full table scan.
is_preferred on product_suppliers	A product may have multiple approved suppliers for redundancy. is_preferred enables a simple ORDER BY is_preferred DESC to get the primary supplier first without application-level sorting logic.
Surrogate integer keys everywhere (SERIAL)	Natural keys (SKU, email) change over time and make JOINs fragile. Surrogate integer keys are stable, compact (4 bytes vs VARCHAR), and fast to join on.


5. Assumptions Made
The following assumptions were documented due to incomplete requirements. Each would be verified with the product team before production deployment.

#	Assumption	Impact If Wrong
1	PostgreSQL is the target database.	SERIAL becomes IDENTITY or AUTO_INCREMENT; NUMERIC behaviour is consistent across DBs but syntax differs.
2	SKUs are unique platform-wide (not per company).	If per-company, the UNIQUE constraint changes to UNIQUE (sku, company_id).
3	Suppliers are global/shared entities, not company-owned.	If company-owned, suppliers needs company_id FK and all supplier queries need tenant scoping.
4	A users table exists (for created_by FK in inventory_transactions).	If not yet built, created_by should be nullable or dropped until users is defined.
5	Bundle inventory is tracked as physical stock, not computed from components.	If computed, the inventory table has no row for bundles and stock checks require recursive CTEs.
6	Soft-delete (is_active) is preferred over hard-delete.	If hard-delete is used, ON DELETE CASCADE rules would need to expand to protect related rows.
7	Prices and costs are single-currency (INR or USD — not specified).	If multi-currency, add currency_code columns and a FX rate table.
8	low_stock_threshold is per product (not per product-warehouse combination).	If per warehouse-product, the column moves to inventory and Part 3 queries change accordingly.
9	An order line (order_items) can reference a specific warehouse for fulfilment.	If orders are warehouse-agnostic, remove warehouse_id from order_items.
10	transaction_type values are: sale, restock, adjustment, transfer_in, transfer_out, write_off.	Additional types may be needed (e.g. return, sample, damage) — expand the allowed value set.

 
Part 3: API Implementation — Low-Stock Alerts
•  Stack: Node.js / Express / PostgreSQL (pg)

1. Approach & Architecture
The endpoint follows a clean three-layer pattern: authentication middleware validates the caller, a service layer runs a single optimised SQL query using CTEs, and a formatter maps the raw rows to the expected JSON shape. No ORM is used — raw SQL gives full control over the query plan and makes the CTE logic transparent.

•	Single SQL round-trip — all filtering, joining, and aggregation happen inside one CTE query; no N+1 loops.
•	CTE 1 (recent_sales) computes per-product-warehouse average daily sales over the last 30 days.
•	CTE 2 (preferred_supplier) picks one supplier per product — is_preferred first, any other as fallback.
•	Main SELECT joins inventory to both CTEs, filters where quantity < low_stock_threshold, and computes days_until_stockout inline.
•	All parameters are passed as $1/$2 placeholders — no string concatenation, no SQL injection risk.
•	The route handler owns HTTP concerns only (auth check, status codes, JSON serialisation). Business logic lives in the service layer.


2. Project File Structure
The implementation is split into three files following Express best-practice separation of concerns.


stockflow/
  src/
    middleware/
      auth.js          // JWT validation, populates req.user
    routes/
      alerts.js        // Route handler — HTTP layer only
    services/
      alertService.js  // Business logic + SQL query
    db/
      pool.js          // pg connection pool (shared singleton)
  app.js               // Express app setup



3. Database Pool — db/pool.js
A shared pg connection pool. Exported as a singleton so all services reuse the same pool across requests.


// db/pool.js
// Shared PostgreSQL connection pool.
// Using a singleton avoids creating a new connection per request.
 
const { Pool } = require('pg');
 
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // Assumption: DATABASE_URL is set in environment variables.
  // Example: postgres://user:pass@localhost:5432/stockflow
  max: 20,             // max concurrent connections in pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
 
// Propagate pool errors to prevent silent crashes
pool.on('error', (err) => {
  console.error('Unexpected PostgreSQL pool error:', err);
});
 
module.exports = pool;



4. Auth Middleware — middleware/auth.js
Validates the Bearer JWT, extracts the company_id claim, and attaches it to req.user. Every protected route uses this middleware. It is the only place in the codebase that reads the token.


// middleware/auth.js
// Validates JWT Bearer token and populates req.user.
// Assumption: JWT payload contains { userId, companyId }.
 
const jwt = require('jsonwebtoken');
 
function requireAuth(req, res, next) {
  const authHeader = req.headers['authorization'];
 
  // Must be 'Bearer <token>'
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing or malformed Authorization header' });
  }
 
  const token = authHeader.split(' ')[1];
 
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = {
      userId:    payload.userId,
      companyId: payload.companyId,   // used for tenant scoping in every query
    };
    next();
  } catch (err) {
    // Expired or tampered token
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}
 
module.exports = { requireAuth };



5. Service Layer — services/alertService.js
Contains the core business logic and the single optimised SQL query. Every comment explains the why, not just the what.


// services/alertService.js
// Business logic for low-stock alerts.
// Single SQL query using CTEs — no N+1, no ORM magic.
 
const pool = require('../db/pool');
 
// Assumption: 'recent' means sales in the last 30 days.
// This is configurable via the RECENT_SALES_DAYS env var.
const RECENT_SALES_DAYS = parseInt(process.env.RECENT_SALES_DAYS || '30', 10);
 
/**
 * Returns low-stock alerts for all active products across all
 * warehouses belonging to the given company.
 *
 * Only products with at least one sale in the last RECENT_SALES_DAYS
 * days are included — purely dormant stock is not alerted.
 *
 * @param {number} companyId - The tenant company ID (from JWT, not URL)
 * @returns {Promise<Array>} Array of alert objects
 */
async function getLowStockAlerts(companyId) {
  // ── CTE 1: recent_sales ──────────────────────────────────────
  // Aggregates sales per product+warehouse over the lookback window.
  // avg_daily_sales is used to estimate days_until_stockout.
  // INNER JOIN here acts as a filter — products with zero recent
  // sales will have no row in this CTE and will be excluded from
  // the final result set automatically.
  //
  // ── CTE 2: preferred_supplier ────────────────────────────────
  // Gets one supplier per product using DISTINCT ON (PostgreSQL-specific).
  // ORDER BY is_preferred DESC means TRUE rows sort first, so the
  // preferred supplier wins. If none is marked preferred, any
  // supplier is returned (fallback). LEFT JOIN in main query
  // means products with no supplier still appear with supplier = null.
  //
  // ── Main SELECT ───────────────────────────────────────────────
  // Joins inventory to products, warehouses, both CTEs.
  // WHERE clause enforces: low stock + active product + active warehouse
  //   + tenant isolation (both warehouse and product scoped to company).
  // ROUND(quantity / NULLIF(avg_daily_sales, 0)) = days_until_stockout.
  //   NULLIF prevents division-by-zero if avg rounds to 0.
  //   CAST to FLOAT ensures decimal division, not integer truncation.
 
  const query = `
    WITH recent_sales AS (
      SELECT
        oi.product_id,
        oi.warehouse_id,
        -- Total units sold divided by window = avg units per day
        SUM(oi.quantity)::FLOAT / $2   AS avg_daily_sales
      FROM order_items oi
      JOIN orders o ON o.id = oi.order_id
      WHERE
        o.company_id  = $1
        AND o.created_at >= NOW() - ($2 || ' days')::INTERVAL
        AND o.status   != 'cancelled'   -- exclude cancelled orders
      GROUP BY oi.product_id, oi.warehouse_id
    ),
 
    preferred_supplier AS (
      -- DISTINCT ON picks one row per product_id.
      -- ORDER BY is_preferred DESC puts TRUE first -> preferred wins.
      SELECT DISTINCT ON (ps.product_id)
        ps.product_id,
        s.id            AS supplier_id,
        s.name          AS supplier_name,
        s.contact_email AS supplier_email
      FROM product_suppliers ps
      JOIN suppliers s ON s.id = ps.supplier_id
      ORDER BY ps.product_id, ps.is_preferred DESC
    )
 
    SELECT
      p.id                       AS product_id,
      p.name                     AS product_name,
      p.sku,
      w.id                       AS warehouse_id,
      w.name                     AS warehouse_name,
      inv.quantity               AS current_stock,
      p.low_stock_threshold      AS threshold,
      -- NULL if avg_daily_sales is zero (avoid division-by-zero)
      ROUND(inv.quantity::FLOAT / NULLIF(rs.avg_daily_sales, 0))
                                 AS days_until_stockout,
      ps.supplier_id,
      ps.supplier_name,
      ps.supplier_email
    FROM inventory inv
    JOIN products   p  ON p.id  = inv.product_id
    JOIN warehouses w  ON w.id  = inv.warehouse_id
    -- INNER JOIN: only products with recent sales survive
    JOIN recent_sales rs
      ON rs.product_id   = inv.product_id
      AND rs.warehouse_id = inv.warehouse_id
    -- LEFT JOIN: supplier is optional — alert still fires if missing
    LEFT JOIN preferred_supplier ps ON ps.product_id = p.id
    WHERE
      w.company_id   = $1          -- tenant isolation for warehouses
      AND p.company_id = $1        -- tenant isolation for products
      AND p.is_active  = TRUE      -- skip deactivated products
      AND w.is_active  = TRUE      -- skip closed warehouses
      AND inv.quantity < p.low_stock_threshold  -- the core alert condition
    ORDER BY inv.quantity ASC      -- most urgent (lowest stock) first
  `;
 
  const { rows } = await pool.query(query, [companyId, RECENT_SALES_DAYS]);
 
  // Map flat DB rows to the nested JSON shape expected by the response
  return rows.map(row => ({
    product_id:          row.product_id,
    product_name:        row.product_name,
    sku:                 row.sku,
    warehouse_id:        row.warehouse_id,
    warehouse_name:      row.warehouse_name,
    current_stock:       row.current_stock,
    threshold:           row.threshold,
    // days_until_stockout is null if avg_daily_sales ~ 0
    days_until_stockout: row.days_until_stockout !== null
      ? parseInt(row.days_until_stockout, 10)
      : null,
    supplier: row.supplier_id ? {
      id:            row.supplier_id,
      name:          row.supplier_name,
      contact_email: row.supplier_email,
    } : null,
  }));
}
 module.exports = { getLowStockAlerts };
6. Route Handler — routes/alerts.js
The route handler is intentionally thin. It owns only HTTP concerns: parsing the URL param, checking authorisation, calling the service, and formatting the response. No SQL or business logic lives here.


// routes/alerts.js
// HTTP layer for alert endpoints.
// Thin handler — business logic lives in alertService.
 
const express  = require('express');
const router   = express.Router();
const { requireAuth }        = require('../middleware/auth');
const { getLowStockAlerts }  = require('../services/alertService');
 
/**
 * GET /api/companies/:company_id/alerts/low-stock
 *
 * Returns low-stock alerts for the specified company.
 * The caller must be authenticated AND must belong to that company.
 */
router.get(
  '/companies/:company_id/alerts/low-stock',
  requireAuth,   // 1. Validate JWT, populate req.user
  async (req, res) => {
    try {
      const requestedCompanyId = parseInt(req.params.company_id, 10);
 
      // ── Authorisation check ────────────────────────────────
      // Even if the JWT is valid, the user may only query their
      // own company. Prevents horizontal privilege escalation.
      if (isNaN(requestedCompanyId)) {
        return res.status(400).json({ error: 'company_id must be a valid integer' });
      }
 
      if (req.user.companyId !== requestedCompanyId) {
        // 403 not 404 — the resource exists, but access is denied
        return res.status(403).json({ error: 'Access denied to this company' });
      }
 
      // ── Delegate to service layer ──────────────────────────
      const alerts = await getLowStockAlerts(requestedCompanyId);
 
      // ── Success response ───────────────────────────────────
      return res.status(200).json({
        alerts,
        total_alerts: alerts.length,
      });
 
    } catch (err) {
      // Log full error server-side; send safe message to client
      console.error(`[low-stock] company=${req.params.company_id}`, err);
      return res.status(500).json({ error: 'Internal server error' });
    }
  }
);
 
module.exports = router;



7. Express App Entry Point — app.js
Wires together middleware and routes. The global error handler is a last-resort catch-all for unhandled promise rejections not caught at the route level.


// app.js
const express      = require('express');
const alertsRouter = require('./src/routes/alerts');
 
const app = express();
 
app.use(express.json());   // parse JSON request bodies
 
// Mount routes under /api prefix
app.use('/api', alertsRouter);
 
// Global last-resort error handler
// Catches any error passed via next(err)
app.use((err, req, res, _next) => {
  console.error('[unhandled]', err);
  res.status(500).json({ error: 'Internal server error' });
});
 
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`StockFlow API running on port ${PORT}`));
 
module.exports = app;  // export for testing



8. Sample Response
Example of the JSON response this endpoint produces for a company with two low-stock alerts:


// GET /api/companies/42/alerts/low-stock
// HTTP 200 OK
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    },
    {
      "product_id": 200,
      "product_name": "Gear B",
      "sku": "GER-002",
      "warehouse_id": 457,
      "warehouse_name": "South Depot",
      "current_stock": 2,
      "threshold": 15,
      "days_until_stockout": null,
      "supplier": null
      // null: no supplier linked — purchasing team must act manually
    }
  ],
  "total_alerts": 2
}



9. Edge Cases Handled
The following edge cases were identified and handled in the implementation:

Edge Case	Severity	How It Is Handled
Product has no supplier linked	Medium	LEFT JOIN on preferred_supplier — supplier field returns null. Alert still fires so the team knows to reorder manually.
avg_daily_sales rounds to zero	Medium	NULLIF(avg_daily_sales, 0) in the division prevents a division-by-zero error. days_until_stockout returns null — serialised as null in JSON.
Caller queries another company's data	Critical	After JWT validation, req.user.companyId is compared to the URL param. Mismatch returns 403 immediately — no DB query is executed.
company_id is non-numeric in URL	High	parseInt check returns NaN — caught before the DB call and returns 400 Bad Request.
Product active but warehouse deactivated	High	WHERE w.is_active = TRUE filters out closed warehouses — no ghost alerts for non-operational locations.
Product in multiple warehouses	Medium	One alert row per warehouse is returned intentionally — each location needs its own restock action.
All sales in window were cancelled orders	Medium	WHERE o.status != 'cancelled' excludes cancelled orders from avg_daily_sales — prevents artificially inflated or false sales velocity.
DB connection pool exhausted	High	pg pool throws — caught by the try/catch in the route handler, logs the error, returns 500 with a safe message.
Company has no warehouses	Low	Query returns zero rows — response is { alerts: [], total_alerts: 0 }. No error.
Product has recent sales but stock is zero	High	0 < threshold is TRUE — zero-stock items correctly appear as the most urgent alerts (ORDER BY quantity ASC puts them first).


10. Assumptions Made
The following assumptions were made due to incomplete requirements and are documented for the live session discussion.

#	Assumption	Impact If Wrong / Alternative
1	Node.js with Express and the pg library is the target stack.	If using Sequelize or Knex, the raw SQL CTE would be wrapped in a query builder call — logic unchanged.
2	'Recent sales activity' = at least 1 sale in the last 30 days. Configurable via RECENT_SALES_DAYS env var.	If the product team defines a different window (7 days, 90 days), only the env var changes — no code change.
3	low_stock_threshold is a column on the products table (per-product, not per-warehouse).	If threshold is per product-warehouse, it moves to the inventory table and the WHERE clause changes to inv.quantity < inv.low_stock_threshold.
4	JWT payload contains { userId, companyId }.	If the auth system uses a different shape, the auth middleware decode logic changes — the route and service are unaffected.
5	Cancelled orders are excluded from sales velocity calculations.	If returns/refunds should also be excluded, add AND o.status NOT IN ('cancelled', 'returned').
6	A product with multiple suppliers uses is_preferred = TRUE to identify the primary contact. Falls back to any supplier.	If multiple preferred suppliers exist (bad data), DISTINCT ON picks one arbitrarily — a DB-level CHECK should enforce at most one preferred supplier per product.
7	Alerts are ordered by current_stock ASC — most urgent (lowest stock) shown first.	If the product team wants ordering by days_until_stockout instead, change ORDER BY to days_until_stockout ASC NULLS LAST.
8	Pagination is not required for this endpoint.	If large catalogues make responses slow, add LIMIT $3 OFFSET $4 query params and return a total_count from a COUNT(*) sub-query.
9	Bundle products are tracked as physical inventory (not computed from components).	If computed, bundles need a recursive CTE to derive effective stock from component quantities.
10	DATABASE_URL and JWT_SECRET are available as environment variables.	Standard 12-factor app convention — if using a secrets manager, the pool.js and auth.js config loading changes but the rest is unaffected.


11. Key Reasoning Summary
The most important design decisions and why they matter — prepared for the live walkthrough session.

•	Single CTE query over multiple queries: Running one SQL round-trip eliminates N+1 problems. For a company with 500 low-stock products, a naive approach would run 500 supplier lookups. The CTE collapses this to one.
•	INNER JOIN to recent_sales as the activity filter: Using an INNER JOIN (not a WHERE EXISTS subquery) is the cleanest way to exclude products with no recent sales — any product not in recent_sales simply has no matching row and disappears from the result set.
•	NULLIF for division safety: avg_daily_sales can technically be zero if all quantities sold round down to zero after the FLOAT division. NULLIF(avg, 0) makes the division return NULL instead of throwing a runtime error — null is a valid and meaningful response ('we cannot estimate this').
•	LEFT JOIN for suppliers: Making the supplier join outer (LEFT) means a product with no supplier record still generates an alert. A missing supplier is a data quality issue, not a reason to suppress the alert — the purchasing team needs to see it.
•	Authorisation at route level, tenant scoping at DB level: The 403 check in the route stops the request before any DB work. The WHERE company_id = $1 in the SQL is a second line of defence — belt-and-suspenders for tenant isolation.
•	Service / route separation: Keeping SQL out of the route handler makes the service independently testable. A unit test can call getLowStockAlerts() directly without spinning up an HTTP server.
•	ORDER BY quantity ASC: The most urgent items (closest to stockout) appear first. Operations teams scanning the alert list see critical items at the top without having to sort.

