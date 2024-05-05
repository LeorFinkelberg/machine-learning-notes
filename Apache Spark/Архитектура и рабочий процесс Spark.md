- [ ] Приложение^[Приложение (Application) -- программа, которая состоит из _программы-драйвера_ и _исполнителей в кластере_] устанавливает соединение с ведущим узлом кластера Spark. После этого оно сообщает кластеру, что необходимо делать: приложение управляет кластером. В рассматриваемом здесь варианте ведущий узел (master) начинает с загрузки файла в формате CSV, а заканчивает сохранением полученного результата в базе данных.

#### Установление соединения с ведущим узлом

Для каждого _приложения_ (application) Spark первой операцией является установление соединения с _ведущим узлом_ (master) Spark и _создание сеанса_ (session) Spark
```java
SparkSession spark = SparkSession
  .builder()
  .appName("CSV to DB")
  .master("local")  // ведущий узел
  .getOrCreate();
```

#### Загрузка или потребления содержимого CSV-файла

Загрузка (loading), потребление (ingesting) и считывание (reading) -- это синонимы. Распределенное потребление (distributed ingestion) означает, что вы обращаетесь к рабочим узлам для одновременного выполнения операции потребления.

_Ведущий узел_ приказывает _рабочим узлам_ загрузить указанный файл. Spark будет потреблять этот CSV-файл распределенным методом. Файл обязательно должен находиться на совместно используемом накопителе (диске), в распределенной файловой системе (например, HDFS) или использоваться через специализированный механизм совместно используемой файловой системы, такой как Dropbox, Nextcloud или ownCloud.

_Раздел_ (partion) -- это специально выделенная ==область в памяти== _рабочего узла_. Каждый рабочий узел имеет свою собственную память, которую он использует через разделы (partions). 

_Рабочие узлы_ создают _задачи_ (tasks) для считывания CSV-файла. Каждый _рабочий узел_ получает доступ к собственной памяти и назначает _раздел памяти_ (партицию) для _задачи_.

Когда _задача_ (task) потребляет строки файла, она сохраняет считанные данные в выделенном _разделе памяти_ (partion).

Основные положения [[Литература#^61ef72]]<c. 68>:
- Набор данных никогда не остается целиком в _приложении_ (драйвере). Набор данных распределяется между _разделами памяти_ на _рабочих узлах_, а ==не на драйвере==.
- Весь процесс обработки данных выполняется на рабочих узлах,
- Рабочие узлы сохраняют обработанные данные из собственных разделов памяти в базу данных. 

Сводка:
- _Приложение_ является _драйвером_. 
- Драйвер устанавливает соединение с _ведущим узлом_ и _создает сеанс_. Данные будут прикреплены к этому сеансу. Сеанс определяет жизненный цикл данных на рабочих узлах.
- Ведущий узел может быть _локальным_ (ваш локальный компьютер) или _удаленным кластером_. При использовании локального режима не требуется создание кластера.
- Данные распределяются и обрабатываются в определенном разделе памяти.

#### Пример записи кадра данных в PostgreSQL

Рассмотрим простой пример с записью кадра данных в PostgreSQL. Предварительно создадим базу данных `spark_labs`
```bash
$ sudo systemctl start postgresql
$ psql -U postgres -p 5432
postgres=# \l  -- выведет список баз данных
postgres=# CREATE DATABASE spark_labs;
postgres=# \q
```
И скачаем драйвер JDBC https://jdbc.postgresql.org/download/  `postgresql-42.7.3.jar`. Теперь можно подготовить Python-сценарий
```python
# run.py

from pyspark.sql import SparkSession, functions as F

MODE = "overwrite"
TABLE_NAME = "ch02"
JDBC_DRIVER_PATH = (
	"/home/leorfinkelberg/Documents/postgres-jdbc/postgresql-42.7.3.jar"
)
# драйвер:диалект://хост:порт/имя_базы_данных
DB_URL = "jdbc:postgresql://localhost:5432/spark_labs"

prop = {
	"driver": "org.postgresql.Driver",
	"user": "postgres",
}

spark = SparkSession \
    .builder \
    .appName("CSV to DB") \
    .master("local[*]") \
    .config("spark.jars", JDBC_DRIVER_PATH) \
    .getOrCreate()

df = spark.read.csv(
	header=True,
	inferSchema=True,
	path="./file.csv"
)

df = df.withColumn(
	"full_name",
	F.concat(F.col("name"), F.lit(", "), F.col("lastname"))
)

df.write.jdbc(
	mode=MODE,
	url=DB_URL,
	table=TABLE_NAME,
	properties=prop,
)

spark.stop()
```

Запустить приложение можно так
```bash
(spark) $ export SPARK_CLASSPATH=~/Documents/postgres-jdbc/postgresql-42.7.3.jar
(spark) $ spark-submit --driver-class-path ${SPARK_CLASSPATH} run.py
```

Spark прочитать csv-файл, создать новый столбец, представляющий результат конкатенации двух столбцов и записать новый кадр данных в таблицу `ch02` базы данных `spark_labs` PostgreSQL.

Подключаемся к нашей базе данных `spark_labs`
```bash
$ psql -U postgres -p 5432
```
Проверяем таблицу `ch02` в базе данных `spark_labs`
```sql
postgres=# \l  -- в списке должна быть база данных spark_labs
postgres=# \conninfo
You are connected to database "postgres" as user "postgres" via socket in "/run/postgresql" at port "5432"
postgres=# \c spark_labs
You are connected to database "spark_labs" as user "postgres"
spark_labs=# \dt+  -- должны увидеть таблицу ch02
spark_labs=# TABLE ch02;
spark_labs=# \q
```

Можно изучить таблицу с помощью `psycopg2`
```python
import psycopg2

# DSN-формат
DB_URL = "postgresql:postgres@localhost:5432/spark_labs"

with psycopg2.connect(DB_URL) as con:
    cur = con.cursor()
    cur.execute("TABLE ch02;")

    for name, lastname, full_name in cur.fetchall():
        print(f"{name=}, {lastname=}, {full_name=})
```

Примечание:
- [DSN](https://python3.info/database/sqlalchemy/connection-dsn.html) -- Database Source Name
- Format: `db://user:password@host:port/database?opt1=val1&opt2=val2`
- RFC 1738 -- Uniform Resource Locator (URL)