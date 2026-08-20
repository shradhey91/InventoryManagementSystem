# Inventory Management System

Role-based system for tracking products, stock, suppliers, customers, purchases, and sales.
Each feature lists who can do it, what happens, and what changes.

## Roles

- **Super Admin** — everything, including users and system settings
- **Admin** — catalog, suppliers, customers, purchase order approval
- **Storekeeper** — receiving goods, stock counts, adjustments, returns
- **Sales Staff** — customers and sales orders

## Global Rules

- Permissions are enforced server-side; hiding a button in the UI is cosmetic only
- Deletes are soft deletes — anything referenced by an order is deactivated, never removed
- Totals, stock availability, and permissions are always recomputed server-side

---

## 1. Authentication & Authorization

**User Login** — anyone
- Verify email exists → compare password hash → check account active and unlocked
- Issue a short-lived access token and a long-lived refresh token
- Record login timestamp, land on the role-filtered dashboard
- On failure show a generic "invalid credentials" message, never revealing whether the email exists
- Increment the failed-attempt counter and lock the account after a threshold

**User Logout** — any logged-in user
- Delete the refresh token row, client discards the access token
- The access token stays valid for its few remaining minutes but can no longer be renewed

**User Management** — Super Admin full, Admin view only
- Create: enter name, email, role → create pending account → email a one-time setup link → user sets their own password
- Change role: assign new role, log old and new values, takes effect on next token refresh
- Deactivate: mark inactive and delete all their refresh tokens, logging them out everywhere
- Email must be unique; a Super Admin cannot demote or deactivate themselves; at least one active Super Admin must always remain

**Role-Based Access Control** — system, on every request
- Verify token signature and expiry → load the role's permission set → check the route's required permission → allow or return 403
- Check permissions by name (`product.delete`), not by role name. Roles are named bundles of permissions, so adding a role later is a data change rather than a code change across every route

---

## 2. Product Management

**Add Product** — Admin and above
- Load active categories and suppliers into dropdowns
- Enter SKU, name, description, category, supplier, cost price, selling price, opening quantity, reorder level, unit of measure
- Validate, create the record, and if opening quantity is above zero write an `ADJUSTMENT` transaction of reason "opening balance"
- SKU must be unique, enforced by a database constraint rather than application code alone
- Category and supplier must be active; prices and reorder level must be zero or greater

**Update Product** — Admin and above
- Load current values → edit → validate → save with an audit entry recording which fields changed
- Quantity is **not editable here**. Stock changes only through Stock Adjustment, so every movement carries a reason and a ledger row
- Existing orders are unaffected because order lines store their own price snapshot

**Delete Product** — Admin and above
- Check whether the product appears on any purchase or sales order
- If referenced, mark it inactive and hide it from selection lists
- If never referenced and stock is zero, the record may be removed outright
- Cannot delete a product holding stock or sitting on an open order

**View Product / View All Products** — all roles
- Paginated, filtered to active products by default
- Filter by category, supplier, or stock status; sort by any column
- Detail view shows attributes, current stock, and recent transaction history
- Sales Staff see selling price but not cost price; Storekeeper sees quantity read-only

**Search Products** — all roles
- Minimum two characters, debounced so the request fires after typing stops
- Partial match against SKU and name, paginated, ordered by relevance then name
- A simple partial match is fine at small scale; add a full-text index when it slows down

---

## 3. Category Management

**Create Category** — Admin and above
- Enter name and optional description, check the name is unique among active categories, create

**Update Category** — Admin and above
- Edit name or description, validate uniqueness, save
- Products keep their link because they reference the category by id, not by name

**Delete Category** — Admin and above
- Count products in the category
- If above zero, block the delete and report how many products must be reassigned first
- If zero, mark the category inactive
- Decide explicitly between blocking and auto-reassigning to an "Uncategorized" fallback, then apply that choice consistently

**View Categories** — all roles
- List all active categories, each with a count of the products it contains

**View Products by Category** — all roles
- Product list filtered by category, with the standard product filters and sorting still available

---

## 4. Supplier Management

**Add Supplier** — Admin and above
- Enter company name, contact person, email, phone, address, payment terms
- Validate email format, run a duplicate check on name and email, warn rather than hard-block

**Update Supplier** — Admin and above
- Edit and save; existing purchase orders are unaffected since they store the address used at order time

**Delete Supplier** — Admin and above
- Check for open purchase orders with this supplier
- If any exist, block and list them; otherwise mark inactive so historical orders keep their link

**View Suppliers** — Admin and above, Storekeeper read-only
- Paginated list; detail shows contact information, products sourced from them, and purchase order history
- Sales Staff have no access, because supplier records expose cost information

**Search Suppliers** — Admin and above, Storekeeper
- Partial match on company name, contact person, email, or phone

---

## 5. Stock Management

Stock is not a separate table. Every view below is the product table filtered by quantity against
reorder level, which is why `reorder_level` is a required field — without it, three of these four
views have no definition.

