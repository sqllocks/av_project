
# Medallion Architecture Design

## Overview

Seven CSV files were provided as the source for the data pipeline:

- `employees.csv` (~70,098 rows)
- `departures.csv` (~31,669 rows)
- `salaries.csv` (~81,291 rows)
- `dept_emp.csv` (~126,432 rows)
- `dept_manager.csv` (~24 rows)
- `departments.csv` (~10 rows)
- `titles.csv` (~8 rows)

The files were uploaded to an **Amazon S3 bucket**. Using **FiveTran**, they were synced into a **Snowflake** database, structured by the *Medallion Architecture* (Bronze, Silver, Gold, Platinum layers).
ETL and data quality were managed with **dbt**; business reporting leveraged **Tableau** dashboards built on the Platinum layer views.

***

## Data Quality Issues

Many common data quality issues were identified and required cleaning at the Silver layer. These issues impacted data reliability and reporting requirements:


| File | Issue Type | Description | Impact | Records |
| :-- | :-- | :-- | :-- | :-- |
| employees.csv | Missing Values | Possible null or missing salary | ETL failure, inaccurate headcount or aging | 50625.0 |
| employees.csv | Missing Values | Missing Department Info | ETL failure | 42941.0 |
| employees.csv | Missing Values | Complete null row | ETL failure | 777.0 |
| employees.csv | No Location | No location, office, or geography columns | Cannot do “where” analysis in reporting |  |
| employees.csv | No Generation/Derived | Only birth_date, not generation category | Further ETL logic needed for generational analysis |  |
| salaries.csv | Orphan Records | emp_no values don’t always map to employees.csv | Incomplete compensation reporting | 62595.0 |
| salaries.csv | No Effective Date | No indication if salary is start, current, or final | Point-in-time reporting impossible |  |
| salaries.csv | Possible Outliers | Negative, zero, or extreme/unexpected salaries possible | Skewed averages, invalid compensation analysis |  |
| salaries.csv | Duplicate emp_no | Possible multiple salary rows for an employee | Ambiguous reporting |  |
| salaries.csv | Unknown Units/Currency | No field for salary units or currency | Cannot normalize/compare globally |  |
| departures.csv | No Descriptive Reason | exitreason is codes only, not mapped to business-friendly text | User confusion, poor chart labeling in reporting |  |
| departures.csv | Orphan Records | emp_no values may not exist in employees.csv | Headcount, turnover denominator errors | 17637.0 |
| departures.csv | Invalid/Null Values | Possible null or invalid values in any field | Loss in record completeness/accuracy |  |
| departures.csv | Invalid Dates | exitdate can be future or before hire | Tenure/attrition calculations may be invalid |  |
| departures.csv | Duplicate emp_no | Multiple departures for same emp_no, but with no period or reason | Over-counted separations |  |
| departures.csv | No Org Context | No department or title at time of departure | Can’t analyze attrition by org/time |  |
| dept_emp.csv | Multi-Dept Assignment | Employees can exist in multiple depts, no “primary” department flag | Double-counted headcount, ambiguity |  |
| dept_emp.csv | Orphan Records | Assignments for non-existent employees or departments | Poor referential integrity |  |
| dept_emp.csv | No Dates | No start or end date columns | Can’t track transfers or headcount over time |  |
| dept_emp.csv | Composite/Parsing | Fields may be concatenated (e.g., empno/deptno together) | Requires parsing, leads to errors |  |
| dept_emp.csv | No Audit Columns | Lack of ETL/load metadata | Hard to troubleshoot ETL issues |  |
| dept_manager.csv | No Dates | Lacks manager start/end dates | Can’t track reporting lines or org structure |  |
| dept_manager.csv | Orphan Records | Dept or emp_no may not map to reference tables | Integrity of org chart is at risk |  |
| dept_manager.csv | Multiple Managers | Could have >1 manager per dept, no context | Org reporting ambiguity |  |
| departments.csv | Name Mismatch | Department names/codes don’t match what is shown in report | Code-to-display mapping required |  |
| departments.csv | Duplicates/Nulls | Possible missing values or duplicates | Dead departments or over-counts |  |
| departments.csv | Missing Group/Type | No hierarchy or business grouping | Can’t do higher-level aggregation |  |
| departments.csv | Headers as Data | File may include header row, not filtered | ETL error, wrong record counts |  |
| titles.csv | Headers as Data | Header row present, not filtered before use | Wrong row counts, parsing errors |  |
| titles.csv | Duplicates/Nulls | Potential missing or duplicate title_id | Chart and filter misses |  |
| titles.csv | No Grouping/Levels | Lacks job family/group for reporting aggregations | Only raw title reporting supported |  |


