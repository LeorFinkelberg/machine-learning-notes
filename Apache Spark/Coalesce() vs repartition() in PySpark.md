Подробности можно найти на странице https://sparkbyexamples.com/pyspark/pyspark-repartition-vs-coalesce/.

Если коротко, то метод `.repartition()` изменяет (увеличивает или уменьшает) количество партиций, в то время как метод `.coalecse()` может только уменьшать количество партиций.

==NB! Методы `.repartition()` и `.coalecse()` -- это _очень дорогие операции_, потому как они обычно приводят к перетасовке данных между партициями. Поэтому следует стараться использовать эти операции _как можно реже_==

В PySpark метод `sparkContext.parallelize()` используется для распаралеливания коллекций устойчивым распределенным наборам данных (RDD). 
```python
rdd1 = spark.sparkContext.parallelize(range(25), 6)
rdd1.getNumPartitions()  # 6
```

Второй аргумент метода `sparkContext.parallelize()` определяет количество партиций, на которые будут разбиты данные.

Теперь запишем RDD на диск и посмотрим содержимое партиций
```python
rdd1.saveAsTextFile("/tmp/partitions/")

Partition 1 : 0 1 2
Partition 2 : 3 4 5
Partition 3 : 6 7 8 9
Partition 4 : 10 11 12
Partition 5 : 13 14 15
Partition 6 : 16 17 18 19
```

Если при выполнении этой операции возникает исключение 
```bash
Py4JJavaError: An error occured while calling o97.saveAsTextFile
```
то скорее всего директория с указанными именем уже существует и нужно либо задать путь до новой директории, либо удалить существующую.

Разумеется таким способом можно записать и кадр данных
```python
df.rdd.saveAsTextFile("/tmp/partitions")
```

В указанной директории будет лежать заданное количество файлов (`part-XXXX`), содержащих строковое представление Row-объектов
```python
# part-00000
# Сюда мы записывали табилцу с атрибутами id, rand и hypo
Row(id=0, rand=-0.673..., hypo=0.397...)
```

### repartition()

Метод `.repartition()` перераспределяет данные по заданному числу партиций. Когда мы вызываем `.repartition(n)`, где `n` -- желаемое число партиций, Spark _перетасовывает_ данные между партициями. 

Если изменить (уменьшить или увеличить) количество партиций с помощью метода `.repartition()`, Spark выполнит ==полную перетасовку данных по всему кластеру==, что может быть очень дорогостоящей операцией, особенно для больших наборов.

### coalecse()

 Метод `.coalesce()` _уменьшает_ (!) количество партиций _без перетасовки данных по всему кластеру_. Когда мы вызываем `.coalesce(n)`, где `n` -- это желаемое количество партиций, Spark _сливает_ (merges) существующие партиции таким образом, чтобы получилось заданное количество партиций `n`.

### PySpark DataFrame repartition() vs coalesce()

Как и в случае с RDD, вы НЕ МОЖЕТЕ управлять партиционированием/параллелизмом при создании кадров данных.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("SparkByExamples.com") \
    .master("local[5]").getOrCreate()

df = spark.range(20)
df.rdd.getNumPartitions()  # 5

