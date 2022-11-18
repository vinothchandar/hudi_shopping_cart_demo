# Introduction 101





# Use-case : Tracking abandoned shopping carts

## Old school Batch Processing


## Incremental Processing with Hudi




# Setup

1. Follow the [Kafka Quickstart](https://kafka.apache.org/quickstart), get it runing locally on port 9092
2. Install kcat, a command-line utility to publish/consume from kafka topics, using  `brew install kcat`.
3. Follow the [Spark Quickstart](https://spark.apache.org/docs/latest/quick-start.html) to get Apache Spark installed, with `spark-shell`, `spark-submit` working.
4. Download/Clone this repo and `cd build_on_oss_S1E7`
5. Spin up a Spark Shell as below


```
spark-shell \
 --packages org.apache.hudi:hudi-spark3.2-bundle_2.12:0.12.1 \
 --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
 --conf 'spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog' \
 --conf 'spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension' \
 --conf 'spark.sql.warehouse.dir=file:///tmp/hudi-warehouse'
```



# [Step 0] Setup "users" table

Create a Hudi table from the csv file containing users

```
import scala.collection.JavaConversions._
import org.apache.spark.sql.SaveMode._
import org.apache.hudi.DataSourceReadOptions._
import org.apache.hudi.DataSourceWriteOptions._
import org.apache.hudi.config.HoodieWriteConfig._

val df = spark.read.option("header","true").option("inferSchema", "true").csv("file:///tmp/hudi-demo-files/user_details.csv")

df.write.format("hudi").
  option(RECORDKEY_FIELD_OPT_KEY, "user_id").
  option(TABLE_NAME, "user_details").
  option(OPERATION_OPT_KEY,"bulk_insert").
  mode(Overwrite).
  save("file:///tmp/user_details")
```



# [Step 1] Pump some user activity events into Kafka

```
cat mock_data_batch1.json | kcat -b localhost -t user_events -P
```

To check if the new topic shows up, use
```
kcat -b localhost -L -J | jq .
```

# [Step 2] Kickoff Hudi Streamer in continuous mode

Download the hudi utilities jar

```
wget  https://repo1.maven.org/maven2/org/apache/hudi/hudi-utilities-slim-bundle_2.12/0.12.1/hudi-utilities-slim-bundle_2.12-0.12.1.jar
```

```
spark-submit \
 --packages org.apache.hudi:hudi-spark3.2-bundle_2.12:0.12.1 \
 --conf 'spark.serializer=org.apache.spark.serializer.KryoSerializer' \
 --class org.apache.hudi.utilities.deltastreamer.HoodieDeltaStreamer \
 hudi-utilities-slim-bundle_2.12-0.12.1.jar \
 --table-type COPY_ON_WRITE \
 --op BULK_INSERT \
 --source-class org.apache.hudi.utilities.sources.JsonKafkaSource \
 --target-base-path file:///tmp/user_events_cow \
 --target-table user_events_cow --props file:///tmp/hudi-demo-files/kafka-source.properties \
 --continuous \
 --min-sync-interval-seconds 60 \
 --schemaprovider-class org.apache.hudi.utilities.schema.FilebasedSchemaProvider
 ```

# [Step 3] Querying table in Spark SQL

```
val user_events_df = spark.read.format("hudi").load("file:///tmp/user_events_cow")
user_events_df.createOrReplaceTempView("user_events")

val user_profile_df = spark.read.format("hudi").load("file:///tmp/user_details")
user_profile_df.createOrReplaceTempView("user_profiles")

// Query the table to get all purchased events after certain date
spark.sql("select  count(*) from user_events where event_time_date > '2022-11-10' and action_type='purchased'").show(100, false)

// Query the table to get count of users who has a non empty cart in last one week
spark.sql("select count(*) from user_events where event_time_date > '2022-11-10' and cart_empty = false").show(100, false)
```

# [Step 4] Incremental Query to compute abandoned carts

Fetch change stream from user_events table:

```
val user_events_df = spark.read.format("hudi").
     | option(QUERY_TYPE_OPT_KEY, QUERY_TYPE_INCREMENTAL_OPT_VAL).
     | option(BEGIN_INSTANTTIME_OPT_KEY, 0).
     | load("file:///tmp/user_events_cow")
```

Enrich with email id
```
val user_events_projected_df =  user_events_df.select("user_id","cart_empty","event_time_ts","last_logged_on")
val user_profiles_projected_df=user_profile_df.select(col("user_id").alias("user_profile_id"),col("email"))
val user_cart_status_df = user_events_projected_df.join(user_profiles_projected_df, user_events_projected_df("user_id") === user_profiles_projected_df("user_profile_id"), "left")
```

Upsert into user_casrt_status Hudi table
```
user_cart_status_df.write.format("hudi").
  option(RECORDKEY_FIELD_OPT_KEY, "user_id").
  option(TABLE_NAME, "user_cart_status").
  option(PARTITIONPATH_FIELD_OPT_KEY, "").
  option(KEYGENERATOR_CLASS_NAME.key, "org.apache.hudi.keygen.NonpartitionedKeyGenerator").
  option(PRECOMBINE_FIELD_OPT_KEY, "event_time_ts").
  option(OPERATION_OPT_KEY,"upsert").
  option(PAYLOAD_CLASS_OPT_KEY, "org.apache.hudi.common.model.DefaultHoodieRecordPayload").
  mode(Append).
  save("file:///tmp/user_cart_status")
```

Query abandoned cart from last 7 days up unitl 5 hours ago.

# [Step 5] Reverse Stream this into Kafka

Coming soon ;) ! 
