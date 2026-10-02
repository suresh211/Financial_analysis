# Credit Card Portfolio Analytics — DAX Measures for Power BI

A set of 15 DAX measures built for a client-level credit card dataset, covering transaction growth, credit utilization, revenue, churn, delinquency risk, and customer segmentation. Designed to power KPI cards, trend visuals, and risk-flagging tables in a Power BI report for a banking institution.

## Repo Structure

```
credit-card-dax-analytics/
├── README.md
├── data/
│   ├── raw/credit_card_data_RAW.csv       # before cleaning
│   └── clean/credit_card_data_CLEAN.csv   # after cleaning
├── dax/measures.dax                       # all 15 formulas, plain text
├── docs/
│   ├── Credit_Card_Analytics.pptx         # problem + solution deck
│   └── data_dictionary.md                 # column-by-column reference
├── reports/findings.md                    # actual results once run in Power BI
└── images/screenshots/                    # report screenshots
```

> **Note on the data:** `data/raw` and `data/clean` in this repo are a synthetic sample built to the same schema as the source dataset, for structure and practice. Replace both with your actual cleaned/raw files before treating any numbers here as findings — see `reports/findings.md`.

## Table of Contents

- [Data Model](#data-model)
- [Formatting Convention](#formatting-convention)
- [1. Transaction Trends](#1-transaction-trends)
- [2. Utilization & Revenue](#2-utilization--revenue)
- [3. Risk & Retention](#3-risk--retention)
- [4. Segmentation Insights](#4-segmentation-insights)
- [Key Takeaways](#key-takeaways)

## Data Model

| Table | Role | Key fields |
|---|---|---|
| `CC` | Client-level fact/dimension table | `CLIENTNUM`, `creditLimit`, `Income`, `Avg_Utilization_Ratio`, `Total_Revolving_Bal`, `Total_Trans_Amt`, `Interest_Earned`, `Delinquent_Acc`, `Card_Category`, `Personal_loan`, `Cust_Satisfaction_Score` |
| `Date` | Standard `CALENDAR()` date table, marked as the active date table, related to a transaction-date column | `Date` |

`Acquisition_Cost` is assumed present on `CC` for the CAC measure — substitute your actual cost field if it's named differently.

> **Note:** If your source file is a single client-level snapshot (no transaction dates), measures 1, 2, 3, and 9 are the standard time-intelligence pattern to apply once a dated fact table is loaded — they won't resolve against a flat snapshot alone.

## Formatting Convention

Ratio-based measures (growth rates, utilization, interest yield, delinquency rate) use Power BI's built-in **Percentage** format applied directly to the measure. `DIVIDE()` results are left as decimals (0–1) — there's no manual `* 100` in the DAX itself.

The one exception is the **Credit Risk Score** (#11): it's a weighted 0–100 point score, not a ratio, so it keeps `* 100` scaling on each component and uses **Decimal Number** formatting instead.

---

## 1. Transaction Trends

### Problem 1 — Running Total of Credit Card Transactions

A cumulative sum of transaction amount up to the current date in context — each point on the trend shows everything spent so far, not just that period.

```dax
Running Total Trans Amt =
CALCULATE(
    SUM(CC[Total_Trans_Amt]),
    FILTER(
        ALLSELECTED(CC),
        CC[Txn_Date] <= MAX(CC[Txn_Date])
    )
)
```

**Business read:** Plot on a line chart by date to reveal acceleration or plateaus in overall spend.

### Problem 2 — 4-Week Moving Average of Credit Limit

Smooths short-term noise by averaging `creditLimit` over the trailing 28 days for each client, so limit-policy trends read clearly.

```dax
CreditLimit 4W Avg =
AVERAGEX(
    DATESINPERIOD(
        'Date'[Date],
        MAX('Date'[Date]),
        -28, DAY
    ),
    CALCULATE(AVERAGE(CC[creditLimit]))
)
```

**Business read:** A widening gap between the raw value and this average signals a recent limit change.

### Problem 3 — MoM % and WoW % Growth on Transaction Amount

Compares current transaction amount against the same measure one month, and one week, earlier — the two standard growth-rate lenses. Format both as **Percentage**.

```dax
MoM % Growth =
VAR PM =
    CALCULATE(
        [Total Trans Amt],
        DATEADD('Date'[Date], -1, MONTH)
    )
RETURN
DIVIDE([Total Trans Amt] - PM, PM)

WoW % Growth =
VAR PW =
    CALCULATE(
        [Total Trans Amt],
        DATEADD('Date'[Date], -7, DAY)
    )
RETURN
DIVIDE([Total Trans Amt] - PW, PW)
```

**Business read:** Use WoW for operational alerts and MoM for board-level trend reporting.

### Problem 4 — Customer Acquisition Cost as a % of Transaction Amount

Expresses what was spent to acquire clients as a percentage of the transaction revenue those clients generate — an efficiency check on acquisition spend. Format as **Percentage**.

```dax
CAC to Trans Amt % =
DIVIDE(
    SUM(CC[Acquisition_Cost]),
    [Total Trans Amt]
)
```

**Business read:** A rising ratio over time means acquisition spend is outpacing the revenue it produces.

---

## 2. Utilization & Revenue

### Problem 5 — Yearly Average of Avg_Utilization_Ratio

Rolls up every client's utilization ratio into one portfolio-level figure per year, the headline number for overall credit-line usage. Format as **Percentage**.

```dax
Yearly Avg Utilization =
CALCULATE(
    AVERAGE(CC[Avg_Utilization_Ratio]),
    ALLSELECTED(CC[CLIENTNUM])
)
```

**Business read:** Trend this by year to spot whether the book is drifting toward heavier utilization.

### Problem 6 — Interest Earned as a % of Total Revolving Balance

Shows how much interest yield is being generated per client relative to the balance they are carrying — a proxy for revolving-book profitability. Format as **Percentage**.

```dax
Interest % of Revolving Bal =
DIVIDE(
    SUM(CC[Interest_Earned]),
    SUM(CC[Total_Revolving_Bal])
)
```

**Business read:** Compare this against the delinquency rate for the same client — high yield with high delinquency is a red flag, not a win.

### Problem 7 — Top 5 Clients by Total Transaction Amount

Ranks clients dynamically within whatever filter context is active, so the leaderboard updates as the user slices the report.

```dax
Top 5 Trans Amt =
CALCULATE(
    [Total Trans Amt],
    TOPN(
        5, ALLSELECTED(CC[CLIENTNUM]),
        [Total Trans Amt], DESC
    )
)
```

**Business read:** Pair with `RANKX` as a visual-level filter to build a self-updating Top 5 table.

### Problem 8 — Clients Whose Avg_Utilization_Ratio Exceeds 80%

Counts and isolates the clients running their credit lines hot — the pool most likely to need a limit review or proactive outreach.

```dax
High Utilization Clients =
CALCULATE(
    DISTINCTCOUNT(CC[CLIENTNUM]),
    CC[Avg_Utilization_Ratio] > 0.8
)
```

**Business read:** Feed this count into a card visual alongside the total client count for an instant exposure ratio.

---

## 3. Risk & Retention

### Problem 9 — Customer Churn Indicator

Flags any client with zero transaction amount across the trailing six months — a simple, fast proxy for disengagement.

```dax
Churn Flag =
VAR Last6M =
    CALCULATE(
        [Total Trans Amt],
        DATESINPERIOD(
            'Date'[Date], MAX('Date'[Date]),
            -6, MONTH
        )
    )
RETURN
IF(Last6M = 0, "Churned", "Active")
```

**Business read:** Surface this as a KPI tile so retention teams can prioritise outreach before an account lapses further.

### Problem 10 — Delinquency Rate

The share of the client base carrying at least one delinquent account — the single most-watched credit-quality metric. Format as **Percentage**.

```dax
Delinquency Rate % =
DIVIDE(
    CALCULATE(
        DISTINCTCOUNT(CC[CLIENTNUM]),
        CC[Delinquent_Acc] > 0
    ),
    DISTINCTCOUNT(CC[CLIENTNUM])
)
```

**Business read:** Break this out by `Card_Category` to see whether risk concentrates in a specific tier.

### Problem 11 — Credit Risk Score

A weighted composite of utilization, delinquency, and revolving balance, so each client gets one comparable 0–100 risk figure instead of three separate ones. Format as **Decimal Number** — this is a point score, not a ratio.

```dax
Credit Risk Score =
VAR UtilPart = CC[Avg_Utilization_Ratio] * 100 * 0.4
VAR DelinqPart =
    MIN(CC[Delinquent_Acc] * 20, 100) * 0.35
VAR RevolvPart =
    DIVIDE(
        CC[Total_Revolving_Bal],
        MAXX(ALL(CC), CC[Total_Revolving_Bal])
    ) * 100 * 0.25
RETURN
UtilPart + DelinqPart + RevolvPart
```

**Business read:** Weights (40/35/25) are a starting point — recalibrate them once you can validate the score against actual write-offs.

### Problem 12 — Income vs Credit Limit Correlation

DAX has no built-in `CORREL`, so the Pearson coefficient is built by hand from its five summed components across all clients.

```dax
Income-CreditLimit Correlation =
VAR N = COUNTROWS(CC)
VAR SX = SUMX(CC, CC[Income])
VAR SY = SUMX(CC, CC[creditLimit])
VAR SXY = SUMX(CC, CC[Income] * CC[creditLimit])
VAR SX2 = SUMX(CC, CC[Income] ^ 2)
VAR SY2 = SUMX(CC, CC[creditLimit] ^ 2)
VAR Num = (N * SXY) - (SX * SY)
VAR Den =
    SQRT((N * SX2 - SX ^ 2) * (N * SY2 - SY ^ 2))
RETURN
DIVIDE(Num, Den)
```

**Business read:** A coefficient near 1 supports income-based limit setting; near 0 suggests other factors drive limit decisions.

---

## 4. Segmentation Insights

### Problem 13 — Average Customer Satisfaction Score by Card Category

One reusable measure that recalculates automatically for whichever `Card_Category` the visual is sliced by.

```dax
Avg Satisfaction Score =
AVERAGE(CC[Cust_Satisfaction_Score])
```

**Business read:** Place on a clustered bar by `Card_Category` — a satisfaction gap between tiers usually points to a benefits or service issue.

### Problem 14 — Loan Approval vs Credit Limit

Splits average credit limit between clients who hold a personal loan and those who don't, to see whether loan status tracks with limit size.

```dax
Avg CL - With Loan =
CALCULATE(
    AVERAGE(CC[creditLimit]),
    CC[Personal_loan] = "Yes"
)

Avg CL - No Loan =
CALCULATE(
    AVERAGE(CC[creditLimit]),
    CC[Personal_loan] = "No"
)
```

**Business read:** A wide gap suggests credit limit is a meaningful input — or an output — of the loan approval decision.

### Problem 15 — High Risk Clients Flag

Combines two conditions — revolving balance above 90% of the limit, and utilization above 80% — into a single actionable flag.

```dax
High Risk Flag =
VAR UtilPct = CC[Avg_Utilization_Ratio]
VAR BalRatio =
    DIVIDE(CC[Total_Revolving_Bal], CC[creditLimit])
RETURN
IF(
    BalRatio > 0.9 && UtilPct > 0.8,
    "High Risk", "Low Risk"
)
```

**Business read:** This is the tightest of the three risk lenses — use it to prioritise the shortlist for manual review.

---

## Key Takeaways

1. Growth measures (1–4) tell you if the portfolio is expanding — utilization and revenue measures (5–8) tell you at what cost.
2. Churn and delinquency (9–10) catch clients drifting away; the risk score and correlation (11–12) explain why.
3. High-utilization, high-risk-flag, and loan-vs-limit views (8, 14, 15) triangulate the same clients from three angles — use them together, not alone.
4. Satisfaction by card tier (13) turns a risk-focused report into a retention one — pair it with the churn flag before acting.

---

**Author:** Suresh Deuja — [LinkedIn](https://www.linkedin.com/in/suresh-deuja-209305391)
