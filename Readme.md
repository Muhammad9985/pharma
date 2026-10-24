# 🏥 MediFlow Pro & MedSupply PK — Master Architectural Blueprint & Implementation Plan
### Complete Pharmacy, Medical Store, Multi-Variant Inventory, POS, Double-Entry Accounting & Hospital LP Supply Management Desktop Suite

---

## Executive Summary & Vision

This blueprint outlines the full architectural specification and execution roadmap for **MediFlow Pro**, a production-grade, offline-first Flutter Windows desktop software. The application is uniquely architected to handle the dual business model prevalent in the Pakistani pharmaceutical market:
1. **High-Velocity Retail Pharmacy & POS:** Over-the-counter and prescription sales, fractional packaging breakdown (Carton → Box → Strip → Tablet), FEFO batch control, barcode scanning, and instant cash closing.
2. **Institutional Hospital LP (Local Purchase) Supply:** Multi-department hospital tender/LP lifecycle management, direct procurement against LP, batch reservations, multi-page delivery challans, joint inspection reports, and institutional credit invoicing strictly matching real-world Sindh/Punjab public healthcare templates (e.g., SMBB Institute of Trauma, Karachi).

---

## 1. High-Level Architecture & System Topology

```mermaid
graph TB
    subgraph ClientLayer["🖥️ Presentation Layer (Flutter Desktop - Win32/x64)"]
        UI_Nav["Sidebar & Global Navigation Hub"]
        UI_POS["⚡ High-Speed Keyboard POS"]
        UI_LP["🏥 Hospital LP Tender & Supply Hub"]
        UI_Stock["📦 Variant & Batch Stock Manager"]
        UI_Acct["💰 Accounting & Ledger Central"]
        UI_Reports["📊 Analytics & Document Printing (PDF/Excel)"]
    end

    subgraph StateLayer["⚡ Application Logic & State Layer (Bloc / Riverpod)"]
        BL_Auth["Single-Login & Multi-Business Controller"]
        BL_Packaging["Unit Conversion & Fraction Engine"]
        BL_FEFO["FEFO Batch & Expiry Optimizer"]
        BL_LP["Hospital Order State Machine"]
        BL_Finance["Double-Entry Ledger & Balance Calculator"]
    end

    subgraph DomainLayer["⚙️ Domain Services & Business Rules"]
        SRV_Inventory["Stock Movement & Valuation Service (FIFO/Actual)"]
        SRV_Procure["Purchase & Goods Receipt Note (GRN) Service"]
        SRV_LP["Challan, Joint Inspection & Invoicing Pipeline"]
        SRV_Doc["Template-Accurate PDF & Thermal Print Renderer"]
        SRV_Backup["Automated Snapshot & Disaster Recovery Engine"]
    end

    subgraph PersistenceLayer["💾 Persistence Layer (Local SQLite Engine)"]
        DB_Driver["sqflite_common_ffi (Encrypted / WAL Mode)"]
        DB_Schema["Strict Relational Schema (32+ Normalized Tables)"]
        DB_Trx["Atomic Transactions & Audit Log Journals"]
    end

    ClientLayer --> StateLayer
    StateLayer --> DomainLayer
    DomainLayer --> PersistenceLayer
```

---

## 2. Multi-Business Profile & Single-Login Architecture

```mermaid
flowchart TD
    Start([Launch MediFlow Application]) --> Login[Single Secure Login Interface]
    Login --> Verify{Credentials Valid?}
    Verify -- No --> Login
    Verify -- Yes --> BizPicker{Active Business Selected?}
    
    BizPicker -- First Run / None --> BizSetup[Create Business Profile / Enter NTN / Upload Logo]
    BizSetup --> SetContext[Mount Business Workspace]
    
    BizPicker -- Existing --> SelectBiz[Choose Active Business Profile]
    SelectBiz --> SetContext
    
    subgraph DataIsolation["🔒 Strict Multi-Tenant Data Isolation (Local Offline DB)"]
        SetContext --> ScopedQuery["Inject active_business_id into Repository Context"]
        ScopedQuery --> TableProducts["tbl_products (business_id)"]
        ScopedQuery --> TableStock["tbl_batches & tbl_stock (business_id)"]
        ScopedQuery --> TableLP["tbl_hospital_orders (business_id)"]
        ScopedQuery --> TableAcct["tbl_journal_entries (business_id)"]
    end
    
    SetContext --> Dashboard[Open Primary Dashboard]
```

