# System Architecture - MVP Design
## Enterprise Supermarket Billing System

**Version:** 1.0  
**Date:** 2026-06-15  
**Architect:** Principal Architect  
**Status:** Design Document  

---

## 1. HIGH-LEVEL ARCHITECTURE

### 1.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│                                                                       │
│  ┌──────────────────────┐         ┌──────────────────────────┐      │
│  │   React Web App      │         │   Mobile Responsive UI   │      │
│  │                      │         │   (Tablet/Mobile)        │      │
│  │ - POS Dashboard      │         │                          │      │
│  │ - Inventory Module   │         │ - Cash Register View     │      │
│  │ - Reporting Screens  │         │ - Quick Barcode Scan     │      │
│  │ - Admin Panel        │         │ - Receipt Print          │      │
│  └──────────────────────┘         └──────────────────────────┘      │
└─────────┬──────────────────────────────────────────────────┬────────┘
          │                                                  │
          │         HTTPS + JWT Token in Header             │
          │                                                  │
┌─────────┴──────────────────────────────────────────────────┴────────┐
│                    API GATEWAY & AUTH LAYER                          │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Express Middleware Stack:                                      │ │
│  │ • JWT Verification                                             │ │
│  │ • CORS Validation                                              │ │
│  │ • Rate Limiting (100 req/min per IP)                           │ │
│  │ • Request Logging & Audit Trail                                │ │
│  │ • Error Handling & Response Formatting                         │ │
│  │ • Branch Context Extraction                                    │ │
│  │ • Permission Validation (RBAC)                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────┬──────────────────────────────────────────────────────────┘
          │
┌─────────┴──────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                              │
│                    (Node.js + Express)                              │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   Auth Service   │  │   POS Service    │  │  Inventory Svc   │  │
│  │                  │  │                  │  │                  │  │
│  │ • Login/Logout   │  │ • Transactions   │  │ • Stock Mgmt     │  │
│  │ • Token Gen      │  │ • Billing        │  │ • Alerts         │  │
│  │ • 2FA (future)   │  │ • GST Calc       │  │ • Stock Transfer │  │
│  │ • User Mgmt      │  │ • Receipt Gen    │  │ • Expiry Track   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ Customer Service │  │ Supplier Service │  │ Reporting Svc    │  │
│  │                  │  │                  │  │                  │  │
│  │ • Customer Data  │  │ • PO Mgmt        │  │ • Sales Reports  │  │
│  │ • Loyalty Track  │  │ • Supplier Data  │  │ • P&L Analysis   │  │
│  │ • Analytics      │  │ • Payment Track  │  │ • GST Reports    │  │
│  │ • Search         │  │ • Reconcile      │  │ • Dashboards     │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐                         │
│  │ Payment Service  │  │ Audit Service    │                         │
│  │                  │  │                  │                         │
│  │ • Payment Gateway│  │ • Audit Logs     │                         │
│  │ • Reconciliation │  │ • Compliance     │                         │
│  │ • Settlement     │  │ • Investigation  │                         │
│  └──────────────────┘  └──────────────────┘                         │
└─────────┬──────────────────────────────────────────────────────────┘
          │
          │  SQL Queries / Cache Invalidation
          │
┌─────────┴──────────────────────────────────────────────────────────┐
│                    DATA & CACHE LAYER                               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │            PostgreSQL Database                               │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                   │  │
│  │  │  Primary DB     │  │  Read Replica   │                   │  │
│  │  │  (Write Master) │  │  (Analytics)    │                   │  │
│  │  └─────────────────┘  └─────────────────┘                   │  │
│  │  Tables: Users, Products, Inventory, Transactions,           │  │
│  │          Customers, Suppliers, Invoices, Audit Logs, etc.   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │            Redis Cache (2 Deployments)                        │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │ Session Cache:          | Real-time Cache:             │ │  │
│  │  │ • JWT Tokens            | • Product Catalog            │ │  │
│  │  │ • User Sessions         | • Inventory Levels           │ │  │
│  │  │ • 2FA OTP               | • Exchange Rates (future)    │ │  │
│  │  │ • Rate Limit Counters   | • Supplier Lists             │ │  │
│  │  │ TTL: 30 min             | TTL: 1 hour / event-driven   │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Architectural Patterns

| Pattern | Implementation | Benefit |
|---------|----------------|---------|
| **Microservices** | Service per domain (POS, Inventory, etc.) | Independent scaling, modularity |
| **API-First** | REST APIs with OpenAPI specs | Frontend/backend decoupling |
| **JWT Authentication** | Token-based stateless auth | Scalability, security |
| **RBAC** | Role-Permission matrix enforced | Fine-grained access control |
| **Database per Service** | Shared DB (MVP), separate (future) | Transaction consistency, scalability path |
| **Caching Strategy** | Redis 2-tier (sessions + real-time) | Performance optimization |
| **Audit Trail** | Immutable event log table | Compliance, debugging |
| **Error Handling** | Centralized middleware | Consistent error responses |

### 1.3 Technology Stack

```
Frontend:
├── React 18.2 (Core framework)
├── Redux Toolkit (State management)
├── React Router v6 (Navigation)
├── Axios (HTTP client)
├── Material-UI v5 (UI Components)
├── Chart.js/Recharts (Dashboards)
├── React Query (Data caching)
├── Formik + Yup (Form validation)
└── ESLint + Prettier (Code quality)

Backend:
├── Node.js 18+ LTS
├── Express.js 4.18 (Framework)
├── Passport.js (Authentication)
├── Joi (Input validation)
├── Winston (Logging)
├── Jest + Supertest (Testing)
├── PM2 (Process management)
└── dotenv (Config management)

Database:
├── PostgreSQL 14+
├── Sequelize ORM (MVP simplicity)
├── Migration tools (db-migrate)
├── Backup & replication setup
└── Read replicas (future)

Cache:
├── Redis 7+
├── RedisJSON (Complex data)
└── Redis Pub/Sub (Real-time notifications)

DevOps:
├── Docker + Docker Compose
├── Git + GitHub
├── GitHub Actions (CI/CD)
├── AWS (EC2, RDS, S3, Route53)
└── Nginx (Reverse proxy)
```

---

## 2. COMPONENT DIAGRAM

