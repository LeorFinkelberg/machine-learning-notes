Детали можно найти на странице документации [Serving MLflow models](https://mlserver.readthedocs.io/en/latest/examples/mlflow/README.html#examples-mlflow-readme--page-root).

### Обучение модели

Первым шагом обучим и сериализуем MLflow-модель. Для этого рассмотрим пример с моделью линейной регрессии
```python
# %load src/train.py
# Original source code and more details can be found in:
# https://www.mlflow.org/docs/latest/tutorials-and-examples/tutorial.html

# The data set used in this example is from
# http://archive.ics.uci.edu/ml/datasets/Wine+Quality
# P. Cortez, A. Cerdeira, F. Almeida, T. Matos and J. Reis.
# Modeling wine preferences by data mining from physicochemical properties.
# In Decision Support Systems, Elsevier, 47(4):547-553, 2009.

import warnings
import sys

import pandas as pd
import numpy as np
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
from sklearn.model_selection import train_test_split
from sklearn.linear_model import ElasticNet
from urllib.parse import urlparse
import mlflow
import mlflow.sklearn
from mlflow.models.signature import infer_signature

import logging

logging.basicConfig(level=logging.WARN)
logger = logging.getLogger(__name__)


def eval_metrics(actual, pred):
    rmse = np.sqrt(mean_squared_error(actual, pred))
    mae = mean_absolute_error(actual, pred)
    r2 = r2_score(actual, pred)
    return rmse, mae, r2


if __name__ == "__main__":
    warnings.filterwarnings("ignore")
    np.random.seed(40)

    # Read the wine-quality csv file from the URL
    csv_url = (
        "http://archive.ics.uci.edu/ml"
        "/machine-learning-databases/wine-quality/winequality-red.csv"
    )
    try:
        data = pd.read_csv(csv_url, sep=";")
    except Exception as e:
        logger.exception(
            "Unable to download training & test CSV, "
            "check your internet connection. Error: %s",
            e,
        )

    # Split the data into training and test sets. (0.75, 0.25) split.
    train, test = train_test_split(data)

    # The predicted column is "quality" which is a scalar from [3, 9]
    train_x = train.drop(["quality"], axis=1)
    test_x = test.drop(["quality"], axis=1)
    train_y = train[["quality"]]
    test_y = test[["quality"]]

    alpha = float(sys.argv[1]) if len(sys.argv) > 1 else 0.5
    l1_ratio = float(sys.argv[2]) if len(sys.argv) > 2 else 0.5

    with mlflow.start_run():
        lr = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, random_state=42)
        lr.fit(train_x, train_y)

        predicted_qualities = lr.predict(test_x)

        (rmse, mae, r2) = eval_metrics(test_y, predicted_qualities)

        print("Elasticnet model (alpha=%f, l1_ratio=%f):" % (alpha, l1_ratio))
        print("  RMSE: %s" % rmse)
        print("  MAE: %s" % mae)
        print("  R2: %s" % r2)

        mlflow.log_param("alpha", alpha)
        mlflow.log_param("l1_ratio", l1_ratio)
        mlflow.log_metric("rmse", rmse)
        mlflow.log_metric("r2", r2)
        mlflow.log_metric("mae", mae)

        tracking_url_type_store = urlparse(mlflow.get_tracking_uri()).scheme
        model_signature = infer_signature(train_x, train_y)

        # Model registry does not work with file store
        if tracking_url_type_store != "file":

            # Register the model
            # There are other ways to use the Model Registry,
            # which depends on the use case,
            # please refer to the doc for more information:
            # https://mlflow.org/docs/latest/model-registry.html#api-workflow
            mlflow.sklearn.log_model(
                lr,
                "model",
                registered_model_name="ElasticnetWineModel",
                signature=model_signature,
            )
        else:
            mlflow.sklearn.log_model(lr, "model", signature=model_signature)

```

Теперь запускаем сценарий
```bash
$ python ./src/train.py
```

### Serving

Теперь можно сервировать модель. Для этого подготовим конфигурационный файл `model-settings.json`
```bash
# model-settings.json
{
  "name": "wine-classifier",
  "implementation": "mlserver_mlflow.MLflowRuntime",
  "parameters": {
    "uri": "{model_path}"
  }
}
```

Запускаем MLServer
```bash
$ mlserver start .
```

Посылаем POST-запрос
```python
import requests

inference_request = {
    "inputs": [
        {
          "name": "fixed acidity",
          "shape": [1],
          "datatype": "FP32",
          "data": [7.4],
        },
        {
          "name": "volatile acidity",
          "shape": [1],
          "datatype": "FP32",
          "data": [0.7000],
        },
        {
          "name": "citric acid",
          "shape": [1],
          "datatype": "FP32",
          "data": [0],
        },
        {
          "name": "residual sugar",
          "shape": [1],
          "datatype": "FP32",
          "data": [1.9],
        },
        {
          "name": "chlorides",
          "shape": [1],
          "datatype": "FP32",
          "data": [0.076],
        },
        {
          "name": "free sulfur dioxide",
          "shape": [1],
          "datatype": "FP32",
          "data": [11],
        },
        {
          "name": "total sulfur dioxide",
          "shape": [1],
          "datatype": "FP32",
          "data": [34],
        },
        {
          "name": "density",
          "shape": [1],
          "datatype": "FP32",
          "data": [0.9978],
        },
        {
          "name": "pH",
          "shape": [1],
          "datatype": "FP32",
          "data": [3.51],
        },
        {
          "name": "sulphates",
          "shape": [1],
          "datatype": "FP32",
          "data": [0.56],
        },
        {
          "name": "alcohol",
          "shape": [1],
          "datatype": "FP32",
          "data": [9.4],
        },
    ]
}

endpoint = "http://localhost:8080/v2/models/wine-classifier/infer"  # <= NB
response = requests.post(endpoint, json=inference_request)

response.json()
```

Можно послать тот же POST-запрос для протокола MLflow
```python
import requests

inference_request = {
    "dataframe_split": {
        "columns": [
            "alcohol",
            "chlorides",
            "citric acid",
            "density",
            "fixed acidity",
            "free sulfur dioxide",
            "pH",
            "residual sugar",
            "sulphates",
            "total sulfur dioxide",
            "volatile acidity",
        ],
        "data": [[7.4,0.7,0,1.9,0.076,11,34,0.9978,3.51,0.56,9.4]]
    }
}

endpoint = "http://localhost:8080/invocations"  # <= NB
response = requests.post(endpoint, json=inference_request)

response.json()
```

Пример подготовки конфигурационного файла `model-settings.json` для модели из проекта Маяк
```python
...
def create_models_configs(models: list[tuple[str, ModelVersion]]):
    for tp in models:
        guid, model = tp
        logging.info("Create config for %s", model.name)
        settings = {
            "name": guid,
            "implementation": "mlserver_mlflow.MLflowRuntime",
            "parameters": {
                "uri": f"models:/{model.name}/{model.version}",
                "version": model.version,
            },
        }
        model_path = os.path.join("models", guid)
        json_path = os.path.join(model_path, "model-settings.json")
        if not os.path.exists(model_path):
            os.makedirs(model_path)
        with open(json_path, "w", encoding="UTF-8") as file:
            json.dump(settings, file, indent=4)
        if not os.path.exists(json_path):
            logging.error("Fail to create model-settings using this path %s", json_path)
...

if __name__ == "__main__":
    loaded_models = get_models()
    create_models_configs(loaded_models)
	mlserver_handler = subprocess.Popen(["mlserver", "start", "."])
    uvicorn.run(
        app, port=int(os.getenv("MLSERVER_UPDATE_PORT", "8000")), host="0.0.0.0"
    )
```

