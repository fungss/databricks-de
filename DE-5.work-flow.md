# Orchestration with Databricks Workflow Jobs

**Objective**
1. Delta Lake transactional guarantees -> foundation for data lakehouse architecture
2. Delta Lake DDL
3. CREATE TABLE as SELECT
4. Spark SQL to perform complete and incremental updates to existing tables

**Key Notes**
- building data lakehouse on top of cloud storage
- a storage layer
- built for scalable metadata handling
- ACID -> Atomicity, Consistency, Isolation, Durability

## DE 5.1 - Scheduling Tasks with the Jobs UI
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

## DE 5.2L - Jobs Lab