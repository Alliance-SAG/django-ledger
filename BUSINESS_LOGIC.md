# Django Ledger - Business Logic & Model Architecture

## Overview

Django Ledger is a **double-entry accounting system** designed to support business operations for entities that need to track both revenue (money coming in) and expenses (money going out). This document clarifies the business logic, accounting principles, and model architecture to help system architects understand how to integrate and use this library.

---

## Key Question: What Type of Business is Represented?

Django Ledger represents **your business entity** - the organization using this system to track its financial activities. The models are designed from **your business's perspective**:

- **Customers** buy from you → You send them **Invoices** (Accounts Receivable)
- **Vendors** sell to you → They send you **Bills** (Accounts Payable)

---

## Core Accounting Principles

### Double-Entry Accounting

Every financial transaction affects **at least two accounts**:
- One account is **debited** (increased for assets/expenses, decreased for liabilities/equity/revenue)
- Another account is **credited** (decreased for assets/expenses, increased for liabilities/equity/revenue)

### Account Types & Normal Balances

| Account Type | Normal Balance | Increase | Decrease | Examples |
|--------------|----------------|----------|----------|----------|
| **Assets** | Debit | Debit | Credit | Cash, Accounts Receivable, Inventory |
| **Liabilities** | Credit | Credit | Debit | Accounts Payable, Loans, Deferred Revenue |
| **Equity** | Credit | Credit | Debit | Capital, Retained Earnings |
| **Revenue** | Credit | Credit | Debit | Sales, Service Income |
| **Expenses** | Debit | Debit | Credit | Cost of Goods Sold, Salaries, Rent |

---

## Business Document Models

### 1. Invoice Model (`invoice.py`)

**Purpose:** Sales documents that **YOU issue TO customers** for goods/services provided.

**Accounting Direction:**
- Represents money **owed TO your business** (Accounts Receivable)
- **Asset account** - Normal DEBIT balance
- When approved, **DEBITS** Accounts Receivable and **CREDITS** Revenue

**Key Characteristics:**
```python
# From invoice.py
from django_ledger.io import ASSET_CA_RECEIVABLES  # Uses A/R account

class InvoiceModel:
    IS_DEBIT_BALANCE = True  # Asset accounts have debit balances
```

**Typical Journal Entry (when approved):**
```
DR  Accounts Receivable    $1,000
    CR  Sales Revenue               $1,000
```

**Lifecycle:**
1. **Draft** - Being prepared, no ledger impact
2. **In Review** - Pending approval, no ledger impact  
3. **Approved** - Creates journal entries, increases A/R
4. **Paid** - Customer pays, cash received
5. **Void/Canceled** - Reverses entries if needed

**Related Models:**
- `CustomerModel` - The entity receiving the invoice
- `EstimateModel` - Can be converted to an invoice

**Use Case Example:**
> Your consulting firm completes a project for Acme Corp and sends them an invoice for $10,000 with 30-day payment terms.

---

### 2. Bill Model (`bill.py`)

**Purpose:** Purchase invoices/bills that **YOU receive FROM vendors** for goods/services purchased.

**Accounting Direction:**
- Represents money **owed BY your business** (Accounts Payable)
- **Liability account** - Normal CREDIT balance
- When approved, **DEBITS** Expense/Asset and **CREDITS** Accounts Payable

**Key Characteristics:**
```python
# From bill.py
from django_ledger.io import LIABILITY_CL_ACC_PAYABLE  # Uses A/P account

class BillModel:
    IS_DEBIT_BALANCE = False  # Liability accounts have credit balances
```

**Typical Journal Entry (when approved):**
```
DR  Office Supplies Expense    $500
    CR  Accounts Payable                $500
```

**Lifecycle:**
1. **Draft** - Being reviewed, no ledger impact
2. **In Review** - Pending approval, no ledger impact
3. **Approved** - Creates journal entries, increases A/P
4. **Paid** - Vendor paid, cash disbursed
5. **Void/Canceled** - Reverses entries if needed

