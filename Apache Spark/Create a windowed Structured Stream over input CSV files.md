Детали можно найти здесь https://github.com/cartershanklin/pyspark-cheatsheet?tab=readme-ov-file#create-a-windowed-structured-stream-over-input-csv-files

```python
from pyspark.sql.functions import avg, count, current_timestamp, window
from pyspark.sql.types import (
    StructField,
    StructType,
    DoubleType,
    IntegerType,
    StringType,
)

input_location = "streaming/input"
schema = StructType(
    [
        StructField("mpg", DoubleType(), True),
        StructField("cylinders", IntegerType(), True),
        StructField("displacement", DoubleType(), True),
        StructField("horsepower", DoubleType(), True),
        StructField("weight", DoubleType(), True),
        StructField("acceleration", DoubleType(), True),
        StructField("modelyear", IntegerType(), True),
        StructField("origin", IntegerType(), True),
        StructField("carname", StringType(), True),
        StructField("manufacturer", StringType(), True),
    ]
)
df = spark.readStream.csv(path=input_location, schema=schema).withColumn(
    "timestamp", current_timestamp()
)

aggregated = (
    df.groupBy(window(df.timestamp, "1 minute"), "manufacturer")
    .agg(
        avg("horsepower").alias("avg_horsepower"),
        avg("timestamp").alias("avg_timestamp"),
        count("modelyear").alias("count"),
    )
    .coalesce(10)
)
summary = aggregated.orderBy("window", "manufacturer")
query = (
    summary.writeStream.outputMode("complete")
    .format("console")
    .option("truncate", False)
    .start()
)
query.awaitTermination()
```