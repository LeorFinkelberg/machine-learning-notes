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

Разумеется данные можно разбить и с помощью $k$-блочной перекрестной проверки
```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator

lr = LogisticRgression(maxIter=10, featureCol="features", labelCol="label")

paramGrid = ParamGridBuilder() \
    .addGrid(lr.regParam, [0.1, 0.01]) \
    .addGrid(lr.elasticNetParam, [0.0, 0.5, 1.0]) \
    .build()

crossval_clf = CrossValidator(
	estimator=lr,
	estimatorParamMaps=paramGrid,
	evaluator=BinaryClassificationEvaluator(),
	numFolds=3,
)

model = crossval_clf.fit(assembled_df)
```

Здесь 3 параметра для ElasticNet, 2 параметра для регуляризации и 3 фолда. Таким образом $3 * 2 * 3 = 18$ моделей. Перекрестная проверка вычислительно дорогая, но полезная для подбора гиперпараметров.

Еще бывает полезно вычислить _индекс стабильности популяции_ (Population Stability Index, PSI). Он позволяет оценить стабильность признаков на обучающем и валидационном поднаборах. Если $PSI < 0.1$, то считается, что у признака низкая вариативность и его можно использовать в матрице признакового описания объекта [[Литература#^ef7d57]],<p. 214>.

Коэффициент детерминации $R^2$ -- это смещенная оценка, так как зависит от числа признаков. Если добавить случайный признак или зашумленный, то $R^2$ возрастет. Поэтому лучше использовать _скорректированную оценку_ $R^2$ [[Литература#^ef7d57]]<p. 228>
$$
R^2_{adj} = 1 - \dfrac{(1 - R^2)\,(N - 1)}{N - p - 1},
$$
где $R^2$ -- коэффициент детерминации, $N$ -- число наблюдений, $p$ -- число признаков.

Кривая точность-полнота (Precision-Recall Curve) полезна для _несбалансированных наборов данных_. 
```python
from sklarn.metrics import precision_recall_curve, average_precision_score

rand_precision, rand_recall, _ = precision_recall_curve(rand_y, rand_prob)
pr = average_precision_score(rand_y, rand_prob)
```

_Коэффициент корреляции Метьюса_ (Matthews correllation coefficient, [MCC](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.matthews_corrcoef.html)) используется и в задачах _бинарной классификации_, и в задачах _мультиклассовой классификации_. В документации sklearn говориться, что MCC считается _сбалансированной метрикой_ и может применяться даже в тех случаях, когда классы сильно перекошены (имеют дисбаланс по числу экземпляров). MCC изменяется в диапзоне от -1 до +1. Значение коэффициента +1 означает идеальный прогноз, 0 -- случайное гадание, а -1 означает обращенный прогноз.