df.write.mode("overwrite").csv("/tmp/partitions.csv")
```

В этом примере будет создано 5 партиций (количество задается через `master("local[5]")`) и затем данные будут распределены по этим 5 партициям.

Когда Spark-сессия запускается с локальным ведущим узлом, использующим все доступные ресурсы, то есть `.master("local[*]")`, то кадры данных будут разбиваться на число партиций, равное числу вычислительных блоков (ядер процессора) -- `multiprocessing.cpu_count()`.

#### DataFrame repartition()

Метод `.repartition()` перераспределяет данные равномерно по заданному числу партиций, оптимизируя параллелизм и утилизацию ресурсов. Метод вызывает полную перетасовку данных и это полезно при подготовке схемы для последующих (нижестоящих) операций (downstream operations), таких как _объединения_ (joins) и _агрегации_ (aggregations).

В следующем примере число партиций увеличивается с 5 до 6 за счет перераспределения данных во всех партициях
```python
df2 = df.repartition(6)
df2.rdd.getNumPartitions()  # 6
```

И _даже в случае уменьшения числа партиций_ метод `.repartition()` все равно вызывает перемещение данных между всеми партициями. Поэтому в случае уменьшения числа партиций рекомендуется использовать метод `.coalecse()`.

#### DataFrame coalesce()

Метод `.coalesce()` используется исключительно для снижения числа партиций
```python
df3 = df.coalesce(2)
df3.add.getNumPartitions()
```

### Default Shuffle Partition

По умолчанию Spark задает 200 партиций (shuffle partitions). Этим значением можно управлять через параметр
```bash
spark.sql.shuffle.partitions
```

```python
df4 = df.groupBy("id").count()
df4.rdd.getNumPartitions()
```

### PySpark repartition vs coalesce

|                      | `.reparation()`                                                                                                   | `.coalesce()`                                                                                                          |
| -------------------- | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Описание             | Регулирует количество партиций, перераспределяя данные между заданным числом партиций                             | Уменьшает число партиций _без перетасовки данных_, сливая существующие партиции                                        |
| Full Shuffle         | ==Да==                                                                                                            | _Нет_ (!)                                                                                                              |
| Дороговизна операции | Может быть очень доорогой, особенно в случае больших наборов данных, так как вызывает _полную перетасовку данных_ | Эта операция менее затратная, чем `.repartition()`, так как минимизирует перемещение данных, комбинируя партиции       |
| Data Movement        | Распределяет данные _равномерно_, выравнивая партиции по мощности (размеру)                                       | Может создавать партиции, несбалансированные по мощности                                                               |
| Use Cases            | Полезен, когда требуется получить сбалансированные по мощности партиции                                           | Полезен, когда требуется уменьшить число партиций _без накладных расходов на полную перетасовку данных_ (full shuffle) |

Чтобы по ошибке не использовать метод `.repartition()` в случаях, когда требуется снизить число партиций, можно написать функцию, которая для редуцирования числа партиций будет использовать метод `.coalesce()`, а для случаев, когда требуется увеличить количество партиций (пусть и с полной перетасовкой) -- метод `.repartition()`
```python
import typing as t
from pyspark.sql import dataframe

SparkDataFrame: t.TypeAliase = dataframe.DataFrame

def make_repartition(
	df: SparkDataFrame,
    /,  # параметр `df` передается строго позиционно
    *,  # параметр `target_n_partitions` передается строго поименно
	target_n_partitions: int
) -> SparkDataFrame:
    current_n_partitions = df.rdd.getNumPartitions()
    _msg = (
        f"-> Current number of partitions: {current_n_partitions}.\n"
        f"-> Target number of partitions: {target_n_partions}"
    )

    if current_n_partitions == target_n_partitions:
        raise <CustomException>("...")

    if target_n_partitions < current_n_partitions:
        print(f"{_msg}. \nREMARK: Using `.coalesce()` method ...")
        df = df.coalesce(target_n_partitions)
    else:
        print(f"{_msg}. \nREMARK: Using `.repartition()` method ...")
        df = df.repartition(target_n_partitions)

    return df


df: SparkDataFrame = ...
df.rdd.getNumPartitions()  # 8

repar_df = make_repartition(df, target_n_partitions=4)
repar_df.rdd.getNumPartitions()  # 4
repar_df.rdd.saveAsText("/tmp/partitions/")
```

В директории `/tmp/partitions` будут лежать фрагменты
```bash
$ tree -h /tmp/partitions
part-00000
part-00001
part-00002
part-00003
...
_SUCCESS
```

Чтобы избежать рисков, для _алгоритма логического вывода_ по умолчанию установлен тип `String`, который ==трудно поддается оптимизации и обработке== [[Литература#^61ef72]]<c. 516>.

