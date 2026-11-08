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
 +-----------+  +------------+  +-----------+
 | MailSource|  | DriveSource|  | CSVSource |
 | - Emails  |  | - Files    |  | - CSV File|
 +-----+-----+  +-----+------+  +-----+-----+
       |  update      |  update       |  update
       |  (Receipts)  |               |
       +--------------+---------------+
                      v
          +-----------------------+
          |     ReceiptLedger     |
          +-----------+-----------+
                      | read
                      v
 +-------------+   +-----------------------+   +----------------+
 | YNABSource  |   |       Accountant      |-->| AccountLedger  |
 | - YNAB      |   |  - Scanner            |   +-------+--------+
 |   Budget    |   |  - Scorer             |           | read
 +------+------+   |  - Classifier         |           v
        | update   +-----------+-----------+   +----------------+
        v                      ^ read          |    YNABSink    |
 +-------------------+         |               | - YNAB Budget  |
 | TransactionLedger |---------+               +----------------+
 +-------------------+
```

## Data Flow

1. **Receipt ingest:** MailSource, DriveSource, and CSVSource use MailScan/MailFetch, DriveScan/DriveFetch, or CSVScan to obtain raw records, then TextReader to extract Receipt metadata (and ReceiptItems). Each source adds/updates the resulting Receipts in the ReceiptLedger.
2. **Transaction ingest:** YNABSource reads Transactions from the YNAB Budget into the TransactionLedger.
3. **Matching:** Accountant reads the ReceiptLedger and TransactionLedger. Its Scanner proposes candidate associations and its Scorer scores them by amount, date window, and merchant similarity. Accepted candidates are written to the AccountLedger by reference.
4. **Classification:** Accountant's Classifier assigns a category_name to each Transaction. The proposal is held until write-back.
5. **Write-back:** YNABSink writes confirmed matches and categories to the YNAB Budget, only after user confirmation.

## Data Model

### Receipt
A Receipt provides evidence supporting a transaction.
Each Receipt has a common form: 
- id: UUID String
- source: String
- source_reference: String
- date: ISO `YYYY-MM-DD`
- payee_name: String
- memo: String
- amount: Integer Milliunits (1000 = 1.00; always positive)
- itemized[ReceiptItem (1..*)]

### ReceiptItem
A Receipt consists of one or more ReceiptItems..
Each ReceiptItem has a common form:
- id: UUID String
- receipt_id: UUID String
- memo: String
- category: String/Enum
- amount: Integer Milliunits (1000 = 1.00; always positive)

Each Receipt has one or more ReceiptItems contributing to the total amount.

### Transaction
A Transaction is a single account movement held in YNAB (or imported into it).
Each Transaction has a common form:
- id: UUID String (e.g., YNAB id)
- date: ISO `YYYY-MM-DD`
- amount: Integer Milliunits (1000 = 1.00; outflows negative)
- memo: String
- payee_name: String
- category_name: String/Enum (`Split` when subtransactions)
- account_id: UUID String
- cleared: String/Enum (`cleared` | `uncleared` | `reconciled`)
- approved: Boolean
- subtransactions[SubTransaction (0..*)]

A Transaction with no `Split` has an empty `subtransactions` list.

### SubTransaction
A SubTransaction is one line of a split Transaction. 
Each SubTransaction has a common form:
- id: UUID String
- transaction_id: UUID String (parent id)
- amount: Integer Milliunits (1000 = 1.00; outflows negative)
- memo: String
- payee_name: String
- category_name: String/Enum

## Stores

### ReceiptLedger

Maintains a local record of canonical Receipts from all receipt sources.

### TransactionLedger

Maintains a record of canonical transactions.
A TransactionLedger may be local or remote (e.g., YNAB API).

### AccountLedger

Maintains an association of Receipts to Transactions.
This ledger takes the following form:
- transaction_id: UUID String
- receipt_id: UUID String
- match: Enum (`missing` | `full` | `partial` | `confirmed`)
- confidence: Integer

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

### CSVSource

Adds/updates Receipts in a ReceiptLedger from a CSV File.
Uses **CSVScan** to list CSV records. 
Uses **TextReader** to extract Receipt metadata from each record.

### YNABSource

Adds/updates Transactions in a TransactionLedger.

### YNABSink

Updates Transactions in a remote TransactionLedger (e.g., YNAB API).

### Accountant

Reads from the ReceiptLedger and TransactionLedger. 
Updates the AccountLedger by associating Receipts to Transactions by reference.
Composed of a **Scanner**, **Scorer**, and **Classifier**.
- The **Scanner** uses fuzzy search to identify potential associations.
- The **Scorer** scores those associations across multiple dimensions.
  Considers amount, date range, and payee_name.
- The **Classifier** assigns a category_name for each Transaction.

## Constraints and Decisions