### Key Data Isolation Rules
1. **Zero Data Bleed:** Every primary query contains `WHERE business_id = ?`.
2. **Document Header Immutability:** Historical invoices and challans record the snapshot of business legal name, NTN, STRN, and address at time of issue.
3. **No Hardcoding:** Business details, hospitals, suppliers, manufacturers, and formulas are 100% database driven.

---

## 3. Product Variant & Multi-Level Packaging Mathematical Engine

Pharmaceutical products in Pakistan have distinct strengths, dosage forms, and varying unit conversions. For example:
- **Panadol:** 500mg Tablet vs. Panadol CF vs. Panadol Elixir 120ml.
- **Conversion Example:**
  - 1 Carton = 24 Boxes
  - 1 Box = 10 Strips
  - 1 Strip = 10 Tablets
  - $\rightarrow$ Base Unit = 1 Tablet (Total Base Qty in Carton = $24 \times 10 \times 10 = 2,400$ Tablets).

```mermaid
graph LR
    subgraph PackagingHierarchy["Hierarchical Packaging Tree"]
        C[Carton<br>Factor: 2400] -->|x24| B[Box<br>Factor: 100]
        B -->|x10| S[Strip<br>Factor: 10]
        S -->|x10| T[Tablet / Capsule<br>Factor: 1 (Base Unit)]
    end

    subgraph LiquidHierarchy["Simple / Liquid Hierarchy"]
        Bottle[Bottle / Pack<br>Factor: 120] -->|x120| ML[Millilitre (ml)<br>Factor: 1 (Base Unit)]
    end
```

### Mathematical Conversion Equations
Let $U_1, U_2, \dots, U_k$ be packaging levels from largest to smallest ($U_k$ is base unit with factor $f_k = 1$).
- **Conversion Factor:**
  $$f_i = \prod_{j=i}^{k-1} \text{multiplier}_j$$
- **Deduction upon selling quantity $Q$ of level $U_i$:**
  $$\Delta \text{BaseStock} = Q \times f_i$$
- **Pretty-Print Decomposition (e.g. 5,345 tablets):**
  $$\text{Cartons} = \lfloor 5345 / 2400 \rfloor = 2 \quad (\text{rem } 545)$$
  $$\text{Boxes} = \lfloor 545 / 100 \rfloor = 5 \quad (\text{rem } 45)$$
  $$\text{Strips} = \lfloor 45 / 10 \rfloor = 4 \quad (\text{rem } 5)$$
  $$\text{Tablets} = 5$$
  $$\mathbf{\text{Display: } \text{"2 Cartons, 5 Boxes, 4 Strips, 5 Tablets"}}$$

---

## 4. Complete Hospital LP (Local Purchase) Supply Chain Workflow

The application precisely implements the operational flow seen in Pakistani tertiary care hospitals (e.g., SMBB Trauma Centre, Civil Hospital, JPMC):

```mermaid
sequenceDiagram
    autonumber
    actor Hosp as Hospital / Ward
    participant App as MediFlow LP Module
    participant Stock as Batch Inventory
    participant Supp as Pharma Supplier
    actor Insp as Joint Inspection Board
    participant Acct as Accounting Ledger

    Hosp->>App: Submits LP Slip / Quotation Request (Slip #, Dept, Generic, Qty)
    App->>Stock: Check Available FEFO Stock for Medicine Variants
    alt Sufficient Stock Available
        App->>Stock: Reserve Batch Quantities (Prevent POS from Selling)
    else Stock Deficit
        App->>Supp: Generate Purchase Order Linked to LP Order
        Supp-->>App: Goods Delivery & Supplier Bill
        App->>Stock: GRN Posted -> Batches Allocated to LP
    end
    App->>App: Generate Multi-Page Delivery Challan (PDF)
    App->>Hosp: Dispatch Goods with Delivery Challan
    Hosp->>Insp: Physical Verification at Receiving Counter
    Insp->>App: Joint Inspection Report (Accepted vs. Rejected Qty Recorded)
    alt Rejected Items Present
        App->>Stock: Quarantine Stock Return / Supplier Debit Note
    end
    App->>App: Generate Final Sales Invoice (Calculates MRP, Disc %, Net Rate & Amount)
    App->>Acct: Post Debit to Hospital Receivable, Credit to Sales & COGS
    Hosp-->>Acct: Issue Payment / Cross Cheque against File #
    Acct->>Acct: Reconcile Outstanding LP Balance
```

