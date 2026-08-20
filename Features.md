# Inventory Management System


## 1. Login and User Roles

**Login**
- User types email and password
- System checks if the password is correct
- If correct, user gets a token and goes to the dashboard
- If wrong, show "invalid email or password" (don't tell them which one was wrong)
- After too many wrong tries, lock the account for some time

**Logout**
- Delete the user's token
- Send them back to the login page

**Managing users** (only Super Admin)
- Add a user by entering name, email and role
- Send them a link so they can set their own password
- Change someone's role
- Turn off an account when someone leaves
- Two rules: emails must be different for each user, and there must always be at least one Super Admin

**Roles**
- Every user has one role
- Each role has a list of things it is allowed to do
- Before doing anything, the system checks if that user's role is allowed
- Important: check this on the **backend**, not just by hiding buttons in the frontend

---

## 2. Products

**Add a product**
- Fill in: SKU (unique code), name, category, supplier, cost price, selling price, quantity, reorder level, unit (kg, pieces, box)
- SKU must be unique — no two products can have the same one
- Cost price = what you paid. Selling price = what you charge. You need both.
- Reorder level = when stock falls to this number, it counts as "low"

**Edit a product**
- Change any field except quantity
- Quantity can only change through Stock Adjustment, so there is always a reason recorded

**Delete a product**
- If the product is on any old order → don't really delete it, just mark it inactive
- If it was never used and stock is 0 → you can delete it fully
- Deleting a product that is on an old order would break your order history

**View products**
- Show a list with page numbers (don't load all products at once)
- Filter by category, supplier, or stock status
- Sales Staff should not see the cost price

**Search products**
- Search by name or SKU
- Wait until the user stops typing before searching

---

## 3. Categories

**Add / Edit**
- Just a name and description
- Two categories can't have the same name

**Delete**
- If products are using this category → don't allow delete, show how many products are using it
- If empty → mark it inactive

**View**
- Show all categories with how many products are in each
- Click a category to see its products

---

## 4. Suppliers

Suppliers are the people you **buy** from.

**Add / Edit**
- Fill in: company name, contact person, email, phone, address
- Show a warning if a similar supplier already exists

**Delete**
- If there are pending purchase orders → don't allow it
- Otherwise mark inactive

**View / Search**
- Show a list with page numbers
- Click a supplier to see their details and past orders
- Search by name, email or phone
- Sales Staff should not see suppliers (it shows cost prices)

---

## 5. Stock

There is no separate stock table. Stock is just the quantity column in the products table.
These pages are the same product list, filtered differently:

- **Current stock** — all products with their quantity
- **In stock** — quantity is more than the reorder level
- **Low stock** — quantity is between 1 and the reorder level
- **Out of stock** — quantity is 0

From the low stock page, the Admin can quickly create a purchase order for those items.

**Stock adjustment** (Storekeeper and Admin)
- Sometimes real stock does not match the system — items get damaged, expired, stolen, or miscounted
- The storekeeper enters the real counted quantity
- Then picks a reason: Damaged, Expired, Lost, Wrong Count
- The system saves the difference with that reason
- Reason must be compulsory, otherwise you will never know why stock changed

---

## 6. Purchase Orders (buying from suppliers)

**Important:** a purchase order is just an order you placed. The goods have not arrived yet.
So stock should **not** increase when you create it.

**Create a purchase order**
- Pick a supplier
- Add products with quantity and price
- The total is calculated automatically
- Save as draft, then submit it for approval
- Admin approves it, then you send it to the supplier
- Stock does not change here

**Edit a purchase order**
- Draft → you can change anything
- Approved → only small things like notes
- Already received → cannot change

**Receive a purchase order** (Storekeeper)
- This happens when the goods actually arrive at the store
- The storekeeper enters how many items actually came
- Sometimes only some of them arrive, so allow partial receiving
- Now the stock increases
- Save a record of it in the transaction table
- If everything arrived → mark as received. If only some → mark as partly received

**Cancel a purchase order**
- If nothing arrived yet → just cancel it
- If some items already arrived → only cancel the remaining ones (you already have the other goods)
- If everything arrived → you cannot cancel, you must return it instead

**Return to supplier**
- If the goods are damaged or wrong
- Stock goes down and a record is saved

---

## 7. Sales Orders (selling to customers)

**Create a sales order**
- Pick a customer (or add a new one right there)
- Add products with quantity and price
- Show the available stock next to each product
- When confirmed: check stock again, reduce the stock, and save a record
- If any product does not have enough stock → cancel the whole thing, don't save half of it

Two important things:

1. **Save the price inside the order.** Don't just link to the product. If you change the product price next month, all your old bills would change too, which is wrong.
2. **Lock the product row while reducing stock.** If two sales staff sell the last item at the same second, both would succeed and your stock would go negative.

**Edit a sales order**
- Only if it is not shipped yet
- Easiest way: put back all the old stock, then subtract the new quantities again
- This is much simpler than calculating the difference for each line

**Cancel a sales order**
- Put the stock back
- Save a new record for the cancellation — don't delete the old sale record
- If it is already delivered, you cannot cancel it, use a return

**View sales orders**
- Sales Staff sees only their own orders
- Admin sees all orders
- Filter by date, customer or status

**Customer return**
- Customer brings back the goods
- If the item is still good → add it back to stock
- If it is damaged → do not add it back, just record the loss

---

## 8. Customers

Customers are the people you **sell** to.

**Add / Edit**
- Fill in: name, email, phone, billing address, delivery address
- Warn if a customer with the same email or phone already exists
- You should be able to add a customer directly from the sales order page

**Delete** (only Admin)
- If they have pending orders → don't allow it
- Otherwise mark inactive
- Sales Staff can add and edit customers but not delete them

**View / Search**
- Show contact details and all their past orders
- Search by name, email or phone

---

## 9. Transactions (stock history)

This is a history table. Every time stock changes, add one row here.
**Never edit or delete these rows.** If something is wrong, add a new row to fix it.

Each row saves: which product, what type, how much, which order caused it, which user did it, and the time.

Use plus and minus numbers so you can just add them all up to get the current stock.

**Stock goes up (+)**
- Purchase received
- Customer returned a good item
- Stock adjustment upward
- Sales order was cancelled

**Stock goes down (−)**
- Sale confirmed
- Returned something to the supplier
- Stock adjustment downward

**Viewing history**
- Filter by product, date, type or user
- Click any row to see which order caused it
- Good check: adding up all rows of a product should equal the quantity in the product table. If not, something is wrong somewhere.

---

## 10. Dashboard

The first page after login. Show only what that role is allowed to see.

**Counts**
- Total products
- Total categories
- Total suppliers
- Total stock quantity

**Alerts**
- Low stock products
- Out of stock products
- Clicking these should take you to that filtered list

**Recent activity**
- Today's sales
- Last 5 sales orders
- Last 5 purchase orders

**Two tips**
- This page runs many queries. Run them together, not one after another, and store the result for a minute or two.
- "Total stock quantity" is a bit useless if some products are in kg and some in pieces. You can't add 5 kg and 3 boxes. Better to show total stock **value** (quantity × cost price).

---

## Order statuses

**Purchase order**
- Draft → Waiting for approval → Approved → Partly received → Received
- Can be cancelled any time before it is fully received
- Stock only increases at the receiving step

**Sales order**
- Draft → Confirmed → Shipped → Delivered → Returned
- Can be cancelled before shipping
- Stock reduces when confirmed, and comes back if cancelled or returned

