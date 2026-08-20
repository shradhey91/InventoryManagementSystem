# Inventory Management System


## 1. Login and User Roles

**Login**
- User types email and password
- System checks if the password is correct
- If correct, user gets a token and goes to the dashboard
- If wrong, show "invalid email or password"

**Logout**
- Delete the user's token
- Send back to the login page

**Managing users** (only Super Admin)
- Add a user by entering name, email and role
- Change someone's role

---

## 2. Products

**Add a product**
- Fill: SKU , name, category, supplier, cost price, selling price, quantity,  unit (kg, pieces, box)
- SKU must be unique
- Cost price = what we paid. Selling price = what we charge

**Edit a product**

**Delete a product**
- If the product is on any old order we will not delete it
- If it was never used and stock is 0 we can delete it 

**View products**
- Show a list with page numbers per page 10
- Filter by category, supplier, or stock status

**Search products**
- Search by name or SKU

---

## 3. Categories

**Add / Edit**
- Just a name and description
- Two categories can't have the same name

**Delete**
- If products are using this category don't allow delete
- If empty → mark it inactive

**View**
- Show all categories with how many products are in each
- Click a category to see its products

---

## 4. Suppliers

**Add / Edit**
- Fill : company name, contact person, email, phone, address

**Delete**
- If there are pending purchase orders → don't allow it
- Otherwise mark inactive

**View / Search**
- Show a list with page numbers
- Click a supplier to see their details and past orders
- Search by name, email or phone

---

## 5. Stock

- **Current stock** — all products with their quantity
- **In stock** — quantity is more than the reorder level
- **Out of stock** — quantity is 0

---

## 6. Purchase Orders (buying from suppliers)

**Create a purchase order**
- Pick a supplier
- Add products with quantity and price
- The total is calculated automatically

**Edit a purchase order**
- Draft → we can change anything
- Approved → only small things like notes
- Already received → cannot change

**Receive a purchase order** (Storekeeper)

**Cancel a purchase order**

**Return to supplier**
- If the goods are damaged or wrong

---

## 7. Sales Orders (selling to customers)

**Create a sales order**

**Edit a sales order**

**Cancel a sales order**

**View sales orders**

**Customer return**
- Customer brings back the goods

---

## 8. Customers


**Add / Edit**
- Fill in: name, email, phone, billing address, delivery address

**Delete** (only Admin)
- If they have pending orders → don't allow it
- Otherwise mark inactive

**View / Search**
- Show contact details and all their past orders
- Search by name, email or phone

---

## 9. Transactions (stock history)


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

---

## 10. Dashboard


**Counts**
- Total products
- Total categories
- Total suppliers
- Total stock quantity

**Alerts**
- Low stock products
- Out of stock products

**Recent activity**
- Today's sales
- Last 5 sales orders
- Last 5 purchase orders

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