---

## 5. Document Reproduction Matching Provided PDF Templates

The three provided PDF documents in the project folder have been reverse-engineered down to typography, header layouts, borders, and column data structures:

### Comparative Template Specification Matrix

| Feature / Field | Delivery Challan (`Delevry challan.pdf`) | Joint Inspection Report (`Joint_inspection_report.pdf`) | Sales Invoice (`Sales invoice.pdf`) |
| :--- | :--- | :--- | :--- |
| **Top Title** | `MEDICAL CORPORATION` (Configurable) | `MEDICAL CORPORATION` (Configurable) | `MEDICAL CORPORATION` (Configurable) |
| **Document Label** | `DELIVERY CHALAN` (Navy bordered box) | `JOINT INSPECTION REPORT` (Navy bordered box) | `INVOICE` (Navy bordered box) |
| **Top Meta** | M/S Hospital Name, Address, NTN | M/S Hospital Name, Address, NTN | M/S Hospital Name, Address, NTN |
| **Right Meta** | `FILE #: 3.002.00`, `DATE: 02-Mar-2026` | `FILE #: 3.002.00`, `DATE: 02-Mar-2026` | `FILE #: 3.002.00`, `DATE: 02-Mar-2026` |
| **Column 1** | `Sr#` | `Sr#` | `Sr#` |
| **Column 2** | `DATE / SLIP #` (e.g. `02-Mar-2026 / 002544`) | `DATE / SLIP #` (e.g. `02-Mar-2026 / 002544`) | `DATE / SLIP #` (e.g. `02-Mar-2026 / 002544`) |
| **Column 3** | `REQUESTED DEPARTMENT` | `REQUESTED DEPARTMENT` | `REQUESTED DEPARTMENT` |
| **Column 4** | `ITEM GENERIC` | `ITEM GENERIC` | `ITEM GENERIC` |
| **Column 5** | `REQ. QTY.` | `REQ. QTY.` | `REQ. QTY.` |
| **Column 6** | `SUPPLIED ITEM NAME` (Brand/Strength) | `SUPPLIED ITEM NAME` (Brand/Strength) | `SUPPLIED ITEM NAME` (Brand/Strength) |
| **Column 7** | `SUP. QTY.` | `SUP. QTY.` | `SUP. QTY.` |
| **Pricing Columns** | *None (Blank verification columns)* | *Inspection checkboxes & Remarks* | `M.R.P. /PCS` \| `DISC. %` \| `NET RATE` \| `NET AMOUNT` |
| **Bottom Totals** | Dispatched Summary & Signature lines | Accepted/Rejected summary & 3 Signatures | **Grand Totals:** Total MRP & Total Net Amount (e.g. `141,742.70 PKR`) |

---

## 6. Entity Relationship Diagram (Database Schema)

```mermaid
erDiagram
    BUSINESS_PROFILE ||--o{ USER_ACCOUNT : has
    BUSINESS_PROFILE ||--o{ COMPANY : manages
    BUSINESS_PROFILE ||--o{ SUPPLIER : deals_with
    BUSINESS_PROFILE ||--o{ HOSPITAL : supplies
    BUSINESS_PROFILE ||--o{ PRODUCT : owns

    HOSPITAL ||--o{ HOSPITAL_DEPARTMENT : contains
    PRODUCT ||--o{ PRODUCT_VARIANT : has_variants
    PRODUCT_VARIANT ||--o{ PACKAGING_CONVERSION : defines_units
    PRODUCT_VARIANT ||--o{ BATCH : stocked_in

    SUPPLIER ||--o{ PURCHASE_ORDER : receives
    PURCHASE_ORDER ||--o{ PURCHASE_ITEM : contains
    PURCHASE_ITEM ||--o{ GOODS_RECEIPT_NOTE : fulfilled_by
    GOODS_RECEIPT_NOTE ||--o{ BATCH : generates

    BATCH ||--o{ STOCK_MOVEMENT : tracks
    BATCH ||--o{ POS_SALE_ITEM : sold_in
    BATCH ||--o{ DELIVERY_CHALLAN_ITEM : allocated_to

    HOSPITAL ||--o{ LP_ORDER : places
    LP_ORDER ||--o{ LP_ORDER_ITEM : includes
    LP_ORDER ||--o{ DELIVERY_CHALLAN : dispatched_via
    DELIVERY_CHALLAN ||--o{ DELIVERY_CHALLAN_ITEM : contains
    DELIVERY_CHALLAN ||--o{ JOINT_INSPECTION : inspected_by
    JOINT_INSPECTION ||--o{ LP_SALES_INVOICE : bills_to

    LP_SALES_INVOICE ||--o{ FINANCIAL_TRANSACTION : creates_receivable
    POS_SALE ||--o{ FINANCIAL_TRANSACTION : creates_cash_bank
```

