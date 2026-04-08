# StockFlow – Inventory Management System (B2B SaaS)

**Backend Engineering Intern — Case Study Submission**  
**Submitted by:** Siddharth Kumar  
**Date:** April 2026

---

## Table of Contents

1. [Part 1: Code Review & Debugging](#part-1-code-review--debugging)
2. [Part 2: Database Design](#part-2-database-design)
3. [Part 3: Low Stock Alerts API](#part-3-low-stock-alerts-api)

---

## Part 1: Code Review & Debugging

### Issues Identified

| # | Issue | Risk |
|---|-------|------|
| 1 | No input validation | Crashes on missing fields |
| 2 | No SKU uniqueness check | Duplicate products created |
| 3 | Multiple DB commits | Inconsistent database state |
| 4 | No error handling | Server crashes on exceptions |
| 5 | No authentication | Security vulnerability |
| 6 | Float used for price | Floating-point precision errors |
| 7 | `warehouse_id` not validated | Invalid foreign-key references |
| 8 | Wrong HTTP status code | Incorrect API semantics |

### Impact Summary

- **Data corruption** — duplicate SKUs enter the database
- **Inconsistent state** — partial writes from multiple commits
- **Security vulnerabilities** — unauthenticated endpoint
- **Poor API reliability** — unhandled exceptions crash the server
- **Financial inaccuracies** — float arithmetic errors on prices

---

### Fixed Implementation

```python
from flask import request, jsonify, g
from decimal import Decimal
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
@require_auth
def create_product():
    data = request.get_json()
    if not data:
        return jsonify({'error': 'Invalid JSON'}), 400

    # 1. Validate required fields
    required = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [f for f in required if f not in data]
    if missing:
        return jsonify({'error': f'Missing fields: {missing}'}), 400

    # 2. Validate price (use Decimal to avoid float precision issues)
    try:
        price = Decimal(str(data['price']))
        if price < 0:
            raise ValueError
    except:
        return jsonify({'error': 'Invalid price'}), 400

    # 3. Validate warehouse belongs to the authenticated company
    warehouse = Warehouse.query.filter_by(
        id=data['warehouse_id'],
        company_id=g.current_user.company_id
    ).first()
    if not warehouse:
        return jsonify({'error': 'Invalid warehouse'}), 404

    # 4. Enforce SKU uniqueness
    if Product.query.filter_by(sku=data['sku'].upper()).first():
        return jsonify({'error': 'SKU exists'}), 409

    # 5. Single atomic transaction
    try:
        product = Product(
            name=data['name'],
            sku=data['sku'].upper(),
            price=price,
            company_id=g.current_user.company_id
        )
        db.session.add(product)
        db.session.flush()  # Get product.id without committing

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)
        db.session.commit()

        return jsonify({'product_id': product.id}), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({'error': 'DB error'}), 500
```

---

## Part 2: Database Design

### Schema Overview

Designed for a **multi-tenant B2B SaaS** system where each company independently manages its own warehouses, products, and inventory.

---

### Core Tables

#### `companies`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `name` | TEXT | — |
| `created_at` | TIMESTAMP | — |

> Root tenant entity. All other data is scoped to a company.

---

#### `warehouses`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `company_id` | FK → `companies.id` | Tenant isolation |
| `name` | TEXT | — |
| `is_active` | BOOLEAN | Soft disable |

> A company can have multiple warehouses.

---

#### `products`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `company_id` | FK | Tenant scoping |
| `name` | TEXT | — |
| `sku` | TEXT UNIQUE | Globally unique, uppercase |
| `price` | NUMERIC | Avoids float precision errors |
| `low_stock_threshold` | INTEGER | Alert trigger level |
| `is_active` | BOOLEAN | Soft delete |

---

#### `inventory`
| Column | Type | Notes |
|--------|------|-------|
| `product_id` | FK | — |
| `warehouse_id` | FK | — |
| `quantity` | INTEGER | Current stock level |
| UNIQUE | `(product_id, warehouse_id)` | Prevents duplicate rows |

> Represents real-time stock for each product per warehouse.

---

#### `inventory_transactions`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `product_id` | FK | — |
| `warehouse_id` | FK | — |
| `change_qty` | INTEGER | +/– delta |
| `created_at` | TIMESTAMP | Indexed for analytics |

> Append-only audit log. Never updated or deleted.

---

#### `suppliers`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `name` | TEXT | — |
| `contact_email` | TEXT | — |

---

#### `product_suppliers`
| Column | Type | Notes |
|--------|------|-------|
| `product_id` | FK | — |
| `supplier_id` | FK | — |
| `is_preferred` | BOOLEAN | Used by alert logic |

> Supports multiple suppliers per product.

---

#### `orders`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `company_id` | FK | — |
| `created_at` | TIMESTAMP | — |
| `status` | TEXT | — |

---

#### `order_items`
| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | — |
| `order_id` | FK | — |
| `product_id` | FK | — |
| `warehouse_id` | FK | — |
| `quantity` | INTEGER | Used for sales velocity |
| `created_at` | TIMESTAMP | — |

> Used to calculate sales velocity for low-stock alert logic.

---

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| `NUMERIC` for price | Eliminates floating-point errors in financial calculations |
| Separate `inventory` + `inventory_transactions` | Fast reads on current stock + full audit trail |
| `UNIQUE(product_id, warehouse_id)` | Prevents duplicate inventory rows |
| Append-only transaction log | Preserves historical integrity; nothing is ever deleted |
| Index on `product_id`, `created_at` | Optimises analytics and velocity queries |

---

### Assumptions

- SKUs are **globally unique** across all companies
- Suppliers are **shared** across companies (not tenant-scoped)
- `low_stock_threshold` is **per product** (not per warehouse)
- Orders represent **completed sales** activity
- Inventory quantity **cannot go negative**

---

### Open Questions

1. Is SKU uniqueness **global** or **per company**?
2. What window defines "recent sales" — 7, 30, or 90 days?
3. Is `low_stock_threshold` per product or **per warehouse**?
4. Are suppliers **global or company-specific**?
5. How should **bundle products** calculate available stock?
6. Are **warehouse-to-warehouse transfers** required?
7. Is **multi-currency** support needed?
8. Should deletes be **soft** (flag) or **hard** (removal)?

---

## Part 3: Low Stock Alerts API

### Endpoint

```
GET /api/companies/:company_id/alerts/low-stock
```

**Auth required:** Yes (`requireAuth` middleware)

---

### Alert Logic

The endpoint returns low-stock alerts derived from:

- Current inventory levels vs. per-product thresholds
- Recent sales activity (last 30 days)
- Calculated sales velocity and estimated stockout time
- Preferred supplier contact info

#### Processing Steps

1. **Tenant isolation** — filter all data by `company_id`
2. **Join inventory** with products and warehouses to get current stock and metadata
3. **Filter low stock** — `quantity < low_stock_threshold`
4. **Filter active products** — only include products with sales in the last 30 days
5. **Calculate sales velocity** — `avg_daily_sales = total_sold / days`
6. **Estimate stockout** — `days_until_stockout = quantity / avg_daily_sales`
7. **Attach supplier info** — prefer supplier where `is_preferred = true`

---

### Implementation (Node.js)

```javascript
router.get('/companies/:company_id/alerts/low-stock', requireAuth, async (req, res) => {
  try {
    const companyId = parseInt(req.params.company_id);

    // Validate path param
    if (isNaN(companyId)) {
      return res.status(400).json({ error: 'Invalid company_id' });
    }

    // Enforce tenant isolation — users can only query their own company
    if (req.user.companyId !== companyId) {
      return res.status(403).json({ error: 'Access denied' });
    }

    const alerts = await getLowStockAlerts(companyId);

    return res.status(200).json({
      alerts,
      total_alerts: alerts.length
    });

  } catch (err) {
    console.error(err);
    return res.status(500).json({ error: 'Internal server error' });
  }
});
```

---

*End of submission.*