### 2.1 Service Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                     FRONTEND (React)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  POS Dashboard │ Inventory │ Customers │ Reports │ Admin │   │
│  └──────────────┬─────────────────────────────────────────┘    │
└────────────────┼─────────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌─────────────┬──────────────┬──────────────┐
│   Auth      │  Inventory   │  Reporting   │
│  Service    │  Service     │  Service     │
│             │              │              │
│ ┌─────────┐ │ ┌─────────┐ │ ┌─────────┐ │
│ │validate │ │ │stockMgmt│ │ │generate │ │
│ │token    │ │ │updateLvl│ │ │Reports  │ │
│ │genToken │ │ │expiry   │ │ │getData  │ │
│ └─────────┘ │ └─────────┘ │ └─────────┘ │
└─────┬───────┴────┬────────┴──────┬──────┘
      │            │               │
      ▼            ▼               ▼
   ┌────────────────────────────────────┐
   │    Shared Middleware Layer         │
   │  ┌──────────────────────────────┐  │
   │  │ CORS | Logging | ErrorHandle │  │
   │  │ RateLimit | AuditTrail       │  │
   │  └──────────────────────────────┘  │
   └────────┬───────────────────────────┘
            │
    ┌───────┼───────┐
    │       │       │
    ▼       ▼       ▼
  ┌──────────────────────────────┐
  │   PostgreSQL               │
  │  ┌────────────────────────┐│
  │  │ Users │ Products │     ││
  │  │ Transactions │ Logs   ││
  │  │ Customers │ Suppliers ││
  │  └────────────────────────┘│
  └─────────┬──────────────────┘
            │
    ┌───────┴───────┐
    │               │
    ▼               ▼
  ┌──────────┐  ┌──────────┐
  │  Redis   │  │  Redis   │
  │ (Session)│  │(Catalog) │
  └──────────┘  └──────────┘
```

### 2.2 Service-to-Service Communication

```
POS Service
├─ Calls Inventory Service → Check stock
├─ Calls Customer Service → Get loyalty
├─ Calls Payment Service → Process payment
├─ Calls Reporting Service → Log transaction
└─ Calls Audit Service → Log transaction

Inventory Service
├─ Reads from PostgreSQL
├─ Updates Redis cache
├─ Publishes to Redis Pub/Sub (stock alerts)
└─ Calls Audit Service

Payment Service
├─ Integrates with Razorpay API
├─ Calls Reporting Service → Log payment
├─ Caches payment gateway responses
└─ Calls Audit Service

Reporting Service
├─ Reads from PostgreSQL (Main DB)
├─ Reads from Read Replica (Analytics)
├─ Aggregates data
└─ Returns to Frontend
```

### 2.3 External Integrations

```
┌─────────────────────────────┐
│   Payment Gateways          │
├─────────────────────────────┤
│ • Razorpay (Cards/UPI)      │
│ • PayU (Wallet/EMI future)  │
│ • Bank Reconciliation APIs  │
└─────────┬───────────────────┘
          │
          ▼
   ┌─────────────────┐
   │ Payment Service │
   └─────────────────┘

┌─────────────────────────────┐
│   GST/Tax APIs              │
├─────────────────────────────┤
│ • GST SUSI Portal (future)  │
│ • Tax Calculation Engine    │
│ • E-Invoice Generation      │
└─────────┬───────────────────┘
          │
          ▼
   ┌─────────────────┐
   │ Billing Service │
   └─────────────────┘

┌─────────────────────────────┐
│   Email/SMS Services        │
├─────────────────────────────┤
│ • AWS SES (Email)           │
│ • AWS SNS (SMS)             │
│ • Twilio (SMS backup)       │
└─────────┬───────────────────┘
          │
          ▼
   ┌─────────────────┐
   │ Notification    │
   │ Service         │
   └─────────────────┘
