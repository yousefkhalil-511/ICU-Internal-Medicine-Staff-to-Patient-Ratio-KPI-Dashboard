# ICU & Internal Medicine: Staff-to-Patient Ratio KPI Dashboard

A Power BI report that measures how nursing staff resources are used and whether they match patient demand in two inpatient units: the **Intensive Care Unit (ICU)** and the **Internal Medicine Ward**. It compares actual staffing against unit targets and regulatory RN limits, and adds workload, skill-mix and labor-cost views.

- **Tool:** Power BI Desktop
- **Period covered:** 1 Jan 2026 to 30 Apr 2026 (120 days)
- **Canvas:** 16:9, 1920 × 1080 px
- **Report pages:** 6 (5 analysis pages + 1 drill-through)

---

## 1. Business questions

1. Are we staffed to the target patient-to-caregiver ratio and the target hours per patient day (HPPD)?
2. How often does RN staffing exceed the mandated maximum patients per RN?
3. Which shifts, weekdays and units are most often understaffed?
4. Does staffing respond to census and acuity?
5. How much of the labor cost and hours come from premium labor (overtime, agency/traveler, float pool)?

---

## 2. Dataset

| File | Type | Grain | Rows | Key |
|---|---|---|---|---|
| `Dim_Unit.csv` | Dimension | One row per unit | 2 | Unit_ID |
| `Dim_Shift.csv` | Dimension | One row per shift template | 5 | Shift_ID |
| `Dim_Staff.csv` | Dimension | One row per employee | 99 | Staff_ID |
| `Dim_Staffing_Targets.csv` | Dimension (targets) | One row per unit × shift | 5 | Unit_ID + Shift_ID |
| `Fact_Patient_Census.csv` | Fact | One row per unit per shift per day | 600 | Shift_Instance_Key |
| `Fact_Staffing_Actuals.csv` | Fact | One row per staff member per shift | 4,379 | Shift_Instance_Key + Staff_ID |
| `Data_Dictionary.docx` | Documentation | n/a | n/a | n/a |

**Units and shifts**

| Unit | Beds | Shift model | Shifts |
|---|---|---|---|
| ICU (`U_ICU`) | 18 | 12-hour | S12_DAY, S12_NIGHT |
| Internal Medicine (`U_INT`) | 48 | 8-hour | S08_DAY, S08_EVE, S08_NIGHT |

`Shift_Instance_Key` = Date + Unit_ID + Shift_ID (for example `2026-01-01_U_ICU_S12_DAY`). It links the two fact tables.

### Differences between the files and the data dictionary

| Item | Data dictionary | Actual files |
|---|---|---|
| Fact_Staffing_Actuals hours | `Indirect_Hours`, `Total_Paid_Hours` | `Productive_Hours` (= Direct + Indirect, verified) |
| Fact_Staffing_Actuals extra columns | not listed | `License_Status`, `Paid_Productive_Hourly_Rate`, `Total_Labor_Cost` |
| Dim_Staff | no License_Status | `License_Status` present |
| Dim_Date | listed as a conforming dimension | not supplied, created in the model |

The cost columns are what make the labor-cost page possible.

---

## 3. Data model

A constellation (galaxy) schema: two fact tables sharing conformed dimensions through a bridge table.

| Table | Role |
|---|---|
| `Dim_Date` | Calendar (created in the model, marked as date table) |
| `Dim_Unit` | Unit attributes and bed capacity |
| `Dim_Shift` | Shift templates (sorted chronologically) |
| `Dim_Staff` | Employee attributes (FTE, license, default employment type) |
| `Dim_Staffing_Targets` | Target ratio, target HPPD, mandated max patients per RN, target RN skill mix |
| `Dim_Shift_Instance` (bridge) | One row per `Shift_Instance_Key`, created in Power Query from the census table |
| `Fact_Patient_Census` | Patients, admissions, discharges, acuity, patient days |
| `Fact_Staffing_Actuals` | Staff on duty, role, employment category, hours, cost |

**Relationships** (single direction, one-to-many)

