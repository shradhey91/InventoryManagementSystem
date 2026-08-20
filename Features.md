# Inventory Management System

## 1. Login and User Roles

**Login**
- User types email and password
- System checks if the password is correct
- If correct, user gets a token and goes to the dashboard else we show invalid email or password

**Logout**
- delelte user's token and send back to login page

**Managing users** (only Super Admin)
- Add a user by entering name, email and role
- Change user's role

---

## 2. Products

**Add a product**
- Fill: SKU, name, category, cost price, selling price, quantity, unit (kg, pieces, box), reorder level
- Cost price = what we paid. Selling price = what we charge

**Edit a product**

**Delete a product**
- If the product is on any old order we will not delete it
- If it was never used and stock is 0 we can delete it

**View products**
- Show a list with page numbers, per page 10
- Filter by category or stock status

**Search products**
- Search by name or SKU

---

## 3. Categories

**Add / Edit**
- Just a name and description
- Two categories can't have the same name

**Delete**
- If products are using this category don't allow delete
- If empty mark it inactive

**View**
- Show all categories with how many products are in each
- Click a category to see its products

---

## 4. Stock

- **Current stock** - all products with their quantity
- **In stock** - quantity is more than the reorder level
- **Low stock** - quantity is less than or equal to reorder level but not 0
- **Out of stock** - quantity is 0

**Stock adjustment**
- select a product to increase or decrease quantity with a reason
- Reasons:  Damaged, Expired, Lost, CountCorrection
- Only Admin can do this

---

## 5. Sales Orders (selling to customers)

**Create a sales order**
- select a customer
- Add products with quantity and price

**Edit a sales order**
- Draft we can change anything
- Confirmed or after cannot change

**Cancel a sales order**
- Only before it is shipped

**View sales orders**

**Customer return**
- Customer brings back the goods
- If the item is still good the stock comes back
- If it is damaged we only save the record, stock does not come back

---

## 6. Customers

**Add / Edit**
- Fill in: name, email, phone, billing address, delivery address

**Delete** (only Admin)
- If they have pending orders, don't allow it
- Otherwise mark inactive

**View / Search**
- Show contact details and all their past orders
- Search by name, email or phone

---

## 7. Transactions (stock history)

**Stock goes up (+)**
- Product created with a starting quantity
- Customer returned a item
- Stock adjustment upward
- Sales order was cancelled

**Stock goes down (-)**
- Sale confirmed
- Stock adjustment downward

**Viewing history**
- Filter by product, date, type or user

---

## 8. Dashboard

**Counts**
- Total products
- Total categories
- Total stock quantity
- Total stock value 

**Alerts**
- Low stock products
- Out of stock products

**Recent activity**
- Today's sales
- Last 5 sales orders
- Last 5 stock adjustments

---

## Order status

**Sales order**
- Draft -> Confirmed -> Shipped -> Delivered -> Returned

---

## Schema

**Users Table**
- id
- name
- email (unique)
- password
- role (SuperAdmin, Admin, Staff)
- isActive
- createdAt

**Categories Table**
- id
- name
- description
- isActive

**Products Table**
- id
- sku (unique)
- name
- description
- categoryId (FK - Categories)
- costPrice
- sellingPrice
- quantity
- unit (kg, pieces, box)
- reorderLevel
- isActive
- createdBy (FK - Users)
- createdAt

**Customers Table**
- id
- name
- email
- phone
- billingAddress
- deliveryAddress
- isActive

**SalesOrder Table**
- id
- customerId (FK - Customers)
- status (Draft, Confirmed, Shipped, Delivered, Cancelled, Returned)
- orderDate
- subTotal
- discount
- tax
- total
- createdBy (FK - Users)
- shippedAt
- deliveredAt

**SalesOrderItems Table**
- id
- salesOrderId (FK - SalesOrder)
- productId (FK - Products)
- quantity
- unitPrice
- TotalPrice 


**SalesReturns Table**
- id
- salesOrderId (FK - SalesOrder)
- customerId (FK - Customers)
- reason
- createdAt

**SalesReturnItems Table**
- id
- salesReturnId (FK - SalesReturns)
- productId (FK - Products)
- quantity

**StockAdjustments Table**
- id
- productId (FK - Products)
- quantityChange
- reason (Restock, Damaged, Expired, Lost)
- notes
- createdBy (FK - Users)
- createdAt

**StockTransactions Table**
- id
- productId (FK - Products)
- type (OpeningStock, AdjustmentIn, SaleCancelled, CustomerReturn, Sale, AdjustmentOut)
- quantityChange
- quantityAfter 
- type (SalesOrder, SalesReturn, StockAdjustment, Product)
- referenceId
- userId (FK - Users)
- createdAt


## Relations

**One to many**
- Categories to Products 
- Users to Products
- Users to SalesOrder 
- Users to StockAdjustments
- Users to StockTransactions
- Customers to SalesOrder 
- SalesOrder to SalesOrderItems
- SalesOrder to SalesReturns 
- SalesReturns to SalesReturnItems
- Products to SalesOrderItems
- Products to SalesReturnItems
- Products to StockAdjustments
- Products to StockTransactions

**Many to many**
- Products and SalesOrder (through SalesOrderItems)
- Products and SalesReturns (through SalesReturnItems)

