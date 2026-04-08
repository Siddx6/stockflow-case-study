# StockFlow – Inventory Management System (B2B SaaS)

**Backend Engineering Intern Case Study Submission**  
**Submitted by:** Siddharth Kumar  
**Date:** April 2026  

---

# Part 1: Code Review & Debugging

## 🔍 Issues Identified

1. No input validation → crashes on missing fields  
2. No SKU uniqueness check → duplicate products  
3. Multiple DB commits → inconsistent state  
4. No error handling → server crashes  
5. No authentication → security risk  
6. Float used for price → precision issues  
7. warehouse_id not validated → invalid references  
8. Wrong HTTP status code  

---

## ⚠️ Impact

- Data corruption (duplicate SKUs)  
- Inconsistent database state  
- Security vulnerabilities  
- Poor API reliability  
- Financial inaccuracies  

---

## ✅ Fixed Implementation

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

    required = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [f for f in required if f not in data]
    if missing:
        return jsonify({'error': f'Missing fields: {missing}'}), 400

    try:
        price = Decimal(str(data['price']))
        if price < 0:
            raise ValueError
    except:
        return jsonify({'error': 'Invalid price'}), 400

    warehouse = Warehouse.query.filter_by(
        id=data['warehouse_id'],
        company_id=g.current_user.company_id
    ).first()

    if not warehouse:
        return jsonify({'error': 'Invalid warehouse'}), 404

    if Product.query.filter_by(sku=data['sku'].upper()).first():
        return jsonify({'error': 'SKU exists'}), 409

    try:
        product = Product(
            name=data['name'],
            sku=data['sku'].upper(),
            price=price,
            company_id=g.current_user.company_id
        )
        db.session.add(product)
        db.session.flush()

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



---

# Part 2: Database Design

## 🏗️ Schema Overview

The database is designed for a **multi-tenant B2B SaaS system** where each company manages its own warehouses, products, and inventory independently.

---

## 📦 Core Tables

### companies
- id (PK)
- name
- created_at

👉 Root tenant entity. All data is scoped to a company.

---

### warehouses
- id (PK)
- company_id (FK → companies.id)
- name
- is_active

👉 A company can have multiple warehouses.

---

### products
- id (PK)
- company_id (FK)
- name
- sku (UNIQUE)
- price (NUMERIC)
- low_stock_threshold
- is_active

👉 SKU is globally unique. Price uses NUMERIC to avoid float precision errors.

---

### inventory
- product_id (FK)
- warehouse_id (FK)
- quantity
- UNIQUE(product_id, warehouse_id)

👉 Represents current stock for each product in each warehouse.

---

### inventory_transactions
- id (PK)
- product_id
- warehouse_id
- change_qty
- created_at

👉 Append-only log to track inventory changes over time.

---

### suppliers
- id (PK)
- name
- contact_email

---

### product_suppliers
- product_id (FK)
- supplier_id (FK)
- is_preferred

👉 Supports multiple suppliers per product.

---

### orders
- id (PK)
- company_id
- created_at
- status

---

### order_items
- id (PK)
- order_id (FK)
- product_id (FK)
- warehouse_id (FK)
- quantity
- created_at

👉 Used to calculate sales velocity for alerts.

---

## ❓ Missing Requirements

1. Is SKU uniqueness global or per company?  
2. What defines "recent sales" (7, 30, 90 days)?  
3. Is low_stock_threshold per product or per warehouse?  
4. Are suppliers global or company-specific?  
5. How should bundle products calculate stock?  
6. Are product transfers between warehouses required?  
7. Is multi-currency needed?  
8. Is soft delete or hard delete expected?  

---

## 🧠 Design Decisions

- **NUMERIC for price** → prevents floating point errors in financial calculations  
- **inventory + transactions separation** → fast reads + full audit trail  
- **UNIQUE(product_id, warehouse_id)** → prevents duplicate inventory rows  
- **Append-only transaction log** → ensures historical integrity  
- **Indexes on product_id, created_at** → optimizes analytics queries  

---

## ⚠️ Assumptions

- SKUs are globally unique  
- Suppliers are shared across companies  
- Low stock threshold is per product  
- Orders represent completed sales activity  
- Inventory cannot be negative  

---

# Part 3: Low Stock Alerts API

## 🚀 Endpoint

GET /api/companies/:company_id/alerts/low-stock

---

## 🧠 Approach

The endpoint is designed to return low-stock alerts based on:

- Current inventory levels  
- Product-specific thresholds  
- Recent sales activity  
- Supplier availability  

---

## ⚙️ Implementation Logic

1. **Filter by company**
   - Ensure tenant isolation using company_id

2. **Join inventory with products & warehouses**
   - Get current stock and metadata

3. **Filter low stock**
   - `quantity < low_stock_threshold`

4. **Filter recent sales**
   - Only include products with sales in last 30 days

5. **Calculate sales velocity**
   - `avg_daily_sales = total_sold / days`

6. **Estimate stockout**
   - `days_until_stockout = quantity / avg_daily_sales`

7. **Attach supplier info**
   - Prefer supplier marked as `is_preferred`

---

## 💻 Implementation (Node.js)

```javascript
router.get('/companies/:company_id/alerts/low-stock', requireAuth, async (req, res) => {
  try {
    const companyId = parseInt(req.params.company_id);

    if (isNaN(companyId)) {
      return res.status(400).json({ error: 'Invalid company_id' });
    }

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

