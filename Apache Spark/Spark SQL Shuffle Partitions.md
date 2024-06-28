Детали можно найти здесь https://sparkbyexamples.com/spark/spark-shuffle-partitions/

Перетасовка в Spark это очень дорогая операция, поскольку она вызывает перемещение данных между партициями по всему кластеру и ее следует по возможности избегать.

Перетасовку данных в Spark вызывают преобразования типа `groupBy()`, `groupByKey()`, `reduceByKey()`, `join()` etc. Перетасовка обходится дорого, так как вызывает дисковый ввод-вывод, сетевой ввод-вывод и сериализацию-десериализацию.

Когда мы выполняем `reduceByKey()` для агрегации данных по ключам, Spark выполняет следующее:
- Spark запускает _задачи отображения_ (map tasks) на всех партициях, которые группируют значения по ключу,
- Результаты этих задач сохраняются в памяти,
- Если данные не помещаются в памяти, Spark их сохраняет на диск,
- Spark перетасовывает данные между партициями; иногда перетасованные данные могут сохраняться на диске для повторного использования,
- Запускает сборщик мусора,
- И, наконец, запускает _задачи агрегации_ (reduce tasks) на ключах для каждой партиции.
### Spark RDD Shuffle

Перетасовку данных в Spark RDD вызывают `repartition()`, `groupByKey()`, `reduceByKey()`, `cogroup()` и `join()`, но не `countByKey()`.
```scala
val spark:SparkSession = SparkSession.builder()
    .master("local[5]")
    .appName("SparkByExamples.com")
    .getOrCreate()

val sc = spark.sparkContext

val rdd:RDD[String] = sc.textFile("src/main/resources/test.txt")

println("RDD Parition Count :"+rdd.getNumPartitions)
val rdd2 = rdd.flatMap(f=>f.split(" "))
  .map(m=>(m,1))

//ReduceBy transformation
val rdd5 = rdd2.reduceByKey(_ + _)

println("RDD Parition Count :"+rdd5.getNumPartitions)

#Output
RDD Parition Count : 3
RDD Parition Count : 3
```

Здесь хотя `reduceByKey()` и запускает перетасовку данных, это не изменяет количество партиций, так как размер партиции наследуется от родительского RDD.

### Spark SQL DataFrame Shuffle

В отличие от RDD, _Spark SQL DataFrame API_ увеличивает количество партиций, когда операции вызывают перетасовку. В случае DataFrame перетасовку вызывают `join()` и все агрегатные функции (`approx_count_distinct()`, `avg()`, `kurtosis()`, `min()`, `sum()`, `stddev()`
```scala
import spark.implicits._

val simpleData = Seq(("James","Sales","NY",90000,34,10000),
    ("Michael","Sales","NY",86000,56,20000),
    ("Robert","Sales","CA",81000,30,23000),
    ("Maria","Finance","CA",90000,24,23000),
    ("Raman","Finance","CA",99000,40,24000),
    ("Scott","Finance","NY",83000,36,19000),
    ("Jen","Finance","NY",79000,53,15000),
    ("Jeff","Marketing","CA",80000,25,18000),
    ("Kumar","Marketing","NY",91000,50,21000)
  )
val df = simpleData.toDF("employee_name","department","state","salary","age","bonus")

val df2 = df.groupBy("state").count()

println(df2.rdd.getNumPartitions)  // 200
```

Spark автоматически увеличивает число партиций после операций, вызывающих перетасовку (`join()` и агрегатные функции).  Изменить это значение можно так
```scala
spark.conf.set("spark.sql.shuffle.partitions", 100)
```

Когда мы работаем с небольшими наборами данных имеет смысл уменьшить количество партиций. В противном случае придется работать с большим числом партиций, каждая из которых будет содержать небольшое число записей.

Подобрать правильное количество перетасовочных партиций всегда сложно и предполагает множество запусков для определения оптимального значения.