- `Dim_Shift_Instance` → `Fact_Patient_Census` on Shift_Instance_Key
- `Dim_Shift_Instance` → `Fact_Staffing_Actuals` on Shift_Instance_Key
- `Dim_Date`, `Dim_Unit`, `Dim_Shift` → `Dim_Shift_Instance`
- `Dim_Staff` → `Fact_Staffing_Actuals` on Staff_ID
- `Dim_Staff[Primary_Unit_ID]` is deliberately not related to `Dim_Unit` (it would create an ambiguous path).

> **TODO (complete before publishing):** describe how `Dim_Staffing_Targets` is connected in this build (for example merged into the bridge on Unit_ID + Shift_ID in Power Query, or related another way). The separate `Dim_Unit_Shift` table from the original plan was not created.

**Why a bridge table?** The facts have different grains (one census row versus many staff rows per shift). The bridge gives both facts one shared shift-level key, so patients and staff for the same shift can be compared shift by shift (needed for the compliance and understaffing measures) and slicers filter both facts together.

---

## 4. Measures

All measures are in the `_Measures` table. Aggregated ratios are always a **ratio of sums**, never an average of shift-level ratios. Target measures are built **row by row, then aggregated**, so they stay correct for any mix of units and shifts.

### Workload
| Measure | Logic |
|---|---|
| Patient Count | Sum of Patient_Count |
| Patient Days | Sum of Patient_Days (Patient_Count × Shift_Hours ÷ 24) |
| Shifts | Row count of the bridge table |
| Avg Census | Patient Count ÷ Shifts |
| Occupancy % | Patient Count ÷ sum of bed capacity per shift row |
| Avg Acuity (wtd) | Acuity weighted by patient count |
| Turnover Index | (Admissions + Discharges/Transfers) ÷ Patient Count |

### Staffing ratio
| Measure | Logic |
|---|---|
| Staff Shifts | Count of staff-shift rows (one caregiver working one shift) |
| RN Shifts | Staff Shifts where role is Registered Nurse |
| Actual Patients per Staff | Patient Count ÷ Staff Shifts (compare to target ratio) |
| Actual Patients per RN | Patient Count ÷ RN Shifts (compare to RN mandate) |
| Required Staff (Target) | Per shift: Patient_Count ÷ target ratio, summed |
| Target Patients per Staff | Patient Count ÷ Required Staff (blended target) |
| Staffing Variance (FTE-shifts) | Staff Shifts − Required Staff (negative means understaffed) |
| Ratio Variance % | (Actual − Target) ÷ Target |
| Patients per Staff 7D | 7-day moving average of Actual Patients per Staff |

### Hours and skill mix
| Measure | Logic |
|---|---|
| Direct Hours | Sum of Direct_Care_Hours |
| Actual HPPD | Direct Hours ÷ Patient Days |
| Target Hours / Target HPPD | Per shift: Patient_Days × target HPPD, summed, then ÷ Patient Days |
| HPPD Variance | Actual HPPD − Target HPPD |
| RN Skill Mix % | RN direct hours ÷ all direct hours |
| Target Skill Mix % | Direct-hours-weighted target skill mix |

### Compliance
| Measure | Logic |
|---|---|
| Mandated Max Patients per RN (blended) | Patient Count ÷ (sum of Patient_Count ÷ mandated max, per shift) |
| RN Mandate Breach Shifts | Shifts where patients per RN exceed the mandate, or where no RN is on shift |
| RN Mandate Compliance % | 1 − Breach Shifts ÷ Shifts |
| Excess over Mandate | Actual Patients per RN − blended mandate |
| Understaffed Shifts / Understaffed % | Shifts where the all-caregiver ratio exceeds target |

### Labor cost
| Measure | Logic |
|---|---|
| Total Labor Cost | Sum of Total_Labor_Cost |
| Cost per Patient Day | Total Labor Cost ÷ Patient Days |
| Cost per Direct Hour | Total Labor Cost ÷ Direct Hours |
| Premium Labor % | Cost share of Overtime, Agency/Traveler and Float Pool |
| Agency % | Hours share of Agency/Traveler staff |

### Shift detail (Page 6)
| Measure | Logic |
|---|---|
| Shift Status | "No RN on shift", "Over RN mandate", "Above target ratio" or "Within standard" |
| Shift Detail Title | Dynamic title built from the drilled-through Shift_Instance_Key |

