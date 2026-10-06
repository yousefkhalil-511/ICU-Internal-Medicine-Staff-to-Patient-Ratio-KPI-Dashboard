# Data Dictionary: Hospital Staff-to-Patient Ratio & Labor Analytics

## 1. Schema Architecture & Overview

The database uses a **Constellation (Galaxy) Star Schema** designed for healthcare workforce analytics, patient census tracking, and labor productivity monitoring.

* **Timeframe:** January 1, 2026 – April 30, 2026

* **Units Monitored:** Intensive Care Unit (`U_ICU`, 12-hour shift cycle) and Internal Medicine Ward (`U_INT`, 8-hour shift cycle)

* **Core Fact Tables:** `Fact_Patient_Census` and `Fact_Staffing_Actuals`

* **Shared Natural Key:** `Shift_Instance_Key` ($\text{Date} + \text{Unit\_ID} + \text{Shift\_ID}$)

```
                      +-------------------+
                      |     Dim_Date      |
                      +---------+---------+
                                |
             +------------------+------------------+
             |                                     |
             v                                     v
+------------------------+             +------------------------+
|  Fact_Patient_Census   |             | Fact_Staffing_Actuals  |
+-----------+------------+             +-----------+------------+
            |                                      |
            +------------------+-------------------+
                               |
                   [ Shift_Instance_Key ]
                               |
        +----------------------+----------------------+
        |                      |                      |
        v                      v                      v
+---------------+      +---------------+      +---------------+
|   Dim_Unit    |      |   Dim_Shift   |      |   Dim_Staff   |
+-------+-------+      +-------+-------+      +---------------+
        |                      |
        +----------+-----------+
                   |
                   v
     +---------------------------+
     |   Dim_Staffing_Targets    |
     +---------------------------+

```

## 2. Dimension Tables

### 2.1. `Dim_Unit` (Department Master)

Stores metadata for inpatient hospital departments, care levels, and physical bed capacity.

* **Grain:** 1 row per inpatient care unit

* **Primary Key:** `Unit_ID`

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Unit_ID` | `VARCHAR(10)` | No | PK | `U_ICU` | Unique identifier for the hospital department/unit. | 
| `Unit_Name` | `VARCHAR(50)` | No | None | `Intensive Care Unit` | Human-readable name of the clinical department. | 
| `Care_Level` | `VARCHAR(30)` | No | None | `Critical Care` | Clinical classification category (e.g., `Critical Care`, `Acute / General Inpatient`). | 
| `Bed_Capacity` | `INTEGER` | No | None | `18` | Total physical licensed beds available. Integer $> 0$. | 
| `Shift_Model` | `VARCHAR(20)` | No | None | `12-Hour` | Standard operational shift scheduling model (`12-Hour` or `8-Hour`). | 

### 2.2. `Dim_Shift` (Operational Shift Window)

Defines the operational shift configuration and scheduled elapsed duration.

* **Grain:** 1 row per shift configuration template

* **Primary Key:** `Shift_ID`

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Shift_ID` | `VARCHAR(10)` | No | PK | `S12_DAY` | Unique code identifying shift duration and window (e.g., `S12_DAY`, `S08_NIGHT`). | 
| `Shift_Name` | `VARCHAR(30)` | No | None | `12h Day` | Display label for operational dashboards. | 
| `Shift_Hours` | `DECIMAL(4,2)` | No | None | `12.00` | Total scheduled elapsed elapsed hours per shift ($8.00$ or $12.00$). | 
| `Start_Time` | `TIME` | No | None | `07:00:00` | Scheduled shift start time in 24-hour military notation. | 
| `End_Time` | `TIME` | No | None | `19:00:00` | Scheduled shift end time in 24-hour military notation. | 

### 2.3. `Dim_Staff` (Caregiver Master Roster)

Stores clinical identity, licensing classification, and contractual parameters for all clinical staff.

* **Grain:** 1 row per staff member