---

## 7. Fast Keyboard-Driven Retail POS Workflow

```mermaid
stateDiagram-v2
    [*] --> Idle: Cashier Screen Ready
    Idle --> Scanning: Scan Barcode / F1 Search Generic or Brand
    Scanning --> VariantPicker: Multiple Strengths Detected (e.g., 500mg, Extra, Susp)
    VariantPicker --> UnitSelection: Select Variant & Sale Unit (Tab/Arrow keys)
    UnitSelection --> FEFO_Suggestion: Auto-select earliest expiring batch
    FEFO_Suggestion --> CartAdded: Enter Qty & Press Enter
    CartAdded --> Payment: F12 Checkout Shortcut
    Payment --> SplitMethod: Cash / Credit Card / Customer Account
    SplitMethod --> AtomicCommit: Post Transaction
    state AtomicCommit {
        DeductBatchStock
        RecordStockMovement
        DebitCash_CreditSales
        CalculateCOGS_GrossProfit
    }
    AtomicCommit --> PrintReceipt: Auto-trigger 80mm / A4 Printer
    PrintReceipt --> Idle
```

---

## 8. Double-Entry Accounting & Financial Ledger Engine

To provide reliable financial tracking, MediFlow uses a clean, standardized chart of accounts:

```mermaid
graph TD
    subgraph ChartOfAccounts["Core Chart of Accounts"]
        Assets["1000 - Assets"]
        Liabilities["2000 - Liabilities"]
        Equity["3000 - Equity"]
        Revenue["4000 - Revenue"]
        Expenses["5000 - Cost & Expenses"]
        
        Assets --> A1["1010 Cash on Hand"]
        Assets --> A2["1020 Bank Accounts"]
        Assets --> A3["1030 Retail Customer Receivables"]
        Assets --> A4["1040 Hospital LP Receivables"]
        Assets --> A5["1050 Medicine Stock Asset (Inventory Valuation)"]
        
        Liabilities --> L1["2010 Supplier Accounts Payable"]
        Liabilities --> L2["2020 Taxes Payable"]
        
        Revenue --> R1["4010 Retail POS Sales"]
        Revenue --> R2["4020 Hospital LP Sales"]
        
        Expenses --> E1["5010 Cost of Goods Sold (COGS)"]
        Expenses --> E2["5020 Shop Operating Expenses"]
        Expenses --> E3["5030 Expired & Damaged Stock Write-off"]
    end
```

### Standard Transaction Journal Rules:
1. **Purchase Goods Posted:**
   - Debit: `1050 Medicine Stock Asset`
   - Credit: `2010 Supplier Accounts Payable`
2. **Retail Sale Completed:**
   - Debit: `1010 Cash on Hand` (or `1020 Bank`)
   - Credit: `4010 Retail POS Sales`
   - *Simultaneous COGS Entry:*
     - Debit: `5010 Cost of Goods Sold` (based on batch actual purchase cost)
     - Credit: `1050 Medicine Stock Asset`
3. **Hospital LP Invoice Billed:**
   - Debit: `1040 Hospital LP Receivables (Specific Hospital & File #)`
   - Credit: `4020 Hospital LP Sales`
4. **Hospital Payment Received:**
   - Debit: `1020 Bank Account`
   - Credit: `1040 Hospital LP Receivables`

---

## 9. Comprehensive Reports & Analytics Suite

