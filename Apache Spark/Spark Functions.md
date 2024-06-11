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
df.select(F.when(df.id == 2, 3).otherwise(4).alias("age")).show()

df.select(
	F.when(F.col("col3") == "green", "GREEN").otherwise("---").aliase("result")
).show()

df.select(
	F.when((F.col("col3") > 0) & (F.col("col1") > 0), "bla") \
	    .otherwise("not bla").alias("res")
).show()
```

Пусть требуется создать в кадре данных новый столбец, содержащий значения `NULL` в тех строках, в которых атрибут `id` принимает одно из следущих значений: 3, 5 или 8. Прочие значения нового столбца выставляются в 1
```python
# PySpark
df.withColumn(
	"new_col",
	F.when(F.col("id").isin([3, 5, 8]), None
).otherwise(F.lit(1)))
```

В `pandas` эту задачу можно было бы решить, например, так
```python
df["new_col"] = 1
df.loc[[3, 5, 8], "new_col"] = np.nan
```

Если требуется вставить `NaN` в разные строки разных столбцов кадра данных, то можно использовать туже самую цепочку вызовов
```python
df.withColumn(
	"x",
	F.when(
	    F.col("id").isin([3, 9]),
	    F.lit(float("nan")
	)
).otherwise(F.col("x")) \
    .withColumn(
        "y",
        F.when(
            F.col("id").isin([0, 10]),
            F.lit(float("nan"))
        )
    )
```
### `.coalesce()`

_Функция_ `F.coalesce()` (==а не метод!==) в Spark выполняет ту же работу, что и метод `.fillna()` в `pandas`, но реагирует на пропущенные значения, представленные как `NULL` (если нужно заполнять пропуски , представленные как `NaN`, то следует воспользоваться функцией `.nanval`). Функция `.coalesce()` принимает объект столбца, в котором нужно заменить пропущенные значения (`NULL`), и либо литератльное значение (`F.lit()`), либо объект другого столбца, значения которого будут использоваться вместо `NULL`
```python
# функция .coalesce() реагирует на NULL
df.withColumn(
	"target_col",
	F.coalesce(F.col("target_col"), F.lit(0))
)
df.withColumn(
	"target_col",
	F.coalesce(F.col("target_col"), F.col("another_col"))
)
```

А в `pandas` эту задачу можно было бы решать так
```python
df["target_col"] = df["target_col"].fillna(0)
```

### `.nanvl()`

Функция `.nanvl()` похожа на функцию `.coalesce()`, но при заполнении пропусков реагирует не на `NULL` (как функция `.coalesce()`), а на `NaN`
```python
df.withColumn("target_column", F.nanval(F.col("col1"), F.lit(0)))
df.withColumn("target_column", F.nanval(F.col("col1"), F.col("another_column")))
```