```

---

## 3. FOLDER STRUCTURE

### 3.1 Backend Structure (Node.js + Express)

```
supermarket-billing-backend/
├── src/
│   ├── config/
│   │   ├── database.js          # PostgreSQL connection config
│   │   ├── redis.js              # Redis client setup
│   │   ├── environment.js         # Environment variables
│   │   └── constants.js           # App-wide constants
│   │
│   ├── middleware/
│   │   ├── auth.middleware.js     # JWT verification
│   │   ├── rbac.middleware.js     # Role-based access control
│   │   ├── errorHandler.js        # Global error handling
│   │   ├── requestLogger.js       # Request/response logging
│   │   ├── rateLimiter.js         # Rate limiting
│   │   ├── auditTrail.js          # Audit logging
│   │   └── cors.js                # CORS configuration
│   │
│   ├── services/
│   │   ├── auth/
│   │   │   ├── AuthService.js     # Login, token generation
│   │   │   ├── JWTService.js      # JWT operations
│   │   │   └── PasswordService.js # Hashing, validation
│   │   │
│   │   ├── pos/
│   │   │   ├── TransactionService.js    # Transaction processing
│   │   │   ├── BillingService.js        # Bill calculation, GST
│   │   │   ├── ReceiptService.js        # Receipt generation
│   │   │   └── RefundService.js         # Refund processing
│   │   │
│   │   ├── inventory/
│   │   │   ├── InventoryService.js      # Stock management
│   │   │   ├── StockAlertService.js     # Low-stock alerts
│   │   │   ├── ExpiryService.js         # Expiry tracking
│   │   │   └── StockTransferService.js  # Inter-branch transfers
│   │   │
│   │   ├── customer/
│   │   │   ├── CustomerService.js       # Customer master
│   │   │   ├── LoyaltyService.js        # Loyalty points
│   │   │   └── SegmentationService.js   # Customer analytics
│   │   │
│   │   ├── supplier/
│   │   │   ├── SupplierService.js       # Supplier management
│   │   │   ├── PurchaseOrderService.js  # PO lifecycle
│   │   │   └── SupplierPerformanceService.js
│   │   │
│   │   ├── payment/
│   │   │   ├── PaymentGatewayService.js # Razorpay integration
│   │   │   ├── PaymentReconciliation.js # Settlement
│   │   │   └── TransactionLog.js        # Payment logging
│   │   │
│   │   ├── reporting/
│   │   │   ├── SalesReportService.js    # Sales reports
│   │   │   ├── FinancialReportService.js# P&L reports
│   │   │   ├── GSTReportService.js      # GST compliance
│   │   │   ├── DashboardService.js      # Real-time dashboard
│   │   │   └── ExportService.js         # CSV/PDF export
│   │   │
│   │   ├── audit/
│   │   │   ├── AuditService.js          # Audit logging
│   │   │   └── ComplianceService.js     # Compliance checks
│   │   │
│   │   ├── cache/
│   │   │   ├── CacheService.js          # Redis operations
│   │   │   └── CacheInvalidation.js     # Cache sync
│   │   │
│   │   └── email/
│   │       └── EmailService.js          # Email notifications
│   │
│   ├── controllers/
│   │   ├── auth/
│   │   │   ├── AuthController.js        # Login/logout endpoints
│   │   │   └── UserController.js        # User management
│   │   │
│   │   ├── pos/
│   │   │   ├── TransactionController.js
│   │   │   ├── ReceiptController.js
│   │   │   └── RefundController.js
│   │   │
│   │   ├── inventory/
│   │   │   ├── ProductController.js
│   │   │   ├── StockController.js
│   │   │   └── AlertController.js
│   │   │
│   │   ├── customer/
│   │   │   ├── CustomerController.js
│   │   │   └── LoyaltyController.js
│   │   │
│   │   ├── supplier/
│   │   │   ├── SupplierController.js
│   │   │   └── PurchaseOrderController.js
│   │   │
│   │   ├── reporting/
│   │   │   ├── ReportController.js
│   │   │   └── DashboardController.js
│   │   │
│   │   └── payment/
│   │       └── PaymentController.js
│   │
│   ├── models/
│   │   ├── User.js
│   │   ├── Product.js
│   │   ├── Inventory.js
│   │   ├── Transaction.js
│   │   ├── Customer.js
│   │   ├── Supplier.js
│   │   ├── PurchaseOrder.js
│   │   ├── Invoice.js
│   │   ├── AuditLog.js
│   │   ├── LoyaltyPoints.js
│   │   └── PaymentLog.js
│   │
│   ├── routes/
│   │   ├── index.js                   # Route aggregator
│   │   ├── auth.routes.js
│   │   ├── pos.routes.js
│   │   ├── inventory.routes.js
│   │   ├── customer.routes.js
│   │   ├── supplier.routes.js
│   │   ├── reporting.routes.js
│   │   └── payment.routes.js
│   │
│   ├── validators/
│   │   ├── authValidator.js           # Login request validation
│   │   ├── transactionValidator.js    # Transaction validation
│   │   ├── inventoryValidator.js      # Inventory operations
│   │   ├── customerValidator.js
│   │   ├── supplierValidator.js
│   │   └── gstValidator.js            # GST calculation rules
│   │
│   ├── utils/
│   │   ├── errorHandler.js            # Custom error classes
│   │   ├── responseFormatter.js       # API response template
│   │   ├── gstCalculator.js           # GST calculation logic
│   │   ├── barcodeGenerator.js        # Barcode creation
│   │   ├── pdfGenerator.js            # Receipt/Report PDF
│   │   ├── dateUtils.js
│   │   ├── encryptionUtils.js         # Password hashing
│   │   └── logger.js                  # Winston logger setup
│   │
│   ├── migrations/
│   │   ├── 001_create_users_table.js
│   │   ├── 002_create_products_table.js
│   │   ├── 003_create_inventory_table.js
│   │   ├── 004_create_transactions_table.js
│   │   ├── 005_create_audit_logs_table.js
│   │   └── ... (30+ migrations)
│   │
│   ├── seeders/
│   │   ├── seedUsers.js
│   │   ├── seedProducts.js
│   │   ├── seedCustomers.js
│   │   └── seedSuppliers.js
│   │
│   ├── constants/
│   │   ├── roles.js                  # RBAC roles and permissions
│   │   ├── gstRates.js               # GST tax slabs
│   │   ├── errorCodes.js             # Error code definitions
│   │   └── businessRules.js          # Business logic constants
│   │
│   └── app.js                        # Express app entry point
│
├── tests/
│   ├── unit/                        # Service unit tests
│   ├── integration/                 # API integration tests
│   └── fixtures/                    # Test data
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .env.example                     # Environment template
├── package.json
├── package-lock.json
├── server.js                        # Server entry point
└── README.md
```

### 3.2 Frontend Structure (React)

```
supermarket-billing-frontend/
├── public/
│   ├── index.html
│   ├── favicon.ico
│   └── manifest.json
│
├── src/
│   ├── components/
│   │   ├── auth/
│   │   │   ├── LoginForm.jsx
│   │   │   ├── LogoutButton.jsx
│   │   │   └── ProtectedRoute.jsx
│   │   │
│   │   ├── pos/
│   │   │   ├── POSTerminal.jsx          # Main POS interface
│   │   │   ├── BarcodeScanner.jsx
│   │   │   ├── CartItems.jsx
│   │   │   ├── BillSummary.jsx
│   │   │   ├── PaymentModal.jsx
│   │   │   ├── ReceiptPreview.jsx
│   │   │   └── TransactionHistory.jsx
│   │   │
│   │   ├── inventory/
│   │   │   ├── ProductCatalog.jsx
│   │   │   ├── StockForm.jsx
│   │   │   ├── StockAlerts.jsx
│   │   │   ├── ExpiryTracker.jsx
│   │   │   ├── StockTransfer.jsx
│   │   │   └── InventorySearch.jsx
│   │   │
│   │   ├── customers/
│   │   │   ├── CustomerList.jsx
│   │   │   ├── CustomerForm.jsx
│   │   │   ├── LoyaltyPoints.jsx
│   │   │   └── CustomerSearch.jsx
│   │   │
│   │   ├── suppliers/
│   │   │   ├── SupplierList.jsx
│   │   │   ├── PurchaseOrderForm.jsx
│   │   │   ├── SupplierPerformance.jsx
│   │   │   └── PaymentDue.jsx
│   │   │
│   │   ├── reporting/
│   │   │   ├── Dashboard.jsx             # Main dashboard
│   │   │   ├── SalesChart.jsx
│   │   │   ├── InventoryChart.jsx
│   │   │   ├── FinancialChart.jsx
│   │   │   ├── ReportBuilder.jsx
│   │   │   ├── ReportList.jsx
│   │   │   └── ExportOptions.jsx
│   │   │
│   │   ├── admin/
│   │   │   ├── UserManagement.jsx
│   │   │   ├── RolePermissions.jsx
│   │   │   ├── SystemSettings.jsx
│   │   │   └── AuditLogs.jsx
│   │   │
│   │   ├── common/
│   │   │   ├── Header.jsx
│   │   │   ├── Sidebar.jsx
│   │   │   ├── Modal.jsx
│   │   │   ├── Table.jsx
│   │   │   ├── Form.jsx
│   │   │   ├── LoadingSpinner.jsx
│   │   │   ├── ErrorBoundary.jsx
│   │   │   └── Notifications.jsx
│   │   │
│   │   └── layout/
│   │       └── MainLayout.jsx
│   │
│   ├── pages/
│   │   ├── LoginPage.jsx
│   │   ├── Dashboard.jsx
│   │   ├── POSPage.jsx
│   │   ├── InventoryPage.jsx
│   │   ├── CustomerPage.jsx
│   │   ├── SupplierPage.jsx
│   │   ├── ReportingPage.jsx
│   │   ├── AdminPage.jsx
│   │   └── NotFoundPage.jsx
│   │
│   ├── services/
│   │   ├── authService.js
│   │   ├── posService.js
│   │   ├── inventoryService.js
│   │   ├── customerService.js
│   │   ├── supplierService.js
│   │   ├── reportingService.js
│   │   ├── apiClient.js              # Axios configured instance
│   │   └── localStorage.js            # Persistent storage
│   │
│   ├── redux/
│   │   ├── store.js                  # Redux store
│   │   ├── slices/
│   │   │   ├── authSlice.js
│   │   │   ├── posSlice.js
│   │   │   ├── inventorySlice.js
│   │   │   ├── customerSlice.js
│   │   │   ├── uiSlice.js
│   │   │   └── notificationSlice.js
│   │   └── selectors/
│   │       ├── authSelector.js
│   │       └── uiSelector.js
│   │
│   ├── hooks/
│   │   ├── useAuth.js
│   │   ├── useFetch.js
│   │   ├── usePermission.js
│   │   ├── useNotification.js
│   │   └── useForm.js
│   │
│   ├── styles/
│   │   ├── globals.css
│   │   ├── variables.css
│   │   ├── responsive.css
│   │   └── themes/
│   │       ├── light.css
│   │       └── dark.css
│   │
│   ├── utils/
│   │   ├── formatters.js             # Date, number, currency
│   │   ├── validators.js             # Form validation
│   │   ├── gstCalculator.js          # GST calculation
│   │   ├── errorHandler.js
│   │   └── constants.js
│   │
│   ├── App.jsx                       # Root component
│   ├── index.js                      # Entry point
│   └── config.js                     # API endpoints, env
│
├── public/
│   └── index.html
│
├── package.json
├── .env.example
├── .eslintrc.json
├── .prettierrc
└── README.md
```

### 3.3 Database Structure

```
PostgreSQL Database: supermarket_billing

