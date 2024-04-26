Детали можно найти на странице документации [Develop ML model with MLflow and deploy to Kubernetes](https://mlflow.org/docs/latest/deployment/deploy-model-to-kubernetes/tutorial.html).

В этом руководстве демонстрируется как использовать MLflow от начала и до конца:
- Обучение модели линейной регрессии с помощью MLflow Tracking.
- Подбор гиперпараметров.
- Упаковка весов модели и ее зависимостей как MLflow Model.
- Локальное тестирование модели с поддержкой MLServer командой `mlflow models serve`.
- Развертывание модели на кластере Kubernetes с использованием KServe и MLflow.

### Introduction: Scalable Model Serving with KServe and MLServer

MLflow предоставляет простой в использовании интерфейс для развертывания моделей с помощью сервера логического вывода на базе Flask. ТЕОРЕТИЧЕСКИ можно использовать тот же самый сервер логического вывода и на кластере Kubernetes, запаковав модель с помощью команды `mlflow models build-docker`, ОДНАКО этот подход может не подойти для промышленной эксплуатации. Дело в том, что Flask не проектировался под высокую нагрузку. Кроме того ручное управление несколькими экземплярами сервера логического вывода связано с большими сложностями.

#### Step 1: Installing MLflow and Additional Dependencies

Сначала нужно установить MLflow с дополнительными зависимостями
```bash
$ pip install mlflow[extras]
```

#### Step 2: Setting Up a Kubernetes Cluster

Если уже есть доступ к Kubernetes-кластеру, то KServe можно установить по инструкциям [официальной документации](https://github.com/kserve/kserve#hammer_and_wrench-installation). Краткое описание автономной установки (Standalone Installation):
- [Serverless Installation](https://kserve.github.io/website/master/admin/serverless/serverless/):  по умолчанию KServe устанавливает Knative для _бессерверного развертывания_ InferenceService.
- [Raw Deployment Installation](https://kserve.github.io/website/master/admin/kubernetes_deployment):  в сравнении с бессерверным развертыванием этот вариант проще, но не поддерживает канареечного развертывания и автомасштабирования.
- [ModelMesh Installation](https://kserve.github.io/website/master/admin/modelmesh/): для обеспечения высокого масштабирования, плотности и частого изменения моделей дополнительно можно установить ModelMesh.
- [Quick Installation](https://kserve.github.io/website/master/get_started/): устанавливает KServe на локальную машину.

Можно установить KServe на локальный кластер Kubernetes с помощью Kind (Kubernetes in Docker)^[https://kserve.github.io/website/master/get_started/#install-the-kserve-quickstart-environment].

Теперь когда есть кластер Kubernetes, запущенный в качестве цели для развертывания модели, можно приступить к созданию MLflow-модели.
#### Step 3: Training the Model

Простой пример модели
```python
import mlflow

import numpy as np
from sklearn import datasets, metrics
from sklearn.linear_model import ElasticNet
from sklearn.model_selection import train_test_split


def eval_metrics(pred, actual):
    rmse = np.sqrt(metrics.mean_squared_error(actual, pred))
    mae = metrics.mean_absolute_error(actual, pred)
    r2 = metrics.r2_score(actual, pred)
    return rmse, mae, r2


# Set th experiment name
mlflow.set_experiment("wine-quality")

# Enable auto-logging to MLflow
mlflow.sklearn.autolog()

# Load wine quality dataset
X, y = datasets.load_wine(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.25)

# Start a run and train a model
with mlflow.start_run(run_name="default-params"):
    lr = ElasticNet()
    lr.fit(X_train, y_train)

    y_pred = lr.predict(X_test)
    metrics = eval_metrics(y_pred, y_test)
```

Теперь у нас есть обученная модель и можно проверить что получилось с помощью MLflow UI
```bash
$ mlflow ui --port 5000
```

Открываем `http://localhost:5000`.

Теперь запустим процедуру подбора гиперпараметров
```python
from scipy.stats import uniform
from sklearn.model_selection import RandomizedSearchCV

lr = ElasticNet()

# Define distribution to pick parameter values from
distributions = dict(
    alpha=uniform(loc=0, scale=10),  # sample alpha uniformly from [-5.0, 5.0]
    l1_ratio=uniform(),  # sample l1_ratio uniformlyfrom [0, 1.0]
)

# Initialize random search instance
clf = RandomizedSearchCV(
    estimator=lr,
    param_distributions=distributions,
    # Optimize for mean absolute error
    scoring="neg_mean_absolute_error",
    # Use 5-fold cross validation
    cv=5,
    # Try 100 samples. Note that MLflow only logs the top 5 runs.
    n_iter=100,
)
					   
mlflow.sklearn.autolog()

mlflow.set_experiment("wine-quality")
# Start a parent run
with mlflow.start_run(run_name="hyperparameter-tuning"):
    search = clf.fit(X_train, y_train)

    # Evaluate the best model on test dataset
    y_pred = clf.best_estimator_.predict(X_test)
    rmse, mae, r2 = eval_metrics(y_pred, y_test)
    mlflow.log_metrics(
        {
            "mean_squared_error_X_test": rmse,
            "mean_absolute_error_X_test": mae,
            "r2_score_X_test": r2,
        }
    )
```

Директория `model/` выглядит следующим образом
```bash
model
├── MLmodel
├── model.pkl
├── conda.yaml
├── python_env.yaml
└── requirements.txt
```

Файл `model.pkl` содержит веса сериализованной модели. `MLmodel` содержит метаданные, которые объясняют MLflow как требуется загружать модель.
#### Step 6: Testing Model Serving Locally

Перед развертыванием модели полезно выяснить может ли она быть запущена локально. Важно не забыть указать флаг `--enable-mlserver`, который сообщит MLflow, что в качестве локального сервера логического вывода следует использовать MLServer
```bash
$ mlflow models serve \
    -m runs:/<run_id_for_your_best_run>/model \
    -p 1234 \
    --enable-mlserver  # Using MLServer Seldon as Inference Server
```

Эта команда запустит локальный сервер на порту 1234. Теперь можно отправить запрос
```bash
$ curl -X POST -H "Content-Type:application/json" --data '{"inputs": [[14.23, 1.71, 2.43, 15.6, 127.0, 2.8, 3.06, 0.28, 2.29, 5.64, 1.04, 3.92, 1065.0]]' http://127.0.0.1:1234/invocations

{"predictions": [-0.03416275504140387]}
```
#### Step 7: Deploying the Model to KServe

Наконец-то мы готовы к развертыванию модели на кластере Kubernetes. Создадим пространство имен
```bash
$ kubectl create namespace mlflow-kserve-test
```

Создадим yaml-файл с инструкциями по развертыванию модели на KServe. Существует два способа указать модель для развертывания в конфигурационном файле KServe:
- Построить Docker-образ с моделью и указать URI образа.
- Указать URI модели на прямую (это работает только если модель хранится на удаленном хранилище).

##### Using Docker-image

В случае с Docker-образом поступаем так:
1. Собираем Docker-образ MLflow-модели
```bash
mlflow models build-docker \
    -m runs:/<run_id_for_your_best_run>/model \
    -n <your_dockerhub_user_name>/mlflow-wine-classifier \
    --enable-mlserver
```
Эта команда соберет Docker-образ с моделью и ее зависимостями и повесит тег `mlflow-wine-classifier:latest`.
2. Публикуем Docker-образ на DockerHub (или какой-то другой реестр)
```bash
$ docker push <your_dockerhub_user_name>/mlflow-wine-classifier
```
3. Создаем yaml-файл
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlflow-wine-classifier"
  namespace: "mlflow-kserve-test"
spec:
  predictor:
    containers:
      - name: "mlflow-wine-classifier"
        image: "<your_docker_user_name>/mlflow-wine-classifier"
        ports:
          - containerPort: 8080
            protocol: TCP
        env:
          - name: PROTOCOL
            value: "v2"
```

##### Using Model URI

В конфигурации KServe можно прямым способом указывать URI модели. Однако, URI-схемы, специфичные для MLflow, такие как `runs:/`, `model:/`, а также `file:///` не разрешаются. Требуется указать URI-модели на удаленном хранилище, таком как `s3://xxx` или `gs://xxx`. 

По умолчанию MLflow хранит модели на локальной файловой системе, поэтому нужно указать, чтобы MLflow сохранял модели на удаленном хранилище.

Конфигурационный файл для модели на удаленном хранилище будет выглядеть так
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlflow-wine-classifier"
  namespace: "mlflow-kserve-test"
spec:
  predictor:
    model:
      modelFormat:
        name: mlflow
      protocolVersion: v2
      storageUri: "<your_model_uri>"
```

Для развертывания сервиса логического вывода на Kuber-кластере выполняем команду
```bash
$ kubectl apply -f YOUR_CONFIG_FILE.yaml

inferenceservice.serving.kserve.io/mlflow-wine-classifier created
```

Можно проверить статус развертывания
```bash
$ kubectl get inferenceservice mlflow-wine-classifier

NAME                     URL                                                     READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION
mlflow-wine-classifier   http://mlflow-wine-classifier.mlflow-kserve-test.local   True             100                    mlflow-wine-classifier-100
```

Чтобы получить подробную информацию о запуске выполняем
```bash
kubectl get inferenceservice mlflow-wine-classifier -oyaml
```

Чтобы послать тестовый запрос на сервер нужно сперва подготовить входной json-файл. Нужно убедиться, что данные в запросе отформатированы в соответствии с протоколом [V2 Inference Protocol](https://kserve.github.io/website/latest/modelserving/inference_api/#inference-request-json-object), потому что мы создали модель с `protocolVersion: v2`
```bash
# test-input.json
{
    "inputs": [
      {
        "name": "input",
        "shape": [13],
        "datatype": "FP32",
        "data": [14.23, 1.71, 2.43, 15.6, 127.0, 2.8, 3.06, 0.28, 2.29, 5.64, 1.04, 3.92, 1065.0]
      }
    ]
}
```

##### Kubernetes Cluster

Посылаем запрос на сервис (предполагается, что кластер доступен через LoadBalancer)
```bash
$ SERVICE_HOSTNAME=$(kubectl get inferenceservice mlflow-wine-classifier -n mlflow-kserve-test -o jsonpath='{.status.url}' | cut -d "/" -f 3)
$ curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./test-input.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/mlflow-wine-classifier/infer
```

##### Local Machine Emulation

Обычно Kuber-кластеры предоставляют доступ к сервисам через LoadBalancer, но у локального кластера, созданного с помощью Kind его нет. В этом случае получить доступ к сервису логического вывода можно с помощью переадресации портов
```bash
$ INGRESS_GATEWAY_SERVICE=$(kubectl get svc -n istio-system --selector="app=istio-ingressgateway" -o jsonpath='{.items[0].metadata.name}')
$ kubectl port-forward -n istio-system svc/${INGRESS_GATEWAY_SERVICE} 8080:80

Forwaring from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Затем выполняем
```bash
$ SERVICE_HOSTNAME=$(kubectl get inferenceservice mlflow-wine-classifier -n mlflow-kserve-test -o jsonpath='{.status.url}' | cut -d "/" -f 3)
$ curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./test-input.json \
  http://localhost:8080/v2/models/mlflow-wine-classifier/infer
```