###  Разбиение данных на обучение и тест

Разбить исходный набор данных на обучающий и тестовый поднаборы в простейшем случае можно так
```python
from pyspark.sql.dataframe import DataFrame

X: DataFrame
Xy_train, Xy_test = X.randomSplit([0.7, 0.3], seed=42)

# Можно разбить сразу на обучение, валидацию и тест
Xy_train, Xy_val, Xy_test = X.randomSplit([0.7, 0.2, 0.1], seed=42)
```

Также можно разбить обучающий набор данных на обучение и тест с подбором гиперпараметров. Здесь предполагается, что мы с помощью класса [`VectorAssembler`](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.feature.VectorAssembler.html) получили кадр даных со столбцом `features`
```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml.classification import LigisticRegression
from pyspark.ml.tuning import ParamGridBuidler, TrainValidationSplit

lr = LogisticRegression(maxIter=10, featureCol="features", labelCol="label")

param_grid = ParamGridBuilder() \
    .addGrid(lr.regParam, [0.1, 0.01]) \
    .addGrid(lr.elasticNetParam, [0.0, 0.5, 1.0]) \
    .build()

# 70% данных под обучение, остальные под тест
train_valid_clf = TrainValidationSplit(
	estimator=lr,
	estimatorParamMaps=paramGrid,
	evaluator=BinaryClassificationEvaluator(),
	trainRatio=0.7,
)

model = train_valid_clf.fit(assembled_df)
```

TODO: Способ стратифицированного разбиения набора данных под вопросом. Чистого, однозначного способа не нашел.

Еще бывает полезно вычислить _индекс стабильности популяции_ (Population Stability Index, PSI). Он позволяет оценить стабильность признаков на обучающем и валидационном поднаборах. Если $PSI < 0.1$, то считается, что у признака низкая вариативность и его можно использовать в матрице признакового описания объекта [[Литература#^ef7d57]],<p. 214>.