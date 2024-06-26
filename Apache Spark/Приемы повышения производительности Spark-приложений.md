Детали можно найти здесь https://sparkbyexamples.com/spark/spark-performance-tuning/

### Use DataFrame/Dataset over RDD

Предпочтительнее использовать DataFrame/Dataset, а не RDD; даже когда мы используем DataFrame или Dataset на самом деле мы неявно работаем с RDD, но эффективным способом, за который отвечают Project Tungsten и оптимизатор Catalyst. Использование RDD напрямую приводит к проблемам с производительностью, так как Spark не знает как применить техники оптимизации, а RDD сериализует/десериализует данные, когда они распределяются по кластеру. Сериализация/десериализация очень дорогостоящие операции, так как большую часть времени мы тратим на сериализацию, а не на выполнение операции. Tungsten это компонент Spark SQL, который позволяет повысить производительность запросов, переписывая Spark-операции на байткод во время выполнения. Поскольку DataFrame это колоночный формат, содержащий дополнительную метаинформацию, Spark может выполнить некоторую оптимизацию запроса. Сначала оптимизатор Catalyst создает логический план выполнения запроса, а затем он выполняется движком Tungsten. 
### Use coalesce() over repartition()

Если нужно уменьшить количество партиций, то лучше использовать `coalesce()` , а не `repartition()`. Так как `coalesce()` не вызывает полной перетасовки данных.
- лучше использовать `mapPartitions()`  вместо `map`
```scala
 val df4 = df2.mapPartitions(iterator => {
    val util = new Util()
    val res = iterator.map(row=>{
        val fullName = util.combine(
            row.getString(0), row.getString(1), row.getString(2)
        )
        (fullName, row.getString(3),row.getInt(5))
    })
    res
  })

  val df4part = df4.toDF("fullName","id","salary")
```

### Use mapPartitions() over map()

### Use Serialized data format’s

Использовать сериализуемые форматы данных -- Avro (строковый формат данных особенно эффективный в связке с Kafka; сериализует данные в компактном двоичном формате; схема в виде JSON описывает имена полей и их типы), Kryo, Parquet (колоночный формат данных) etc. --, а не в CSV, JSON etc. 
### Avoid UDF’s (User Defined Functions)

Избегать пользовательских функций (UDFs) любой ценой. Нужно стараться избегать Spark/PySpark UDFs. UDF для Spark -- это "черный ящик", поэтому он не может применить оптимизацию. Прежде чем писать UDF, следует проверить список [встроенных функций Spark](https://sparkbyexamples.com/spark/spark-sql-functions/). 
### Persisting & Caching data in memory

Используя `cache()` и `persist()` можно попростить Spark закешировать промежуточные результаты и повторно использовать в следующих действиях 
```scala
df.where(col("State") === "PR").cache()
```

При кэшировании используется колоночный формат хранения данных в памяти. Настраивая параметр `batchSize`, можно повысить производительность 
```python
spark.conf.set("spark.sql.inMemoryColumnarStorage.compressed", true)
spark.conf.set("spark.sql.inMemoryColumnarStorage.batchSize",10000)
```
### Reduce expensive Shuffle operations

Перетасовка (shuffling) -- это механизм, который Spark использует для перераспределения данных между исполнителями. Spark запускает перетасовку, когда мы выполняем `groupByKey()`, `reduceByKey()`, `join()` и т.д. на RDD и DataFrame.

Перетасовка -- дорогостоящая операция, так как включает в себя: дисковый ввод-вывод, сетевой ввод-вывод и сериализацию-десериализацию. Мы не можем полностью избежать операций перетасовки, но по возможности стараемся сократить количество операций. 
```python
spark.conf.set("spark.sql.shuffle.partitions",100)
sqlContext.setConf("spark.sql.shuffle.partitions", "100") // older version
```
### Disable DEBUG & INFO Logging