Tables:
├── Core
│   ├── users (id, username, password_hash, email, phone, role, branch_id, created_at, updated_at)
│   ├── roles (id, name, description, permissions_json)
│   └── permissions (id, resource, action)
│
├── Products & Inventory
│   ├── products (id, sku, name, category, price, hsn_code, gst_rate, barcode, unit, created_at)
│   ├── inventory (id, product_id, branch_id, quantity, reorder_level, max_level, last_updated)
│   ├── stock_movements (id, product_id, branch_id, quantity_change, type, reference_id, created_at)
│   ├── batch_tracking (id, product_id, batch_number, expiry_date, quantity, created_at)
│   └── product_pricing (id, product_id, branch_id, price, discount_pct, effective_from, effective_to)
│
├── Transactions & Billing
│   ├── transactions (id, transaction_no, branch_id, cashier_id, total_amount, tax_amount, gst_amount, payment_method, status, created_at)
│   ├── transaction_items (id, transaction_id, product_id, quantity, unit_price, line_total, tax_amount, created_at)
│   ├── invoices (id, invoice_no, transaction_id, gst_number, billing_address, total_before_tax, total_tax, total_after_tax, created_at)
│   ├── refunds (id, transaction_id, refund_amount, reason, approved_by, status, created_at)
│   └── payment_logs (id, transaction_id, payment_method, amount, gateway_ref, status, created_at)
│
├── Customers & Loyalty
│   ├── customers (id, phone, email, name, address, gstin, customer_type, created_at)
│   ├── loyalty_points (id, customer_id, transaction_id, points_earned, points_redeemed, balance, created_at)
│   └── customer_transactions (id, customer_id, transaction_id, created_at)
│
├── Suppliers & PO
│   ├── suppliers (id, name, contact_person, phone, email, address, gstin, payment_terms, created_at)
│   ├── supplier_pricing (id, supplier_id, product_id, cost_price, moq, lead_time_days, effective_from, effective_to)
│   ├── purchase_orders (id, po_number, supplier_id, total_amount, expected_delivery, status, created_at)
│   ├── po_items (id, po_id, product_id, quantity, unit_cost, total_cost, received_qty, created_at)
│   └── goods_receipt (id, po_id, received_qty, received_date, quality_check, created_at)
│
├── Reporting & Analytics
│   ├── sales_summary (id, branch_id, transaction_date, total_sales, total_tax, total_refunds, transaction_count)
│   ├── inventory_summary (id, branch_id, summary_date, total_value, stock_count, low_stock_items)
│   └── gst_summary (id, branch_id, period_month, period_year, taxable_amount, gst_amount, gst_rate)
│
├── Audit & Compliance
│   ├── audit_logs (id, user_id, action, resource, resource_id, old_value, new_value, ip_address, created_at)
│   ├── system_logs (id, level, message, stack_trace, created_at)
│   ├── failed_transactions (id, transaction_data, error_message, retry_count, status, created_at)
│   └── data_access_logs (id, user_id, data_accessed, created_at)
│
└── Configuration
    ├── branches (id, name, address, phone, manager_id, opening_time, closing_time, created_at)
    ├── system_config (key, value, description)
    └── gst_config (id, product_category, gst_rate, effective_from, effective_to)
