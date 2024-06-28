Детали можно найти здесь https://sparkbyexamples.com/spark/spark-sql-performance-tuning-configurations/

Управлять конфигурацией можно, например, через `spark-submit`
```bash
$ spark-submit \
    --conf "key=value" \
    --conf "key=value"
```
### Use Columnar Format When Caching

Рекомендуется при _кэшировании данных_ DataFrame / SQL использовать _колоночный формат хранения данных в памяти_ (in-memory columnar format). 

Когда выполняются операции над столбцами в DataFrame / SQL, Spark извлекает только необходимые, что приводит к меньшему количеству операций поиска и меньшему использованию памяти.

Включить поддержку _колоночного хранения данных в памяти_ можно так
```scala
spark.conf.set("spark.sql.inMemoryColumnarStorage.compressed", true)
```
### Spark Cost-Based Optimizer

При работе с несколькими соединениями (multiple joins) рекомендуется использовать Cost-Based Optimizer, поскольку он улучшает план выполнения запроса, основываясь на статистике таблиц и столбцов.

Этот режим поддерживается по умолчанию, но если вдруг он выключен, включить его можно так
```scala
spark.conf.set("spark.sql.cbo.enabled", true)
```

NB! Перед выполнением множественного соединения, следует запустить команду `ANALYZE TABLE` https://spark.apache.org/docs/3.0.0-preview/sql-ref-syntax-aux-analyze-table.html для всех столбцов, которые участвуют в соединении. Эта команда собирает статистику по таблицам и столбцам для Cost-Based Optimizer.
```sql
ANALYZE TABLE table_name COMPUTE STATISTICS FOR COLUMNS col1, col2
```
### Use Optimal value for Shuffle Partitions

Когда мы выполняем операцию, которая вызывает _перетасовку_ -- shuffling -- `groupByKey()`, `reduceByKey()`, `join()`, `groupBy()` etc. -- Spark по умолчанию создает 200 партиций. Так как по умолчанию
```scala
spark.conf.set("spark.sql.shuffle.partitions", 200)
```

В большинстве случаев это значение создает проблемы с производительностью. Если набор данных большой, то следует задать более высокое значение, а в случае небольшого набора данных -- меньшее.
```scala
spark.conf.set("spark.sql.shuffle.partitions", 30) //Default value is 200
```

Перетасовка в Spark (Spark Shuffle) -- это очень затратная операция, так как:
- вызывает дисковый ввод-вывод,
- сериализацию-десериализацию,
- сетевой ввод-вывод.
### Use Broadcast Join when your Join data can fit in memory

Broadcast-hash-join -- эффективная стратегия соединения, но она может быть применена только, если одна из соединяемых таблиц значительно меньше другой и помещается в памяти в пределах порога, который можно задать так
```scala
//100 MB by default
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10485760) 
```
### Spark 3.0 -- Using coalesce() & repartition() on SQL

В контексте Spark SQL теперь для изменения числа партиций можно использовать `COALESCE`, `REPARTITION` и `REPARTITION_BY_RANGE`
```sql
SELECT /*+ COALESCE(3) */ * FROM EMP_TABLE
SELECT /*+ REPARTITION(3) */ * FROM EMP_TABLE
SELECT /*+ REPARTITION(c) */ * FROM EMP_TABLE
SELECT /*+ REPARTITION(3, dept_col) */ * FROM EMP_TABLE
SELECT /*+ REPARTITION_BY_RANGE(dept_col) */ * FROM EMP_TABLE
SELECT /*+ REPARTITION_BY_RANGE(3, dept_col) */ * FROM EMP_TABLE
```
### Spark 3.0 -- Enable Adaptive Query Execution

_Адаптивное выполнение запроса_ (Adaptive Query Execution) -- это фича Spark 3.0, которая повышает эффективность выполнения запроса за счет повторной оптимизации плана выполнения на основе статистики, собранной на каждом этапе.

Включить можно так
```scala
spark.conf.set("spark.sql.adaptive.enabled", true)
```
### Spark 3.0  -- Coalescing Post Shuffle Partitions

В Spark 3.0 после каждого этапа (stage) задания (job), Spark динамически определяет оптимальное число партиций на основе метрик завершенных этапов (stage). Включить это можно так
```scala
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", true)
```
### Spark 3.0 -- Optimizing Skew Join

Иногда данные неравномерно распределяются по партициям -- это называет перекосом данных. На таких партициях операции соединения выполняются очень медленно. Включив AQE, Spark проверяет статистику этапов и проверяет есть ли скошенные соединения. Если скошенные соединени обнаруживаются, Spark оптимизирует их, разбивая большие партиции на более мелкие (в соответствие с размером соответствующих партиций другой таблицы)
```scala
spark.conf.set("spark.sql.adaptive.skewJoin.enabled",true)
```