***

## Layer Overview

| Layer | Key Tables | Row Counts | Major Processing Steps |
| :-- | :-- | :-- | :-- |
| **Bronze** | 7 base tables | 70k–126k (as loaded) | Raw ingest; no cleaning |
| **Silver** | 7 cleansed tables | 7k–69k (facts); 7–9 (dims) | Remove orphans, dedupe, parse, mapping |
| **Gold** | 5 dims + 3 facts | Dims: 7–69k; Facts: 7k–69k; Snapshots: ~12.5M | Star schema creation, keys, joins |
| **Platinum** | Views (agg/analysis) | N/A | Reporting queries for Tableau |


***

## ETL Steps by Layer

**Bronze → Silver:**

- Remove header/empty/null rows
- Standardize date fields
- Filter and deduplicate by `emp_no`
- Remove orphans (unmatched employees/departments)
- Map exit reason codes to descriptions
- Map department codes to names
- Assign primary department (if needed)
- Build audit/ETL metadata

**Silver → Gold:**

- Build deduplicated star-schema dimensions
- Assign surrogate keys
- Derive employee generation from birth year
- Generate monthly employee-by-month snapshots
- Map codes to human-readable labels
- Ensure referential integrity
- Add row versioning and audit columns

**Gold → Platinum (Reporting):**

- Aggregate and blend joins for dashboards
- Apply department and exit reason mappings
- Filter/label data to match dashboard requirements

***

## Entity-Relationship (ER) Diagrams

### Bronze Layer Model
![Bronze Layer](images/AV_Homework_Models-bronze.jpg)

### Silver Layer Model
![Silver Layer](images/AV_Homework_Models-silver.jpg)

### Gold Layer Model
![Gold Layer](images/AV_Homework_Models-gold.jpg)

***

## Medallion Architecture Flow

The process moves raw files through four distinct layers, each adding structure, quality, and analytic value:

1. **Bronze Layer:** Raw, unfiltered tables directly loaded from source files.
2. **Silver Layer:** Data cleansed, mapped, and validated.
3. **Gold Layer:** Star schema dimensions and facts for reporting, snapshot tables for trend analysis.
4. **Platinum Layer:** Business-ready views optimized for Tableau dashboards.

***

## Platinum Layer: Reporting Views

**Key output views for Tableau:**

- `vw_fluctuation_monthly_summary`:
*Headcount, hires, departures, net change, turnover rate; by month for trend/KPI cards*
- `vw_fluctuation_by_department_monthly`:
*Monthly headcount, hires, departures by department; department mapped for Tableau filters*
- `vw_departures_by_exit_reason_monthly`:
*Departure counts by month and exit reason; supports charts and filter choices*
- `vw_current_employee_snapshot`:
*Current active roster; used for filter population (department, title, generation)*
- `vw_turnover_by_department`:
*Departures, headcount, turnover % by department; drives departmental analysis*

***

## Data Dictionary \& Modeling Highlights

### Dimensions

- **Employee:** `emp_no`, name fields, sex, birth/hire dates, generation attributes, validity indicators
- **Department:** `dept_no`, name, category, status
- **Title:** `title_id`, name, category, level
- **Exit Reason:** code, description, category
- **Date:** key fields, value, day/month/year attributes, holidays


### Facts

- **Employee Snapshot:** Monthly employment status, headcount, active flags, salary states
- **Employee Hire:** New hires, starting salary, age, department/title/dates
- **Employee Departure:** Departures by reason, tenure, final salary, exit date

***

## Project Notes

- Diagrams are provided as image references for clarity and GitHub rendering.
- All transformations are fully documented in dbt scripts, and audit columns are maintained throughout each stage.
- Data modeling emphasizes referential integrity, deduplication, and readiness for business KPIs and dashboarding.
