# Architecture

## Purpose

The system categorizes YNAB transactions to automate budget reconciliation.
Categorization decisions depend upon input from multiple sources to include:
* YNAB: e.g., historical transactions
* Gmail: e.g., labeled emails conveying text receipts
* Drive: e.g., cloud folders conveying text/scanned receipts
* CSV: e.g., a record conveying multiple receipts

## Context

```text
+-------------------------+                   +--------------------------+
| MailSource              |<- read -----------|                          |
| receipts: emails        |   (Receipts)      |                          |
+-------------------------+                   |                          |
                                              |                          |
+-------------------------+                   |                          |
| DriveSource             |<- read -----------|                          |
| receipts: files         |   (Receipts)      |                          |
+-------------------------+                   |                          |
                                              |   Accountant             |
+-------------------------+                   |  (matching service)      |
| TransactionSource       |<- read -----------|                          |
| transactions            |   (Transactions)  |                          |
+-------------------------+                   |                          |
                                              |                          |
+-------------------------+                   |                          |
| DataSource              |<- read -----------|                          |
| receipts: records       |   (Receipts)      |                          |
+-------------------------+                   +--------------------------+
                                                            | (Transactions)
                                                            v
                                              +--------------------------+
                                              |   TransactionSink        |
                                              |   local ledger           |
                                              +--------------------------+
```

## Data Model

### Receipt
A Receipt provides evidence supporting a transaction.
Each Receipt has a common form: 
- id: UUID String (e.g., ABCDEF123)
- source: String (e.g., Google Drive)
- source_reference: String (e.g., purchase-yyyymmdd.pdf)
- date: YYYY-MM-DD Date (e.g., 2026-02-15) 
- payee_name: String (e.g., "Acme, Inc.")  
- memo: String (e.g., "Groceries for trip")
- amount: Integer Milliunits (1000 = 1.00)
- itemized[ReceiptItem (1..*)]

### ReceiptItem
A Receipt consists of one or more ReceiptItems..
Each ReceiptItem has a common form:
- id: UUID String (e.g., ABCDEF123)
- receipt_id: UUID String (i.e., Parent id)
- memo: String  (e.g., "Snacks")
- category: String/Enum (e.g., Household::Grocery)
- amount: Integer Milliunits (1000 = 1.00)

Each Receipt has one or more ReceiptItems contributing to the total amount.

### Transaction
A Transaction is a single account movement held in YNAB (or imported into it).
Each Transaction has a common form:
- id (string, UUID; YNAB transaction id)
- date (ISO `YYYY-MM-DD`)
- amount (integer milliunits; 1000 = 1.00; outflows negative)
- memo (string or null)
- payee_name (string or null; merchant equivalent)
- category_name (string or null; `Split` when the Transaction has subtransactions)
- account_id (UUID) 
- account_name (string)
- cleared (`cleared` | `uncleared` | `reconciled`)
- approved (boolean)
- subtransactions[SubTransaction (0..*)]

### SubTransaction
A SubTransaction is one line of a split Transaction. 
Each SubTransaction has a common form:
- id (string, UUID)
- transaction_id (string; parent Transaction id)
- amount (integer milliunits; sum of all SubTransaction amounts equals the parent amount)
- memo (string or null)
- payee_name (string or null)
- category_name (string or null)

A Transaction with no `Split` has an empty `subtransactions` list.

### ReceiptLedger

Maintains a local record of canonical Receipts from all receipt sources.

### TransactionLedger

Maintains a local record of canonical transactions from a remote source (YNAB).

### AccountLedger

Maintains an association of Receipts to Transactions.

## Components

The following components manipulate objects in the data model.

### MailSource

Adds/updates Receipts in a ReceiptLedger from GMail.
Uses **MailScan** to list and **MailFetch** to fetch email messages.
Email messages must carry either the `receipts` or `ynab` label.
Uses **TextReader** to extract Receipt metadata from each email message.

### DriveSource

Adds/updates Receipts in a ReceiptLedger from Google Drive.
Uses **DriveScan** to list files, **DriveFetch** to download files. 
Files must be either PDF or plain text (applying OCR on receipt scans).
Uses **TextReader** to extract Receipt metadata from each file.

### DataSource

Adds/updates Receipts in a ReceiptLedger from a CSV File.
Uses **CSVReader** to extract Receipt metadata from each CSV record.

### TransactionSource

Reads Transactions in a TransactionLedger (YNAB).

### TransactionSink

Updates Transactions in a TransactionLedger (YNAB).

### Accountant

Reads the ReceiptLedger and TransactionLedger. 
Updates the AccountLedger by associating Receipts to Transactions by reference.
Composed of a **Scanner**, **Scorer**, and **Classifier**.
Pairs use amount, date window, and merchant similarity.
Partial or ambiguous matches are flagged rather than forced.

## Data Flow

1. MailSource and DriveSource (and DataSource for CSV files) add/update Receipts in the ReceiptLedger.
2. TransactionSource reads Transactions from YNAB into the TransactionLedger.
3. MailSource, DriveSource, and DataSource use MailScan/MailFetch, DriveScan/DriveFetch, or CSVReader to obtain raw records, then TextReader to extract Receipt metadata (and ReceiptItems) normalised into the ReceiptLedger.
4. Accountant reads the ReceiptLedger and TransactionLedger, pairing Receipts to Transactions by amount, date window, and merchant similarity, and updates the AccountLedger by reference.
5. Matches above the confidence threshold are written out via TransactionSink.
   Partial or ambiguous matches are flagged for manual review rather than forced.

## Constraints and Decisions