**Time intelligence:** only the 7-day moving average is used. Prior-period, to-date, cumulative and year-over-year measures were not needed.

---

## 5. Report design

### Canvas and conventions
- 16:9, 1920 × 1080 px.
- **Week convention:** all weekday analysis uses **Day Name** (Mon to Sun, sorted by day-of-week number).
- **Colours:** blue = actual, grey = target, red = breach or understaffed, amber = overstaffed or above target.
- **No small multiples.** ICU and Internal Medicine have very different scales (ratio about 2 versus 4.5 to 6, acuity about 4.1 versus 2.2). Use the **Unit** slicer to view one unit at a time. Select a single unit when reading ratio charts, because a combined view blends two incomparable scales.

### Analysis page layout (Pages 1 to 5)
| Zone | Content |
|---|---|
| Top left | Page name |
| Top right | Page navigator buttons |
| Under the navigator | Date slicer (between) |
| Middle left | Unit, Shift and Day Type slicers |
| Rest of canvas | Charts |

### Shift detail page layout (Page 6)
| Zone | Content |
|---|---|
| Top | Page name (left), page navigator buttons (right) |
| Row 1 | Dynamic title |
| Row 2 | Six KPI cards |
| Left middle | Shift Status card and role composition bar chart |
| Rest of canvas | Staff roster table |

Page 6 has **no slicers** and no context comparison chart. It is a hidden drill-through page.

---

## 6. Pages

### Page 1: Executive overview
*Purpose: are we staffed to standard, and what does it cost?*

| Visual | Type | Why |
|---|---|---|
| Six KPI cards: Patients per Staff, HPPD, RN Mandate Compliance %, Understaffed %, Occupancy %, Cost per Patient Day | Cards | Fast scan, each value shown with its target |
| Patients per Staff: Actual vs Target | Line chart | Shows drift and the gap to target |
| Ratio Variance % by Unit and Shift | Diverging bar | Only measure comparable across all five unit-shifts |

### Page 2: Staffing adequacy
*Purpose: find where and when staffing misses the standard.*

| Visual | Type | Why |
|---|---|---|
| Actual vs Target vs Mandate: Patients per Caregiver, by Shift | Combo (columns + markers) | Shows all three standards side by side |
| Understaffed Shift Rate by Weekday and Shift | Matrix heatmap | Exposes weekday and shift patterns |
| Census vs Staff on Duty | Scatter (acuity as size) | Tests whether staffing follows demand |
| 7-Day Moving Average: Patients per Staff | Line chart | Smooths daily noise |

### Page 3: Compliance and skill mix
*Purpose: regulatory exposure and RN presence.*

| Visual | Type | Why |
|---|---|---|
| RN Mandate Compliance % by Unit and Shift | Bar | Pass/fail share on a 0 to 100% axis |
| RN Mandate Breaches over time | Stacked column + line | Is compliance improving or worsening? |
| RN Skill Mix vs Target | Combo (columns + markers) | Target is an hours proportion |
| Top 15 Shifts Furthest Over the RN Mandate | Table (drill-through source) | Lists the specific shifts to investigate |

The table uses **Shift_Instance_Key** (not Date) so that drill-through passes exactly one shift to Page 6.

### Page 4: Workload and capacity
*Purpose: the demand side, so staffing gaps are read in context.*

| Visual | Type | Why |
|---|---|---|
| Occupancy % over time | Line | Main demand driver |
| Admissions, Discharges and Turnover Index | Combo | Churn creates workload that census alone hides |
| Average Patient Acuity by Unit | Line (axis fixed 1 to 5) | Honest scale for the ICU vs ward gap |
| Acuity vs HPPD Variance | Scatter | Do sicker shifts get fewer hours than planned? |

### Page 5: Labor mix and cost
*Purpose: what drives cost and reliance on premium labor.*

| Visual | Type | Why |
|---|---|---|
| Direct Care Hours by Employment Category | 100% stacked column | Mix shift independent of volume |
| Cost per Patient Day by Unit | Clustered column | Normalised, so the larger unit is comparable |
| Premium Labor % and Agency % Trend | Line | Early warning of core-staff gaps |
| Role Mix by Unit | 100% stacked bar | Links cost to skill mix |

