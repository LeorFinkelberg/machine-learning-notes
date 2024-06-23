Детали здесь https://sparkbyexamples.com/pyspark/pyspark-broadcast-variables/

В PySpark broadcast-переменные -- это переменные доступные _только для чтения_ на всех рабочих узлах кластера.

Когда запускается Spark-приложение с broadcast-переменными, PySpark делает следующее:
- PySpark разбивает задание (job) на этапы (stages).
- Позже этапы разбиваются на задачи (tasks).
- Spark транслирует (broadcast) общие данные, нужные задачам, на каждом этапе.
- Транслируемые данные (broadcast data) кэшируются в сериализованном формате и десериализуются перед выполнением каждой задачи.

Нужно создавать broadcast-переменные для данных, которые могут быть переиспользованы в разных задачах и на разных этапах.

NB! Broadcast-переменные не отправляются на рабочие узлы с помощью вызова `sc.broadcast(variable)`. Вместо этого они будут отправлены на рабочие узлы при первом использовании.

Создать broadcast-переменную можно так
```python
broadcast_var = spark.sparkContext.broadcast({"key1": 10, "key2": 20})
broadcast_var.value  # {"key1": 10, "key2": 20}
```

Пример
```python
import pyspark
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName('SparkByExamples.com').getOrCreate()

states = {"NY":"New York", "CA":"California", "FL":"Florida"}
broadcastStates = spark.sparkContext.broadcast(states)

data = [("James","Smith","USA","CA"),
    ("Michael","Rose","USA","NY"),
    ("Robert","Williams","USA","CA"),
    ("Maria","Jones","USA","FL")
  ]

columns = ["firstname","lastname","country","state"]
df = spark.createDataFrame(data = data, schema = columns)
df.printSchema()
df.show(truncate=False)

def state_convert(code):
    return broadcastStates.value[code]

result = df.rdd.map(
	lambda x: (
	    x[0],
	    x[1],
	    x[2],
	    state_convert(x[3])
	)
).toDF(columns)
result.show(truncate=False)
```

Еще можно использовать broadcast-переменные при фильтрации и в соединениях
```python
df.filter(F.col("state").isin(
	set(broadcastStates.value.keys())
)).show()
```