```

---

## 4. API ARCHITECTURE

### 4.1 API Design Principles

- **RESTful Design:** Resource-based endpoints, HTTP methods follow conventions
- **Consistent Response Format:** Every endpoint returns standardized JSON
- **Error Handling:** Comprehensive error codes and messages
- **Versioning:** All endpoints prefixed with `/api/v1/`
- **Pagination:** Limit/offset for list endpoints
- **Filtering:** Query parameters for search/filter
- **Rate Limiting:** Token bucket algorithm (100 req/min per user)
- **Authentication:** JWT in Authorization header
- **Documentation:** OpenAPI/Swagger spec maintained

### 4.2 Authentication Flow

```
1. USER LOGIN
   ┌─────────────────────────────────────┐
   │ POST /api/v1/auth/login             │
   │ Body: {username, password}          │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ AuthService validates credentials   │
   │ - Check user exists                 │
   │ - Verify password hash              │
   │ - Check user is active              │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Generate JWT Token                  │
   │ - Payload: {userId, role, branch}   │
   │ - Expiry: 24 hours                  │
   │ - Secret: HMAC-SHA256               │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Cache token in Redis                │
   │ - Key: jwt:{user_id}                │
   │ - Value: token + metadata           │
   │ - TTL: 24 hours                     │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Return Response:                    │
   │ {                                   │
   │   token: "eyJhbGc...",              │
   │   user: {id, name, role},           │
   │   permissions: [...]                │
   │ }                                   │
   └─────────────────────────────────────┘

2. AUTHENTICATED REQUEST
   ┌─────────────────────────────────────┐
   │ GET /api/v1/inventory/products      │
   │ Header: Authorization: Bearer {jwt} │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Middleware: Verify JWT              │
   │ - Decode token                      │
   │ - Verify signature                  │
   │ - Check Redis blacklist             │
   │ - Check expiry                      │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Middleware: Check RBAC              │
   │ - Extract permissions from token    │
   │ - Match with endpoint requirements  │
   │ - Allow or deny access              │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Middleware: Audit Logging           │
   │ - Log request details               │
   │ - Store in audit_logs table         │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │ Controller Process Request          │
   │ - Call Service layer                │
   │ - Return Response                   │
   └─────────────────────────────────────┘
```

### 4.3 Core API Endpoints

#### Authentication Endpoints
```
POST   /api/v1/auth/login              # User login
POST   /api/v1/auth/logout             # User logout
POST   /api/v1/auth/refresh-token      # Refresh JWT
GET    /api/v1/auth/me                 # Get current user
POST   /api/v1/auth/change-password    # Change password
```

#### POS Endpoints
```
POST   /api/v1/pos/transactions              # Create transaction (checkout)
GET    /api/v1/pos/transactions/{id}         # Get transaction details
POST   /api/v1/pos/transactions/{id}/refund  # Refund transaction
POST   /api/v1/pos/transactions/{id}/receipt/reprint
GET    /api/v1/pos/terminal/status           # Terminal status
POST   /api/v1/pos/payment/process           # Process payment
```

#### Inventory Endpoints
```
GET    /api/v1/inventory/products            # List products (paginated)
GET    /api/v1/inventory/products/{sku}      # Get product details
POST   /api/v1/inventory/products            # Create product
PATCH  /api/v1/inventory/products/{sku}      # Update product
GET    /api/v1/inventory/stock?branch={id}   # Get stock levels
POST   /api/v1/inventory/adjust              # Adjust inventory
GET    /api/v1/inventory/alerts/low-stock    # Low stock alerts
GET    /api/v1/inventory/expiry/due-soon     # Expiry alerts
POST   /api/v1/inventory/transfer            # Inter-branch transfer
GET    /api/v1/inventory/movement/{id}       # Stock movement history
```

#### Customer Endpoints
```
GET    /api/v1/customers                     # List customers
POST   /api/v1/customers                     # Create customer
GET    /api/v1/customers/{id}                # Get customer details
PATCH  /api/v1/customers/{id}                # Update customer
GET    /api/v1/customers/{id}/loyalty-points # Get loyalty points
POST   /api/v1/customers/{id}/loyalty-redeem # Redeem points
GET    /api/v1/customers/{id}/transactions   # Customer transaction history
DELETE /api/v1/customers/{id}                # Delete customer (GDPR)
```

#### Supplier Endpoints
```
GET    /api/v1/suppliers                     # List suppliers
POST   /api/v1/suppliers                     # Create supplier
GET    /api/v1/suppliers/{id}                # Get supplier details
PATCH  /api/v1/suppliers/{id}                # Update supplier
GET    /api/v1/suppliers/{id}/performance    # Supplier performance
GET    /api/v1/suppliers/{id}/pricing        # Supplier pricing
POST   /api/v1/purchase-orders               # Create PO
GET    /api/v1/purchase-orders/{id}          # Get PO details
PATCH  /api/v1/purchase-orders/{id}/status   # Update PO status
POST   /api/v1/purchase-orders/{id}/receive  # Receive goods
```

#### Reporting Endpoints
```
GET    /api/v1/reports/dashboard             # Real-time dashboard
GET    /api/v1/reports/sales-daily?date=...  # Daily sales
GET    /api/v1/reports/sales-monthly?month=... # Monthly sales
GET    /api/v1/reports/pl-statement          # P&L report
GET    /api/v1/reports/gst-compliance        # GST report
GET    /api/v1/reports/inventory-summary     # Inventory status
GET    /api/v1/reports/cashier-performance   # Cashier report
POST   /api/v1/reports/export                # Export report (PDF/CSV)
POST   /api/v1/reports/schedule              # Schedule recurring report
```

#### Admin Endpoints
```
GET    /api/v1/admin/users                   # List users
POST   /api/v1/admin/users                   # Create user
PATCH  /api/v1/admin/users/{id}              # Update user
DELETE /api/v1/admin/users/{id}              # Delete user
GET    /api/v1/admin/roles                   # List roles
POST   /api/v1/admin/roles                   # Create role
PATCH  /api/v1/admin/roles/{id}              # Update role
GET    /api/v1/admin/audit-logs              # Audit logs
GET    /api/v1/admin/system-config           # System configuration
PATCH  /api/v1/admin/system-config           # Update configuration
```

### 4.4 Request/Response Format

**Standard Response Format:**
```json
{
  "status": "success",         // "success" or "error"
  "code": 200,                 // HTTP status code
  "message": "Operation successful",
  "data": {                    // Actual response data
    "id": 123,
    "name": "Product Name"
  },
  "meta": {
    "timestamp": "2026-06-15T10:30:00Z",
    "request_id": "uuid-1234-5678",
    "execution_time_ms": 145
  }
}
```

**Error Response Format:**
```json
{
  "status": "error",
  "code": 400,
  "message": "Invalid input",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format"
    }
  ],
  "meta": {
    "timestamp": "2026-06-15T10:30:00Z",
    "request_id": "uuid-1234-5678"
  }
}
```

### 4.5 Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| 1001 | 400 | Invalid request parameters |
| 1002 | 401 | Unauthorized - invalid token |
| 1003 | 403 | Forbidden - insufficient permissions |
| 1004 | 404 | Resource not found |
| 1005 | 409 | Resource already exists |
| 2001 | 400 | Invalid transaction data |
| 2002 | 422 | Inventory insufficient |
| 2003 | 422 | Payment gateway error |
| 3001 | 500 | Database error |
| 3002 | 503 | Service unavailable |

---

## 5. SECURITY ARCHITECTURE

### 5.1 Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION LAYER                      │
│                                                              │
│  Username/Password → Hash(bcrypt) → JWT Token               │
│  Token Payload: {userId, role, branch, permissions}         │
│  Signed with: HMAC-SHA256                                   │
│  Expires: 24 hours                                          │
│  Stored: Redis (for revocation/blacklist)                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────────┐
│                 AUTHORIZATION LAYER (RBAC)                   │
│                                                              │
│  Role: CASHIER                                              │
│  ├─ Transaction.Create ✅                                   │
│  ├─ Transaction.Read (Own) ✅                               │
│  ├─ Refund.Request ✅                                       │
│  ├─ Product.Read ✅                                         │
│  ├─ Report.Read ❌                                          │
│  └─ User.Manage ❌                                          │
│                                                              │
│  Role: STORE_MANAGER                                        │
│  ├─ Transaction.Create ✅                                   │
│  ├─ Transaction.Read (All Branch) ✅                        │
│  ├─ Refund.Approve ✅                                       │
│  ├─ Inventory.Read ✅                                       │
│  ├─ Report.Read ✅                                          │
│  ├─ Report.Export ✅                                        │
│  ├─ User.Manage (Branch) ✅                                 │
│  └─ System.Config ❌                                        │
│                                                              │
│  Role: ADMIN                                                │
│  └─ All permissions ✅                                      │
└─────────────────────────────────────────────────────────────┘
```

