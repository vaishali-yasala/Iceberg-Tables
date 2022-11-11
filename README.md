# Iceberg-Tables

Iceberg is a high-performance format for huge analytic tables. Iceberg brings the reliability and simplicity of SQL tables to big data, while making it possible for engines like Spark, Trino, Flink, Presto, Hive and Impala to safely work with the same tables, at the same time.

The Iceberg table format offers many features to help power your data lake architecture.

1. Schema evolution 
2. Expressive SQL
3. Partition evolution 
4. Time travel and rollback
5. Transactional consistency

To test these features, let us get started by running a local Spark instance with Iceberg integration using Docker. In this environment, we can explore Iceberg's above given features in Jupyter Notebook.

### Launching the Notebook

First, install Docker and Docker Compose if you don't already have them. Next, create a docker-compose.yaml file with the following content.

![img](/Input_and_Output_images/docker_compose.png)

In the same directory as the docker-compose.yaml file, run the following commands to start the runtime and launch an Iceberg-enabled Spark notebook server.

> docker-compose up -d </br>
> docker exec -it spark-iceberg pyspark-notebook

This opens a fully functional notebook server running at http://localhost:8888. 

### A Minimal Runtime
The runtime provided by the docker compose file is far from a large scale production-grade warehouse, but it does let you demo Iceberg's wide range of features. Let's quickly cover this minimal runtime.

1. Spark 3.3.0 in local mode (the Engine)
2. A JDBC catalog backed by a Postgres container (the Catalog)
3. A docker-compose setup to wire everything together
4. A %%sql magic command to easily run SQL in a notebook cell

### Look at the Spark Session Version in Notebook
![Img](/Input_and_Output_images/sparksession.png)

### Let us load Data 
For this session, I am using the New York City Taxi and Limousine Commission Trip Record Data that's available on the AWS Open Data Registry. The data I picked is of yellow taxis January 2022 trip records in parquet format. Let us save this into an iceberg table called *trips*. 

To get accurate test results, let us drop the table if it already exists. 
![Img](/Input_and_Output_images/drop_table.png)

Let us the read and save the data as a Table under folder tlc in local with trips as table name. 
![Img](/Input_and_Output_images/read_and_save_data.png)

Look at the extensive information of the table trips. 
![img](/Input_and_Output_images/Description_1.png)
![img](/Input_and_Output_images/Description_2.png)

Look at the number of trip records available in the database.
![img](/Input_and_Output_images/count.png)

Now that the data is read, let us explore some of the Iceberg's features given above.

## 1. Schema Evolution 
Schema evolution supports add, drop, update, or rename, and has no side-effects. 

- Add- add a new column to the table or to a nested struct
- Drop - remove an existing column from the table or a nested struct
- Rename - rename an existing column or field in a nested struct
- Update - widen the type of a column, struct field, map key, map value, or list element
- Reorder - change the order of columns or fields in a nested struct

Iceberg schema updates are metadata changes, so no data files need to be rewritten to perform the update.

Below is the snapshot of some of ways to alter the table with Schema Evolution.
![img](/Input_and_Output_images/Schema_evolution.png)

Description of the new altered table
![img](/Input_and_Output_images/schema_evolution_description.png)

Let us look at a part of the newly updated table where VendorId is 1. 
![img](/Input_and_Output_images/Select.png)

## 2. Expressive SQL 
Iceberg supports flexible SQL commands to merge new data, update existing rows, and perform targeted deletes. Iceberg can eagerly rewrite data files for read performance, or it can use delete deltas for faster updates.

1.  Merge new Data -  Requires Iceberg Spark extensions </br>
- Iceberg supports MERGE INTO by rewriting data files that contain rows that need to be updated in an *overwrite* commit. </br>
- MERGE INTO is recommended instead of INSERT OVERWRITE because Iceberg can replace only the affected data files, and because the data overwritten by a dynamic overwrite may change if the table's partitioning changes. </br>
- MERGE INTO updates a table, called the target table, using a set of updates from another query, called the source. The update for a row in the target table is found using the ON clause that is like a join condition.

#### MERGE INTO SYNTAX

