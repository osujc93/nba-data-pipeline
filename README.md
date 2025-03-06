# NBA-Data-Pipeline
Data pipeline for analyzing NBA player boxscores using Kafka, Spark, Dbt, HDFS &amp; more.

![Diagram](/assets/data_diagram.png)

## Extract

- Kafka to fetch player boxscores from NBA stats API and send to Kafka topic.

- Took data with initial batch ingestion of boxscores from 1946 to current date. and then daily from the last ingestion date.

- To not fire off requests and cause 443 or 503 HTTP errors, used a token-bucket rate limiter to space out requests, one every seven seconds. 

- And set ThreadPoolExecutor with max_workers=1 so each API request finishes fully before the next one starts. This helped with not flooding the API and prevent timeouts. 

- After the initial batch finishes, it inserts a last-ingestion date into Postgres, so that subsequent incremental runs know exactly where to resume from.

- The incremental runs look up the last-ingested date in Postgres, fetches only the new data from that date forward, sends it to Kafka, then updates the last-ingested date for the next run.

### Load

- Spark job to process data from topic to Iceberg tables stored in HDFS.

- Initially, Spark reads from Kafka’s earliest offset to capture all boxscores. 

- Then, incrementally, Spark only reads new records from the latest offset, processes them, and appends to the same Iceberg table.

### Transform 

- Run dbt model to apply SCD Type 2 merges in Iceberg tables to track historical changes

- Cumulative tables for current-season and all-seasons.

### Storage

- Data is stored in HDFS volumes.

- The volumes store file blocks (including Iceberg tables) on DataNodes, metadata on NameNodes, and edit logs in JournalNodes.

- Sets a built-in HDFS Erasure Coding policy (RS-6-3-1024k) that uses Reed–Solomon encoding with 6 data blocks, 3 parity blocks, and a 1 MB cell size. 

- This lets HDFS reconstruct data if up to three blocks are lost, while requiring less total disk space than the usual 3× replication.

- ZooKeeper is used for automatic NameNode failover by keeping track of which NameNode is active and which is standby. If the active NameNode fails, ZooKeeper coordinates switching the standby node to active, preventing both from being active at once.

- JournalNodes store the shared edit logs used by the NameNodes. If the active NameNode fails, the standby NameNode can read the latest edits from the JournalNodes and take over without data loss.

### Visualize

- Use ThriftServer's JDBC endpoint as a datasource in Superset to visualize data from Iceberg tables through interactive charts and dashboards.

### Automate

- Once all apps are up and running, Airflow automates entire workflow

- Set <code>is_paused_upon_creation</code> to False so that dag starts once Airflow is running