**RBAC Implementation:**
```javascript
// roles.js - Define roles and permissions
const roles = {
  ADMIN: {
    name: 'Administrator',
    permissions: ['*'] // wildcard for all
  },
  STORE_MANAGER: {
    name: 'Store Manager',
    permissions: [
      'transaction:read', 'transaction:create',
      'refund:read', 'refund:approve',
      'inventory:read', 'inventory:adjust',
      'customer:read', 'customer:create',
      'report:read', 'report:export'
    ]
  },
  CASHIER: {
    name: 'Cashier',
    permissions: [
      'transaction:read', 'transaction:create',
      'refund:request',
      'product:read'
    ]
  }
};

// Middleware enforcement
const checkPermission = (requiredPermission) => {
  return (req, res, next) => {
    const userRole = req.user.role;
    const hasPermission = roles[userRole].permissions.includes(requiredPermission) ||
                          roles[userRole].permissions.includes('*');
    
    if (!hasPermission) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};

// Route usage
router.post('/transactions', 
  authenticate, 
  checkPermission('transaction:create'),
  transactionController.create
);
```

### 5.2 Data Encryption

```
┌─────────────────────────────────────────┐
│        ENCRYPTION STRATEGY              │
│                                         │
│ Data in Transit (TLS 1.2+):             │
│ • All HTTP → HTTPS redirection          │
│ • Certificate pinning (mobile app)      │
│ • Perfect Forward Secrecy               │
│                                         │
│ Data at Rest:                           │
│ ┌──────────────────────────────────┐   │
│ │ Sensitive Fields (AES-256-GCM):  │   │
│ │ • passwords (bcrypt)             │   │
│ │ • payment_tokens (tokenized)     │   │
│ │ • customer.gstin                 │   │
│ │ • supplier.bank_account          │   │
│ └──────────────────────────────────┘   │
│                                         │
│ Database Level:                         │
│ • Column-level encryption for PII      │
│ • Key management: AWS KMS              │
│ • Key rotation: Quarterly              │
│                                         │
│ API Keys & Secrets:                    │
│ • Stored in environment (.env)         │
│ • Never in version control             │
│ • Rotated every 90 days                │
└─────────────────────────────────────────┘
```

### 5.3 API Security

```
┌─────────────────────────────────────────────────┐
│         API GATEWAY SECURITY                     │
│                                                  │
│ 1. Authentication Check                         │
│    ├─ JWT validation                            │
│    ├─ Token expiry check                        │
│    ├─ Blacklist check (Redis)                   │
│    └─ Signature verification                    │
│                                                  │
│ 2. Rate Limiting                                │
│    ├─ 100 requests/minute per user              │
│    ├─ 1000 requests/minute per IP               │
│    ├─ Token bucket algorithm                    │
│    └─ Redis-backed counters                     │
│                                                  │
│ 3. Input Validation                             │
│    ├─ Schema validation (Joi)                   │
│    ├─ Type checking                             │
│    ├─ Length limits                             │
│    ├─ Whitelist approach                        │
│    └─ SQL injection prevention                  │
│                                                  │
│ 4. CORS Policy                                  │
│    ├─ Allowed origins: frontend URL only        │
│    ├─ Allowed methods: GET, POST, PATCH, DELETE │
│    ├─ Allowed headers: Authorization, Content   │
│    └─ No credentials in preflight               │
│                                                  │
│ 5. CSRF Protection                              │
│    ├─ SameSite=Strict cookie flag               │
│    └─ Token validation on state-changing ops   │
└─────────────────────────────────────────────────┘
```

### 5.4 Audit Logging & Compliance

```
┌──────────────────────────────────────────────┐
│      AUDIT TRAIL ARCHITECTURE                 │
│                                               │
│ Every Action Logged:                          │
│ ┌────────────────────────────────────────┐   │
│ │ User ID      │ john.doe (cashier001)  │   │
│ │ Action       │ transaction.create     │   │
│ │ Resource     │ Transaction#5678       │   │
│ │ Timestamp    │ 2026-06-15T10:30:00Z   │   │
│ │ IP Address   │ 192.168.1.100          │   │
│ │ Change       │ amount: 0 → 5500.00    │   │
│ │ Status       │ SUCCESS                │   │
│ │ Session ID   │ uuid-12345             │   │
│ └────────────────────────────────────────┘   │
│                                               │
│ Storage Strategy:                             │
│ ├─ PostgreSQL (audit_logs table)             │
│ ├─ Immutable: No delete/update allowed       │
│ ├─ Hash verification: Detect tampering       │
│ ├─ Retention: 10 years (GST compliance)      │
│ └─ Backup: Daily + Archive                   │
│                                               │
│ Compliance Checks:                            │
│ ├─ GST data access audit                      │
│ ├─ User permission changes tracked            │
│ ├─ Failed login attempts flagged              │
│ ├─ Data export logged                         │
│ └─ Sensitive operation approvals              │
└──────────────────────────────────────────────┘
```

