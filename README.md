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
</br>
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