> MERGE INTO prod.db.target t   -- a target table </br>
> USING (SELECT ...) s          -- the source updates </br>
> ON t.id = s.id                -- condition to find updates for target rows </br>
> WHEN ...                      -- updates


Creating a source table and inserting values to understand MERGE INTO query functionality.

![img](/Input_and_Output_images/create_table.png)
![img](/Input_and_Output_images/Insert_and_merge.png)

Before merge into was used, table data for a particular row
![img](/Input_and_Output_images/before_merge_into.png)

After merge into was used, table data for a particular row
![img](/Input_and_Output_images/after_merge_into.png)

Only one record in the source data can update any given row of the target table, or else an error will be thrown.

2. Targeted Deletes - Row-level delete requires Spark extensions
- With Iceberg tables, DELETE queries can be used to perform row-level deletes. This is as simple as providing the table name and a WHERE predicate. If the filter matches an entire partition of the table, Iceberg will intelligently perform a metadata-only operation where it simply deletes the metadata for that partition.

Let's perform a row-level delete for all rows that have a fare_per_distance_unit greater than 4 or a distance less than 2. This should leave us with the number of trips that are priced moderately. 

![img](/Input_and_Output_images/delete_from_and_count.png)

The above count 555,430 is way less than the total trips of 2,463,931 (we have begun with this number of trips).

3. Update existing rows -  Requires Iceberg Spark extensions
- Update queries accept a filter to match rows to update.

![img](/Input_and_Output_images/update_set.png)

Before updating a particular row, table data is given below
![img](/Input_and_Output_images/before_update_set.png)

After updating a particular row, table data is given below
![img](/Input_and_Output_images/after_update_set.png)

## 3. Partitioning

Partitioning is a way to make queries faster by grouping similar rows together when writing. Iceberg can partition timestamps by year, month, day, and hour granularity. 

The problem with Hive partition (which was being used before) is: 
- when there is timestamp such as '2022-01-01 10:40:15' in a column and we are trying to find the records in the table between 10 and 12 of a particular date. 
- if we query '01-01-2022' instead of '2022-01-01' which is the wrong format, it produces silently incorrect result rather than query failures. 

So, it depends on the user to write queries correctly to avoid silently incorrect results. 
Working queries are tied to the table's partitioning scheme, so partitioning configuration cannot be changed without breaking queries. 

What does Iceberg do differently?
- Other tables formats like Hive support partitioning, but Iceberg supports hidden partitioning.
- Iceberg handles the tedious and error-prone task of producing partition values for rows in a table.
- Iceberg avoids reading unnecessary partitions automatically. Consumers don't need to know how the table is partitioned and add extra filters to their queries.
- Iceberg partition layouts can evolve as needed.

ICEBERG'S HIDDEN PARTITIONING -
A table's partitioning can be updated in place and applied only to newly written data. Query plans are then split, using the old partition scheme for data written before the partition scheme was changed, and using the new partition scheme for data written after. People querying the table don't even have to be aware of this split. Simple predicates in WHERE clauses are automatically converted to partition filters that prune out files with no matches. This is what's referred to in Iceberg as Hidden Partitioning.

![img](/Input_and_Output_images/partition.png)
![img](/Input_and_Output_images/partition_description.png)

Iceberg can partition timestamps by year, month, day, and hour granularity.

Adding a partition field is a metadata operation and does not change any of the existing table data. New data will be written with the new partitioning, but existing data will remain in the old partition layout. Old data files will have null values for the new partition fields in metadata tables.

Partition fields can be removed using DROP PARTITION FIELD. 

> ALTER TABLE tlc.trips DROP PARTITION FIELD tpep_dropoff_datetime

Note that although the partition is removed, the column will still exist in the table schema.

Dropping a partition field is a metadata operation and does not change any of the existing table data. New data will be written with the new partitioning, but existing data will remain in the old partition layout.

A partition field can be replaced by a new partition field in a single metadata update by using REPLACE PARTITION FIELD query. 

![img](/Input_and_Output_images/replace_partition.png)
![img](/Input_and_Output_images/replace_partition_description.png)