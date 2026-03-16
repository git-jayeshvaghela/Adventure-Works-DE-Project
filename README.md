# Adventure-Works-DE-Project

A **production-grade ETL pipeline** implementing the **Medallion Architecture** (Bronze → Silver → Gold) for Adventure Works business data. This end-to-end data engineering solution demonstrates enterprise best practices in data ingestion, transformation, quality testing, and analytics.

**Live Architecture:** GitHub → Azure Data Factory → Databricks → Azure Synapse → Power BI

## 📊 Project Overview
 
### Business Objective
Transform raw Adventure Works data into a curated, analytics-ready data warehouse with automated ETL workflows, comprehensive data quality checks, and interactive business intelligence dashboards.
 
### Dataset
**10 CSV files** from Adventure Works:
- AdventureWorks_Calendar
- AdventureWorks_Customers
- AdventureWorks_Product_Categories
- AdventureWorks_Product_Subcategories
- AdventureWorks_Products
- AdventureWorks_Returns
- AdventureWorks_Sales_2015 / 2016 / 2017
- AdventureWorks_Territories
 
---

---
 
## 🏗️ Architecture: Medallion Pattern
 
```
┌──────────────────────────────────────────────────────────────┐
│                    BRONZE LAYER (Raw)                        │
│  Raw CSV files from GitHub (10 files, ~20MB unprocessed)    │
│  ✓ No transformations   ✓ Full historical data              │
│  ✓ Immutable storage    ✓ Source of truth                   │
└─────────────────────────┬──────────────────────────────────┘
                          │
                    [Azure Data Lake]
                          │
┌─────────────────────────▼──────────────────────────────────┐
│              SILVER LAYER (Cleaned)                         │
│  De-duplicated, validated, enriched fact/dimension tables  │
│  ✓ Data quality checks    ✓ Type conversions               │
│  ✓ Null handling          ✓ Business logic applied         │
│  ✓ SCD Type 2 dimensions  ✓ Lineage tracking              │
└─────────────────────────┬──────────────────────────────────┘
                          │
                  [Apache Spark / Databricks]
                          │
┌─────────────────────────▼──────────────────────────────────┐
│              GOLD LAYER (Analytics)                         │
│  Star schema dimensional models optimized for BI            │
│  ✓ Fact tables    ✓ Dimension tables    ✓ Aggregations    │
│  ✓ Performance optimized   ✓ Business metrics            │
└─────────────────────────┬──────────────────────────────────┘
                          │
                 [Azure Synapse SQL DW]
                          │
┌─────────────────────────▼──────────────────────────────────┐
│         PRESENTATION LAYER (Visualization)                  │
│            Power BI Dashboards & Reports                    │
│  ✓ Executive KPIs     ✓ Real-time analytics               │
│  ✓ Drill-down views   ✓ Trend analysis                    │
└──────────────────────────────────────────────────────────┘
```
 
---

## 📋 Bronze Layer: Data Ingestion
 
### Dynamic Parameterization
All 10 files ingested via a **single parameterized pipeline** (no hardcoding):
 
**Pipeline Parameters:**
```json
{
  "files": [
    {
      "p_relativeurl": "anshlamba03/Adventure-Works-Data-Engineering-Project/refs/heads/main/Data/AdventureWorks_Calendar.csv",
      "p_sink_folder_dynamiz": "Calendar",
      "p_filename_dynamaic": "AdventureWorks_Calendar.csv"
    },
    {
      "p_relativeurl": "anshlamba03/Adventure-Works-Data-Engineering-Project/refs/heads/main/Data/AdventureWorks_Sales_2015.csv",
      "p_sink_folder_dynamiz": "Sales",
      "p_filename_dynamaic": "AdventureWorks_Sales_2015.csv"
    }
    // ... 8 more files
  ]
}
```
 ### ADF Ingestion Pipeline Features
✅ **Parameterized linked services** - GitHub HTTP authentication  
✅ **ForEach activity** - Loop through 10 files concurrently  
✅ **Copy activity** - Stream data to ADLS Bronze layer  
✅ **Error handling** - Skip headers, handle encoding  
✅ **Logging** - Pipeline run metadata captured  
 