### 5.5 Sensitive Data Protection

```
PII (Personally Identifiable Information) Handling:

Customer Phone/Email:
├─ Encrypted at rest (AES-256)
├─ Not logged in audit trails (except hashed for tracking)
├─ GDPR: Deleted after 2 years of inactivity
└─ Export: Requires manager approval

Payment Information:
├─ Never stored locally (Razorpay tokenization)
├─ Token stored with TTL
├─ PCI-DSS compliance enforced
└─ All payment logs encrypted

Employee/Supplier Bank Accounts:
├─ Encrypted at rest
├─ Limited access (Finance only)
├─ Audit logged
└─ TLS in transit

Deletion Workflow (GDPR):
1. Soft delete (flag deleted_at)
2. 30-day grace period (recovery possible)
3. Hard delete of PII data
4. Archive transaction records (7-year retention)
5. Log deletion action
```

### 5.6 Secrets Management

```
Environment Variables (.env not in git):

CRITICAL SECRETS:
├─ DB_PASSWORD
├─ JWT_SECRET
├─ RAZORPAY_KEY
├─ RAZORPAY_SECRET
├─ REDIS_PASSWORD
├─ AWS_SECRET_KEY
├─ ENCRYPTION_KEY
└─ PAYMENT_WEBHOOK_SECRET

Management Strategy:
├─ Development: .env.local (local machine only)
├─ Staging: AWS Secrets Manager
├─ Production: AWS Secrets Manager + Key rotation
└─ Rotation: Every 90 days

Access Control:
├─ Only CI/CD pipeline accesses secrets
├─ No hardcoding in code
├─ Logs never print secrets
├─ Developers use local .env.example (with dummy values)
└─ GitHub Actions secrets for CI/CD
```

---

## 6. DEPLOYMENT ARCHITECTURE

### 6.1 Development Workflow

```
Developer Machine
├─ Git clone repository
├─ Run docker-compose up (PostgreSQL + Redis)
├─ npm install (backend + frontend)
├─ Create .env.local with dummy values
├─ npm run dev (runs both frontend & backend)
└─ Access http://localhost:3000

Git Workflow:
├─ Feature branch: feature/pos-module
├─ Commit to feature
├─ Create Pull Request
├─ Code review required
├─ Automated tests run (GitHub Actions)
├─ Merge to main branch
└─ CI/CD pipeline triggers → Staging → Production
```

### 6.2 Deployment Stages

```
┌─────────────────────────────────────────────────────────────┐
│            DEPLOYMENT PIPELINE (GitOps)                      │
│                                                              │
│  Stage 1: Development                                       │
│  ├─ Local machine with docker-compose                       │
│  └─ Database: PostgreSQL 14 (local container)               │
│                                                              │
│  Stage 2: Staging (AWS)                                     │
│  ├─ EC2 t3.medium (2vCPU, 4GB RAM)                          │
│  ├─ RDS PostgreSQL (db.t3.small, 20GB)                      │
│  ├─ ElastiCache Redis (cache.t3.micro)                      │
│  ├─ ALB (Application Load Balancer)                         │
│  ├─ Domain: staging.billing.company.com                    │
│  └─ SSL/TLS: AWS Certificate Manager                        │
│                                                              │
│  Stage 3: Production (AWS Multi-Region)                     │
│  ├─ Primary Region (us-east-1)                              │
│  │  ├─ EC2 t3.large x2 (Auto Scaling Group)                 │
│  │  ├─ RDS PostgreSQL (db.t3.medium, 100GB)                 │
│  │  │  ├─ Multi-AZ (Automatic Failover)                     │
│  │  │  └─ Read Replica (us-west-2)                          │
│  │  ├─ ElastiCache Redis (cache.t3.small)                   │
│  │  ├─ ALB + Auto Scaling                                   │
│  │  ├─ CloudFront CDN                                       │
│  │  ├─ WAF (Web Application Firewall)                       │
│  │  ├─ CloudWatch Monitoring                                │
│  │  └─ RTO: 4 hours, RPO: 15 minutes                        │
│  │                                                           │
│  └─ DR Region (us-west-2) - Standby                         │
│     └─ Read-only replicas (cross-region)                    │
│     └─ Can be promoted to primary if needed                 │
│                                                              │
│  Deployment Frequency: 2-3 times per week                   │
│  Rollback Strategy: Blue-green deployment                   │
│  Monitoring: CloudWatch + Custom dashboards                │
│  Alerting: SNS + PagerDuty                                  │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Containerization (Docker)

**Backend Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package.json package-lock.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY src ./src

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node healthcheck.js

# Start application
CMD ["node", "server.js"]
```

**docker-compose.yml (Development):**
```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: billing_user
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: supermarket_billing
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U billing_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Backend API
  backend:
    build: ./backend
    environment:
      NODE_ENV: development
      DB_HOST: postgres
      DB_PORT: 5432
      REDIS_HOST: redis
      JWT_SECRET: dev_secret
    ports:
      - "5000:5000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend/src:/app/src

  # Frontend
  frontend:
    build: ./frontend
    environment:
      REACT_APP_API_URL: http://localhost:5000/api
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres_data:
```

### 6.4 Infrastructure as Code (Terraform)

```hcl
# main.tf - AWS Infrastructure

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# RDS PostgreSQL
resource "aws_db_instance" "main" {
  allocated_storage    = 100
  storage_type         = "gp2"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = "db.t3.medium"
  identifier           = "billing-db-prod"
  username             = var.db_username
  password             = random_password.db.result
  multi_az             = true
  publicly_accessible  = false
  backup_retention_period = 30
  
  tags = {
    Name = "billing-database"
  }
}

# ElastiCache Redis
resource "aws_elasticache_cluster" "main" {
  cluster_id           = "billing-redis"
  engine               = "redis"
  node_type            = "cache.t3.small"
  num_cache_nodes      = 2
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  port                 = 6379
  
  tags = {
    Name = "billing-cache"
  }
}

# EC2 Auto Scaling Group
resource "aws_launch_template" "backend" {
  name_prefix = "billing-backend-"
  image_id    = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.large"
  
  user_data = base64encode(file("${path.module}/scripts/user_data.sh"))
}

resource "aws_autoscaling_group" "backend" {
  name                = "billing-asg"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = 2
  max_size            = 5
  desired_capacity    = 3
  launch_template {
    id      = aws_launch_template.backend.id
    version = "$Latest"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "billing-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids
}

resource "aws_lb_target_group" "backend" {
  name     = "billing-tg"
  port     = 5000
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
}
```