**View Current Stock** — all roles
- Every active product with quantity, reorder level, and a derived status label
- Filter by category, supplier, or status; export the result

**View In-Stock Products** — all roles
- Filtered to quantity above reorder level

**View Low-Stock Products** — all roles
- Filtered to quantity above zero but at or below reorder level, sorted most urgent first
- Admin can select items here and generate a draft purchase order grouped by supplier. This is the main reason the view exists

**View Out-of-Stock Products** — all roles
- Filtered to quantity of zero
- Sales Staff should check this before promising delivery to a customer

**Stock Adjustment** — Storekeeper, Admin and above
- Open the product, enter the physically counted quantity
- Pick a reason from a fixed list: Damaged, Expired, Lost or Stolen, Count Correction, Opening Balance
- System computes the difference, writes a signed `ADJUSTMENT` transaction with the reason and acting user, then updates the quantity
- Counted quantity must be zero or greater and the reason is mandatory — this is the only place stock changes without an order behind it, so it must never be anonymous

---

## 6. Purchase Order Management

A purchase order is a promise to buy, not inventory. Stock increases only when goods physically
arrive. Getting this wrong makes every in-transit order inflate your stock figures.

**Create Purchase Order** — Admin and above
- Select a supplier, which filters the products that can be added
- Add lines one at a time: product, quantity, unit cost defaulting to the product's cost price
- Total recalculates from the lines on every change
- Save as draft or submit for approval; approval moves it to approved and it is issued to the supplier
- Supplier must be active, at least one line, quantity above zero, unit cost zero or greater
- **Stock does not change at this point**

**Update Purchase Order** — Admin and above
- What may be edited depends on status: everything while draft; everything while pending approval but approval resets; notes and expected date only once approved; nothing once partially received, received, or cancelled
- Validate the transition, recalculate the total, write an audit entry

**Receive Purchase Order** — Storekeeper
- Enter the quantity actually received per line, which may be less than ordered and may arrive across several deliveries
- Inside a single database transaction: increase received quantity per line, increase product stock, write a `PURCHASE` transaction per line referencing this order
- Status becomes received if every line is complete, otherwise partially received
- Cumulative received quantity must never exceed the ordered quantity on any line
- **This is the only way stock enters the system through purchasing**

**Cancel Purchase Order** — Admin and above
- Enter a reason; if nothing has been received the order is cancelled with no stock effect
- If partially received, only the outstanding balance is cancelled — goods already received stay, because they are physically present
- A fully received order cannot be cancelled; raise a purchase return instead

**View Purchase Orders** — Admin and above full, Storekeeper sees approved orders for receiving
- Paginated, filterable by status, supplier, and date range
- Detail shows header, all lines with ordered against received quantities, status history, and linked stock transactions

**Purchase Return** — Storekeeper raises, Admin approves
- Select lines from a received order, enter quantity and reason
- On approval stock decreases and a `RETURN_OUT` transaction is written referencing the original order

---

## 7. Sales / Order Management

**Create Sales Order** — Sales Staff, Admin and above
- Select an existing customer or create one inline
- Add lines: product, quantity, unit price defaulting to the selling price, with available stock shown per line
- On confirm, inside a single database transaction: re-check available stock with the product rows locked, decrease stock per line, write a `SALE` transaction per line, set status to confirmed
- If any line fails the stock check the whole operation rolls back and nothing changes
- **Snapshot the price onto the order line.** Never join to the product for price on an old order, or a future price change would silently rewrite every historical invoice
- **Lock product rows during check-and-decrement.** Two staff selling the last unit at the same moment is a real race condition, not a theoretical one

**Update Sales Order** — Sales Staff for their own orders, Admin and above
- Open a confirmed, unshipped order and modify lines by changing quantity, adding, or removing products
- Reverse the entire original stock effect, then apply the new version, all in one database transaction
- Reversing and reapplying is easier to verify than computing per-line deltas, and added or removed lines go through the same code path
- Order must not be shipped, delivered, or cancelled; new quantities must pass the stock check

**Cancel Sales Order** — Sales Staff for their own orders before shipping, Admin and above
- Enter a reason, return the full quantity of every line to stock
- Write a `CANCELLATION` transaction per line; the original `SALE` rows are never deleted or edited
- A delivered order cannot be cancelled; raise a sales return instead

**View Sales Orders** — Sales Staff see their own, Admin and above see all
- Paginated, filterable by status, customer, and date range
- Detail shows customer, lines with snapshot prices, totals, status history, and linked transactions

**Sales Return** — Sales Staff raises, Admin approves
- Select lines and quantities from a delivered order, enter a reason, mark whether the goods are resaleable
- Resaleable goods increase stock and write a `RETURN_IN` transaction
- Damaged goods do not increase stock; an `ADJUSTMENT` records the write-off
- The return is linked back to the original sales order

---

## 8. Customer Management

**Add Customer** — Sales Staff, Admin and above
- Enter name, email, phone, billing address, shipping address
- Run a duplicate check on email and phone, warning if a match is found
- Can be triggered inline from the sales order screen, so a new customer is not a two-screen detour

