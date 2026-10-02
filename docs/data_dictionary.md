# Data Dictionary

Describes every column in `data/clean/credit_card_data_CLEAN.csv` (and its raw counterpart before cleaning). One row per client.

| Column | Type | Description | Example |
|---|---|---|---|
| `CLIENTNUM` | Integer | Unique client identifier, primary key | `768000012` |
| `Customer_Age` | Integer | Client's age in years | `44` |
| `Gender` | Text | Client's gender, `M` or `F` | `F` |
| `Income` | Decimal | Annual income in local currency | `58338.0` |
| `Card_Category` | Text | Credit card tier: `Blue`, `Silver`, `Gold`, `Platinum` | `Gold` |
| `creditLimit` | Decimal | Approved credit limit on the card | `17435.0` |
| `Total_Revolving_Bal` | Decimal | Balance carried forward (revolving) on the card | `6227.0` |
| `Avg_Utilization_Ratio` | Decimal (0–1) | `Total_Revolving_Bal` as a share of `creditLimit` | `0.3572` |
| `Total_Trans_Amt` | Decimal | Total transaction amount in the period | `3435.95` |
| `Total_Trans_Ct` | Integer | Total number of transactions in the period | `42` |
| `Interest_Earned` | Decimal | Interest revenue earned on this client's balance | `171.42` |
| `Delinquent_Acc` | Integer | Count of delinquent accounts held by the client (0, 1, 2…) | `0` |
| `Personal_loan` | Text | Whether the client holds a personal loan, `Yes` or `No` | `No` |
| `Cust_Satisfaction_Score` | Integer (1–5) | Self-reported customer satisfaction score | `3` |
| `Acquisition_Cost` | Decimal | Cost incurred to acquire this client | `76.56` |
| `Txn_Date` | Date (`YYYY-MM-DD`) | Date associated with the transaction record | `2024-07-22` |

## Known issues fixed between raw and clean

| Issue | Where it showed up | Fix applied |
|---|---|---|
| Missing values | `Income`, `Avg_Utilization_Ratio`, `Cust_Satisfaction_Score`, `Acquisition_Cost`, `Total_Trans_Amt` | *(document your actual fix here — e.g. median imputation, row removal)* |
| Inconsistent text casing | `Card_Category` (`blue`/`Blue`/`BLUE`), `Personal_loan` (`Yes`/`yes`/`Y`/`N`) | Standardised to title case / `Yes`-`No` |
| Out-of-range values | `Avg_Utilization_Ratio` > 1.0, `Customer_Age` > 120 | *(document how you handled these — capped, removed, flagged)* |
| Negative amounts | `Total_Trans_Amt` with negative values | *(document your fix — investigated as refunds, corrected, or removed)* |
| Mixed date formats | `Txn_Date` in multiple formats (`DD/MM/YYYY`, `MM-DD-YYYY`, `DD-Mon-YYYY`) | Standardised to ISO `YYYY-MM-DD` |
| Duplicate rows | Full-row duplicates | Removed |

> Fill in the "Fix applied" cells above with what you actually did once you clean the real dataset — this table should describe your process, not a template.
