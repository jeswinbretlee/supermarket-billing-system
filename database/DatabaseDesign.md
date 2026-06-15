# PostgreSQL Database Schema Design
## Enterprise Supermarket Billing System

**Version:** 1.0  
**Date:** 2026-06-15  
**Architect:** Senior Database Architect  
**Target:** 50,000+ products, 100+ branches, 100,000+ daily transactions  

---

## TABLE OF CONTENTS

1. [Database Overview](#database-overview)
2. [Design Principles](#design-principles)
3. [Entity Relationship Diagram](#entity-relationship-diagram)
4. [Core Tables](#core-tables)
5. [Product & Inventory Tables](#product--inventory-tables)
6. [Transaction Tables](#transaction-tables)
7. [Supplier Tables](#supplier-tables)
8. [Customer Tables](#customer-tables)
9. [Financial Tables](#financial-tables)
10. [Audit & Compliance Tables](#audit--compliance-tables)
11. [Indexes Strategy](#indexes-strategy)
12. [Views & Materialized Views](#views--materialized-views)
13. [Partitioning Strategy](#partitioning-strategy)

---

## DATABASE OVERVIEW

### 1.1 Database Name and Encoding

```sql
CREATE DATABASE supermarket_billing
  OWNER billing_admin
  ENCODING 'UTF8'
  LC_COLLATE 'en_US.UTF-8'
  LC_CTYPE 'en_US.UTF-8'
  TEMPLATE template0;
```

### 1.2 Database Statistics

| Metric | Value |
|--------|-------|
| Expected Size (Year 1) | 50-100 GB |
| Expected Size (Year 5) | 500-800 GB |
| Transaction Tables | 15+ tables |
| Daily Transactions | 100,000+ |
| Concurrent Connections | 200-500 |
| Read/Write Ratio | 70/30 |

### 1.3 Naming Conventions

| Object | Convention | Example |
|--------|-----------|---------|
| Table | snake_case | `users`, `sale_items` |
| Column | snake_case | `created_at`, `branch_id` |
| Primary Key | id | `id` (auto-increment) |
| Foreign Key | `{table}_id` | `user_id`, `branch_id` |
| Index | `idx_{table}_{column}` | `idx_users_email` |
| Unique Constraint | `uq_{table}_{column}` | `uq_users_email` |
| Check Constraint | `ck_{table}_{column}` | `ck_inventory_qty` |

---

## DESIGN PRINCIPLES

### 2.1 Database Design Strategy

1. **Normalization:** 3NF (Third Normal Form) for consistency
2. **Denormalization:** Strategic for performance-critical queries
3. **Partitioning:** By date for transaction tables (sales, payments)
4. **Replication:** Primary + Read Replica for analytics
5. **Backup:** Daily incremental + Weekly full
6. **Data Integrity:** Foreign keys + Check constraints enforced

### 2.2 Scalability Considerations

- Partitioning on `created_at` for transaction tables (monthly)
- Sharding key: `branch_id` for multi-branch scaling
- Archive old data (>2 years) to separate schema
- Materialized views for expensive aggregations
- Time-series data optimization for reporting

### 2.3 Performance Optimization

- Composite indexes on frequently filtered columns
- BRIN indexes for time-series data
- Partial indexes for filtered queries
- Covering indexes to avoid table lookups
- Query statistics (ANALYZE) on schedule

---

## ENTITY RELATIONSHIP DIAGRAM

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER & SECURITY                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   users      │  │   roles      │  │ permissions  │           │
│  │   1:N        │──│              │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│         │                                                         │
│         └────────────────────────────────────────────────┐       │
└─────────────────────────────────────────────────┬────────────────┘
                                                  │
┌─────────────────────────────────────────────────┴────────────────┐
│                      ORGANIZATIONAL                               │
│  ┌──────────────┐  ┌──────────────┐                              │
│  │   branches   │  │ branch_users │                              │
│  │   1:N        │──│              │                              │
│  └──────────────┘  └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   PRODUCTS & INVENTORY                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ categories   │  │   products   │  │  inventory   │           │
│  │   1:N        │──│   1:N        │──│              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                           │                    │                 │
│                           │         ┌──────────┴──────────┐      │
│                           │         │                     │      │
│                    ┌──────┴────────┐ ▼                    ▼      │
│                    │  batch_       │┌─────────────┐ ┌──────────┐│
│                    │  tracking     ││ stock_      │ │ stock_   ││
│                    │   (expiry)    ││ movements   │ │ alerts   ││
│                    └───────────────┴┴─────────────┴ └──────────┘│
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  TRANSACTIONS & BILLING                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   sales      │  │  sale_items  │  │   payments   │           │
│  │   1:N        │──│              │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│         │                                      │                 │
│         ▼                                      ▼                 │
│  ┌──────────────┐                    ┌──────────────────┐       │
│  │   invoices   │                    │ payment_logs     │       │
│  │              │                    │                  │       │
│  └──────────────┘                    └──────────────────┘       │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   returns    │                                               │
│  │              │                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   SUPPLIERS & PROCUREMENT                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  suppliers   │  │ supplier_    │  │ purchase_    │           │
│  │   1:N        │──│ pricing      │  │ orders       │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                            │                    │
│                                            ▼                    │
│                                    ┌──────────────────┐         │
│                                    │  po_items        │         │
│                                    │                  │         │
│                                    └──────────────────┘         │
│                                            │                    │
│                                            ▼                    │
│                                    ┌──────────────────┐         │
│                                    │  goods_receipt   │         │
│                                    │                  │         │
│                                    └──────────────────┘         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      CUSTOMERS & LOYALTY                          │
│  ┌──────────────┐  ┌──────────────────┐                         │
│  │  customers   │  │  loyalty_points  │                         │
│  │   1:N        │──│                  │                         │
│  └──────────────┘  └──────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    FINANCIAL & COMPLIANCE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ gst_config   │  │ gst_summary  │  │ audit_logs   │           │
│  │              │  │              │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## CORE TABLES

### 3.1 Users Table

**Purpose:** Store all system users with authentication credentials and profile info

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | Auto-incrementing primary key |
| username | VARCHAR(50) | UNIQUE, NOT NULL | Login credential |
| email | VARCHAR(100) | UNIQUE, NOT NULL | Contact email |
| password_hash | VARCHAR(255) | NOT NULL | bcrypt hashed password |
| phone | VARCHAR(15) | | Phone number |
| first_name | VARCHAR(100) | NOT NULL | User's first name |
| last_name | VARCHAR(100) | NOT NULL | User's last name |
| role_id | BIGINT | FK → roles.id | User's role |
| branch_id | BIGINT | FK → branches.id | Primary branch assignment |
| is_active | BOOLEAN | DEFAULT true | Account status |
| last_login_at | TIMESTAMP | | Last login time |
| password_changed_at | TIMESTAMP | | Last password change |
| failed_login_attempts | INTEGER | DEFAULT 0 | Failed login counter |
| locked_until | TIMESTAMP | | Account lock expiry |
| created_at | TIMESTAMP | DEFAULT NOW() | Record creation time |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update time |
| created_by | BIGINT | FK → users.id | Audit trail |
| updated_by | BIGINT | FK → users.id | Audit trail |

**CREATE TABLE:**
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  phone VARCHAR(15),
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  role_id BIGINT NOT NULL REFERENCES roles(id),
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  is_active BOOLEAN DEFAULT true,
  last_login_at TIMESTAMP,
  password_changed_at TIMESTAMP,
  failed_login_attempts INTEGER DEFAULT 0,
  locked_until TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by BIGINT REFERENCES users(id),
  updated_by BIGINT REFERENCES users(id),
  CONSTRAINT ck_username_length CHECK (LENGTH(username) >= 3),
  CONSTRAINT ck_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);
```

### 3.2 Roles Table

**Purpose:** Define role types and their descriptions for RBAC

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | Auto-incrementing |
| role_name | VARCHAR(50) | UNIQUE, NOT NULL | Role identifier |
| description | TEXT | | Role description |
| permissions | JSONB | NOT NULL | Permission list (JSON array) |
| created_at | TIMESTAMP | DEFAULT NOW() | |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE roles (
  id BIGSERIAL PRIMARY KEY,
  role_name VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  permissions JSONB NOT NULL DEFAULT '[]'::jsonb,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Sample data
INSERT INTO roles (role_name, description, permissions) VALUES
('ADMIN', 'Administrator', '["*"]'::jsonb),
('STORE_MANAGER', 'Store Manager', '["transaction:read","transaction:create","refund:read","refund:approve","inventory:read","customer:read","report:read"]'::jsonb),
('CASHIER', 'Cashier', '["transaction:create","transaction:read","product:read","refund:request"]'::jsonb),
('INVENTORY_MANAGER', 'Inventory Manager', '["inventory:*","product:read","supplier:read"]'::jsonb),
('ACCOUNTS', 'Accounts Officer', '["transaction:read","payment:read","report:read","gst:read"]'::jsonb);
```

### 3.3 Branches Table

**Purpose:** Store branch/location details for multi-branch operations

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| branch_code | VARCHAR(10) | UNIQUE, NOT NULL | Branch identifier (e.g., BR001) |
| branch_name | VARCHAR(100) | NOT NULL | Branch display name |
| address | TEXT | NOT NULL | Street address |
| city | VARCHAR(50) | NOT NULL | City name |
| state | VARCHAR(50) | NOT NULL | State/Province |
| postal_code | VARCHAR(10) | | ZIP/Postal code |
| phone | VARCHAR(15) | | Branch phone |
| email | VARCHAR(100) | | Branch email |
| manager_id | BIGINT | FK → users.id | Branch manager |
| gstin | VARCHAR(15) | | GST registration number |
| opening_time | TIME | | Store opening time |
| closing_time | TIME | | Store closing time |
| is_active | BOOLEAN | DEFAULT true | Branch status |
| created_at | TIMESTAMP | DEFAULT NOW() | |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE branches (
  id BIGSERIAL PRIMARY KEY,
  branch_code VARCHAR(10) UNIQUE NOT NULL,
  branch_name VARCHAR(100) NOT NULL,
  address TEXT NOT NULL,
  city VARCHAR(50) NOT NULL,
  state VARCHAR(50) NOT NULL,
  postal_code VARCHAR(10),
  phone VARCHAR(15),
  email VARCHAR(100),
  manager_id BIGINT REFERENCES users(id),
  gstin VARCHAR(15),
  opening_time TIME,
  closing_time TIME,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_branch_code CHECK (LENGTH(branch_code) >= 3)
);
```

### 3.4 Permissions Table

**Purpose:** Master list of all permissions in the system

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| resource | VARCHAR(50) | NOT NULL | Resource type (e.g., 'transaction') |
| action | VARCHAR(50) | NOT NULL | Action type (e.g., 'create', 'read') |
| description | TEXT | | Permission description |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE permissions (
  id BIGSERIAL PRIMARY KEY,
  resource VARCHAR(50) NOT NULL,
  action VARCHAR(50) NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(resource, action)
);
```

---

## PRODUCT & INVENTORY TABLES

### 4.1 Categories Table

**Purpose:** Product category hierarchy

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| category_name | VARCHAR(100) | NOT NULL | Category name |
| description | TEXT | | Category description |
| parent_category_id | BIGINT | FK → categories.id | For hierarchical categories |
| is_active | BOOLEAN | DEFAULT true | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE categories (
  id BIGSERIAL PRIMARY KEY,
  category_name VARCHAR(100) NOT NULL,
  description TEXT,
  parent_category_id BIGINT REFERENCES categories(id),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.2 Products Table

**Purpose:** Master product catalog for all branches

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| sku | VARCHAR(50) | UNIQUE, NOT NULL | Stock Keeping Unit |
| barcode | VARCHAR(50) | UNIQUE | Barcode/EAN |
| product_name | VARCHAR(200) | NOT NULL | Product name |
| category_id | BIGINT | FK → categories.id | Product category |
| description | TEXT | | Product description |
| unit | VARCHAR(20) | NOT NULL | Unit of measure (kg, liter, box, etc.) |
| price | DECIMAL(10,2) | NOT NULL | Base price |
| hsn_code | VARCHAR(10) | NOT NULL | HSN code for GST |
| gst_rate | DECIMAL(5,2) | NOT NULL | GST rate (5, 12, 18, 28) |
| is_active | BOOLEAN | DEFAULT true | Product status |
| reorder_level | INTEGER | DEFAULT 100 | Minimum stock level |
| created_at | TIMESTAMP | DEFAULT NOW() | |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  sku VARCHAR(50) UNIQUE NOT NULL,
  barcode VARCHAR(50) UNIQUE,
  product_name VARCHAR(200) NOT NULL,
  category_id BIGINT NOT NULL REFERENCES categories(id),
  description TEXT,
  unit VARCHAR(20) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  hsn_code VARCHAR(10) NOT NULL,
  gst_rate DECIMAL(5,2) NOT NULL,
  is_active BOOLEAN DEFAULT true,
  reorder_level INTEGER DEFAULT 100,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_price CHECK (price >= 0),
  CONSTRAINT ck_gst_rate CHECK (gst_rate IN (0, 5, 12, 18, 28))
);
```

### 4.3 Inventory Table

**Purpose:** Stock levels per branch per product

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| product_id | BIGINT | FK → products.id | Product reference |
| branch_id | BIGINT | FK → branches.id | Branch reference |
| quantity_on_hand | DECIMAL(10,3) | NOT NULL | Current stock |
| quantity_available | DECIMAL(10,3) | NOT NULL | Available (on-hand - reserved) |
| quantity_reserved | DECIMAL(10,3) | DEFAULT 0 | Reserved for orders |
| reorder_level | DECIMAL(10,3) | NOT NULL | Min stock threshold |
| max_level | DECIMAL(10,3) | | Max stock threshold |
| last_movement_date | TIMESTAMP | | Last stock movement |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE inventory (
  id BIGSERIAL PRIMARY KEY,
  product_id BIGINT NOT NULL REFERENCES products(id),
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  quantity_on_hand DECIMAL(10,3) NOT NULL DEFAULT 0,
  quantity_available DECIMAL(10,3) NOT NULL DEFAULT 0,
  quantity_reserved DECIMAL(10,3) DEFAULT 0,
  reorder_level DECIMAL(10,3) NOT NULL,
  max_level DECIMAL(10,3),
  last_movement_date TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(product_id, branch_id),
  CONSTRAINT ck_qty_positive CHECK (quantity_on_hand >= 0),
  CONSTRAINT ck_available CHECK (quantity_available >= 0)
);
```

### 4.4 Stock Movements Table

**Purpose:** Audit trail of all inventory changes

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| inventory_id | BIGINT | FK → inventory.id | Inventory reference |
| movement_type | VARCHAR(20) | NOT NULL | SALE, PURCHASE, ADJUSTMENT, TRANSFER, RETURN |
| quantity_change | DECIMAL(10,3) | NOT NULL | Quantity delta |
| reference_id | BIGINT | | FK to source table (sales.id, po.id) |
| reference_type | VARCHAR(50) | | Source type (sale, purchase_order) |
| notes | TEXT | | Movement notes |
| created_by | BIGINT | FK → users.id | User who made change |
| created_at | TIMESTAMP | DEFAULT NOW() | MUST partition on this column |

**CREATE TABLE:**
```sql
CREATE TABLE stock_movements (
  id BIGSERIAL PRIMARY KEY,
  inventory_id BIGINT NOT NULL REFERENCES inventory(id),
  movement_type VARCHAR(20) NOT NULL,
  quantity_change DECIMAL(10,3) NOT NULL,
  reference_id BIGINT,
  reference_type VARCHAR(50),
  notes TEXT,
  created_by BIGINT REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_movement_type CHECK (movement_type IN ('SALE', 'PURCHASE', 'ADJUSTMENT', 'TRANSFER', 'RETURN'))
) PARTITION BY RANGE (DATE_TRUNC('month', created_at));
```

### 4.5 Batch Tracking Table

**Purpose:** Track product batches with expiry dates

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| inventory_id | BIGINT | FK → inventory.id | |
| batch_number | VARCHAR(50) | NOT NULL | Batch/Lot number |
| expiry_date | DATE | NOT NULL | Product expiry date |
| quantity | DECIMAL(10,3) | NOT NULL | Quantity in batch |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE batch_tracking (
  id BIGSERIAL PRIMARY KEY,
  inventory_id BIGINT NOT NULL REFERENCES inventory(id),
  batch_number VARCHAR(50) NOT NULL,
  expiry_date DATE NOT NULL,
  quantity DECIMAL(10,3) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_expiry_future CHECK (expiry_date > CURRENT_DATE)
);
```

### 4.6 Stock Alerts Table

**Purpose:** Alerts for low stock, overstock, or expiry

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| inventory_id | BIGINT | FK → inventory.id | |
| alert_type | VARCHAR(20) | NOT NULL | LOW_STOCK, OVERSTOCK, EXPIRY_SOON |
| threshold_value | DECIMAL(10,3) | | Threshold triggered |
| current_value | DECIMAL(10,3) | NOT NULL | Current value |
| is_acknowledged | BOOLEAN | DEFAULT false | Alert acknowledged |
| acknowledged_by | BIGINT | FK → users.id | |
| acknowledged_at | TIMESTAMP | | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE stock_alerts (
  id BIGSERIAL PRIMARY KEY,
  inventory_id BIGINT NOT NULL REFERENCES inventory(id),
  alert_type VARCHAR(20) NOT NULL,
  threshold_value DECIMAL(10,3),
  current_value DECIMAL(10,3) NOT NULL,
  is_acknowledged BOOLEAN DEFAULT false,
  acknowledged_by BIGINT REFERENCES users(id),
  acknowledged_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_alert_type CHECK (alert_type IN ('LOW_STOCK', 'OVERSTOCK', 'EXPIRY_SOON'))
);
```

---

## TRANSACTION TABLES

### 5.1 Sales Table

**Purpose:** Main sales transaction header

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| transaction_no | VARCHAR(50) | UNIQUE, NOT NULL | Transaction receipt number |
| branch_id | BIGINT | FK → branches.id | Transaction branch |
| cashier_id | BIGINT | FK → users.id | Cashier who processed |
| customer_id | BIGINT | FK → customers.id | Customer (nullable for cash) |
| total_items | INTEGER | NOT NULL | Item count |
| subtotal_amount | DECIMAL(12,2) | NOT NULL | Amount before tax |
| discount_amount | DECIMAL(12,2) | DEFAULT 0 | Total discount |
| taxable_amount | DECIMAL(12,2) | NOT NULL | Amount subject to tax |
| total_tax | DECIMAL(12,2) | NOT NULL | Total GST |
| total_amount | DECIMAL(12,2) | NOT NULL | Final amount |
| payment_method | VARCHAR(20) | NOT NULL | CASH, CARD, UPI, CHEQUE |
| transaction_status | VARCHAR(20) | DEFAULT 'COMPLETED' | PENDING, COMPLETED, CANCELLED |
| payment_status | VARCHAR(20) | DEFAULT 'PENDING' | PENDING, SUCCESS, FAILED |
| return_status | VARCHAR(20) | DEFAULT 'NONE' | NONE, PARTIAL_RETURN, FULL_RETURN |
| created_at | TIMESTAMP | DEFAULT NOW() | PARTITION KEY |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE sales (
  id BIGSERIAL PRIMARY KEY,
  transaction_no VARCHAR(50) UNIQUE NOT NULL,
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  cashier_id BIGINT NOT NULL REFERENCES users(id),
  customer_id BIGINT REFERENCES customers(id),
  total_items INTEGER NOT NULL,
  subtotal_amount DECIMAL(12,2) NOT NULL,
  discount_amount DECIMAL(12,2) DEFAULT 0,
  taxable_amount DECIMAL(12,2) NOT NULL,
  total_tax DECIMAL(12,2) NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL,
  payment_method VARCHAR(20) NOT NULL,
  transaction_status VARCHAR(20) DEFAULT 'COMPLETED',
  payment_status VARCHAR(20) DEFAULT 'PENDING',
  return_status VARCHAR(20) DEFAULT 'NONE',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_payment_method CHECK (payment_method IN ('CASH', 'CARD', 'UPI', 'CHEQUE', 'WALLET')),
  CONSTRAINT ck_total_amount CHECK (total_amount >= 0)
) PARTITION BY RANGE (DATE_TRUNC('month', created_at));
```

### 5.2 Sale Items Table

**Purpose:** Individual items in a sale transaction

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| sale_id | BIGINT | FK → sales.id | Parent sale |
| product_id | BIGINT | FK → products.id | Product sold |
| quantity | DECIMAL(10,3) | NOT NULL | Quantity purchased |
| unit_price | DECIMAL(10,2) | NOT NULL | Price at time of sale |
| discount_pct | DECIMAL(5,2) | DEFAULT 0 | Item-level discount % |
| discount_amount | DECIMAL(10,2) | DEFAULT 0 | Item discount amount |
| line_total | DECIMAL(12,2) | NOT NULL | Qty × Unit Price |
| tax_rate | DECIMAL(5,2) | NOT NULL | GST rate applied |
| tax_amount | DECIMAL(10,2) | NOT NULL | Tax on item |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE sale_items (
  id BIGSERIAL PRIMARY KEY,
  sale_id BIGINT NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
  product_id BIGINT NOT NULL REFERENCES products(id),
  quantity DECIMAL(10,3) NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  discount_pct DECIMAL(5,2) DEFAULT 0,
  discount_amount DECIMAL(10,2) DEFAULT 0,
  line_total DECIMAL(12,2) NOT NULL,
  tax_rate DECIMAL(5,2) NOT NULL,
  tax_amount DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_quantity CHECK (quantity > 0),
  CONSTRAINT ck_line_total CHECK (line_total >= 0)
);
```

### 5.3 Payments Table

**Purpose:** Payment details for transactions

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| sale_id | BIGINT | FK → sales.id | Associated sale |
| payment_method | VARCHAR(20) | NOT NULL | Payment type |
| amount | DECIMAL(12,2) | NOT NULL | Amount paid |
| reference_no | VARCHAR(100) | | Check/Card reference |
| gateway_response_id | VARCHAR(100) | | Payment gateway transaction ID |
| payment_status | VARCHAR(20) | DEFAULT 'PENDING' | PENDING, SUCCESS, FAILED |
| created_at | TIMESTAMP | DEFAULT NOW() | PARTITION KEY |

**CREATE TABLE:**
```sql
CREATE TABLE payments (
  id BIGSERIAL PRIMARY KEY,
  sale_id BIGINT NOT NULL REFERENCES sales(id),
  payment_method VARCHAR(20) NOT NULL,
  amount DECIMAL(12,2) NOT NULL,
  reference_no VARCHAR(100),
  gateway_response_id VARCHAR(100),
  payment_status VARCHAR(20) DEFAULT 'PENDING',
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_payment_method CHECK (payment_method IN ('CASH', 'CARD', 'UPI', 'CHEQUE', 'WALLET'))
) PARTITION BY RANGE (DATE_TRUNC('month', created_at));
```

### 5.4 Payment Logs Table

**Purpose:** Detailed payment gateway interaction logs

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| payment_id | BIGINT | FK → payments.id | Payment reference |
| gateway_name | VARCHAR(50) | NOT NULL | Razorpay, PayU, etc. |
| request_data | JSONB | | Request sent to gateway |
| response_data | JSONB | | Response from gateway |
| response_code | VARCHAR(10) | | Gateway response code |
| error_message | TEXT | | Error message if failed |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE payment_logs (
  id BIGSERIAL PRIMARY KEY,
  payment_id BIGINT NOT NULL REFERENCES payments(id),
  gateway_name VARCHAR(50) NOT NULL,
  request_data JSONB,
  response_data JSONB,
  response_code VARCHAR(10),
  error_message TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 5.5 Returns Table

**Purpose:** Sales returns and refunds

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| original_sale_id | BIGINT | FK → sales.id | Sale being returned |
| return_date | DATE | NOT NULL | Return date |
| return_type | VARCHAR(20) | NOT NULL | FULL, PARTIAL |
| reason | TEXT | | Reason for return |
| return_amount | DECIMAL(12,2) | NOT NULL | Amount refunded |
| refund_status | VARCHAR(20) | DEFAULT 'PENDING' | PENDING, PROCESSED, REJECTED |
| approved_by | BIGINT | FK → users.id | Manager approval |
| approved_at | TIMESTAMP | | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE returns (
  id BIGSERIAL PRIMARY KEY,
  original_sale_id BIGINT NOT NULL REFERENCES sales(id),
  return_date DATE NOT NULL,
  return_type VARCHAR(20) NOT NULL,
  reason TEXT,
  return_amount DECIMAL(12,2) NOT NULL,
  refund_status VARCHAR(20) DEFAULT 'PENDING',
  approved_by BIGINT REFERENCES users(id),
  approved_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_return_type CHECK (return_type IN ('FULL', 'PARTIAL')),
  CONSTRAINT ck_return_amount CHECK (return_amount >= 0)
);
```

### 5.6 Invoices Table

**Purpose:** GST-compliant invoices for transactions

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| sale_id | BIGINT | FK → sales.id | Associated sale |
| invoice_no | VARCHAR(50) | UNIQUE, NOT NULL | Invoice number |
| invoice_date | DATE | NOT NULL | Invoice date |
| customer_name | VARCHAR(100) | | Customer name on invoice |
| customer_gstin | VARCHAR(15) | | Customer GST ID (B2B) |
| billing_address | TEXT | | Billing address |
| subtotal_before_tax | DECIMAL(12,2) | NOT NULL | |
| total_cgst | DECIMAL(10,2) | DEFAULT 0 | Central GST |
| total_sgst | DECIMAL(10,2) | DEFAULT 0 | State GST |
| total_igst | DECIMAL(10,2) | DEFAULT 0 | Integrated GST |
| total_tax | DECIMAL(10,2) | NOT NULL | Total GST |
| total_after_tax | DECIMAL(12,2) | NOT NULL | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE invoices (
  id BIGSERIAL PRIMARY KEY,
  sale_id BIGINT NOT NULL REFERENCES sales(id),
  invoice_no VARCHAR(50) UNIQUE NOT NULL,
  invoice_date DATE NOT NULL,
  customer_name VARCHAR(100),
  customer_gstin VARCHAR(15),
  billing_address TEXT,
  subtotal_before_tax DECIMAL(12,2) NOT NULL,
  total_cgst DECIMAL(10,2) DEFAULT 0,
  total_sgst DECIMAL(10,2) DEFAULT 0,
  total_igst DECIMAL(10,2) DEFAULT 0,
  total_tax DECIMAL(10,2) NOT NULL,
  total_after_tax DECIMAL(12,2) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_invoice_total CHECK (total_after_tax >= 0)
);
```

---

## SUPPLIER TABLES

### 6.1 Suppliers Table

**Purpose:** Supplier master data

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| supplier_code | VARCHAR(20) | UNIQUE, NOT NULL | Supplier ID |
| supplier_name | VARCHAR(150) | NOT NULL | Supplier name |
| contact_person | VARCHAR(100) | | Primary contact |
| phone | VARCHAR(15) | NOT NULL | Contact phone |
| email | VARCHAR(100) | NOT NULL | Contact email |
| address | TEXT | NOT NULL | Supplier address |
| city | VARCHAR(50) | NOT NULL | |
| state | VARCHAR(50) | NOT NULL | |
| postal_code | VARCHAR(10) | | |
| gstin | VARCHAR(15) | | Supplier GST ID |
| pan | VARCHAR(10) | | PAN number |
| bank_account | VARCHAR(20) | | Bank account (encrypted) |
| bank_name | VARCHAR(100) | | |
| ifsc_code | VARCHAR(11) | | |
| payment_terms | VARCHAR(50) | | Net 30, COD, etc. |
| credit_limit | DECIMAL(12,2) | | |
| is_active | BOOLEAN | DEFAULT true | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE suppliers (
  id BIGSERIAL PRIMARY KEY,
  supplier_code VARCHAR(20) UNIQUE NOT NULL,
  supplier_name VARCHAR(150) NOT NULL,
  contact_person VARCHAR(100),
  phone VARCHAR(15) NOT NULL,
  email VARCHAR(100) NOT NULL,
  address TEXT NOT NULL,
  city VARCHAR(50) NOT NULL,
  state VARCHAR(50) NOT NULL,
  postal_code VARCHAR(10),
  gstin VARCHAR(15),
  pan VARCHAR(10),
  bank_account VARCHAR(20),
  bank_name VARCHAR(100),
  ifsc_code VARCHAR(11),
  payment_terms VARCHAR(50),
  credit_limit DECIMAL(12,2),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 6.2 Supplier Pricing Table

**Purpose:** Product pricing from suppliers

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| supplier_id | BIGINT | FK → suppliers.id | Supplier reference |
| product_id | BIGINT | FK → products.id | Product reference |
| cost_price | DECIMAL(10,2) | NOT NULL | Cost from supplier |
| moq | DECIMAL(10,3) | NOT NULL | Minimum Order Quantity |
| lead_time_days | INTEGER | | Delivery lead time |
| effective_from | DATE | NOT NULL | Price validity start |
| effective_to | DATE | | Price validity end |
| is_current | BOOLEAN | DEFAULT true | Current pricing |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE supplier_pricing (
  id BIGSERIAL PRIMARY KEY,
  supplier_id BIGINT NOT NULL REFERENCES suppliers(id),
  product_id BIGINT NOT NULL REFERENCES products(id),
  cost_price DECIMAL(10,2) NOT NULL,
  moq DECIMAL(10,3) NOT NULL,
  lead_time_days INTEGER,
  effective_from DATE NOT NULL,
  effective_to DATE,
  is_current BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(supplier_id, product_id, effective_from),
  CONSTRAINT ck_cost_price CHECK (cost_price >= 0)
);
```

### 6.3 Purchase Orders Table

**Purpose:** Purchase orders to suppliers

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| po_number | VARCHAR(50) | UNIQUE, NOT NULL | PO identifier |
| supplier_id | BIGINT | FK → suppliers.id | Supplier |
| branch_id | BIGINT | FK → branches.id | Requesting branch |
| order_date | DATE | NOT NULL | PO creation date |
| expected_delivery_date | DATE | NOT NULL | Expected delivery |
| actual_delivery_date | DATE | | Actual delivery date |
| po_amount | DECIMAL(12,2) | NOT NULL | Total PO value |
| po_status | VARCHAR(20) | DEFAULT 'DRAFT' | DRAFT, SENT, ACKNOWLEDGED, SHIPPED, RECEIVED, INVOICED |
| created_by | BIGINT | FK → users.id | PO creator |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE purchase_orders (
  id BIGSERIAL PRIMARY KEY,
  po_number VARCHAR(50) UNIQUE NOT NULL,
  supplier_id BIGINT NOT NULL REFERENCES suppliers(id),
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  order_date DATE NOT NULL,
  expected_delivery_date DATE NOT NULL,
  actual_delivery_date DATE,
  po_amount DECIMAL(12,2) NOT NULL,
  po_status VARCHAR(20) DEFAULT 'DRAFT',
  created_by BIGINT REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_po_status CHECK (po_status IN ('DRAFT', 'SENT', 'ACKNOWLEDGED', 'SHIPPED', 'RECEIVED', 'INVOICED')),
  CONSTRAINT ck_po_amount CHECK (po_amount >= 0)
);
```

### 6.4 PO Items Table

**Purpose:** Line items in purchase orders

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| po_id | BIGINT | FK → purchase_orders.id | Parent PO |
| product_id | BIGINT | FK → products.id | Ordered product |
| quantity_ordered | DECIMAL(10,3) | NOT NULL | Order quantity |
| unit_cost | DECIMAL(10,2) | NOT NULL | Cost per unit |
| line_total | DECIMAL(12,2) | NOT NULL | Qty × Cost |
| quantity_received | DECIMAL(10,3) | DEFAULT 0 | Quantity received so far |
| quantity_rejected | DECIMAL(10,3) | DEFAULT 0 | Quantity rejected |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE po_items (
  id BIGSERIAL PRIMARY KEY,
  po_id BIGINT NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
  product_id BIGINT NOT NULL REFERENCES products(id),
  quantity_ordered DECIMAL(10,3) NOT NULL,
  unit_cost DECIMAL(10,2) NOT NULL,
  line_total DECIMAL(12,2) NOT NULL,
  quantity_received DECIMAL(10,3) DEFAULT 0,
  quantity_rejected DECIMAL(10,3) DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_qty_ordered CHECK (quantity_ordered > 0)
);
```

### 6.5 Goods Receipt Table

**Purpose:** Receipt of goods from PO

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| po_id | BIGINT | FK → purchase_orders.id | Related PO |
| receipt_date | DATE | NOT NULL | Receipt date |
| received_by | BIGINT | FK → users.id | Warehouse staff |
| quality_checked | BOOLEAN | DEFAULT false | QC done |
| quality_checked_by | BIGINT | FK → users.id | QC staff |
| quality_check_notes | TEXT | | QC notes |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE goods_receipt (
  id BIGSERIAL PRIMARY KEY,
  po_id BIGINT NOT NULL REFERENCES purchase_orders(id),
  receipt_date DATE NOT NULL,
  received_by BIGINT REFERENCES users(id),
  quality_checked BOOLEAN DEFAULT false,
  quality_checked_by BIGINT REFERENCES users(id),
  quality_check_notes TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## CUSTOMER TABLES

### 7.1 Customers Table

**Purpose:** Customer master data

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| customer_code | VARCHAR(20) | UNIQUE | Customer ID (optional) |
| phone | VARCHAR(15) | | Primary phone (unique for registration) |
| email | VARCHAR(100) | | Email address |
| first_name | VARCHAR(100) | | First name |
| last_name | VARCHAR(100) | | Last name |
| address | TEXT | | Home address |
| city | VARCHAR(50) | | |
| state | VARCHAR(50) | | |
| postal_code | VARCHAR(10) | | |
| customer_type | VARCHAR(20) | DEFAULT 'REGULAR' | REGULAR, VIP, WHOLESALE |
| gstin | VARCHAR(15) | | GST ID (B2B customers) |
| total_purchases | DECIMAL(12,2) | DEFAULT 0 | Lifetime purchases |
| total_returns | DECIMAL(12,2) | DEFAULT 0 | Lifetime returns |
| is_active | BOOLEAN | DEFAULT true | |
| created_at | TIMESTAMP | DEFAULT NOW() | |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE customers (
  id BIGSERIAL PRIMARY KEY,
  customer_code VARCHAR(20) UNIQUE,
  phone VARCHAR(15),
  email VARCHAR(100),
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  address TEXT,
  city VARCHAR(50),
  state VARCHAR(50),
  postal_code VARCHAR(10),
  customer_type VARCHAR(20) DEFAULT 'REGULAR',
  gstin VARCHAR(15),
  total_purchases DECIMAL(12,2) DEFAULT 0,
  total_returns DECIMAL(12,2) DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_customer_type CHECK (customer_type IN ('REGULAR', 'VIP', 'WHOLESALE'))
);
```

### 7.2 Loyalty Points Table

**Purpose:** Customer loyalty program tracking

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| customer_id | BIGINT | FK → customers.id | Customer reference |
| points_balance | DECIMAL(12,2) | NOT NULL | Current points |
| lifetime_earned | DECIMAL(12,2) | DEFAULT 0 | Total earned |
| lifetime_redeemed | DECIMAL(12,2) | DEFAULT 0 | Total redeemed |
| last_transaction_date | TIMESTAMP | | Last transaction date |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE loyalty_points (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT UNIQUE NOT NULL REFERENCES customers(id),
  points_balance DECIMAL(12,2) NOT NULL DEFAULT 0,
  lifetime_earned DECIMAL(12,2) DEFAULT 0,
  lifetime_redeemed DECIMAL(12,2) DEFAULT 0,
  last_transaction_date TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_points_balance CHECK (points_balance >= 0)
);
```

---

## FINANCIAL TABLES

### 8.1 GST Config Table

**Purpose:** GST configuration and rates

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| hsn_code | VARCHAR(10) | NOT NULL | HSN code range |
| product_category | VARCHAR(100) | | Category description |
| gst_rate | DECIMAL(5,2) | NOT NULL | GST rate (0, 5, 12, 18, 28) |
| effective_from | DATE | NOT NULL | Rate validity |
| effective_to | DATE | | |
| is_active | BOOLEAN | DEFAULT true | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE gst_config (
  id BIGSERIAL PRIMARY KEY,
  hsn_code VARCHAR(10) NOT NULL,
  product_category VARCHAR(100),
  gst_rate DECIMAL(5,2) NOT NULL,
  effective_from DATE NOT NULL,
  effective_to DATE,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_gst_rate CHECK (gst_rate IN (0, 5, 12, 18, 28))
);
```

### 8.2 GST Summary Table

**Purpose:** Monthly GST compliance summary

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| branch_id | BIGINT | FK → branches.id | Branch reference |
| summary_month | INTEGER | NOT NULL | Month (1-12) |
| summary_year | INTEGER | NOT NULL | Year |
| total_taxable_sales | DECIMAL(12,2) | | |
| total_gst_collected | DECIMAL(12,2) | DEFAULT 0 | IGST + SGST + CGST |
| total_sgst | DECIMAL(12,2) | DEFAULT 0 | State GST |
| total_cgst | DECIMAL(12,2) | DEFAULT 0 | Central GST |
| total_igst | DECIMAL(12,2) | DEFAULT 0 | Interstate GST |
| gst_return_filed | BOOLEAN | DEFAULT false | GSTR-3B filed |
| filed_date | DATE | | |
| created_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE gst_summary (
  id BIGSERIAL PRIMARY KEY,
  branch_id BIGINT NOT NULL REFERENCES branches(id),
  summary_month INTEGER NOT NULL,
  summary_year INTEGER NOT NULL,
  total_taxable_sales DECIMAL(12,2),
  total_gst_collected DECIMAL(12,2) DEFAULT 0,
  total_sgst DECIMAL(12,2) DEFAULT 0,
  total_cgst DECIMAL(12,2) DEFAULT 0,
  total_igst DECIMAL(12,2) DEFAULT 0,
  gst_return_filed BOOLEAN DEFAULT false,
  filed_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(branch_id, summary_month, summary_year)
);
```

---

## AUDIT & COMPLIANCE TABLES

### 9.1 Audit Logs Table

**Purpose:** Immutable audit trail for compliance

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| user_id | BIGINT | FK → users.id | User who made change |
| action | VARCHAR(50) | NOT NULL | CREATE, UPDATE, DELETE, READ |
| resource_type | VARCHAR(50) | NOT NULL | Table name (users, sales, etc.) |
| resource_id | BIGINT | | Record ID affected |
| old_value | JSONB | | Previous value |
| new_value | JSONB | | New value |
| ip_address | INET | | IP address of user |
| user_agent | TEXT | | Browser/client info |
| status | VARCHAR(20) | DEFAULT 'SUCCESS' | SUCCESS, FAILURE |
| error_message | TEXT | | Error if failed |
| created_at | TIMESTAMP | DEFAULT NOW() | MUST partition on this |

**CREATE TABLE:**
```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT REFERENCES users(id),
  action VARCHAR(50) NOT NULL,
  resource_type VARCHAR(50) NOT NULL,
  resource_id BIGINT,
  old_value JSONB,
  new_value JSONB,
  ip_address INET,
  user_agent TEXT,
  status VARCHAR(20) DEFAULT 'SUCCESS',
  error_message TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT ck_action CHECK (action IN ('CREATE', 'READ', 'UPDATE', 'DELETE'))
) PARTITION BY RANGE (DATE_TRUNC('month', created_at));
```

### 9.2 System Configuration Table

**Purpose:** System settings and configuration

| Column | Data Type | Constraints | Notes |
|--------|-----------|------------|-------|
| id | BIGSERIAL | PK | |
| config_key | VARCHAR(100) | UNIQUE, NOT NULL | Configuration key |
| config_value | TEXT | | Configuration value |
| config_type | VARCHAR(20) | | STRING, INTEGER, BOOLEAN, JSON |
| description | TEXT | | Purpose |
| updated_at | TIMESTAMP | DEFAULT NOW() | |

**CREATE TABLE:**
```sql
CREATE TABLE system_config (
  id BIGSERIAL PRIMARY KEY,
  config_key VARCHAR(100) UNIQUE NOT NULL,
  config_value TEXT,
  config_type VARCHAR(20),
  description TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## INDEXES STRATEGY

### 10.1 Primary Table Indexes

```sql
-- Users Table Indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_branch_id ON users(branch_id);
CREATE INDEX idx_users_role_id ON users(role_id);
CREATE INDEX idx_users_is_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Products Table Indexes
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_barcode ON products(barcode);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_is_active ON products(is_active) WHERE is_active = true;
CREATE INDEX idx_products_hsn_code ON products(hsn_code);

-- Inventory Table Indexes (Composite for common queries)
CREATE INDEX idx_inventory_branch_product ON inventory(branch_id, product_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_low_stock ON inventory(quantity_on_hand) 
  WHERE quantity_on_hand < reorder_level;
CREATE INDEX idx_inventory_last_movement ON inventory(last_movement_date DESC);

-- Sales Table Indexes (IMPORTANT - heavily queried)
CREATE INDEX idx_sales_branch_id ON sales(branch_id);
CREATE INDEX idx_sales_cashier_id ON sales(cashier_id);
CREATE INDEX idx_sales_customer_id ON sales(customer_id);
CREATE INDEX idx_sales_transaction_no ON sales(transaction_no);
CREATE INDEX idx_sales_created_at ON sales(created_at DESC);
CREATE INDEX idx_sales_payment_status ON sales(payment_status);
CREATE INDEX idx_sales_transaction_status ON sales(transaction_status);
-- Composite for date range queries
CREATE INDEX idx_sales_date_branch ON sales(created_at DESC, branch_id) 
  INCLUDE (total_amount, total_tax);

-- Sale Items Table Indexes
CREATE INDEX idx_sale_items_sale_id ON sale_items(sale_id);
CREATE INDEX idx_sale_items_product_id ON sale_items(product_id);

-- Stock Movements Table Indexes (Time-series)
CREATE INDEX idx_stock_movements_inventory ON stock_movements(inventory_id);
CREATE INDEX idx_stock_movements_created_at ON stock_movements(created_at DESC);
CREATE INDEX idx_stock_movements_type ON stock_movements(movement_type);
CREATE INDEX idx_stock_movements_branch_date ON stock_movements(
  DATE_TRUNC('month', created_at), 
  inventory_id
);

-- Customers Table Indexes
CREATE INDEX idx_customers_phone ON customers(phone);
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_customer_type ON customers(customer_type);
CREATE INDEX idx_customers_is_active ON customers(is_active) WHERE is_active = true;

-- Suppliers Table Indexes
CREATE INDEX idx_suppliers_supplier_code ON suppliers(supplier_code);
CREATE INDEX idx_suppliers_is_active ON suppliers(is_active) WHERE is_active = true;

-- Purchase Orders Indexes
CREATE INDEX idx_purchase_orders_supplier_id ON purchase_orders(supplier_id);
CREATE INDEX idx_purchase_orders_branch_id ON purchase_orders(branch_id);
CREATE INDEX idx_purchase_orders_po_number ON purchase_orders(po_number);
CREATE INDEX idx_purchase_orders_po_status ON purchase_orders(po_status);
CREATE INDEX idx_purchase_orders_created_date ON purchase_orders(order_date DESC);

-- Payments Table Indexes (Time-series)
CREATE INDEX idx_payments_sale_id ON payments(sale_id);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
CREATE INDEX idx_payments_payment_status ON payments(payment_status);
CREATE INDEX idx_payments_gateway ON payments(gateway_response_id);

-- Audit Logs Indexes (Important for compliance)
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- Batch Tracking Indexes (Expiry tracking)
CREATE INDEX idx_batch_tracking_expiry ON batch_tracking(expiry_date);
CREATE INDEX idx_batch_tracking_inventory ON batch_tracking(inventory_id);
```

### 10.2 Covering Indexes for Performance

```sql
-- Sales reporting without table lookup
CREATE INDEX idx_sales_reporting ON sales(branch_id, created_at DESC) 
  INCLUDE (total_amount, total_tax, payment_status, total_items);

-- Inventory status without additional lookup
CREATE INDEX idx_inventory_status ON inventory(branch_id, product_id) 
  INCLUDE (quantity_on_hand, quantity_available, reorder_level);

-- Customer loyalty reporting
CREATE INDEX idx_loyalty_reporting ON loyalty_points(customer_id) 
  INCLUDE (points_balance, lifetime_earned);
```

### 10.3 Partial Indexes for Common Filters

```sql
-- Only active products (most queries use this)
CREATE INDEX idx_products_active ON products(category_id) 
  WHERE is_active = true;

-- Active customers only
CREATE INDEX idx_customers_active_type ON customers(customer_type) 
  WHERE is_active = true;

-- Pending returns
CREATE INDEX idx_returns_pending ON returns(refund_status) 
  WHERE refund_status = 'PENDING';

-- Incomplete orders
CREATE INDEX idx_po_incomplete ON purchase_orders(po_status) 
  WHERE po_status NOT IN ('RECEIVED', 'INVOICED', 'CANCELLED');
```

### 10.4 BRIN Indexes for Time-Series

```sql
-- For very large time-series tables (sales over years)
CREATE INDEX idx_sales_created_brin ON sales USING BRIN (created_at) 
  WITH (pages_per_range = 128);

-- Stock movements time-series
CREATE INDEX idx_stock_movements_created_brin ON stock_movements 
  USING BRIN (created_at) WITH (pages_per_range = 128);

-- Payments time-series
CREATE INDEX idx_payments_created_brin ON payments USING BRIN (created_at) 
  WITH (pages_per_range = 128);

-- Audit logs time-series
CREATE INDEX idx_audit_logs_created_brin ON audit_logs USING BRIN (created_at) 
  WITH (pages_per_range = 256);
```

---

## VIEWS & MATERIALIZED VIEWS

### 11.1 Operational Views

```sql
-- Current Inventory Status View
CREATE VIEW v_inventory_status AS
SELECT 
  p.sku,
  p.product_name,
  b.branch_code,
  inv.quantity_on_hand,
  inv.quantity_available,
  inv.reorder_level,
  CASE 
    WHEN inv.quantity_on_hand < inv.reorder_level THEN 'LOW'
    WHEN inv.quantity_on_hand > inv.max_level THEN 'OVERSTOCK'
    ELSE 'NORMAL'
  END as stock_status
FROM inventory inv
JOIN products p ON inv.product_id = p.id
JOIN branches b ON inv.branch_id = b.id
WHERE p.is_active = true;

-- Daily Sales Summary View
CREATE VIEW v_daily_sales_summary AS
SELECT 
  DATE(s.created_at) as sale_date,
  b.branch_code,
  COUNT(s.id) as transaction_count,
  SUM(s.total_items) as total_items,
  SUM(s.subtotal_amount) as subtotal,
  SUM(s.total_tax) as total_tax,
  SUM(s.total_amount) as gross_sales
FROM sales s
JOIN branches b ON s.branch_id = b.id
WHERE s.transaction_status = 'COMPLETED'
GROUP BY DATE(s.created_at), b.branch_code;

-- Customer Purchase History
CREATE VIEW v_customer_purchase_history AS
SELECT 
  c.customer_code,
  c.first_name || ' ' || c.last_name as customer_name,
  COUNT(DISTINCT s.id) as purchase_count,
  MAX(s.created_at) as last_purchase_date,
  SUM(s.total_amount) as total_spent,
  AVG(s.total_amount) as avg_purchase_amount
FROM customers c
LEFT JOIN sales s ON c.id = s.customer_id AND s.transaction_status = 'COMPLETED'
WHERE c.is_active = true
GROUP BY c.id, c.customer_code, c.first_name, c.last_name;
```

### 11.2 Materialized Views for Reporting

```sql
-- Monthly Sales Summary (Materialized for performance)
CREATE MATERIALIZED VIEW mv_monthly_sales_summary AS
SELECT 
  DATE_TRUNC('month', s.created_at)::DATE as month,
  b.id as branch_id,
  b.branch_code,
  COUNT(s.id) as transaction_count,
  SUM(s.total_items) as total_items_sold,
  SUM(s.subtotal_amount) as subtotal_amount,
  SUM(s.total_tax) as total_tax,
  SUM(s.total_amount) as gross_sales,
  SUM(CASE WHEN s.transaction_status = 'COMPLETED' THEN s.total_amount ELSE 0 END) as completed_sales,
  SUM(CASE WHEN s.return_status IN ('PARTIAL_RETURN', 'FULL_RETURN') THEN s.total_amount ELSE 0 END) as returned_amount
FROM sales s
JOIN branches b ON s.branch_id = b.id
GROUP BY DATE_TRUNC('month', s.created_at), b.id, b.branch_code;

CREATE INDEX idx_mv_monthly_sales ON mv_monthly_sales_summary(month, branch_id);

-- Category Sales Performance (Materialized)
CREATE MATERIALIZED VIEW mv_category_sales_performance AS
SELECT 
  DATE_TRUNC('month', s.created_at)::DATE as month,
  c.category_name,
  COUNT(DISTINCT si.id) as items_sold,
  SUM(si.quantity) as total_quantity,
  SUM(si.line_total) as total_amount,
  AVG(si.line_total) as avg_item_value
FROM sales s
JOIN sale_items si ON s.id = si.sale_id
JOIN products p ON si.product_id = p.id
JOIN categories c ON p.category_id = c.id
WHERE s.transaction_status = 'COMPLETED'
GROUP BY DATE_TRUNC('month', s.created_at), c.category_name;

CREATE INDEX idx_mv_category_sales ON mv_category_sales_performance(month, category_name);

-- Refresh materialized views strategy:
-- Option 1: Daily refresh at 2 AM
-- Option 2: Incremental updates based on triggers
-- Option 3: Manual refresh after month-end close
```

---

## PARTITIONING STRATEGY

### 12.1 Partition Large Tables

```sql
-- Sales table partitioned by month
CREATE TABLE sales_2026_01 PARTITION OF sales
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE sales_2026_02 PARTITION OF sales
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Auto-create partitions for next 2 years using function/trigger

-- Stock Movements partitioned by month
CREATE TABLE stock_movements_2026_01 PARTITION OF stock_movements
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Payments partitioned by month
CREATE TABLE payments_2026_01 PARTITION OF payments
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Audit Logs partitioned by month
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### 12.2 Archive Old Data

```sql
-- Archive function to move data >2 years old to separate schema
CREATE SCHEMA archived;

-- Procedure to archive old sales
CREATE OR REPLACE FUNCTION archive_old_sales()
RETURNS void AS $$
BEGIN
  CREATE TABLE IF NOT EXISTS archived.sales_old AS
  SELECT * FROM sales 
  WHERE created_at < (CURRENT_DATE - INTERVAL '2 years');
  
  DELETE FROM sales 
  WHERE created_at < (CURRENT_DATE - INTERVAL '2 years');
  
  ANALYZE sales;
END;
$$ LANGUAGE plpgsql;

-- Schedule execution monthly
-- SELECT cron.schedule('archive_old_sales', '0 2 1 * *', 'SELECT archive_old_sales()');
```

---

## RELATIONSHIPS & CONSTRAINTS

### 13.1 Foreign Key Constraints

```sql
-- Branch-related
ALTER TABLE users ADD CONSTRAINT fk_users_branch 
  FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE RESTRICT;

ALTER TABLE users ADD CONSTRAINT fk_users_role 
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE RESTRICT;

-- Products
ALTER TABLE products ADD CONSTRAINT fk_products_category 
  FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE RESTRICT;

-- Inventory
ALTER TABLE inventory ADD CONSTRAINT fk_inventory_product 
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE;

ALTER TABLE inventory ADD CONSTRAINT fk_inventory_branch 
  FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE CASCADE;

-- Sales
ALTER TABLE sales ADD CONSTRAINT fk_sales_branch 
  FOREIGN KEY (branch_id) REFERENCES branches(id) ON DELETE RESTRICT;

ALTER TABLE sales ADD CONSTRAINT fk_sales_cashier 
  FOREIGN KEY (cashier_id) REFERENCES users(id) ON DELETE RESTRICT;

-- Supplier relationships
ALTER TABLE supplier_pricing ADD CONSTRAINT fk_supplier_pricing_supplier 
  FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE CASCADE;

ALTER TABLE supplier_pricing ADD CONSTRAINT fk_supplier_pricing_product 
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE;
```

### 13.2 Check Constraints

```sql
-- Data validation
ALTER TABLE products ADD CONSTRAINT ck_gst_valid 
  CHECK (gst_rate IN (0, 5, 12, 18, 28));

ALTER TABLE sales ADD CONSTRAINT ck_amounts_positive 
  CHECK (total_amount >= 0 AND total_tax >= 0);

ALTER TABLE inventory ADD CONSTRAINT ck_qty_valid 
  CHECK (quantity_on_hand >= 0 AND quantity_available >= 0);

ALTER TABLE purchase_orders ADD CONSTRAINT ck_po_dates 
  CHECK (order_date <= expected_delivery_date);
```

---

## SAMPLE QUERIES

### 14.1 Critical Business Queries

```sql
-- Query 1: Daily Sales Report by Branch
SELECT 
  b.branch_code,
  DATE(s.created_at)::DATE as sale_date,
  COUNT(DISTINCT s.id) as num_transactions,
  SUM(s.total_amount) as total_revenue,
  SUM(s.total_tax) as total_gst,
  AVG(s.total_amount) as avg_transaction
FROM sales s
JOIN branches b ON s.branch_id = b.id
WHERE s.created_at >= CURRENT_DATE - INTERVAL '30 days'
  AND s.transaction_status = 'COMPLETED'
GROUP BY b.branch_code, DATE(s.created_at)
ORDER BY DATE(s.created_at) DESC, b.branch_code;

-- Query 2: Low Stock Alert
SELECT 
  p.sku,
  p.product_name,
  b.branch_code,
  inv.quantity_on_hand,
  inv.reorder_level,
  (inv.reorder_level - inv.quantity_on_hand) as shortage
FROM inventory inv
JOIN products p ON inv.product_id = p.id
JOIN branches b ON inv.branch_id = b.id
WHERE inv.quantity_on_hand < inv.reorder_level
  AND p.is_active = true
ORDER BY shortage DESC;

-- Query 3: GST Compliance Report
SELECT 
  g.summary_month,
  g.summary_year,
  b.branch_code,
  g.total_taxable_sales,
  g.total_sgst,
  g.total_cgst,
  g.total_igst,
  (g.total_sgst + g.total_cgst + g.total_igst) as total_gst,
  g.gst_return_filed,
  g.filed_date
FROM gst_summary g
JOIN branches b ON g.branch_id = b.id
WHERE g.gst_return_filed = false
ORDER BY g.summary_year DESC, g.summary_month DESC;

-- Query 4: Top 10 Products by Sales
SELECT 
  p.sku,
  p.product_name,
  SUM(si.quantity) as units_sold,
  SUM(si.line_total) as revenue,
  COUNT(DISTINCT s.id) as transaction_count,
  AVG(si.unit_price) as avg_selling_price
FROM sale_items si
JOIN sales s ON si.sale_id = s.id
JOIN products p ON si.product_id = p.id
WHERE s.created_at >= CURRENT_DATE - INTERVAL '30 days'
  AND s.transaction_status = 'COMPLETED'
GROUP BY p.sku, p.product_name
ORDER BY revenue DESC
LIMIT 10;

-- Query 5: Inventory Valuation
SELECT 
  b.branch_code,
  c.category_name,
  COUNT(DISTINCT p.id) as num_products,
  SUM(inv.quantity_on_hand) as total_units,
  SUM(inv.quantity_on_hand * p.price) as inventory_value
FROM inventory inv
JOIN products p ON inv.product_id = p.id
JOIN branches b ON inv.branch_id = b.id
JOIN categories c ON p.category_id = c.id
WHERE p.is_active = true
GROUP BY b.branch_code, c.category_name
ORDER BY b.branch_code, inventory_value DESC;
```

---

## MAINTENANCE STRATEGIES

### 15.1 Regular Maintenance

```sql
-- Weekly maintenance
ANALYZE; -- Update query planner statistics
REINDEX INDEX CONCURRENTLY idx_sales_created_at; -- Rebuild critical indexes

-- Monthly maintenance
VACUUM FULL;
REINDEX DATABASE CONCURRENTLY supermarket_billing;

-- Quarterly: Archive old data
SELECT archive_old_sales();

-- Monitor table sizes
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-15 | Senior Database Architect | Initial PostgreSQL Schema Design |

**Classification:** Internal - Confidential  
**Last Updated:** 2026-06-15  
**Next Review:** 2026-09-15