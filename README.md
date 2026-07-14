# Fund & Benchmark Data Warehouse on Azure

## Project Overview
Production-grade data warehouse pipeline built on Azure, demonstrating SCD Type 2 implementation, modern ETL orchestration, and CI/CD deployment.

## Architecture
Azure Data Lake Storage Gen2 (CSV files) → Azure Data Factory (Copy Data + Stored Procedure) → Azure SQL Database (Serverless) → Star Schema with SCD Type 2

## Key Features

### Slowly Changing Dimensions
- SCD Type 2 for Fund Dimension - tracks historical name/type changes with effective date ranges
- SCD Type 1 for Benchmark Dimension - current state only

### Data Pipeline
- 4 CSV files ingested from Azure Data Lake Storage Gen2
- ADF Copy Data activities load staging tables
- Stored Procedure executes all transformations in T-SQL
- Idempotent design with duplicate prevention

### Data Quality
- NULL foreign key detection
- Duplicate primary key checks
- Audit logging for all ETL steps
- Row count reconciliation

### CI/CD
- Azure DevOps Git integration
- ARM template generation for infrastructure-as-code
- One-click deployment to any Azure environment

## Technology Stack
| Service | Purpose |
|---------|---------|
| Azure Data Lake Storage Gen2 | Raw data storage |
| Azure Data Factory | ETL orchestration |
| Azure SQL Database (Serverless) | Data warehouse |
| Azure DevOps | Version control & CI/CD |
| T-SQL | All transformation logic |

## Repository Structure
- dataset/ - 8 dataset definitions (4 source + 4 sink)
- factory/ - ADF factory configuration
- linkedService/ - 2 linked services (ADLS + Azure SQL DB)
- pipeline/ - Fund_ETL_Pipeline definition
- publish_config.json
- README.md

## Pipeline Flow
1. Copy_FundMaster - Loads fund dimension data into stg.fund_master
2. Copy_BenchmarkMaster - Loads benchmark dimension data into stg.benchmark_master
3. Copy_FundPerf - Loads fund performance data into stg.fund_perf
4. Copy_BenchmarkPerf - Loads benchmark performance data into stg.benchmark_perf
5. Run_DW_Pipeline - Executes stored procedure for SCD logic and fact loading

## SCD Type 2 Implementation
```sql
UPDATE dw.dim_fund SET eff_end_dt = DATEADD(DAY,-1,GETDATE()), curr_flg = 'N'
FROM stg.fund_master s
WHERE dw.dim_fund.src_fund_id = s.fund_id
  AND dw.dim_fund.fund_name <> s.fund_name;

INSERT INTO dw.dim_fund (src_fund_id, fund_name, fund_type, eff_start_dt, eff_end_dt, curr_flg)
SELECT s.fund_id, s.fund_name, s.fund_type, GETDATE(), '9999-12-31', 'Y'
FROM stg.fund_master s
LEFT JOIN dw.dim_fund d ON s.fund_id = d.src_fund_id AND d.curr_flg = 'Y'
WHERE d.dim_fund_key IS NULL;



Fact Table Surrogate Key Lookup
INSERT INTO dw.fact_fund_perf (date_key, dim_fund_key, nav, return_pct, aum)
SELECT s.nav_date, d.dim_fund_key, s.nav, s.return_pct, s.aum
FROM stg.fund_perf s
INNER JOIN dw.dim_fund d ON s.fund_id = d.src_fund_id
  AND s.nav_date BETWEEN d.eff_start_dt AND d.eff_end_dt;


Author
Shantanu Kulkarni
Data Engineer | Azure | SQL | ETL | Data Warehousing
