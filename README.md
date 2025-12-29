# 🚗 Car Sales Incremental Data Pipeline on Microsoft Fabric

An **end-to-end Data Engineering project** built on **Microsoft Fabric**, demonstrating ingestion from GitHub, watermark-based incremental loading, and dimensional modeling using **SCD Type-1** with **PySpark notebooks**.

The project delivers **curated external Delta tables in a Lakehouse**, designed for **downstream analytics consumption**.

---

## 📌 Project Overview

This project implements a modern **ELT pipeline** that:

- Ingests raw CSV files from **GitHub** into a staging database  
- Performs **incremental loads** using watermark logic  
- Transforms data through **Bronze, Silver, and Gold** layers  
- Builds a **Star Schema** (dimensions & fact)  
- Stores curated data as **external Delta tables** in a Fabric Lakehouse  
- Enables downstream teams to load data as **managed tables** into a **Warehouse** for analytics and Power BI

---

## 🏗 Architecture
<img width="4233" height="2406" alt="ARCHITECTURE" src="https://github.com/user-attachments/assets/ecfc8f69-f80e-4aa0-8177-f5912604b919" />

---

## 🛠 Tech Stack

- **Microsoft Fabric**
- **Data Factory Pipelines**
- **Lakehouse (External Delta Tables)**
- **Warehouse (Downstream Analytics)**
- **PySpark Notebooks**
- **Delta Lake**
- **GitHub** (Source Data)

---

## 📂 Data Layers

- **Bronze**  
  Raw incremental data copied from the source DB  

- **Silver**  
  Cleaned, standardized, and deduplicated data  

- **Gold**  
  Dimensional model stored as **external Delta tables**:
  - `dim_date`
  - `dim_branch`
  - `dim_model`
  - `dim_dealer`
  - `fact_sales`

---

## 🔁 Pipelines

### 1️⃣ GitToDB Pipeline (GitHub → DB)
<img width="1910" height="993" alt="image" src="https://github.com/user-attachments/assets/754c2443-6012-4d5e-9069-05a6ab31e740" />

- Copies CSV files from GitHub into staging DB tables  
- Acts as the **raw landing layer**

---

### 2️⃣ Incremental Pipeline (DB → Lakehouse)
<img width="1905" height="989" alt="image" src="https://github.com/user-attachments/assets/a847dfe5-d4d8-4f6b-bf7f-9c77ff37748f" />

- Uses **watermarking** to track last and current load  
- Copies only **new or changed records** into the Lakehouse (Bronze)  
- Orchestrates PySpark notebooks:
  - Bronze → Silver
  - Silver → Dimensions
  - Dimensions → Fact

---

## 🧪 Incremental Load Logic

A watermark table with last_load column

```sql
SELECT *
FROM SOURCE_CARS
WHERE DATE_ID >= '@{activity('LAST LOAD').output.value[0].last_load}' AND DATE_ID <= '@{activity('CURRENT LOAD').output.value[0].max_date}'
```
---

## 🧹 Data Quality & Deduplication

Because the **GitToDB pipeline uses append mode**, duplicate records can exist in the **staging and Bronze layers**.

Deduplication is handled in the **Silver layer** using PySpark before loading data into the Gold layer.

```python
df = df.dropDuplicates([...])
```

This ensures that only **clean and unique records** flow into the dimensional model.

---

## 🐢 Slowly Changing Dimensions (SCD Type-1)

All dimension tables in this project follow **Slowly Changing Dimension Type-1 (SCD-1)** methodology.

### Characteristics
 
- Existing records are overwritten on change  
- New records are inserted  

Delta Lake `MERGE` is used to implement SCD Type-1 behavior.

```python
from delta.tables import DeltaTable

if spark.catalog.tableExists('dim_dealer'):            #Incremental run
    delta_table=DeltaTable.forPath(spark,'abfss://incre_ws@onelake.dfs.fabric.microsoft.com/lake_san.Lakehouse/Files/gold/dim_dealer')
    delta_table.alias("trg").merge(df_final.alias("src"),"trg.Dealer_ID=src.Dealer_ID")\
        .whenMatchedUpdateAll()\
        .whenNotMatchedInsertAll()\
        .execute()
else:                                                      #Initial run
    df_final.write.format("delta")\
        .mode('overwrite')\
        .option('path','abfss://incre_ws@onelake.dfs.fabric.microsoft.com/lake_san.Lakehouse/Files/gold/dim_dealer')\
        .saveAsTable('dim_dealer')
```

This guarantees **idempotent upserts** and maintains the **latest dimension state**.

---

## ⭐ Dimensional Modeling

The Gold layer follows a **Star Schema** design.


- Central **fact_sales** table with business measures  
- Supporting dimensions: `dim_date`, `dim_branch`, `dim_model`, `dim_dealer`  
- Fact table stores **foreign keys + measures only**  
- Dimensions use **surrogate keys** and follow **SCD Type-1**  
- Optimized for analytical queries and Power BI consumption  

Gold tables are stored as **external Delta tables** in the Fabric Lakehouse.

```text
Files/gold/
├── dim_date
├── dim_branch
├── dim_model
├── dim_dealer
└── fact_sales
```

---

## 📥 Downstream Analytics Enablement

Downstream **Data Analytics (DA)** teams can:

- Load external Delta tables into a **Fabric Warehouse**
- Create **managed tables**
- Perform analytical queries using **SQL**
- Build **Power BI semantic models and reports**

> Visualization and reporting are treated as **downstream responsibilities** and are **out of scope** for this project.

---

## 📂 Repository Structure

```text
.
├── data/
├── notebooks/
├── pipelines/
├── diagrams/
├──LICENSE
└── README.md
```