### Page 6: Shift detail (drill-through)
*Purpose: root cause for one shift.*

- **Drill-through field:** `Shift_Instance_Key`, with Keep all filters = Off.
- **Reached from:** the Page 3 table (right-click a row → Drill through → Shift Detail).
- **Contents:** dynamic title, six KPI cards, Shift Status card, role composition bar, staff roster table (Staff_ID, role, employment category, direct and indirect hours, hourly rate, cost).
- The page is hidden from the tab bar and reachable only through drill-through.

---

## 7. Key definitions and assumptions

1. **Two different ratio comparisons.** All-caregiver patients per staff is compared with the target ratio. RN-only patients per RN is compared with the mandated maximum. Never mix them.
2. **Headcount, not FTE.** Each staff-shift row counts as one caregiver.
3. **Per-shift compliance.** Breaches and understaffing are evaluated shift by shift, not on averages.
4. **Zero-RN shifts** count as breaches (the ratio is undefined but non-compliant). One such shift exists in the data.
5. **RN mandate definition.** The RN-only count follows the data dictionary. If policy allows LPNs to count, change the RN filter in RN Shifts to licensed staff (RN + LPN) and re-check the compliance page.
6. **Direct hours** (not productive hours) are the HPPD numerator, as the dictionary specifies.
7. **No budget data.** Only actual cost exists, so there is no budget-versus-actual cost view.
8. Do not slice the ratio measures by `Dim_Staff[Staff_Role]`: it filters only the staffing fact, not census. Use the role-specific measures instead.

---

## 8. Data quality checks performed

- No orphan keys between census and staffing, in either direction.
- No duplicate Shift_Instance_Key, and no duplicate (shift, staff) rows.
- Every staff ID exists in Dim_Staff; role and unit on the fact rows match Dim_Staff.
- `Productive_Hours` = Direct + Indirect, and `Total_Labor_Cost` = hours × rate, on all rows.
- `Patient_Days` = Patient_Count × Shift_Hours ÷ 24 on all rows.
- No census exceeds bed capacity; no null values in any file.
- Every staff member worked only in their primary unit, so the data contains no cross-unit floating.

### Reference values for validating the report (full period, no filters)

| Unit / Shift | Patients per Staff (all caregivers) | Patients per RN | RN mandate breach shifts (of 120) |
|---|---|---|---|
| ICU Day | ≈ 2.07 (target 2.0) | ≈ 2.48 (max 2.0) | 97 |
| ICU Night | ≈ 2.03 (target 2.0) | ≈ 2.41 (max 2.0) | 97 |
| Internal Medicine Day | ≈ 4.54 (target 4.5) | ≈ 8.27 (max 5.0) | 120 |
| Internal Medicine Evening | ≈ 5.03 (target 5.0) | ≈ 9.27 (max 6.0) | 109 |
| Internal Medicine Night | ≈ 6.08 (target 6.0) | ≈ 11.31 (max 7.0) | 105 |

These are averages of shift-level values, so aggregated visuals (ratio of sums) will be slightly different. Breach counts use the RN-only definition and will change if the definition in section 7 is changed.

Other checks: ICU occupancy ≈ 75% (13.5 average census of 18 beds), Internal Medicine ≈ 82% (39.6 of 48).

---

## 9. Limitations

- Four months of data, so seasonality cannot be assessed and year-over-year comparison is not possible.
- Each weekday and shift cell in the heatmap holds only about 17 shifts, so single cells should be read with caution.
- `Patient_Count` is an average or active census per shift, not a per-patient assignment, so true assignment-level ratios cannot be calculated.
- Cost reflects productive hours only, as supplied.

---

## 10. Using the report

1. Open the `.pbix` file in Power BI Desktop and refresh. If the CSV folder has moved, update the file path in Power Query (Transform data → Data source settings).
2. Use the date slicer first, then Unit, Shift and Day Type.
3. Read ratio visuals with a single unit selected.
4. To investigate a breach, go to Page 3, right-click a row in the table, and choose Drill through → Shift Detail.

---

## 11. Repository layout

```
/data            CSV source files and Data_Dictionary.docx
/report          Power BI report (.pbix)
README.md        This file
data dictionary  Data Dictionary.md  
```
