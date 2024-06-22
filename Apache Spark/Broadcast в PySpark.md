Детали здесь https://sparkbyexamples.com/pyspark/pyspark-broadcast-join-with-example/

Иногда возникает необходимость соединить маленькую таблицу (сотни строк) с большой таблицей (миллионы строк). Чтобы сделать это эффективно, Spark предлагает функцию `.broadcast()`. Таблица меньшего размера транслируется на все рабочие узлы кластера. Это повышает производительность, так как Spark'у не требуется перетасовывать большую таблицу.
```python
df_large.join(F.broadcast(df_small), df_large["id"] == df_small["id"], "left_semi").count()
```

Ремарка: _кардинальность_ -- число уникальных значений переменной. Таким образом, вещественные (непрерывные) переменные можно считать высококардинальными [[Литература#^ef7d57]]<p. 95>.

Традиционные соединения требуют обычно занимают больше времени, так как требуют перетасовки данных и данные всегда собираются на драйвере!

PySpark определяет `pyspark.sql.functions.broadcast()` для трансляции меньшей таблицы для соединения с большей таблицей. Как известно PySpark разбивает данные по различным узлам для параллельной обработки. Когда есть два кадра данных, данные каждого из них будут распределены по всему кластеру. Поэтому когда выполняется обычное соединение, PySpark требует _перетасовки данных_ (shuffle data). Перетасовка необходима, так как записи, связанные с соответствующим значением атрибута соединения могут лежать на разных узлах, а для выполнения соединения требутеся, чтобы значения атрибута соединения лежали на одном и том же узле. Поэтому традиционные соединений очень затратные.

PySpark транслирует (broadcast) меньшую таблицу на все рабочие узлы, а рабочие узлы сохраняют ее в своей памяти. Большая таблица разделяется на фрагменты и распределяется по рабочим узлам. Так что теперь PySpark может выполнить соединение без перетасовки данных, так как у каждого рабочего узла есть нужные данные.

NB! Чтобы выполнять трансляцию меньшей таблицы на рабочие узлы, эта меньшая таблица должна помещаться в памяти драйвера и рабочих узлов. В противном случае очень вероятны ошибки нехватки памяти.

Типы broadcast joins:
- Broadcast Hash Join: строит хеш-таблицу в памяти драйвера, а затем распространяет ее на все рабочие узлы.
- Broadcast nested loop join.

Можно указать максимальный размер кадра данных, когда последний все еще считается маленьким и может транслироваться (broadcast) на рабочие узлы в качестве меньшей таблицы
```python
#Enable broadcast Join and 
#Set Threshold limit of size in bytes of a DataFrame to broadcast
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 104857600)

#Disable broadcast Join
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
```

Можно вызвать метод `.explain(extended=False)` на результате соединения и изучить план выполнения запроса. Например,
```bash
= Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- BroadcastHashJoin [coalesce(UNIT#543, ), isnull(UNIT#543)], [coalesce(code#560, ), isnull(code#560)], Inner, BuildRight, false
   :- GlobalLimit 2000
   :  +- Exchange SinglePartition, ENSURE_REQUIREMENTS, [id=#313]
   :     +- LocalLimit 2000
   :        +- FileScan parquet [NAME#537,STATION#538,LATITUDE#539,LONGITUDE#540,ELEVATION#541,DATE#542,UNIT#543,TAVG#544] Batched: true, DataFilters: [], Format: Parquet, Location: InMemoryFileIndex(1 paths)[dbfs:/mnt/training/weather/StationData/stationData.parquet], PartitionFilters: [], PushedFilters: [], ReadSchema: struct<NAME:string,STATION:string,LATITUDE:float,LONGITUDE:float,ELEVATION:float,DATE:date,UNIT:s...
   +- BroadcastExchange HashedRelationBroadcastMode(ArrayBuffer(coalesce(input[0, string, true], ), isnull(input[0, string, true])),false), [id=#316]
      +- LocalTableScan [code#560, realUnit#561]
```

Читаем справа-налево и начинаем с узла с наибольшим отступом. Таким образом здесь:
- Первым шагом вычитывается parquet-файл и создается большой кадр данных с ограничением на число записей.
- Затем выполняется `BroadcastHashJoin` между маленькой и большой таблицами.

Даже если специально не определять меньшую таблицу как broadcast-таблицу, PySpark все равно автоматически таблицу меньшего размера поместит в память рабочего узла.