**Update Customer** — Sales Staff, Admin and above
- Edit and save; past orders keep the address they were shipped to, since that is snapshotted onto the order

**Delete Customer** — Admin and above only
- Check for open sales orders; if any exist block and list them, otherwise mark inactive
- Sales Staff can create and edit customers but not delete them, so records cannot be quietly removed by whoever created them

**View Customers** — Sales Staff, Admin and above
- Paginated list; detail shows contact information, full order history, total lifetime value, last order date

**Search Customers** — Sales Staff, Admin and above
- Partial match on name, email, or phone, used most often from within the sales order screen

---

## 9. Inventory Transaction Management

This is an append-only ledger. Rows are never updated or deleted — corrections are new rows, the
way accounting works. This is what lets you answer "why does the system say 47 when we counted 45."

Each row records the product, transaction type, a signed quantity, the reference type and id, the
acting user, a reason, and a timestamp. Signed quantity means a product's balance is a plain sum
over its rows.

**Track Products Added** — written automatically by the system, viewed by all roles
- Receiving a purchase order writes `PURCHASE`, positive
- A resaleable customer return writes `RETURN_IN`, positive
- An upward stock adjustment writes `ADJUSTMENT`, positive
- Cancelling a confirmed sale writes `CANCELLATION`, positive

**Track Products Sold** — written automatically, viewed by all roles
- Confirming a sales order writes `SALE`, negative
- A return to a supplier writes `RETURN_OUT`, negative
- A downward stock adjustment writes `ADJUSTMENT`, negative

**Track Product Returns** — written automatically, viewed by all roles
- Both return directions are recorded here and always carry a reference to the order that caused them, so any return traces back to its original transaction

**View Transaction History** — Admin and above and Storekeeper see all read-only, Sales Staff see only their own
- Filter by product, type, date range, or user
- Each row links back to the order or adjustment that produced it
- A per-product view shows a running balance so drift traces to a specific entry
- The sum of a product's ledger rows must equal its stored quantity; a mismatch means something bypassed the stock service, which is worth an automated check

---

## 10. Dashboard

All roles land here on login. Widgets the user lacks permission for are omitted server-side, not
hidden client-side.

**Summary metrics**
- Total products and total categories — counts of active rows, visible to all
- Total suppliers — count of active suppliers, visible to Admin and above and Storekeeper
- Total stock quantity — sum of quantity across active products, visible to all

**Stock alerts**
- Low-stock products — quantity above zero and at or below reorder level, most urgent first
- Out-of-stock products — quantity of zero
- Both are clickable, leading to the filtered stock list and from there to a draft purchase order

**Recent activity**
- Today's sales — sum of today's order totals excluding cancelled, visible to Sales and Admin and above
- Recent sales orders — last five by date descending, visible to Sales and Admin and above
- Recent purchase orders — last five by date descending, visible to Storekeeper and Admin and above

**Notes**
- Ten widgets means ten aggregate queries per page load. Fire them concurrently behind one endpoint and cache the response briefly; two minutes stale is almost always acceptable
- Summing stock quantity across mixed units — kilograms plus pieces plus boxes — produces a meaningless number. Restrict it to a single unit, or show stock value as quantity times cost price instead

---

## Order Statuses

**Purchase order** — draft, pending approval, approved, partially received, received, with cancelled
reachable from any state before receipt. Stock increases only on entering partially received or
received.

**Sales order** — draft, confirmed, shipped, delivered, returned, with cancelled reachable before
shipping. Stock decreases on confirmed, is restored on cancelled, and is restored on returned when
the goods come back resaleable.

---

## Permissions Summary

- **Users** — Super Admin full, Admin view only, others none
- **Products, categories** — Admin and above full, Storekeeper and Sales Staff view only, Sales Staff without cost price
- **Suppliers** — Admin and above full, Storekeeper view, Sales Staff none
- **Customers** — Admin and above full, Sales Staff create, update and view, Storekeeper none
- **Stock** — Admin and above full, Storekeeper view and adjust, Sales Staff view
- **Purchase orders** — Admin and above full, Storekeeper receive and return, Sales Staff none
- **Sales orders** — Admin and above full, Sales Staff create, update and cancel their own, Storekeeper none
- **Transactions** — Super Admin full, Admin and Storekeeper view, Sales Staff view their own
- **Dashboard** — Admin and above see everything, Storekeeper sees stock widgets, Sales Staff sees sales widgets

---

## Cross-Cutting Rules

- **Stock changes in exactly one place.** Every module calls a single stock service and nothing updates quantity directly, which keeps the ledger and the cached quantity in agreement
- **Multi-step operations are atomic.** A sales order touches the order, its lines, product quantities, and the ledger — all of it succeeds or none of it does
- **Prices and addresses are snapshotted onto orders** so historical documents do not change when a product or customer record is later edited
- **Deletes are soft.** Anything referenced by an order is deactivated, never removed
- **The server is the only authority** on totals, permissions, and stock availability