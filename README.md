# 🏦 US Insurance Analytics Dashboard — Power BI

The dashboard delivers end-to-end insurance analytics across premium performance, agent productivity, claims analysis, and carrier profitability — built on a cleaned, fully modelled dataset of 6,000+ policies, 3,500 claims, and 360 months of carrier performance data.

---

## 📊 Dashboard Pages

### Page 1 — Executive Summary & Sales Performance
- **KPI Strip**: Total Written Premium ($50.5M), Active Policies (3.6K), Loss Ratio (1.8%), Combined Ratio (31.5%), Premium Growth YoY (49.2%)
- **Premium Trend Chart**: Current year (solid line) vs prior year (dashed line) with MoM % change tooltips and a What-If target reference line
- **Agent Performance Matrix**: Written Premium, Policy Count, and Commission Earned by Agent × Line of Business with data bar conditional formatting
- **Line of Business Donut Chart**: Premium split with drill-through to policy detail

### Page 2 — Claims & Carrier Performance
- **Loss Ratio Heatmap**: Carrier × Month matrix with RAG status conditional formatting (Green / Amber / Red)
- **Claims Analysis**: Total claims paid by claim type with average days-to-close as a secondary axis
- **Carrier Scatter Plot**: Loss Ratio vs Expense Ratio, bubble-sized by Gross Written Premium, with 100% combined ratio reference line
- **Customer Segment Table**: Policy Count, Claims Frequency, Avg Premium, Loss Ratio, and CLV Tier mix by customer segment

---

## 🗂️ Dataset

| Table | Rows | Description |
|---|---|---|
| Policies | 6,000 | Policy-level data across all lines of business |
| Claims | 3,500 | Individual claims linked to policies |
| Customers | 3,000 | Customer demographics and segmentation |
| Agents | 50 | Agent profile, region, commission, and performance |
| CarrierPerformance | 360 | Monthly carrier-level premium, loss, and expense data |

> All tables contained intentional data quality issues (inconsistent casing, typos, nulls, blank foreign keys) that were resolved in Power Query before any visuals or DAX were built.

---

## 🔧 Data Cleaning (Power Query)

- **PolicyStatus, ClaimStatus, Gender, LineOfBusiness**: Standardised using Replace Values with Match Entire Cell Contents — eliminated typos like "Actve", "ACTIVE", "Commerical" down to clean, consistent values
- **Age**: Null values flagged with a custom boolean column (1 = missing, 0 = present) rather than dropped — preserving record completeness while making gaps visible and filterable. Data type corrected from float to whole number
- **LossAmount / LossRatio**: Loss Ratio without a corresponding Loss Amount treated as a genuine missing value — not defaulted to zero to avoid silently understating loss experience
- **GrossWrittenPremium**: Null values flagged for correction against known carrier WrittenPremium figures rather than treated as true nulls
- **DeductibleAmount**: Nulls replaced with 0 — no deductible reasonably means zero deductible in this context
- **Date columns**: ClaimDate, CloseDate, EffectiveDate, ExpirationDate corrected from text/generic types to proper date columns for time intelligence compatibility
- **CarrierPerformance YearMonth**: Transformed from "2023-06" text format to a proper date column
- **Date Table**: Built to span the full 2022–2024 range with Year, Quarter, Month Number, Month Name, Week Number, Day of Week, and IsWeekend flag — marked as official Date Table so TOTALYTD, SAMEPERIODLASTYEAR, and other time intelligence functions resolve correctly

---

## 🏗️ Data Model

**Star schema** with three fact tables and surrounding dimensions:

```
Policies (Fact) ──── PolicyID ────► Claims (Fact)
     │                                    │
     ├──► Customers (Dim)                 ├──► Customers (Dim)
     ├──► Agents (Dim)                    ├──► Agents (Dim)
     └──► Date (Dim)                      └──► Date (Dim)

CarrierPerformance (Fact) ──► DimCarrier (Dim)
                          └──► Date (Dim)

[Disconnected] LossRatioTarget — drives What-If parameter
[Disconnected] TopNTable — drives dynamic Top N agent slicer
```

- All relationships are **single-directional** except the Date table, where bidirectional filtering was enabled only where visuals genuinely required it
- CarrierPerformance is kept at carrier-month grain — linked through DimCarrier rather than joined back into the policy-level tables to avoid a many-to-many relationship and keep ratio calculations clean
- Disconnected tables sit outside the star schema intentionally — they drive measure logic without filtering fact tables

