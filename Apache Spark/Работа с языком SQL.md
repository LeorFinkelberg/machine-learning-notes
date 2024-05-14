Следует запомнить прием, применяемый, когда необходимо использовать SQL совместно со Spark: требуется определить _представление_ (view), а представление является элементом, к которому вы будете обращаться с запросом.

Чтобы получить возможность использования SQL с таблицами в Spark, необходимо создать представление. Область видимости может быть локальной (для сеанса) или глобальной (для приложения).

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import * 
from contextlib import contextmanager

@contextmanager
def start_spark_sessoin(
	master_url: str = "local[*]",
	app_name: str = "Spark Application",
):
    try:
        spark = SparkSession.builder \
            .master(master_url) \
            .appName(app_name) \
            .getOrCreate()

        yield spark
    finally:
        spark.stop()

def main():
	schema = StructType([
        StructField("geo", StringType(), True),
        StructField("yr1980", DobleType(), True),
	])

    df = spark.read.format("csv") \
        .option("header", True) \
        .schema(schema) \
        .load("./file.csv")

    # регистрируем табилцу как представление
    df.createOrReplaceTempView("geodata")
    query = "SELECT * FROM geodata WHERE yr1980 < 1 ORDER BY 2 LIMIT 5"
	df = spark.sql(query)
    df.printSchema()


if __name__ == "__main__":
    with start_spark_session() as spark:
	    main()
```