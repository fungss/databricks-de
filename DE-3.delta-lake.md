# Delta Lake

**Objective**
1. Delta Lake transactional guarantees -> foundation for data lakehouse architecture
2. Delta Lake DDL
3. CREATE TABLE as SELECT (CTAS)
4. Spark SQL to perform complete and incremental updates to existing tables

**Delta Lake** 
- building data lakehouse on top of cloud storage
- is a storage layer
- built for scalable metadata handling
- ACID -> Atomicity, Consistency, Isolation, Durability
- support versioning

## DE 3.1 - Schemas and Tables
**Objective**
- Use Spark SQL DDL to define schemas and tables
- Describe how the impact of ```LOCATION``` keyword

**Key Notes**
- schema = db
- data in external table persists in designated location after DROP TABLE
- data in managed table do not
- default database location if not specified ```dbfs:/user/hive/warehouse/<schema_name>.db/```

**Key Syntax**
1. Create database
```
CREATE SCHEMA IF NOT EXISTS <database_name>
```

2. Show database metadata, including location of the database
* option EXTENDED will show schema property
```
DESCRIBE SCHEMA EXTENDED <database_name>
```

3. Show table metadata, including location
- ```DESCRIBE EXTENDED <table_name>```for table schema
```
DESCRIBE DETAIL <table_name>
```

4. Create external table
```
USE <database_name>;

CREATE OR REPLACE TEMPORARY VIEW temp_delays USING CSV OPTIONS (
  path = <path_to_data_source_file>,
  header = "true",
  mode = "FAILFAST" -- abort file parsing with a RuntimeException if any malformed lines are encountered
);
CREATE OR REPLACE TABLE external_table LOCATION <path_to_desired_location> AS
  SELECT * FROM temp_delays;

SELECT * FROM external_table; 
```

## DE 3.2 - Set Up Delta Tables
**Objective**
- Use CTAS (Create Table as Select) to create Delta Lake tables
- Create new tables from existing views or tables
- Enrich loaded data with additional metadata
- Set advanced options to control data location, quality enforcement, and partitioning
- Create shallow and deep clones

**Key Notes**
- CTAS statement does not support table schema declaration -> useful only for data ingestion from external source with well-defined schema, eg Parquet files and tables
- DEEP CLONE
    - create new table by copying both metadata and data from source table
- SHALLOW CLONE 
    - create reference table to source by only including metadata
    - fast table creation, enabling testing and experimentation (e.g. test change of column data type or defintion)
- REFRESH TABLE not needed for delta lake tables, vs??????
- Delta Lake enforces schema-on-write -> Databricks support ```NOT NULL``` and ```CHECK``` constraints

**Key Syntax**
1. Create table with CTAS
```
CREATE OR REPLACE TABLE <table_name> AS
SELECT * FROM <file_ext, eg parquet/csv>.`<path_to_file>`
```

2. Data ingestion with csv file
- temp view -> CTAS from temp_vw
```
CREATE OR REPLACE TEMP VIEW sales_tmp_vw
  (order_id LONG, email STRING, transactions_timestamp LONG, total_item_quantity INTEGER, purchase_revenue_in_usd DOUBLE, unique_items INTEGER, items STRING)
USING CSV
OPTIONS (
  path = "${da.paths.datasets}/ecommerce/raw/sales-csv",
  header = "true",
  delimiter = "|"
);

CREATE TABLE sales_delta AS
  SELECT * FROM sales_tmp_vw;
```

3. Alternative to create table by defining table schema with ```Generated Columns```
- define table schema -> ```MERGE INTOP``` and ```WHEN NOT MATCHED THEN```
```
CREATE OR REPLACE TABLE purchase_dates (
  id STRING, 
  transaction_timestamp STRING, 
  price STRING,
  date DATE GENERATED ALWAYS AS (
    cast(cast(transaction_timestamp/1e6 AS TIMESTAMP) AS DATE))
    COMMENT "generated based on `transactions_timestamp` column")

SET spark.databricks.delta.schema.autoMerge.enabled=true; 

MERGE INTO purchase_dates a
USING purchases b
ON a.id = b.id
WHEN NOT MATCHED THEN
  INSERT *
```

4. Add check constraint to existing Delta Lake tables
```
ALTER TABLE <table_name> ADD CONSTRAINT <constraint_name> CHECK (date > '2020-01-01');
```

5. Enrich Tables with Additional Options and Metadata
- Check [this reference](https://spark.apache.org/docs/latest/sql-ref-functions-builtin.html) for more built-in Spark SQL commands
```
CREATE OR REPLACE TABLE users_pii
COMMENT "Contains PII"
LOCATION "${da.paths.working_dir}/tmp/users_pii"
PARTITIONED BY (first_touch_date)
AS
  SELECT *, 
    cast(cast(user_first_touch_timestamp/1e6 AS TIMESTAMP) AS DATE) first_touch_date, 
    current_timestamp() updated,
    input_file_name() source_file
  FROM parquet.`${da.paths.datasets}/ecommerce/raw/users-historical/`;
  
SELECT * FROM users_pii;
```

6. Cloning Delta Lake Tables
```
CREATE OR REPLACE TABLE <clone_table_name>
DEEP/SHALLOW CLONE  <table_name>
```

## DE 3.3 - Load Data into Delta Lake
**Objective**
- Overwrite data tables using ```INSERT OVERWRITE```
- Append to a table using ```INSERT INTO```
- Append, update, and delete using ```MERGE INTO```
- Ingest data incrementally using ```COPY INTO```

**Key Notes**
- ```INSERT OVERWRITE``` can only replace existing table, and only with matched schema, not creating new one
- ```MERGE``` handles updates, inserts, and deletes in a single transaction

**Key Syntax**
1. Create Table with CRAS
- CRAS better than DROP and CREATE coz ACID -> versioning and concurrent access
```
CREATE OR REPLACE TABLE events AS
SELECT * FROM parquet.`<path_to_source_file>`
```

2. Check history of tables
```
DESCRIBE HISTORY <table_name>
```

3. To pick up new files in the source directory automatically
```
COPY INTO <table_name>
FROM "<source_directory>"
FILEFORMAT = PARQUET
```

## DE 3.4 - Load Data Lab

## DE 3.5 - Version and Optimize Delta Tables
**Objective**
- Use ```OPTIMIZE``` to compact small files
- Use ```ZORDER``` to index tables
- Describe directory structure of Delta Lake files
- Review history of table transactions
- Query and roll back to previous table version
- Clean up stale data files with ```VACUUM```

**Key Syntax**
1. Index table and make table size smaller
```
OPTIMIZE <table_name>
ZORDER BY <index_field>
```

2. Work with specific version of a table
```
SELECT * 
FROM <table_name> VERSION AS OF <version_number>
```

3. Restore table
```
RESTORE TABLE <table_name> TO VERSION AS OF <version_number>
```

4. Purge old data with circumventing the 7 days default retention period
```
SET spark.databricks.delta.retentionDurationCheck.enabled = false;
SET spark.databricks.delta.vacuum.logging.enabled = true;
VACUUM <table_name> RETAIN 0 HOURS
```

## DE 3.6 - Manipulate Delta Tables Lab

**Reference**
1. [Easily Clone your Delta Lake for Testing, Sharing, and ML Reproducibility](https://www.databricks.com/blog/2020/09/15/easily-clone-your-delta-lake-for-testing-sharing-and-ml-reproducibility.html)