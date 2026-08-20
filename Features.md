# Inventory Management System

Role-based tracking of products, stock, suppliers, customers, purchases, and sales.

## Roles

- **Super Admin** — everything, incl. users and settings
- **Admin** — catalog, suppliers, customers, PO approval
- **Storekeeper** — receiving, stock counts, adjustments
- **Sales Staff** — customers, sales orders

---

## 1. Authentication & Authorization

**Login** — anyone
- Verify hash → check active and unlocked → issue access + refresh token
- Generic error on failure; never reveal if email exists
- Lock account after repeated failures

**Logout** — any user
- Delete refresh token; access token expires in minutes

**User Management** — Super Admin (Admin: view only)
- Create: name, email, role → setup link → user sets own password
- Change role: takes effect on next token refresh
- Deactivate: mark inactive + delete refresh tokens → logged out everywhere
- Email unique; can't demote self; one active Super Admin must remain

**RBAC** — system, every request
- Verify token → load permissions → check route permission → allow or 403
- Check by permission name (`product.delete`), not role name

---

## 2. Product Management

**Add** — Admin+
- Fields: SKU, name, category, supplier, cost price, selling price, quantity, reorder level, unit
- SKU unique via DB constraint
- Opening quantity writes an `ADJUSTMENT`

**Update** — Admin+
- Quantity **not editable here** — only via Stock Adjustment
- Past orders unaffected (they snapshot price)

**Delete** — Admin+
- Referenced by an order → mark inactive
- Never referenced and stock zero → remove
- Blocked if holding stock or on an open order

**View / View All** — all roles
- Paginated, active-only default; filter by category, supplier, stock status
- Sales Staff: no cost price

**Search** — all roles
- Min 2 chars, debounced, partial match on SKU + name

---

## 3. Category Management

**Create / Update** — Admin+
- Name required and unique
- Products link by id, so renaming is safe

**Delete** — Admin+
- Products in category → block, report count
- Empty → mark inactive

**View Categories** — all roles
- Active categories with product counts

**View Products by Category** — all roles
- Product list filtered by category

---

## 4. Supplier Management

**Add / Update** — Admin+
- Fields: company, contact, email, phone, address, payment terms
- Duplicate check warns, doesn't block

**Delete** — Admin+
- Open POs → block and list them
- Else mark inactive

**View / Search** — Admin+, Storekeeper (read-only)
- Detail: contacts, products sourced, PO history
- Search on name, contact, email, phone
- Sales Staff: no access (exposes cost data)

---

## 5. Stock Management

Not a separate table — all views filter products by quantity vs reorder level.

**Views** — all roles
- Current stock: all active products + status label
- In stock: `qty > reorder_level`
- Low stock: `0 < qty <= reorder_level`, urgent first
- Out of stock: `qty = 0`
- From low stock → generate draft PO grouped by supplier

**Stock Adjustment** — Storekeeper, Admin+
- Enter counted quantity → pick reason → system writes signed `ADJUSTMENT` → updates quantity
- Reasons: Damaged, Expired, Lost/Stolen, Count Correction, Opening Balance
- Reason mandatory — only place stock moves without an order behind it

---

## 6. Purchase Order Management

A PO is a promise to buy, not inventory.

**Create** — Admin+
- Select supplier → add lines (product, qty, unit cost) → total recalcs → draft → approve
- Supplier active, ≥1 line, qty > 0
- **No stock effect**

**Update** — Admin+
- Draft: everything
- Pending approval: everything, approval resets
- Approved: notes and dates only
- Received / partially received / cancelled: nothing

**Receive** — Storekeeper
- Enter qty actually received per line (may be partial, may span deliveries)
- One transaction: bump received qty, increase stock, write `PURCHASE` per line
- All lines complete → received; else partially received
- Received never exceeds ordered
- **Only way stock enters via purchasing**

**Cancel** — Admin+
- Nothing received → cancelled, no stock effect
- Partially received → cancel outstanding balance only
- Fully received → not cancellable, use a return

**View** — Admin+ (all), Storekeeper (approved, for receiving)
- Filter by status, supplier, date; detail shows ordered vs received per line

**Purchase Return** — Storekeeper raises, Admin approves
- Select lines from received order → qty + reason → stock decreases, `RETURN_OUT` written

---

## 7. Sales / Order Management

**Create** — Sales Staff, Admin+
- Select or inline-create customer → add lines → available stock shown per line
- On confirm, one transaction: lock rows, re-check stock, decrease stock, write `SALE` per line
- Any line fails → full rollback
- **Snapshot price onto the line** — else a future price change rewrites old invoices
- **Lock rows** — two staff selling the last unit is a real race condition