---

## 📐 DAX Measures

### Core Measures
```dax
Total Written Premium = SUM(Policies[WrittenPremium])
Total Earned Premium = SUM(CarrierPerformance[EarnedPremium])
Total Claims Paid = SUM(Claims[PaidAmount])
Total Reserved Amount = SUM(Claims[ReservedAmount])
Policy Count = COUNTROWS(Policies)
Active Policy Count = CALCULATE(COUNTROWS(Policies), Policies[PolicyStatus] = "Active")

Loss Ratio = DIVIDE([Total Claims Paid], [Total Earned Premium])
Expense Ratio = DIVIDE(SUM(CarrierPerformance[ExpenseAmount]), [Total Earned Premium])
Combined Ratio = [Loss Ratio] + [Expense Ratio]
Claims Frequency = DIVIDE(COUNTROWS(Claims), [Active Policy Count])
```

### RAG Status (Conditional Formatting)
```dax
Loss Ratio RAG Status =
    IF([Loss Ratio] < 0.60, "Green",
    IF([Loss Ratio] <= 0.75, "Amber", "Red"))
```

### Time Intelligence
```dax
Written Premium YTD = TOTALYTD([Total Written Premium], Date[Date])
Written Premium PYTD = CALCULATE([Total Written Premium], SAMEPERIODLASTYEAR(Date[Date]))
Written Premium YoY % =
    DIVIDE([Written Premium YTD] - [Written Premium PYTD], [Written Premium PYTD])
Written Premium MoM % =
    VAR CurrentMonth = [Total Written Premium]
    VAR PriorMonth = CALCULATE([Total Written Premium], DATEADD(Date[Date], -1, MONTH))
    RETURN DIVIDE(CurrentMonth - PriorMonth, PriorMonth)
Rolling 3M Avg Premium =
    AVERAGEX(DATESINPERIOD(Date[Date], LASTDATE(Date[Date]), -3, MONTH),
             [Total Written Premium])
Rolling 12M Claims =
    CALCULATE([Total Claims Paid], DATESINPERIOD(Date[Date], LASTDATE(Date[Date]), -12, MONTH))
```

### Advanced
- **What-If Parameter**: Loss Ratio Target (50%–90% in 1% increments) — drives "Premium Required to Hit Target" scenario measure on the trend chart
- **Agent Ranking**: RANKX within Region by Commission Earned
- **Top N Slicer**: Disconnected TopNTable (values: 1, 5, 10, 20, All) — dynamically filters agent matrix via RANKX interaction
- **CLV Tier**: Calculated column bucketing customers into Platinum / Gold / Silver / Bronze based on Customer Lifetime Value
- **Retention Rate**: Derived from RenewalFlag column — renewed vs cancelled policy split
- **Policies at Risk**: Active policies with ExpirationDate within 30 days of the latest date in the dataset

---

## 🔐 Row-Level Security (RLS)

Three-tier access model implemented via a separate security table (not hardcoded into the Agents dimension):

| Role | Access |
|---|---|
| Agent | Own policies and claims only (filtered by AgentID) |
| Regional Manager | All agents and policies within their region |
| Admin | Full dataset — no filter applied |

Tested in Power BI Desktop using **View As Role** — switching between a sample agent, regional manager, and Admin to confirm visuals narrow and expand correctly on both pages before publishing.

---

## 🛠️ Tools & Skills

- **Power BI Desktop** — dashboard design, data modelling, RLS
- **Power Query (M)** — data cleaning, type corrections, date table generation, text standardisation
- **DAX** — core measures, time intelligence, What-If parameters, RANKX, disconnected tables, calculated columns
- **Star Schema Design** — fact/dimension separation, cardinality management, bidirectional filter control
- **SQL** — data exploration and pre-analysis
- **Python (Pandas, NumPy)** — data profiling and EDA prior to Power BI import

---

## 📁 Files

| File | Description |
|---|---|
| `assignment_report.pbix` | Full Power BI dashboard file |
| `Insurance_Dataset.xlsx` | Source dataset (5 tables) |
| `Assignment_Write-Up.pdf` | Data model design decisions, cleaning assumptions, RLS approach |

---

## 👤 Author

**Anudeep Sambaraju**
Full-Stack Developer & Data Analyst | AWS Certified
[Portfolio](https://anudeepsambaraju.com) · [LinkedIn](https://linkedin.com/in/anudeep-sambaraju) · [GitHub](https://github.com/AnudeepZero)
