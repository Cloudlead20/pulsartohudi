# Pulsar to Hudi on Dataproc

This project demonstrates how to read data from an Apache Pulsar topic and write it to an Apache Hudi table using a Google Cloud Dataproc cluster.

## Prerequisites

- Google Cloud SDK (`gcloud`) installed and configured
- A Google Cloud project
- Apache Pulsar set up with a broker and topic
- Google Cloud Storage bucket for Hudi table storage

## Step 1: Set Up Dataproc Cluster

Create a Dataproc cluster with the necessary configurations and dependencies:

```sh
gcloud dataproc clusters create hudi-cluster \
    --region <your-region> \
    --zone <your-zone> \
    --single-node \
    --image-version 2.2-debian10 \
    --scopes cloud-platform \
    --properties "spark:spark.jars.packages=org.apache.hudi:hudi-spark-bundle_2.12:0.15.0,org.apache.pulsar:pulsar-spark_2.12:2.10.0"

```

## Step 2: Install Dependencies

Ensure Hudi, Pulsar, and other dependencies are available on your cluster.

Dependencies Installation Metadata
Hudi Version: 0.15.0
Pulsar Spark Connector Version: 2.10.0
Spark Version: 3.x (compatible with Dataproc 2.2)



## Step 3: Create a Script to Read Pulsar Topic and Create Hudi Table
Create a Python script named hudi_pulsar.py:

 ```sh
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType, LongType

# Initialize Spark Session
spark = SparkSession.builder \
    .appName("PulsarToHudi") \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .getOrCreate()

# Pulsar configuration
pulsar_service_url = "pulsar://<your-pulsar-broker>:6650"
pulsar_topic = "<your-pulsar-topic>"

# Define schema based on your data
schema = StructType([
    StructField("record_key", StringType(), True),
    StructField("partition_path", StringType(), True),
    StructField("precombine_field", LongType(), True),
    # Add other fields here
])

# Read from Pulsar topic
pulsar_df = spark.read \
    .format("pulsar") \
    .option("service.url", pulsar_service_url) \
    .option("topic", pulsar_topic) \
    .load()

# Transform data as needed
transformed_df = pulsar_df.selectExpr("CAST(value AS STRING) as json") \
    .select(from_json(col("json"), schema).alias("data")) \
    .select("data.*")

# Hudi options
hudi_options = {
    "hoodie.table.name": "<your_hudi_table_name>",
    "hoodie.datasource.write.recordkey.field": "record_key",
    "hoodie.datasource.write.partitionpath.field": "partition_path",
    "hoodie.datasource.write.precombine.field": "precombine_field",
    "hoodie.datasource.hive.sync.enable": "true",
    "hoodie.datasource.hive.sync.table": "<your_hudi_table_name>",
    "hoodie.datasource.hive.sync.partition.fields": "partition_path",
    "hoodie.datasource.hive.sync.jdbcurl": "jdbc:hive2://<your-hive-server>:10000"
}

# Write to Hudi table
transformed_df.write.format("hudi") \
    .options(**hudi_options) \
    .mode("append") \
    .save("gs://<your-gcs-bucket>/path/to/hudi/table")

# Stop the Spark session
spark.stop()

```

### Script Metadata
# hudi_pulsar.py


- **Script Name:** hudi_pulsar.py
- **Language:** Python

## Spark Configuration

- **Serializer:** Kryo

## Pulsar Configuration

- **Service URL:** pulsar://<your-pulsar-broker>:6650
- **Topic:** <your-pulsar-topic>

## Hudi Configuration

- **Table Name:** <your_hudi_table_name>
- **Record Key Field:** record_key
- **Partition Path Field:** partition_path
- **Precombine Field:** precombine_field
- **Hive Sync Enabled:** true
- **Hive Table Name:** <your_hudi_table_name>
- **Hive Partition Fields:** partition_path
- **Hive JDBC URL:** jdbc:hive2://<your-hive-server>:10000
- **Data Storage Path:** gs://<your-gcs-bucket>/path/to/hudi/table



## Step 4: Submit the Job to Dataproc
Use gcloud to submit the job to Dataproc:

```sh
gcloud dataproc jobs submit pyspark hudi_pulsar.py --cluster=hudi-cluster --region=<your-region>
```

#### Job Submission Metadata

1.Cluster Name: hudi-cluster

2.Region: <your-region>

3.Script Path: hudi_pulsar.py


### Notes
Adjust the schema and transformation logic as per your data structure.
Ensure Pulsar, Hudi, and Spark versions are compatible. Dataproc 2.2 typically uses Spark 3.x.
Configure Hive and Hudi integration properly.


### Handling Version Compatibility Issues
Check Spark, Hudi, and Pulsar versions: Ensure the versions you are using are compatible. Check the documentation for each tool.
Dependencies: Ensure all required dependencies are available on your cluster. You can manually install them if needed.