* **Primary Key:** `Staff_ID`

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Staff_ID` | `VARCHAR(15)` | No | PK | `STF_ICU_001` | Unique employee badge / payroll identifier. | 
| `Primary_Unit_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Unit.Unit_ID` | `U_ICU` | Department to which the caregiver is formally assigned. | 
| `Staff_Role` | `VARCHAR(35)` | No | None | `Registered Nurse` | Clinical role title: `Registered Nurse`, `Licensed Practical Nurse`, `Certified Nursing Assistant`. | 
| `License_Status` | `VARCHAR(15)` | No | None | `Licensed` | State clinical board licensure status: `Licensed` (RN, LPN) or `Unlicensed` (CNA/UAP). | 
| `Default_Employment_Type` | `VARCHAR(20)` | No | None | `Core Staff` | Standard contract type (`Core Staff`, `Overtime`, `Float Pool`, `Agency/Traveler`). | 
| `FTE` | `DECIMAL(3,2)` | No | None | `0.90` | Budgeted Full-Time Equivalent commitment ($0.00 \le \text{FTE} \le 1.00$). | 

### 2.4. `Dim_Staffing_Targets` (Policy & Staffing Matrix)

Defines planned staffing baselines, regulatory compliance boundaries, and skill mix benchmarks by unit and shift.

* **Grain:** 1 row per Unit and Shift pairing

* **Primary Key:** Composite (`Unit_ID`, `Shift_ID`)

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Unit_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Unit.Unit_ID` | `U_ICU` | Foreign key referencing the clinical care unit. | 
| `Shift_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Shift.Shift_ID` | `S12_DAY` | Foreign key referencing the shift schedule template. | 
| `Target_Patient_To_Staff_Ratio` | `DECIMAL(4,2)` | No | None | `2.00` | Target operational patient count per caregiver (e.g., $2.00 = 1:2$ ratio). | 
| `Target_HPPD` | `DECIMAL(4,2)` | No | None | `12.50` | Budgeted target Hours Per Patient Day. | 
| `Mandated_Max_Patient_Per_RN` | `DECIMAL(4,2)` | No | None | `2.00` | Legal threshold for maximum assigned patients per RN. Exceeding flags a violation. | 
| `Target_RN_Skill_Mix_Pct` | `DECIMAL(3,2)` | No | None | `0.90` | Planned target proportion of RN hours relative to total care hours ($0.00 \le \text{Value} \le 1.00$). | 

## 3. Fact Tables

### 3.1. `Fact_Patient_Census` (Clinical Workload Fact)

Captures patient volume, patient flow turnover (admissions and discharges), and acuity intensity per shift window.

* **Grain:** 1 row per unit per shift occurrence (`Shift_Instance_Key`)

