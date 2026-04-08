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
Part 2: Database Design
🏗️ Core Tables
companies
id (PK)
name
created_at
warehouses
id (PK)
company_id (FK)
name
products
id (PK)
company_id (FK)
name
sku (UNIQUE)
price (NUMERIC)
low_stock_threshold
inventory
product_id (FK)
warehouse_id (FK)
quantity
UNIQUE(product_id, warehouse_id)
inventory_transactions
id
product_id
warehouse_id
change_qty
created_at
suppliers
id
name
contact_email
product_suppliers
product_id
supplier_id
orders / order_items
used to track sales activity
❓ Missing Requirements
SKU uniqueness scope (global vs per company)
Definition of “recent sales”
Threshold per product or per warehouse
Supplier ownership model
Bundle stock calculation logic
🧠 Design Decisions
Used NUMERIC for price to avoid floating point errors
Separated inventory (current state) from transactions (history)
Enforced composite keys to prevent duplicates
Indexed frequently queried fields
Part 3: Low Stock Alerts API
🚀 Endpoint

GET /api/companies/:company_id/alerts/low-stock

🧠 Approach
Use SQL CTEs to calculate:
recent sales
supplier mapping
Filter:
low stock items
active products
recent activity
Compute:
days until stockout


⚙️ Implementation (Node.js)
router.get('/companies/:company_id/alerts/low-stock', requireAuth, async (req, res) => {
  const companyId = parseInt(req.params.company_id);

  if (req.user.companyId !== companyId) {
    return res.status(403).json({ error: 'Access denied' });
  }

  const alerts = await getLowStockAlerts(companyId);

  res.json({
    alerts,
    total_alerts: alerts.length
  });
});


⚠️ Edge Cases
No supplier → return null
No recent sales → excluded
Division by zero → handled safely
Cross-company access → blocked


📊 Sample Response
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "current_stock": 5,
      "threshold": 20
    }
  ],
  "total_alerts": 1
}


Assumptions--

Recent sales = last 30 days
SKUs are globally unique
JWT contains companyId
Suppliers are shared across companies
Final Notes
Focused on correctness and scalability
Designed for real-world backend scenarios
Handles important edge cases