**ETL Validation at Bronze:**
- File size checks
- Row count logging
- Arrival time tracking
- Header schema comparison
 
---
 
## 🔄 Silver Layer: Data Transformation & Quality
 
### Data Quality Framework
 
**Implemented Checks:**
1. **Schema Validation**
   - Column count matches source
   - Data type compliance (int, string, date, decimal)
   - Primary key non-nullability
 
2. **Duplicate Detection**
   - Row-level exact duplicates
   - Business key duplicates (e.g., duplicate OrderID)
   - Quarantine strategy: isolated in `duplicates/` folder
 
3. **Null Handling**
   - Null percentage per column
   - Conditional nulls (e.g., optional fields vs required)
   - Default value imputation where business logic allows
 
4. **Referential Integrity**
   - Foreign key validation
   - Orphaned records detection
   - Relationship graphs (customer → order → product)
 
5. **Range & Format Validation**
   - Date range checks (no future dates)
   - Numeric range checks (no negative quantities)
   - String format validation (email, phone)
 
### Silver Layer Transformations (PySpark)
 
**Example: Customers Dimension**
```python
# Load bronze customer data
df_customers = spark.read.csv(
    "abfss://bronze@datalake.dfs.core.windows.net/Customers/*",
    header=True,
    inferSchema=True
)
 
# Quality checks
duplicates = df_customers.groupBy("CustomerID").count().filter(col("count") > 1)
print(f"Duplicate customers found: {duplicates.count()}")
 
# Transformations
df_silver = df_customers \
    .withColumn("LoadDate", current_timestamp()) \
    .withColumn("SourceSystem", lit("AdventureWorks")) \
    .dropDuplicates(["CustomerID"]) \
    .filter(col("CustomerID").isNotNull()) \
    .withColumn("FullName", concat_ws(" ", "FirstName", "LastName"))
 
# Write to Silver layer with schema evolution
df_silver.write \
    .mode("overwrite") \
    .format("delta") \
    .option("mergeSchema", "true") \
    .save("abfss://silver@datalake.dfs.core.windows.net/Customers/")
```
 
### Data Lineage Tracking
Every row in Silver includes:
- `LoadDate` - timestamp of ingestion
- `SourceSystem` - source identifier
- `SourceHash` - hash of Bronze record for traceability
- `ProcessingStatus` - "VALID", "QUARANTINED", "TRANSFORMED"


 ---
 
## 📊 ETL Testing & Data Quality Assurance
 
### Testing Strategy
 
**Unit Tests (PySpark)**
```python
# Test 1: Null check on critical columns
def test_no_nulls_in_critical_columns():
    df = spark.read.delta("silver/Customers")
    assert df.filter(col("CustomerID").isNull()).count() == 0, "Null found in CustomerID"
    assert df.filter(col("Email").isNull()).count() == 0, "Null found in Email"
 
# Test 2: Duplicate detection
def test_no_duplicates():
    df = spark.read.delta("silver/Customers")
    duplicates = df.groupBy("CustomerID").count().filter(col("count") > 1)
    assert duplicates.count() == 0, f"Found {duplicates.count()} duplicate customers"
 
# Test 3: Data type validation
def test_data_types():
    df = spark.read.delta("silver/Sales")
    assert df.schema["SalesAmount"].dataType == DecimalType(), "SalesAmount not decimal"
    assert df.schema["Quantity"].dataType == IntegerType(), "Quantity not integer"
 
# Test 4: Business rule validation
def test_sales_quantity_positive():
    df = spark.read.delta("silver/Sales")
    invalid = df.filter(col("Quantity") <= 0).count()
    assert invalid == 0, f"Found {invalid} rows with non-positive quantities"
 
# Test 5: Referential integrity
def test_foreign_key_integrity():
    orders = spark.read.delta("silver/Orders")
    customers = spark.read.delta("silver/Customers")
    orphaned = orders.join(customers, "CustomerID", "left_anti")
    assert orphaned.count() == 0, f"Found {orphaned.count()} orphaned orders"
```
