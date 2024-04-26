MLflow Model это стандартный формат упаковки моделей машинного обучения. Формат определяет соглашения, которые позволяют сохранить модель в различных "вариантах".

Каждая MFlow Model представляет собой директорию, которая содержит различные файлы, включая файл `MLmodel`  в корне директории.

К примеру, `mlflow.sklearn` сохраняет модель в следующем виде
```bash
# Directory written by mlflow.sklearn.save_model(model, "my_model")
my_model/
├── MLmodel
├── model.pkl
├── conda.yaml
├── python_env.yaml
└── requirements.txt
```

А файл `MLmodel` описывает два варианта
```bash
time_created: 2018-05-25T17:28:53.35

flavors:
  sklearn:
    sklearn_version: 0.19.1
    pickled_model: model.pkl
  python_function:
    loader_module: mlflow.sklearn
```

Сохранить модель можно несколькими способами. Во-первых, MLflow поддерживает интерфейсы нескольких распространенных библиотек. Например, `mlflow.sklearn` содержит следующие функции `save_model`, `log_model` и `load_model` для моделей библиотеки scikit-learn. А, во-вторых, для создания и записи модели можно использовать класс `mlflow.models.Model`.

К примеру, сохранить и загрузить модель [LightGBM](https://mlflow.org/docs/latest/models.html#lightgbm-lightgbm) можно так
```python
from lightgbm import LGBMClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import mlflow
from mlflow.models import infer_signature

data = load_iris()

# Remove special characters from feature names to be able to use them as keys for mlflow metrics
feature_names = [
    name.replace(" ", "_").replace("(", "").replace(")", "")
    for name in data["feature_names"]
]
X_train, X_test, y_train, y_test = train_test_split(
    data["data"], data["target"], test_size=0.2
)
# create model instance
lgb_classifier = LGBMClassifier(
    n_estimators=10,
    max_depth=3,
    learning_rate=1,
    objective="binary:logistic",
    random_state=123,
)

# Fit and save model and LGBMClassifier feature importances as mlflow metrics
with mlflow.start_run():
    lgb_classifier.fit(X_train, y_train)
    feature_importances = dict(zip(feature_names, lgb_classifier.feature_importances_))
    feature_importance_metrics = {
        f"feature_importance_{feature_name}": imp_value
        for feature_name, imp_value in feature_importances.items()
    }
    mlflow.log_metrics(feature_importance_metrics)
    signature = infer_signature(X_train, lgb_classifier.predict(X_train))
    model_info = mlflow.lightgbm.log_model(
        lgb_classifier, "iris-classifier", signature=signature
    )

# Load saved model and make predictions
lgb_classifier_saved = mlflow.pyfunc.load_model(model_info.model_uri)
y_pred = lgb_classifier_saved.predict(X_test)
print(y_pred)
```

Пример обучения модели LightGBM, ее сохранения и чтения
```python
from lightgbm import LGBMClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import mlflow
from mlflow.models import infer_signature

data = load_iris()

# Remove special characters from feature names to be able to use them as keys for mlflow metrics
feature_names = [
    name.replace(" ", "_").replace("(", "").replace(")", "")
    for name in data["feature_names"]
]
X_train, X_test, y_train, y_test = train_test_split(
    data["data"], data["target"], test_size=0.2
)
# create model instance
lgb_classifier = LGBMClassifier(
    n_estimators=10,
    max_depth=3,
    learning_rate=1,
    objective="binary:logistic",
    random_state=123,
)

# Fit and save model and LGBMClassifier feature importances as mlflow metrics
with mlflow.start_run():
    lgb_classifier.fit(X_train, y_train)
    feature_importances = dict(zip(feature_names, lgb_classifier.feature_importances_))
    feature_importance_metrics = {
        f"feature_importance_{feature_name}": imp_value
        for feature_name, imp_value in feature_importances.items()
    }
    mlflow.log_metrics(feature_importance_metrics)
    signature = infer_signature(X_train, lgb_classifier.predict(X_train))
    model_info = mlflow.lightgbm.log_model(
        lgb_classifier, "iris-classifier", signature=signature
    )

# Load saved model and make predictions
lgb_classifier_saved = mlflow.pyfunc.load_model(model_info.model_uri)
y_pred = lgb_classifier_saved.predict(X_test)
print(y_pred)
```

Кроме всего прочего MLflow поддерживает платформу тестирования моделей машинного обучения [Giskard](https://docs.giskard.ai/en/latest/getting_started/index.html) (качество, безопасность и согласованность). Пример использования библиотеки Giskard в связке LightGBM можно найти [здесь](https://docs.giskard.ai/en/latest/reference/notebooks/insurance_prediction_lgbm.html).

##### Кастомизация моделей

MLflow два механизма кастомизации моделей:
- Custom Python Models
- Custom Flavors