**Update** — Sales Staff (own), Admin+
- Reverse the whole original stock effect, then reapply the new version, in one transaction
- Easier to verify than per-line deltas; handles added and removed lines identically
- Blocked once shipped, delivered, or cancelled

**Cancel** — Sales Staff (own, pre-ship), Admin+
- Return full qty to stock, write `CANCELLATION` per line
- Original `SALE` rows never edited
- Delivered → not cancellable, use a return

**View** — Sales Staff (own), Admin+ (all)
- Filter by status, customer, date; detail shows snapshot prices and linked transactions

**Sales Return** — Sales Staff raises, Admin approves
- Select lines from delivered order → qty, reason, resaleable flag
- Resaleable → stock increases, `RETURN_IN`
- Damaged → no stock increase, `ADJUSTMENT` write-off

---

## 8. Customer Management

**Add / Update** — Sales Staff, Admin+
- Fields: name, email, phone, billing + shipping address
- Duplicate check on email and phone
- Can be created inline from the sales order screen
- Past orders keep their shipped-to address

**Delete** — Admin+ only
- Open orders → block and list them; else mark inactive
- Sales Staff deliberately cannot delete

**View / Search** — Sales Staff, Admin+
- Detail: contacts, order history, lifetime value, last order date
- Search on name, email, phone

---

## 9. Inventory Transaction Management

Append-only ledger. Never update or delete a row — corrections are new rows.

Each row: product, type, **signed** quantity, reference, user, reason, timestamp.
Signed quantity means a product's balance is a plain sum.

**Products added (positive)**
- `PURCHASE` — PO received
- `RETURN_IN` — customer return, resaleable
- `ADJUSTMENT` — upward correction
- `CANCELLATION` — confirmed sale cancelled

**Products sold (negative)**
- `SALE` — sales order confirmed
- `RETURN_OUT` — return to supplier
- `ADJUSTMENT` — downward correction

**Returns**
- Both directions recorded here, always referencing the order that caused them

**View history** — Admin+ and Storekeeper (all, read-only), Sales Staff (own)
- Filter by product, type, date, user; each row links back to its source order
- Per-product running balance traces drift to a specific entry
- Ledger sum must equal stored quantity — mismatch means something bypassed the stock service

---

## 10. Dashboard

Widgets the user can't access are omitted server-side.

**Summary**
- Total products, total categories — all roles
- Total suppliers — Admin+, Storekeeper
- Total stock quantity — all roles

**Stock alerts** — all roles
- Low stock, out of stock
- Clickable → filtered stock list → draft PO

**Recent activity**
- Today's sales, recent sales orders — Sales, Admin+
- Recent purchase orders — Storekeeper, Admin+

**Notes**
- Run widget queries concurrently behind one endpoint; cache briefly
- Summing stock across mixed units (kg + pieces + boxes) is meaningless — use one unit or show stock value instead

---

## Order Statuses

**Purchase order**
- Draft → pending approval → approved → partially received → received
- Cancellable any time before full receipt
- Stock increases only on partially received or received

**Sales order**
- Draft → confirmed → shipped → delivered → returned
- Cancellable before shipping
- Stock decreases on confirmed; restored on cancelled and on resaleable returns

---

## Permissions

- **Users** — Super Admin full; Admin view
- **Products, categories** — Admin+ full; others view (Sales Staff without cost price)
- **Suppliers** — Admin+ full; Storekeeper view; Sales Staff none
- **Customers** — Admin+ full; Sales Staff create/update/view
- **Stock** — Admin+ full; Storekeeper view + adjust; Sales Staff view
- **Purchase orders** — Admin+ full; Storekeeper receive + return
- **Sales orders** — Admin+ full; Sales Staff create/update/cancel own
- **Transactions** — Super Admin full; Admin and Storekeeper view; Sales Staff own
- **Dashboard** — Admin+ all; Storekeeper stock widgets; Sales Staff sales widgets

---

## Cross-Cutting Rules

- **One stock service.** Nothing updates quantity directly — keeps ledger and cached quantity in sync
- **Atomic multi-step operations.** Order, lines, quantities, and ledger succeed or fail together
- **Snapshot prices and addresses onto orders.** Historical documents must not change retroactively
- **Soft deletes.** Anything on an order is deactivated, never removed
- **Server is the only authority** on totals, permissions, and stock availability