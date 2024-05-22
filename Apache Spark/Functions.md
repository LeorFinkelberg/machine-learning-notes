Детали можно найти на странице документации https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html.  
### `.explode()`

Возвращает новую строку для каждого элемента в целевом массиве или отображении. Если для новых столбцов не указан псеводним, то метод использует имя `col` в случае массива и имена `key` и `value` в случае отображения.

Метод `.explode()` _удалит строку_, если атрибут, по которому выполняется развертывание значения, содержит пустой массив или отображение!

```python
from pyspark.sql import Row, functions as F

df = spark.createDataFrame([
    Row(col=1, intlist=[10, 20, 30], mapfield={"foo": "bar"}),
    Row(col=10, intlist=[10], mapfield={}),
    Row(col=100, intlist=[], mapfield={"foo1": "bar1", "foo2": "bar2"}),
])

# `.explode()` удалит строку col=100,
# так как атрибут развертывания `intlist` в этой строке содержит пустой массив
df.select("col", "mapfield", F.explode("intlist").alias("intlist")).show()
+---+-----------+-------+
|col|   mapfield|intlist|
+---+-----------+-------+
|  1|{foo -> bar}|    10|
|  1|{foo -> bar}|    20|
|  1|{foo -> bar}|    30|
| 10|          {}|    10|
+---+-----------+-------+
```

### `.explode_outer()`

Для каждого элемента атрибута развертывания (то есть атрибута, по которому выполняется развертывание) возвращает новую строку. Однако в отличие от метода `.explode()`, если в конкретной строке значение атрибута развертывания не задано или представляет собой пустой объект, то `explode_outer()` возвращает `NULL`.

```python
from pyspark.sql import Row, functions as F

df = spark.createDataFrame([
    Row(col=1, intlist=[10, 20, 30], mapfield={"foo": "bar"}),
    Row(col=10, intlist=[10], mapfield={}),
    Row(col=100, intlist=[], mapfield={"foo1": "bar1", "foo2": "bar2"}),
])

# `.explode()` удалит строку col=100,
# так как атрибут развертывания `intlist` в этой строке содержит пустой массив
df.select("col", "mapfield", F.explode_outer("intlist").alias("intlist")).show()
+---+--------------------------+-------+
|col|                  mapfield|intlist|
+---+--------------------------+-------+
|  1|                 {foo -> bar}|  10|
|  1|                 {foo -> bar}|  20|
|  1|                 {foo -> bar}|  30|
| 10|                           {}|  10|
| 100|{foo2 -> bar2, foo1 -> bar1}|NULL|  # <= NB
+---+-----------------------------+----+
```

### `.when()`

Пример
```python
from pyspark.sql import functions as F

df = spark.range(3)
df.select(F.when(df.id) == 2, 3).otherwise(4).alias("age")).show()

df.select(
	F.when(F.col("col3") == "green", "GREEN").otherwise("---").aliase("result")
).show()

df.select(
	F.when((F.col("col3") > 0) & (F.col("col1") > 0), "bla") \
	    .otherwise("not bla").alias("res")
).show()
```