```mermaid
mindmap
  root((📊 MediFlow<br>Reports Central))
    Inventory Reports
      Current Stock & Valuation
      Batch-wise Expiry & Near Expiry Alert
      FEFO Movement & Item Ledger
      Quarantine & Wastage Report
      Low Stock & Minimum Reorder Alert
    Hospital LP Reports
      Hospital-wise Tender & LP Register
      Delivery Challan & Dispatch Log
      Joint Inspection Acceptance Report
      Hospital Aging & Pending Receivables
      Profit Margin per LP Order
    Purchase & Supplier Reports
      Supplier Purchase Register & GRN
      Item-wise Purchase Price History
      Supplier Outstanding & Ledger Statements
      Purchase Returns & Debit Notes
    Sales & POS Reports
      Daily Sales Summary & Cash Closing
      Category & Generic Performance
      Profit & Loss per Prescription / Item
      Cashier Reconciliation
    Financial Statements
      Trial Balance (Debit = Credit)
      Profit & Loss Statement (Income Statement)
      Balance Sheet
      Cash & Bank Flow Statements
```

---

## 10. Phased Step-by-Step Implementation Roadmap

```mermaid
gantt
    title MediFlow Pro - Implementation Timeline
    dateFormat  YYYY-MM-DD
    section Phase 1: Setup & Architecture
    Workspace & Dependencies Config      :p1_1, 2026-10-25, 2d
    SQLite Schema & Migration Layer      :p1_2, after p1_1, 3d
    Multi-Business & Login Core         :p1_3, after p1_2, 2d
    section Phase 2: Masters & UOM Engine
    Settings & Business Profile UI      :p2_1, after p1_3, 3d
    Companies, Suppliers & Hospitals    :p2_2, after p2_1, 3d
    Product Variants & Packaging Engine  :p2_3, after p2_2, 4d
    section Phase 3: Inventory & POS
    Purchasing & GRN Batch Storage      :p3_1, after p2_3, 4d
    Batch Inventory & FEFO Engine       :p3_2, after p3_1, 3d
    Keyboard POS & Instant Billing       :p3_3, after p3_2, 4d
    section Phase 4: Hospital LP Module
    LP Orders & Procurement Linkage     :p4_1, after p3_3, 4d
    Delivery Challan PDF Generator      :p4_2, after p4_1, 3d
    Joint Inspection & LP Invoicing     :p4_3, after p4_2, 4d
    section Phase 5: Accounting & Reports
    Double-Entry Ledger & Cash/Bank     :p5_1, after p4_3, 4d
    Comprehensive Reports & Dashboard   :p5_2, after p5_1, 4d
    section Phase 6: QA, Build & Polish
    End-to-End Testing & Packaging Test :p6_1, after p5_2, 3d
    Windows x64 Native Desktop Release  :p6_2, after p6_1, 2d
```

### Milestone Deliverables Summary:
- **Phase 1:** Core database with 32+ normalized tables, foreign key constraints, migration engine, and session management.
- **Phase 2:** Product variant generator, brand association, and multi-tier recursive packaging conversion engine with validation.
- **Phase 3:** Full batch lifecycle with actual unit cost valuation, barcode detection, and high-speed POS.
- **Phase 4:** Exact replica of the 3 hospital document templates (`Delevry challan.pdf`, `Joint_inspection_report.pdf`, `Sales invoice.pdf`) generating pixel-perfect multi-page printable PDFs.
- **Phase 5:** Automated double-entry financial posting and real-time operational dashboard with interactive charts.
- **Phase 6:** Offline backup/restore engine, release build verification on Windows 10/11 desktop.

---

## 11. Technology Stack & Package Compatibility

| Component | Selected Technology | Purpose & Justification |
| :--- | :--- | :--- |
| **Framework** | **Flutter Desktop (Windows x64)** | High performance, native 60fps desktop rendering, clean UI/UX |
| **Local Database** | `sqflite_common_ffi` | Standard ACID-compliant SQLite engine for Windows desktop |
| **State Management**| `flutter_bloc` or `riverpod` | Predictable state machines for POS and LP order states |
| **PDF Generation** | `pdf` & `printing` | Pixel-perfect programmatic vector PDF generation matching sample forms |
| **Excel Export** | `syncfusion_flutter_xlsio` / `excel` | Native .xlsx spreadsheet generation without cloud dependencies |
| **Charts & Graphs**| `fl_chart` | Smooth, modern interactive dashboard trends and stock charts |
| **Icons & UI** | `lucide_icons` / `fluent_ui` | Clean, modern clinical and accounting design aesthetics |
| **Localization** | `intl` | Pakistani Rupee (`PKR`, `Rs.`), local number grouping, and date formats |
