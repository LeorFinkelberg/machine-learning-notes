Детали можно найти на странице документации [Deploy MLflow models with InferenceService](https://kserve.github.io/website/latest/modelserving/v1beta1/mlflow/v2/).

### Training

Сначала нужно обучить модель и сохранить ее, вызвав `log_model()`
```python
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

Обученная модель сериализуется в виде
```bash
model/
├── MLmodel
├── model.pkl
├── conda.yaml
└── requirements.txt
```

### Testing locally

Для локального тестирования модели, установим сервер логического вывода MLServer
```bash
$ pip install mlserver mlserver-mlflow
```

Подготовим файл конфигурации модели
```bash
# model-settings.json
{
  "name": "mlflow-wine-classifier",
  "version": "v1.0.0",
  "implementation": "mlserver_mlflow.MLflowRuntime"
}
```

Теперь можно запустить сервер
```bash
$ mlserver start .
```

### Deploy with InferenceService

Здесь предполагается, что мы протестировали несколько моделей локально, отправляя запросы на локальный MLServer Seldon, выбрали наиболее производительную модель и поместили ее на удаленное объектное хранилище GCS
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlflow-v2-wine-classifier"
spec:
  predictor:
    model:
      modelFormat:
        name: mlflow
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/mlflow/wine"
```

NB: В конфигурационном файле KServe можно указывать не только URI модели на удаленном объектном хранилище, но и ==URI до Docker-образа в Docker-репозитории==. См. детали на странице [Create Deployment Configuraion](https://mlflow.org/docs/latest/deployment/deploy-model-to-kubernetes/tutorial.html#create-deployment-configuration).

Применяем yaml-файл
```bash
kubectl apply -f mlflow-new.yaml
```
### Testing deployed model

Протестируем развернутую модель, предварительно подготовив файл полезной нагрузки
```bash
{
  "parameters": {
    "content_type": "pd"
  },
  "inputs": [
      {
        "name": "fixed acidity",
        "shape": [1],
        "datatype": "FP32",
        "data": [7.4]
      },
      {
        "name": "volatile acidity",
        "shape": [1],
        "datatype": "FP32",
        "data": [0.7000]
      },
      {
        "name": "citric acid",
        "shape": [1],
        "datatype": "FP32",
        "data": [0]
      },
      {
        "name": "residual sugar",
        "shape": [1],
        "datatype": "FP32",
        "data": [1.9]
      },
      {
        "name": "chlorides",
        "shape": [1],
        "datatype": "FP32",
        "data": [0.076]
      },
      {
        "name": "free sulfur dioxide",
        "shape": [1],
        "datatype": "FP32",
        "data": [11]
      },
      {
        "name": "total sulfur dioxide",
        "shape": [1],
        "datatype": "FP32",
        "data": [34]
      },
      {
        "name": "density",
        "shape": [1],
        "datatype": "FP32",
        "data": [0.9978]
      },
      {
        "name": "pH",
        "shape": [1],
        "datatype": "FP32",
        "data": [3.51]
      },
      {
        "name": "sulphates",
        "shape": [1],
        "datatype": "FP32",
        "data": [0.56]
      },
      {
        "name": "alcohol",
        "shape": [1],
        "datatype": "FP32",
        "data": [9.4]
      }
  ]
}
```

Теперь можно послать запрос
```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice mlflow-v2-wine-classifier -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./mlflow-input.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/mlflow-v2-wine-classifier/infer
```

