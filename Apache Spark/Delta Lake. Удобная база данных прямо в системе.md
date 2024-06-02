Delta Lake -- это база данных в ядре инфраструктуры Spark. Изначально Delta Lake была известна под именем Databricks Delta и была доступна в облачной среде Databricks. На конференции Spark Summit 2019 компания Databricks открыла исходный код Delta под лицензией Apache, и название сменилось на Delta Lake. 

Delta Lake располагается внутри Spark и обеспечивает доступ к одному набору данных из различных сеансов.

При записи данных в Delta Lake используется формат `delta`, но при сохранении на диск данные записываются в более эффективном файловом формате Apache Parquet. Данные сохраняются на _рабочем узле_ .

Delta Lake -- база данных, которая существует в рабочей среде Spark. Можно надежно (постоянно) сохранять кадры данных в Delta Lake (https://delta.io).

Для сокращения количества разделов можно использовать метод `coalesce()`  или `repartation()`.

ВАЖНО! Перед началом работы с базой данных Delta Lake https://github.com/delta-io/delta/releases необходимо убедиться, что Spark будет использовать совместимую версию базы данных (с учетом версии Scala).

Узнать версию Scala для текущей версии Spark можно так
```bash
$ spark-submit --version
```

Совместимость версий Spark и Delta Lake можно проверить на странице проекта https://docs.delta.io/3.2.0/releases.html.

Полезные советы по установке и разрешению конфликтов можно найти на странице Quick Start https://docs.delta.io/3.2.0/quick-start.html.

Например, если Spark версии 3.5.1 (Scala 2.12), Delta Lake версии 3.1.0, то значение `--packages` будет выглядеть так
```bash
# Spark 3.5.1
--packages io.delta:delta-spark_2.12:3.1.0
```

Для проведения тестов и в целом для конфигурирования Spark-сессии для работы с Delta Lake, на странице https://docs.delta.io/3.2.0/quick-start.html#python советуют установить _совместимую_  версию `delta-spark`
```bash
$ pip install delta-spark==3.1.0
```

При создании экземпляра Spark-сессии нужно будет сделать следующее
```python
from pyspark.sql import SparkSession
from delta import *  # NB

builder = SparkSession.builder.appName("MyAPP") \
    .config(
        "spark.sql.extensions",
        "io.delta.sql.DeltaSparkSessionExtension"
    ) \
    .config(
        "spark.sql.catalog.spark_catalog",
        "org.apache.spark.sql.delta.catalog.DeltaCatalog"
    ) 

spark = configure_spark_with_delta_pip(builder).getOrCreate()
```

И как бы этого должно быть достаточно, но на практике запустить Spark-приложение с помощью `spark-submit` без указания
```
--packages io.delta:delta-spark_2.12:3.1.0
```
не получается. Поднимает исключение 
```bash
java.lang.ClassNotFoundException: io.delta.sql.DeltaSparkSessionExtension
```

==Пробовал передавать путь пакета и при создании экземпляра Spark-сессии и через параметр `extra_packages` функции `configure_spark_with_delta_pip`, но ничего не получается. Возможно есть все-таки какая-то несовместимость версий==

Пример работы с Delta Lake. 
```python
# delta_lake.py

from pyspark.sql import SparkSession
from contextlib import contexmanager
from delta import *

@contextmanager
def run_spark_session(
	master_url: str = "local[*]",
	app_name: str = "Spark App",
):
    try:
        extra_packages = ["io.delta:delta-spark_2.12:3.1.0"]

        _builder = SparkSession.builder \
            .appName(app_name) \
            .master(master_url) \
            .config(
                "spark.jars.packages",
                extra_packages[0]
            ) \
            .config(
                "spark.sql.extensions",
                "io.delta.sql.DeltaSparkSessionExtension"
            ) \
            .config(
                "spark.sql.catalog.spark_catalog",
                "org.apache.spark.sql.delta.catalog.DeltaCatalog"
            )

        _spark = configure_spark_with_delta_pip(
            _builder,
            extra_packages=extra_packages,
        ).getOrCreate() 

        yield _spark
    finally:
        spark.stop()

def main():
    spark.range(10).write.format("delta") \
        .mode("overwrite") \
        .save("./results/delta-table/")

    df = spark.read.format("delta").load("./results/delta-table")
    df.show()
    df.printSchema()

if __name__ == "__main__":
    with run_spark_session() as spark:
        main()
        
```

Однако, этот приложение можно запустить, если передать значение параметру `--packages`
```bash
$ spark-submit --packages io.delta:delta-spark_2.12:3.1.0 ./delta_lake.py
```
