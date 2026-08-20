# Inventory Management System

A role-based inventory management application for tracking products, stock levels,
suppliers, customers, purchases, and sales.

---

## Features

### 1. Authentication & Authorization

- User Login
- User Logout
- User Management
- Role-Based Access Control

**Roles:** `Super Admin` · `Admin` · `Storekeeper` · `Sales Staff`

---

### 2. Product Management

- Add Product
- Update Product
- Delete Product
- View Product
- View All Products
- Search Products

**Product attributes include:** SKU, name, category, supplier, unit price, quantity.

---

### 3. Category Management

- Create Category
- Update Category
- Delete Category
- View Categories
- View Products by Category

---

### 4. Supplier Management

- Add Supplier
- Update Supplier
- Delete Supplier
- View Suppliers
- Search Suppliers

---

### 5. Stock Management

- View Current Stock
- View In-Stock Products
- View Low-Stock Products
- View Out-of-Stock Products

---

### 6. Purchase Order Management

- Create Purchase Order
- Update Purchase Order
- Cancel Purchase Order
- View Purchase Orders
- Select Supplier
- Add Products to Purchase Order

---

### 7. Sales / Order Management

- Create Sales Order
- Update Sales Order
- Cancel Sales Order
- View Sales Orders

---

### 8. Customer Management

- Add Customer
- Update Customer
- Delete Customer
- View Customers
- Search Customers

---

### 9. Inventory Transaction Management

- Track Products Added
- Track Products Sold
- Track Product Returns

---

### 10. Dashboard

**Summary Metrics**

- Total Products
- Total Categories
- Total Suppliers
- Total Stock Quantity

**Stock Alerts**

- Low-Stock Products
- Out-of-Stock Products

**Recent Activity**

- Today's Sales
- Recent Sales Orders
- Recent Purchase Orders

---



## Role Permissions

| Module          | Super Admin | Admin | Storekeeper   | Sales Staff              |
| --------------- | ----------- | ----- | ------------- | ------------------------ |
| User Management | Full        | View  | —             | —                        |
| Products        | Full        | Full  | View          | View                     |
| Categories      | Full        | Full  | View          | View                     |
| Suppliers       | Full        | Full  | View          | —                        |
| Customers       | Full        | Full  | —             | Full                     |
| Stock           | Full        | Full  | View + Adjust | View                     |
| Purchase Orders | Full        | Full  | Receive only  | —                        |
| Sales Orders    | Full        | Full  | —             | Create / Update / Cancel |
| Transactions    | Full        | View  | View          | View own                 |
| Dashboard       | Full        | Full  | Stock widgets | Sales widgets            |