**Related Models:**
- `VendorModel` - The entity sending the bill
- `PurchaseOrderModel` - Can be linked to bills

**Use Case Example:**
> Your business receives a bill from Office Depot for $500 worth of supplies with 15-day payment terms.

---

### 3. Estimate Model (`estimate.py`)

**Purpose:** Quotes, proposals, or contracts for potential work with customers.

**Accounting Direction:**
- **No direct ledger impact** (pre-sale stage)
- Can be converted to invoices when accepted
- May track potential revenue for forecasting

**Lifecycle:**
1. **Draft** - Being prepared
2. **In Review** - Sent to customer for review
3. **Approved** - Customer accepts
4. **Converted** - Transformed into invoice(s)
5. **Canceled** - Opportunity lost

**Related Models:**
- Can be parent to multiple invoices, bills, and purchase orders
- Links to `CustomerModel`

**Use Case Example:**
> You send a $50,000 estimate to a potential client for a 3-month project. Once they approve, you convert it to monthly invoices.

---

### 4. Purchase Order Model (`purchase_order.py`)

**Purpose:** Orders placed **BY your business WITH vendors** to purchase goods/services.

**Accounting Direction:**
- Tracks **commitments** but typically no immediate ledger impact
- May create encumbrances in advanced accounting
- Linked to bills when goods/services are received

**Lifecycle:**
1. **Draft** - Being prepared
2. **In Review** - Pending approval
3. **Approved** - Sent to vendor
4. **Fulfilled** - Goods/services received, linked to bill
5. **Canceled** - Order canceled

**Related Models:**
- Links to `VendorModel`
- Can be linked to `BillModel` when fulfilled
- May be linked to `EstimateModel` for project tracking

**Use Case Example:**
> You issue a PO to your supplier for $2,000 worth of inventory. When delivered, you receive a bill that references this PO.

---

## Business Flow Cycles

### Revenue Cycle (Sales Process)

```
1. Customer Inquiry
   ↓
2. Create Estimate (Quote)
   ↓
3. Customer Accepts
   ↓
4. Convert to Invoice
   ↓
5. Approve Invoice
   ↓  (Creates journal entry: DR A/R, CR Revenue)
6. Deliver Goods/Services
   ↓
7. Customer Pays
   ↓  (Creates journal entry: DR Cash, CR A/R)
8. Mark Invoice as Paid
```

**Accounts Affected:**
- **Accounts Receivable** (Asset) - Increases when invoice approved, decreases when paid
- **Sales Revenue** (Revenue) - Increases when invoice approved
- **Cash** (Asset) - Increases when customer pays

---

### Expenditure Cycle (Procurement Process)

```
1. Identify Need
   ↓
2. Create Purchase Order
   ↓
3. Approve PO
   ↓
4. Send to Vendor
   ↓
5. Receive Goods/Services
   ↓
6. Receive Bill from Vendor
   ↓
7. Approve Bill
   ↓  (Creates journal entry: DR Expense/Asset, CR A/P)
8. Schedule Payment
   ↓
9. Pay Vendor
   ↓  (Creates journal entry: DR A/P, CR Cash)
10. Mark Bill as Paid
```

**Accounts Affected:**
- **Accounts Payable** (Liability) - Increases when bill approved, decreases when paid
- **Expenses/Assets** (varies) - Increases when bill approved
- **Cash** (Asset) - Decreases when vendor is paid

---

## Project Tracking with Estimates

Estimates can serve as project containers linking multiple business documents:

```
Estimate (Project Container)
  ├── Multiple Invoices (to customer)
  ├── Multiple Bills (from vendors)
  └── Multiple Purchase Orders (to vendors)
```

