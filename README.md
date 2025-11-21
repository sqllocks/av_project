
```markdown
```
# Medallion Architecture Design

-7 files were given as the SOURCE
-They where then uploaded to an Amazon S3 bucket
-FiveTran was used to sync the files into a Snowflake database using the medallion layout (Bronze, Silver, Gold, Platinum)
-dbt was used to move the data between layers and perform all needed data quality and cleaning.
-The platinum views expose the data for the final Tableau report

## ER Diagrams

## Architecture Diagram

┌─────────────────────────────────────────────────────────────────────────────┐
│                 SOURCE FILES (CSV as provided)                              │
│ ─────────────────────────────────────────────────────────────────────────── │
│  employees.csv      (~70,098 rows)                                          │
│  departures.csv     (~31,669 rows)                                          │
│  salaries.csv       (~81,291 rows)                                          │
│  dept_emp.csv       (~126,432 rows)                                         │
│  dept_manager.csv   (~24 rows)                                              │
│  departments.csv    (~10 rows)                                              │
│  titles.csv         (~8 rows)                                               │
└───────────────────────────────────────────────────┬─────────────────────────┘
                               			  			        │
(No changes. All data is loaded, including    			│
headers, blanks, orphaned and malformed rows.)			│
														                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          BRONZE LAYER (RAW)                                 │
│                   [STAGE 1: RAW INGEST - Landing zone]                      │
│ ─────────────────────────────────────────────────────────────────────────── │
│  bronze_employees      (70,098 rows)                                        │
│  bronze_departures     (31,669 rows)                                        │
│  bronze_salaries       (81,291 rows)                                        │
│  bronze_dept_emp       (126,432 rows)                                       │
│  bronze_dept_manager   (24 rows)                                            │
│  bronze_departments    (10 rows)                                            │
│  bronze_titles         (8 rows)                                             │
└──────────────────────────────────────────────────────┬──────────────────────┘
                                                       │
│   (ETL STEPS to Silver)                              │
│   - Remove header/empty/null rows                    │
│   - Parse and standardize all date fields            │
│   - Filter/de-duplicate based on emp_no integrity    │
│   - Remove orphans (no matching emp_no/dept)         │
│   - Map exitreason codes to descriptions             │
│   - Map dept codes to report-friendly names          │
│   - Assign primary dept (if needed)                  │
│   - Build audit/ETL metadata                         │
													                             ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SILVER LAYER (CLEANSED + CONFORMED)                   │
│                    [STAGE 2: CLEANSING, VALIDATION, MAPPING]                │
│ ─────────────────────────────────────────────────────────────────────────── │
│  silver_employees       (~69,321 rows)         (-777 nulls and errors)      │
│  silver_departures      (~13,881 rows)         (matched/valid only)         │
│  silver_salaries        (~18,696 rows)         (matched/valid only)         │
│  silver_dept_emp        (~38,413 rows)         (matched/valid only)         │
│  silver_departments     (9 rows)               (cleaned, mapped)            │
│  silver_titles          (7 rows)               (cleaned, mapped)            │
│  exit_reason_mapping    (~8 rows)              (custom, mapped)             │
└──────────────────────────────────────────────────────┬──────────────────────┘
│   (ETL STEPS to Gold)                                │
│   - Build star-schema dimensions (deduplicated)      │
│   - Assign surrogate keys (for dims/facts)           │
│   - Derive ‘generation’ from birth_year              │
│   - Generate monthly employee×month snapshots        │
│   - Join all mapped codes to human-readable labels   │
│   - Ensure referential integrity                     │
│   - Add audit/row versioning columns                 │
													                             ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                       		GOLD LAYER 									                     │	
│					(STAR SCHEMA | READY FOR REPORTING)       				                 │
│   [STAGE 3: DIMENSIONAL MODEL CREATION, SNAPSHOTS, DERIVED ATTRIBUTES]     │
│ ───────────────────────────────────────────────────────────────────────────│
│ DIMENSIONS:                                                                │
│  - dim_employee        (~69,321 rows)                                      │
│  - dim_department      (9 rows)                                      		   │
│  - dim_title           (7 rows)                                     	  	 │
│  - dim_exit_reason     (8 rows)                                     	  	 │
│  - dim_date            (~11,688 rows)                                   	 │
│                                                                            │
│ FACTS:                                                                     │
│  - fact_employee_snapshot  (~12,477,780 rows)                              │
│      (monthly status for each employee over the reporting range)           │
│  - fact_employee_hire         (~69,321 rows)                               │
│  - fact_employee_departure    (~13,881 rows)                               │
│                                                                            │
│ KEY OUTPUT VIEWS:                                                          │
│  - vw_fluctuation_report_monthly  (aggregates by month, dept, reason, etc.)│
└──────────────────────────────────────────────────────┬─────────────────────┘
│   (TO REPORTING: Tabled views for Tableau)           │
│   - Use only mapped departments and exit reasons     │
│   - Ensure filters/labels match dashboard            │									
													                             ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    PLATINUM LAYER: Reporting Views                 		    │
│  (Business-Ready Aggregates & Drilldowns for Tableau Consumption)  		    │
└──────────────────────────────────────────────────────┬────────────────────┘
		(Aggregations, blend joins, label exposure)	       │   
													                             ▼
┌────────────────────────────────────────────────────────────────────────-──┐
│                        vw_fluctuation_monthly_summary                     │
│        - Headcount, hires, departures, net change, turnover rate  		    │
│        - By month (supports trend and KPI cards in Tableau)       		    │
│ 																			                                    │ 
│                        vw_fluctuation_by_department_monthly           		│
│        - Monthly headcount, hires, departures by department       	    	│
│        - Department label mapped for Tableau legends/filters      		    │
│																		                                      	│
│                        vw_departures_by_exit_reason_monthly       		    │
│        - Departure counts by month and human-readable exit reason 		    │
│        - Drives pie/bar charts and exit reason filter             		    │
│																			                                      │
│                        vw_current_employee_snapshot              			    │
│        - Current (latest period) active employee roster           		    │
│        - For filter population (department, title, generation)    		    │
│																			                                      │
│                        vw_turnover_by_department                  		    │
│        - Total departures, current headcount, turnover % by dept  		    │
│        - For department-level analysis/bar charts in the report   		    │
└───────────────────────────────────────────────────────────────────────────┘


