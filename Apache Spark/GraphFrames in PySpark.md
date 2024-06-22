Подробности можно найти на странице https://graphframes.github.io/graphframes/docs/_site/quick-start.html. 

Чтобы запустить Spark-приложение, в котором используется графовые кадры данных следует указать пакет `graphframes`
```bash
$ spark-submit --packages graphframes:graphframes:0.8.3-spark3.5-s_2.12
```

Простой пример
```python
from pyspark.sql import SparkSession
from graphframes import *

spark = SparkSession.builder.getOrCreate()

# Create a Vertex DataFrame with unique ID column "id"
v = spark.createDataFrame([
  ("a", "Alice", 34),
  ("b", "Bob", 36),
  ("c", "Charlie", 30),
], ["id", "name", "age"])
# Create an Edge DataFrame with "src" and "dst" columns
e = spark.createDataFrame([
  ("a", "b", "friend"),
  ("b", "c", "follow"),
  ("c", "b", "follow"),
], ["src", "dst", "relationship"])

# Create a GraphFrame
g = GraphFrame(v, e)

# Query: Get in-degree of each vertex.
g.inDegrees.show()

# Query: Count the number of "follow" connections in the graph.
g.edges.filter("relationship = 'follow'").count()

# Run PageRank algorithm, and show results.
results = g.pageRank(resetProbability=0.01, maxIter=20)
results.vertices.select("id", "pagerank").show()
```