**Use Case Example:**
> Construction Project:
> - Estimate: $100,000 total project
> - Invoice customer: 3 progress invoices ($30k, $30k, $40k)
> - Bills received: Materials ($25k), Subcontractors ($35k)
> - POs issued: Equipment rental ($5k)
> 
> Net Project Profit: $100k revenue - $65k costs = $35k profit

---

## Model Relationships

### Core Entity Hierarchy

```
EntityModel (Your Business)
  ├── CustomerModel (Entities that buy from you)
  │     └── InvoiceModel (What you bill them)
  │           ├── ItemTransactionModel (Line items)
  │           └── JournalEntryModel (Accounting entries)
  │
  ├── VendorModel (Entities you buy from)
  │     ├── BillModel (What they bill you)
  │     │     ├── ItemTransactionModel (Line items)
  │     │     └── JournalEntryModel (Accounting entries)
  │     └── PurchaseOrderModel (What you order from them)
  │
  ├── EstimateModel (Quotes to customers)
  │     └── Links to Invoices, Bills, POs
  │
  ├── ChartOfAccountsModel
  │     └── AccountModel (Individual accounts)
  │           └── TransactionModel (Individual debits/credits)
  │
  └── LedgerModel
        └── JournalEntryModel (Groups of balanced transactions)
```

---

## Common Integration Scenarios

### Scenario 1: Service-Based Business

**Example:** Software Consulting Firm

1. **Revenue Side:**
   - Create `CustomerModel` for each client
   - Send `InvoiceModel` for completed work
   - Track A/R and cash collections

2. **Expense Side:**
   - Create `VendorModel` for suppliers (hosting, software, contractors)
   - Receive `BillModel` for expenses
   - Track A/P and cash disbursements

### Scenario 2: Product-Based Business

**Example:** Retail Store

1. **Revenue Side:**
   - Create `CustomerModel` for each customer (or use default)
   - Generate `InvoiceModel` for each sale
   - Track inventory reduction and revenue

2. **Expense Side:**
   - Create `VendorModel` for suppliers
   - Issue `PurchaseOrderModel` for inventory
   - Receive `BillModel` when inventory arrives
   - Track inventory increase and A/P

### Scenario 3: Project-Based Business

**Example:** Construction Company

1. **Project Setup:**
   - Create `EstimateModel` for project bid
   - Link to `CustomerModel`

2. **Revenue Recognition:**
   - Create multiple `InvoiceModel` (progress billing)
   - Link to parent `EstimateModel`

3. **Cost Tracking:**
   - Issue `PurchaseOrderModel` to suppliers
   - Receive `BillModel` for materials and labor
   - Link to parent `EstimateModel` for job costing

---

## Status Workflow

All major document models follow similar status progressions:

### Status Definitions

| Status | Description | Ledger Impact | Can Edit | Can Delete |
|--------|-------------|---------------|----------|------------|
| **Draft** | Initial creation, work in progress | None | Yes | Yes |
| **In Review** | Pending approval/review | None | Limited | No |
| **Approved** | Finalized and impacting books | **YES** | No | No |
| **Paid** | Fully settled | YES | No | No |
| **Void** | Reversed/nullified (entries reversed) | Reversal | No | No |
| **Canceled** | Canceled before approval | None | No | No |

### Key Principle: Approved = In the Books

**IMPORTANT:** Once a document is **Approved**, it creates immutable journal entries. The document cannot be edited or deleted - only voided (which creates reversing entries).

---

## Financial Statements Generated

Django Ledger automatically generates standard financial statements:

### 1. Balance Sheet (Statement of Financial Position)

Shows **what you own** (assets) and **what you owe** (liabilities) at a point in time:

```
ASSETS
  Current Assets
    - Cash
    - Accounts Receivable (from Invoices)
    - Inventory
  Fixed Assets
    - Property, Plant & Equipment

LIABILITIES
  Current Liabilities
    - Accounts Payable (from Bills)
    - Deferred Revenue
  Long-term Liabilities
    - Loans Payable

EQUITY
  - Capital
  - Retained Earnings
```

