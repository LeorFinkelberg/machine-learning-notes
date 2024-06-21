### UDF 

User Defined Functions (UDF) работают на уровне столбцов кадра данных. UDF может принимать от 0 до 22 агрументов и всегда возвращает только одно значение. Возвращаемое значение представляет собой значение, сохраняемое в столбце.

Код выполняется на рабочих узлах! Это означает, что код UDF должен быть _сериализуемым_ (для передачи на рабочий узел) [[Литература#^61ef72]]<c. 380>

Spark берет на себя часть работы, отвечающей за передачу кода. Если для UDF требуется внешние библиотеки, то необходимо обязательно обеспечить их развертывание на рабочих узлах.

Spark помещает все преобразования в ориентированный ациклический граф перед вызовом действия. В момент вызовы действия Spark обращается к Catalyst для оптимизации сформированного ориентированного ациклического графа пред выполнением указанных в нем задач.

==Catalyst не видит содержимое пользовательской функции (UDF)== и воспринимает ее как черный ящик. ==В Spark _нет возможности_ оптимизировать пользовательские функции (UDF)==. Кроме того, Spark _не может проанализировать контекст_, в котором вызвана UDF. Если до или после вызова UDF выполняются вызовы API кадра данных, то Catalyst не сможет оптимизировать весь процесс преобразования в целом [[Литература#^61ef72]]<c. 381>.

Сериализация UDF на рабочие узлы выполняется незаметно для пользователя. 

Простой пример
```python
from pyspark.sql import functions as F
from pyspark.sql.types import IntegerType

# Варианты определения функций
slen = F.udf(lambda s: len(s), IntegerType())

@udf
def to_upper(s):
    if s is not None:
        return s.upper()

@udf(returnType=IntegerType())
def add_one(x):
    if x is not None:
        return x + 1

df.select(slen("name"), to_upper("name"), add_one("age"))
```

Как отмечается в [[Литература#^61ef72]]<c. 394>, UDF не является наилучшим решением любой задачи, и у них есть следующие ограничения:
- _сериализуемость_ -- класс, реализующий саму функцию, ==обязательно должен быть сериализуемым==. Между тем некоторые артефакты Java, статические переменные не являются сериализуемыми.
- выполнение на рабочих узлах -- сама функция UDF выполняется на _рабочих узлах_, следовательно, ресурсы, среда выполнения и зависимости, необходимые для работы этой функции, также должны размещаться на рабочих узлах.
- _черный ящик для механизма оптимизации_ -- Catalyst -- весьма важная компонента, которая оптимизирует ориентированный ациклический граф преобразований. Но ==Catalyst ничего не знает о том, что делает функция (UDF)==.
- не допускается полиморфизм.

Пример _UDF_
```python
from pyspark.sql import functions as F
from pyspark.sql.types import *

@F.udf(returnType=DoubleType())
def make_sum(col1, col2) -> float:
    return col1 + col2

df.select("id", make_sum("col1", "col2")).show()
```

UDF можно применять и в контексте PySpark SQL, но предварительно пользовательскую функцию требуется зарегистрировать
```python
# Обычная Python-функция
def convert_string_to_code(col_name: str) -> int:
    ...

spark.udf.register(
	"convert_string_to_code",
	convert_string_to_code,
	IntegerType()
)

spark.sql("select x, convert_string_to_code(color) from my_table;")
```

### UDAF 

Напрямую Spark User-Defined Aggregate Functions (_UDAF_) не поддерживает, поэтому приходится выполнять агрегацию вручную
```python
@F.udf(returnType=DoubleType())
def make_agg(marker: str, col1: float, col2: float) -> float:
    if marker in ("blue", "red"):
        return col1 + col2 ** 2 
    elif marker == "green":
        return col1 ** 2 - 10
    else:
        return 2 * col1

df.select("id", make_agg("col3", "col1", "col2").alias("result")).show()
```

### UDTF

Подробности по UDTF можно найти на странице документации [Python User-Defined Table Functions (UDTFs)](https://spark.apache.org/docs/latest/api/python/user_guide/sql/python_udtf.html).

Начиная с версии `3.5.0`, PySpark поддерживает User-Defined Table Functions (UDTFs) -- новый тип пользовательских функций. В отличие от скалярных функций, которые возвращаю одно единственное значение для каждого вызова, каждая UDTF возвращает таблицу в качестве результата. Каждая UDTF может 0+ аргументов. Эти аргументы могут быть либо скалярными выражениями, либо таблицами.

Общий шаблон
```python
import typing as t

class PythonUDTF:
    def __init__(self) -> None:
        """
		Это необязательный метод, который инициализирует UDTF.
		Вызывается один раз при создании экземпляра UDTF на стороне исполнителя.
        """
        pass

    def eval(self, *args: t.Any) -> t.Iterator[t.Any]:
        """
		Вычисляет функцию с указанными аргументами. Это обязательный метод
		и должен быть реализован.
        """
        pass

    def terminate(self) -> t.Iterator[t.Any]:
        """
		Этот метод необязательный и вызывается,
		когда UDTF закончила обработку всех строк

        Examples
        ---------
        def terminate(self) -> t.Iterator[t.Any]:
            yield "done", None
        """
        pass
```

Пример использования UDTF
```python
from pyspark.sql.types import *
from pyspark.sql import functions as F

# можно было бы передать строку
# return_type = "num: int, squared: int"
return_type = StructType([
	StructField("num", IntegerType(), False),
	StructField("squared", IntegerType(), False),
])

@F.udtf(returnType=return_type)
class SquaredNumber:
    def eval(self, start: int, end: int):
        for num in range(start, end + 1):
            yield num, num ** 2

SquaredNumber(F.lit(1), F.lit(3)).show()
```

UDTF так же могут использоваться в контексте SQL-запросов. Для этого нужно класс зарегистрировать.

Пример использования UDTF с простым строковым аргументов
```python
import typing as t
from pyspark.sql.types import *
from pyspark.sql import functions as F

@F.udtf(returnType="word: string")
class WordSplitter:
    def __init__(self) -> None:
        self.prefix = 0

    def eval(text: str) -> t.Iterator[str]:
        for word in text.split():
            yield f"{self.prefix}-{word.strip()}",  # одноэлементый кортеж

    def termination(self) -> t.Iterator[str]:
        yield "Done",  # одноэлементный кортеж

spark.udtf.register("split_words", WordSplitter)

spark.sql("SELECT * FROM split_words('hello word')") \
    .withColumnRenamed("world", "result").show()
# Вернет таблицу из двух строк

spark.sql(
	"SELECT * FROM VALUES ('Apache Spark'), ('Fortran Scala') table(text),"
	"LATERAL split_words(text)"
)
```

UDTF можно использовать вместе с латеральными подзапросами, которые, как известно, позволяют ссылаться на столбцы и псевдонимы в контексте ключевого слова `FROM`
```python
spark.sql(
	"SELECT * FROM VALUES ('Hello World'), ('Apache Spark') vtable(text), "
    # перед ключевым словом `LATERAL` обязательно нужна запятая!
	"LATERAL split_words(text)"
).show()
# +------------+------+
# |        text|  word|
# +------------+------+
# | Hello World| Hello|
# | Hello World| World|
# |Apache Spark|Apache|
# |Apache Spark| Spark|
# +------------+------+
```

Apache Arrow -- это организованный в памяти колончатый формат данных, который используется в Spark для эффективной передачи данных между процессами Java и Python. По умолчанию для Python UDTFs Apache Arrow отключен.

Arrow может повысить производительность, когда UDTF для каждой входной строки генерирует большую таблицу результатов.

Чтобы включить поддержку оптимизации Arrow, можно выставить параметр конфигурации в `true`
```bash
spark.sql.execution.pythonUDF.arrow.enabled = true
```

Также можно выставить параметр `useArrow` в `True` при определении UDTF
```python
@F.udtf(returnType="c1: int, c2: int", useArrow=True)
class PlusOne:
    def eval(self, x: int):
        yield x, x + 1
```

Кроме того Python UDTFs могут принимать в качестве входных аргументов таблицы и могут сочетаться с скалярными входными аргументами. По умолчанию разрешается использовать только один табличный аргумент (главным образом из соображений производительности). Если требуется использовать более одного табличного аргумента, то следует выставить параметр `spark.sql.tvf.allowMultipleTableArguments.enabled` в `true`.
```python
from pyspark.sql.functions import udtf
from pyspark.sql.types import Row

@F.udtf(returnType="id: int")
class FilterUDTF:
    def eval(self, row: Row):
        # Таблица, которая подается на вход функции `filter_udtf`
        # будет обрабатываться построчно
		row_id = row["id"]
        if row_id > 5:
            yield row_id,

spark.udtf.register("filter_udtf", FilterUDTF)

spark.sql("SELECT * FROM filter_udtf(TABLE(SELECT * FROM range(10)))").show()
```

Здесь `TABLE(SELECT * FROM range(10))` возвращает одностолбцовую таблицу, каждая строка которой представляет собой одноэлементный Row-объект. То есть в качестве входного аргумента функции `filter_udtf()` можно передать таблицу, которая будет обрабатываться построчно.

Еще один пример
```python
from datetime import datetime, timedelta
from pyspark.sql import functions as F

@F.udtf(returnType="date: string")
class DateExpander:
    date_format = "%Y-%m-%d"
    def eval(self, start_date: str, end_date: str):
        current = datetime.strptime(start_date, date_format)
        end = datetime.strptime(end_date, date_format)

        while current <= end:
            yield current.strftime(date_format),
            current += timedelta(days=1)

DateExpander(F.lit("2023-02-25"), F.lit("2023-03-01")).show()
```

И еще один пример
```python
from pyspark.sql.functions as F

@F.utdf(returnType="cnt: int")
class CounterUDTF:
    def __init__(self):
        self.count = 0

    def eval(self, x: int):
        self.count += 1

    def terminate(self):
        yield self.count, # одноэлементный кортеж

spark.udtf.register("count_udtf", CountUDTF)
spark.sql("SELECT * FROM range(0, 10, 1, 1), LATERAL count_udtf(id)").show()
```