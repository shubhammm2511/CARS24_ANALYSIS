#Cars24 Builder-Analyst Screening Case

## Part A — Answer sheet

| Question | Answer |
|---|---:|
| A1 August lead-to-inspection conversion | 24.04% |
| A2 Unique sellers | 44,701 |
| A3 Duplicate inspection rows | 219 |
| A4 GPS distance >20 km and duration <5 minutes | 352 |
| A5 Vendor-partner share of A4 | 100.00% |
| A6 Lucknow connect rate after 21 July | 24.79% |
| A7 Show-up rate when wait exceeded 48 hours | 71.34% |
| A8 Contribution from old cars in Jaipur and Lucknow | ₹-10,028,551 |
| A9 Offer rows with invalid inspection IDs | 2,293 |
| A10 Purchase rate after revision >10% | 16.04% |
| A11 Purchase rate with no revision | 69.54% |
| A12 Reported conversion-drop points not reproduced | 1.41 percentage points |

### Assumptions

- Conversion follows the raw-data definition used in the analysis notebook.
- Duplicate rows are exact duplicate inspection rows.
- Contribution equals resale price minus offer price, refurbishment cost, and tow cost.
- Positive `revision_pct` values represent downward offer revisions.
- No revision means `revision_pct = 0`.

## Part B — SQL approach

The files were loaded into DuckDB. The analysis used four queries:

1. Daily leads and deduplicated inspections by city, with a seven-day rolling average.
2. Inspector count, median duration, median GPS distance, offer rate, and anomaly flag.
3. Finance reconciliation comparing raw rows, distinct inspection IDs, distinct leads, duplicate rows, and Finance's count.
4. Weekly lead cohorts with inspection completion within 24 hours, 48 hours, and seven days.

The key SQL patterns were:

```sql
COUNT(DISTINCT inspection_id)
MEDIAN(duration_min)
MEDIAN(distance_km)
AVG(metric) OVER (
    PARTITION BY city
    ORDER BY activity_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```

Finance counts completion dates in IST and include re-inspections. Therefore, Finance should be compared with distinct `inspection_id` values, not distinct leads. Duplicate rows explain only part of the difference; the residual requires investigation into re-inspections, timezone handling, and extraction timing.

## Part C — Product and inference

Purchase rate is 16.04% after an offer revision greater than 10%, compared with 69.54% when there is no revision. This is a 53.50 percentage-point association.

This is correlation, not proof of causation. Revisions may reflect vehicle quality, documentation problems, or inspection findings that independently reduce purchase probability. I would segment the result by city, car age, revision reason, inspector, and revision size.

I would test clearer revised-offer explanations and a controlled approval flow. The test should measure purchase rate, seller drop-off, time to purchase, and contribution per purchased car.

More call attempts are associated with lower booking, but difficult leads may receive more attempts. I would not immediately cap attempts. I would first run a randomized holdout test.

## Part D — Unit economics

The Excel model includes live formulas for expected inspections, expected purchases, gross contribution, inspection cost, lead acquisition cost, contribution per 1,000 leads, break-even conversion, and a two-variable sensitivity table.

Break-even conversion:

```excel
=Cost_Per_Lead/(Purchase_Rate*Average_Contribution_Per_Car*(1-Vendor_Commission)-Inspector_Cost)
```

Under the selected assumptions, the break-even lead-to-inspection conversion is approximately 14.0%.

## Part E — Dashboard

The Power BI dashboard contains KPI cards for August conversion, Lucknow connect rate, show-up rate, revised-offer purchase rate, and old-car contribution. It also contains city-level mapping, a funnel-rate bar chart, daily conversion and rolling-average line chart, and offer-revision comparison.

Alert thresholds:

| Alert | Threshold | Owner |
|---|---:|---|
| Conversion | Below 30% | City heads / Supply |
| Lucknow connect rate | Below 35% | Call operations |
| Long-wait show-up | Below 70% | Appointment operations |
| Revised-offer purchase rate | Below 20% | Pricing / Product |
| Old-car contribution | Below ₹0 | Finance / City head |
| GPS anomalies | Above 300 | Inspector operations |
| Invalid inspection IDs | Above 0 | Data Engineering |

If leads fail, conversion and CAC are unavailable. If inspections fail, inspection counts and reconciliation are unavailable. If call logs fail, connect-rate monitoring is unavailable. If Finance fails, the operational count should be labelled provisional.

## Part F — Action plan

### 7 days

1. Finance and Data Engineering reconcile operational and Finance inspection definitions.
2. Inspector Operations investigate GPS and short-duration anomalies.
3. Call Operations review the Lucknow calling process.

### 30 days

1. Pricing and Product test revised-offer explanations.
2. Appointment Operations reduce lead-to-slot wait time.
3. Supply Operations rebalance inspector capacity by city.

### 90 days

1. Marketing and Finance move from raw-lead CAC to inspected-lead CAC.
2. Data Engineering automate source-quality alerts.
3. Supply Operations optimize inspector deployment using capacity, quality, and contribution.

I would not immediately cap call attempts because the relationship between attempts and booking is observational.

## Part G — AI log

Tools used: Python, pandas, DuckDB, Excel, Power BI, and ChatGPT.

Prompt 1: “Profile all files, identify inconsistent date formats, duplicate keys, missing joins, and timezone issues.”

Prompt 2: “Write reproducible SQL for daily conversion, inspector anomalies, Finance reconciliation, and lead cohorts.”

AI error caught: an initial calculation treated positive `revision_pct` values as upward revisions. I checked the raw values and corrected the logic to treat values greater than 10 as downward revisions.

I separately validated duplicate rows, unique sellers, and unique inspected leads. The main lesson was that AI accelerated the work, but all headline numbers required checks against raw rows, unique IDs, dates, and business definitions.