### 6.5 CI/CD Pipeline (GitHub Actions)

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
      redis:
        image: redis:7
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies (Backend)
        run: cd backend && npm ci
      
      - name: Run linter
        run: cd backend && npm run lint
      
      - name: Run unit tests
        run: cd backend && npm test
      
      - name: Run integration tests
        run: cd backend && npm run test:integration
      
      - name: Install dependencies (Frontend)
        run: cd frontend && npm ci
      
      - name: Build frontend
        run: cd frontend && npm run build
      
      - name: Run E2E tests
        run: cd frontend && npm run test:e2e

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker images
        run: |
          docker build -t billing-backend:${{ github.sha }} ./backend
          docker build -t billing-frontend:${{ github.sha }} ./frontend
      
      - name: Push to ECR
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_KEY }}
        run: |
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker push ${{ secrets.ECR_REGISTRY }}/billing-backend:${{ github.sha }}
          docker push ${{ secrets.ECR_REGISTRY }}/billing-frontend:${{ github.sha }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to Staging
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_KEY }}
        run: |
          aws ecs update-service \
            --cluster billing-staging \
            --service billing-service \
            --force-new-deployment
      
      - name: Run smoke tests
        run: |
          ./scripts/smoke-tests.sh staging.billing.company.com

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
      - name: Approve Deployment
        run: echo "Manual approval required"
      
      - name: Blue-Green Deployment
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_KEY }}
        run: |
          ./scripts/blue-green-deploy.sh
      
      - name: Run smoke tests
        run: |
          ./scripts/smoke-tests.sh billing.company.com
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ Production deployment successful",
              "blocks": [{"type": "section", "text": {"type": "mrkdwn", "text": "Production deployment for commit ${{ github.sha }} completed successfully"}}]
            }
```

### 6.6 Monitoring & Observability

```
┌─────────────────────────────────────────────────┐
│        MONITORING STACK                         │
│                                                 │
│  Metrics Collection:                            │
│  ├─ Application Metrics (Prometheus)            │
│  │  ├─ Response time (ms)                       │
│  │  ├─ Requests per second                      │
│  │  ├─ Error rate                               │
│  │  ├─ Database query performance               │
│  │  ├─ Cache hit ratio                          │
│  │  └─ Active users                             │
│  │                                              │
│  ├─ Infrastructure Metrics (CloudWatch)         │
│  │  ├─ CPU utilization                          │
│  │  ├─ Memory usage                             │
│  │  ├─ Disk I/O                                 │
│  │  ├─ Network throughput                       │
│  │  └─ RDS connections                          │
│  │                                              │
│  └─ Custom Metrics                              │
│     ├─ Transactions per minute                  │
│     ├─ Checkout success rate                    │
│     ├─ Inventory accuracy                       │
│     └─ GST calculation accuracy                 │
│                                                 │
│  Alerting Rules:                                │
│  ├─ Error rate > 1% → Alert                     │
│  ├─ Response time > 1s (P95) → Alert            │
│  ├─ Database connection pool > 80% → Alert      │
│  ├─ Disk usage > 80% → Alert                    │
│  └─ System uptime < 99.5% → Escalate            │
│                                                 │
│  Dashboards:                                    │
│  ├─ Real-time Operations Dashboard              │
│  ├─ Business KPI Dashboard                      │
│  ├─ Infrastructure Health Dashboard             │
│  └─ Security & Compliance Dashboard             │
│                                                 │
│  Logging:                                       │
│  ├─ Application Logs (Winston)                  │
│  ├─ Access Logs (Nginx)                         │
│  ├─ Database Logs (Slow query log)              │
│  ├─ Audit Logs (PostgreSQL)                     │
│  └─ Centralized Log Analysis (ELK/CloudWatch)   │
└─────────────────────────────────────────────────┘
```

### 6.7 Disaster Recovery

```
┌──────────────────────────────────────────────┐
│   BACKUP & RECOVERY STRATEGY                 │
│                                              │
│  Database Backups:                           │
│  ├─ Automated daily snapshots                │
│  ├─ Weekly full backups (encrypted)          │
│  ├─ Cross-region replication (daily)         │
│  ├─ Point-in-time recovery (7 days)          │
│  ├─ Retention: 30 days online, 3 years cold  │
│  └─ RTO: 1 hour, RPO: 15 minutes             │
│                                              │
│  Application Recovery:                       │
│  ├─ Multi-AZ deployment                      │
│  ├─ Auto Scaling Group (min 2)               │
│  ├─ ELB automatic health checks              │
│  └─ RTO: 5 minutes (automatic failover)      │
│                                              │
│  Disaster Recovery Plan:                     │
│  ├─ Annual DR drill                          │
│  ├─ Documented runbooks                      │
│  ├─ Failover procedures tested               │
│  ├─ Recovery time tracked                    │
│  └─ Post-incident review                     │
└──────────────────────────────────────────────┘
```

---

## DEPLOYMENT CHECKLIST

### Pre-Deployment
- [ ] All tests passing (unit + integration + E2E)
- [ ] Code review approved
- [ ] Security audit completed
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] SSL certificates valid
- [ ] Monitoring dashboards ready
- [ ] Incident response plan reviewed

### Production Deployment
- [ ] Blue-green deployment setup
- [ ] Smoke tests defined and ready
- [ ] Rollback plan documented
- [ ] Team on-call and available
- [ ] Monitoring alerts active
- [ ] Backup verified
- [ ] DR region health checked

### Post-Deployment
- [ ] Smoke tests executed successfully
- [ ] Monitoring metrics normal
- [ ] Error rates acceptable
- [ ] Performance metrics meet SLA
- [ ] User feedback positive
- [ ] Incident log updated
- [ ] Deployment documented

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-15 | Principal Architect | Initial MVP Architecture Design |

**Classification:** Internal - Confidential