* **Primary Key:** `Shift_Instance_Key`

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Shift_Instance_Key` | `VARCHAR(35)` | No | PK | `2026-01-01_U_ICU_S12_DAY` | Natural key joining census to staffing actuals ($\text{Date} + \text{Unit\_ID} + \text{Shift\_ID}$). | 
| `Date` | `DATE` | No | FK $\rightarrow$ `Dim_Date.Date` | `2026-01-01` | Shift calendar date. | 
| `Unit_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Unit.Unit_ID` | `U_ICU` | Unit identifier where census was captured. | 
| `Shift_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Shift.Shift_ID` | `S12_DAY` | Operational shift window identifier. | 
| `Patient_Count` | `INTEGER` | No | None | `14` | Average occupied beds / active census count ($0 \le \text{Count} \le \text{Bed\_Capacity}$). | 
| `Admissions_Count` | `INTEGER` | No | None | `2` | Patient admissions arriving during the shift window. | 
| `Discharges_Transfers_Count` | `INTEGER` | No | None | `1` | Discharges and outbound transfers completing during the shift window. | 
| `Average_Acuity_Score` | `DECIMAL(3,2)` | No | None | `4.25` | Workload/acuity score index (Standard range: $1.00$ to $5.00$). | 
| `Patient_Days` | `DECIMAL(6,3)` | No | None | `7.000` | Normalized volume metric: $(\text{Patient\_Count} \times \text{Shift\_Hours}) \div 24$. | 

### 3.2. `Fact_Staffing_Actuals` (Labor, Time & Cost Fact)

Captures granular clock-in hours, productive direct/indirect time, hourly rates, and total labor cost per scheduled staff member.

* **Grain:** 1 row per individual caregiver per shift occurrence

* **Primary Key:** Composite (`Shift_Instance_Key`, `Staff_ID`)

| Column Name | Data Type | Nullable | Key / Ref | Example Value | Description & Constraints | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| `Shift_Instance_Key` | `VARCHAR(35)` | No | FK $\rightarrow$ `Fact_Patient_Census` | `2026-01-01_U_ICU_S12_DAY` | Shift instance linkage to patient census. | 
| `Date` | `DATE` | No | FK $\rightarrow$ `Dim_Date.Date` | `2026-01-01` | Calendar date of worked shift. | 
| `Unit_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Unit.Unit_ID` | `U_ICU` | Department where the caregiver actually worked. | 
| `Shift_ID` | `VARCHAR(10)` | No | FK $\rightarrow$ `Dim_Shift.Shift_ID` | `S12_DAY` | Shift scheduled and completed. | 
| `Staff_ID` | `VARCHAR(15)` | No | FK $\rightarrow$ `Dim_Staff.Staff_ID` | `STF_ICU_001` | Unique caregiver employee identifier. | 
| `Staff_Role` | `VARCHAR(35)` | No | None | `Registered Nurse` | Role performed during this shift (`Registered Nurse`, `Licensed Practical Nurse`, `Certified Nursing Assistant`). | 
| `License_Status` | `VARCHAR(15)` | No | None | `Licensed` | Licensure state of the caregiver (`Licensed` vs. `Unlicensed`). | 
| `Employment_Category` | `VARCHAR(20)` | No | None | `Core Staff` | Pay classification code: `Core Staff`, `Overtime`, `Float Pool`, or `Agency/Traveler`. | 
| `Direct_Care_Hours` | `DECIMAL(4,2)` | No | None | `11.50` | Paid bedside direct patient care hours (Shift Hours minus lunch minus indirect duties). | 
| `Indirect_Hours` | `DECIMAL(4,2)` | No | None | `0.00` | Paid hours spent on administrative, charge nurse, or non-bedside duties. | 
| `Productive_Hours` | `DECIMAL(4,2)` | No | None | `11.50` | Total productive working hours: $\text{Direct\_Care\_Hours} + \text{Indirect\_Hours}$. | 
| `Paid_Productive_Hourly_Rate` | `DECIMAL(5,2)` | No | None | `54.20` | Base productive hourly wage. Valid boundary: $\$30.00 \le \text{Rate} \le \$60.00$. | 
| `Total_Labor_Cost` | `DECIMAL(7,2)` | No | None | `623.30` | Shift labor cost: $\text{Productive\_Hours} \times \text{Paid\_Productive\_Hourly\_Rate}$. | 

## 4. Hourly Pay Rate Reference Matrix

Hourly rates are assigned based on clinical role and professional licensure status:

| Clinical Role | License Status | Minimum Rate | Maximum Rate | Target Baseline Rationale | 
| ----- | ----- | ----- | ----- | ----- | 
| **Registered Nurse (RN)** | Licensed | $\$48.00$ | $\$60.00$ | Autonomous assessment, ICU titrations, charge duties. | 
| **Licensed Practical Nurse (LPN)** | Licensed | $\$38.00$ | $\$47.50$ | Medication passes, routine wound care, structured bedside care. | 
| **Certified Nursing Assistant (CNA)** | Unlicensed | $\$30.00$ | $\$36.50$ | Activities of daily living (ADLs), vitals, patient hygiene. | 

## 5. Core Metric Calculations & Dashboard Formulas

### 5.1. Actual Hours Per Patient Day (HPPD)

Measures the volume of bedside nursing care delivered per patient day:

$$
\text{Actual HPPD} = \frac{\sum \text{Direct\_Care\_Hours}}{\sum \text{Patient\_Days}}
$$

### 5.2. Actual Staff-to-Patient Ratio

Calculates the average number of active patients cared for by a single direct-care staff member during a shift:

$$
\text{Actual Ratio (Patients per Staff)} = \frac{\text{Patient\_Count}}{\text{Count of Active Bedside Staff}}
$$

### 5.3. RN Skill Mix Percentage

The proportion of care hours delivered by licensed Registered Nurses:

$$
\text{RN Skill Mix \%} = \frac{\sum (\text{Productive\_Hours where Staff\_Role} = \text{'Registered Nurse'})}{\sum \text{Productive\_Hours}} \times 100
$$

### 5.4. Staffing Variance Hours

Evaluates whether actual labor hours met or exceeded the budgeted standard:

$$
\text{Required Labor Hours} = \sum (\text{Patient\_Days} \times \text{Target\_HPPD})
$$

$$
\text{Staffing Variance Hours} = \sum \text{Productive\_Hours} - \text{Required Labor Hours}
$$

### 5.5. Direct Care Labor Cost per Patient Day

Calculates financial efficiency normalized by patient days:

$$
\text{Labor Cost Per Patient Day} = \frac{\sum \text{Total\_Labor\_Cost}}{\sum \text{Patient\_Days}}
$$

### 5.6. Premium Labor Exposure Rate

The percentage of productive labor hours supplied via non-standard, higher-cost staffing channels:

$$
\text{Premium Labor Rate \%} = \frac{\sum (\text{Productive\_Hours where Employment\_Category} \in \{\text{'Overtime'}, \text{'Agency/Traveler'}\})}{\sum \text{Productive\_Hours}} \times 100
$$