### 2. Income Statement (Profit & Loss)

Shows **revenue earned** and **expenses incurred** over a period:

```
REVENUE
  - Sales (from Invoices)
  - Service Revenue (from Invoices)

EXPENSES
  - Cost of Goods Sold (from Bills)
  - Operating Expenses (from Bills)
  - Salaries
  - Rent

NET INCOME = Revenue - Expenses
```

### 3. Cash Flow Statement

Shows **actual cash movements** over a period:

```
OPERATING ACTIVITIES
  - Cash from customers (Invoice payments)
  - Cash to vendors (Bill payments)

INVESTING ACTIVITIES
  - Purchase of equipment
  - Sale of assets

FINANCING ACTIVITIES
  - Loans received
  - Loan payments
```

---

## Important Conventions

### 1. Entity-Centric Design

All models are scoped to an `EntityModel` - your business entity. If you're building a multi-tenant application, each tenant gets their own entity.

### 2. Immutability After Approval

Once approved, financial documents cannot be edited. This maintains audit integrity. To correct errors:
- **Void** the incorrect document (creates reversing entries)
- Create a new **correct** document

### 3. Accrual vs. Cash Accounting

Django Ledger supports **accrual accounting** by default:
- Revenue recognized when invoice is **approved** (not when paid)
- Expenses recognized when bill is **approved** (not when paid)

Cash flow is tracked separately through payment records.

### 4. Multi-Currency Support

Django Ledger is designed primarily for single-currency operations. If you need multi-currency support, you will need to extend the base models. The architecture provides extension points at the `ItemModel` and transaction level, but multi-currency features are not built-in and would require custom implementation.

---

## Quick Reference: Invoice vs. Bill

| Aspect | Invoice | Bill |
|--------|---------|------|
| **Direction** | YOU → Customer | Vendor → YOU |
| **Represents** | Sales (Revenue) | Purchases (Expenses) |
| **Account Type** | Asset (A/R) | Liability (A/P) |
| **Normal Balance** | Debit | Credit |
| **Related Entity** | CustomerModel | VendorModel |
| **Journal Entry** | DR A/R, CR Revenue | DR Expense, CR A/P |
| **Cash Impact** | Cash IN when paid | Cash OUT when paid |

---

## Integration Checklist for Architects

When evaluating Django Ledger for your project:

- [ ] **Identify your business type** (service, product, project-based)
- [ ] **Map your revenue streams** → Use `InvoiceModel` + `CustomerModel`
- [ ] **Map your expense categories** → Use `BillModel` + `VendorModel`
- [ ] **Determine if you need estimates/quotes** → Use `EstimateModel`
- [ ] **Determine if you need purchase orders** → Use `PurchaseOrderModel`
- [ ] **Plan your Chart of Accounts** → Customize default CoA
- [ ] **Define approval workflows** → Use status transitions
- [ ] **Identify reporting needs** → Balance Sheet, P&L, Cash Flow
- [ ] **Consider multi-entity needs** → Separate `EntityModel` per tenant
- [ ] **Plan for payment tracking** → Link to payment gateways/bank accounts

---

## Additional Resources

- **Main Documentation:** https://django-ledger.readthedocs.io/
- **QuickStart Notebook:** `/notebooks/QuickStart Notebook.ipynb`
- **Model Reference:** `/docs/source/models.rst`
- **API Examples:** See inline docstrings in model files

---

## Support & Contributing

For questions about business logic or accounting principles:
- **Discord Community:** https://discord.gg/c7PZcbYgrc
- **GitHub Issues:** Report bugs or request features
- **Consulting Services:** Contact msanda@arrobalytics.com

Created and maintained by [Miguel Sanda](https://www.miguelsanda.com)

Copyright© EDMA Group Inc - Licensed under GPLv3
