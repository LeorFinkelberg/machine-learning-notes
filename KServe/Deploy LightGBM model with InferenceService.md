Детали можно найти на странице документации [Deploy LightGBM Model with InferenceService](https://kserve.github.io/website/latest/modelserving/v1beta1/lightgbm/).
### Train a LightGBM model

```python
import lightgbm as lgb
from sklearn.datasets import load_iris
import os

model_dir = "."
BST_FILE = "model.bst"

iris = load_iris()
y = iris['target']
X = iris['data']
dtrain = lgb.Dataset(X, label=y, feature_names=iris['feature_names'])

params = {
    'objective':'multiclass', 
    'metric':'softmax',
    'num_class': 3
}
lgb_model = lgb.train(params=params, train_set=dtrain)
model_file = os.path.join(model_dir, BST_FILE)
lgb_model.save_model(model_file)
```
### Deploy LightGBM model with V1 protocol

#### Test the model locally

Установим и запустим сервер [LightGBM Server](https://github.com/kserve/kserve/tree/master/python/lgbserver), используя обученную модель и отправим запрос
```bash
$ python -m lgbserver --model_dir /path/to/model_dir --model_name lgb
```

После локального запуска сервера LightGBM Server, можно отправить запрос на прогноз
```python
import requests

request = {'sepal_width_(cm)': {0: 3.5}, 'petal_length_(cm)': {0: 1.4}, 'petal_width_(cm)': {0: 0.2},'sepal_length_(cm)': {0: 5.1} }
formData = {
    'inputs': [request]
}
res = requests.post('http://localhost:8080/v1/models/lgb:predict', json=formData)
print(res)
print(res.text)
```

#### Deploy with InferenceService

Чтобы развернуть модель на кластере Kubernetes, можно создать сервис логического вывода InferenceService, указав формат модели `modelFormat` и URI `storageUri`
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      storageUri: "gs://kfserving-examples/models/lightgbm/iris"
```

Применяем созданный yaml-файл для создания сервиса логического вывода
```bash
$ kubectl apply -f lightgbm.yaml
# inferenceservice.serving.kserve.io/lightgbm-iris created
```

#### Test the deployed model

Чтобы протестировать развернутую модель сначала требуется определить IP и порт Ingress и задать `INGRESS_HOST` и `INGRESS_PORT`, а затем отправить запрос
```bash
MODEL_NAME=lightgbm-iris
INPUT_PATH=@./iris-input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice lightgbm-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" -H "Content-Type: application/json" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

### Deploy the Model with Open Inference Protocol

#### Test the model locally

Пусть у нас есть артефакт сериализации `model.bst`. Теперь мы можем создать локальный сервер с помощью KServe LightGBM Server
```bash
python3 lgbserver --model_dir /path/to/model_dir --model_name lightgbm-v2-iris
```

#### Deploy InferenceService with REST endpoint

Чтобы развернуть модель LightGBM с использованием протокола Open Inference Protocol, следует установить поле `protocolVersion` в `v2`
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      runtime: kserve-lgbserver
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
```

NB: Если поле `runtime` не задано, то для протокола `V2 Protocol` (`Open Inference Protocol`) по умолчанию в качестве среды выполнения (runtime) будет использоваться `mlserver`.

Применяем yaml-файл для создания сервиса логического вывода с доступом по REST
```bash
kubectl apply -f lightgbm-v2.yaml
# inferenceservice.serving.kserve.io/lightgbm-v2-iris created
```

#### Test the deployed model with curl

Теперь можно протестировать развернутую модель, послав запрос. Важно! Этот запрос должен следовать протоколу [V2 DataPlane Protocol](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2).
```bash
# iris-input-v2.json
{
  "inputs": [
    {
      "name": "input-0",
      "shape": [2, 4],
      "datatype": "FP32",
      "data": [
        [6.8, 2.8, 4.8, 1.4],
        [6.0, 3.4, 4.5, 1.6]
      ]
    }
  ]
}
```

Теперь можно отправить запрос `curl`
```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice lightgbm-v2-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d @./iris-input-v2.json \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/lightgbm-v2-iris/infer
```

### Create the InferenceService with gRPC endpoint

Создадим yaml-файл сервиса логического вывода и откроем gRPC порт. В настоящее время только один порт может предоставлять доступ к HTTP или gRPC (по умолчанию открыт HTTP).

Для случая Serverless 
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris-grpc"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      protocolVersion: v2
      runtime: kserve-lgbserver
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
      ports:
        - name: h2c          # knative expects grpc port name to be 'h2c'
          protocol: TCP
          containerPort: 8081
```

Для случая RawDeployment
```bash

apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "lightgbm-v2-iris-grpc"
spec:
  predictor:
    model:
      modelFormat:
        name: lightgbm
      protocolVersion: v2
      runtime: kserve-lgbserver
      storageUri: "gs://kfserving-examples/models/lightgbm/v2/iris"
      ports:
        - name: grpc-port      # Istio requires the port name to be in the format <protocol>[-<suffix>]
          protocol: TCP
          containerPort: 8081
```

Применяем yaml-файл для создания сервиса логического вывода InferenceService с доступом по gRPC
```bash
$ kubectl apply -f lightgbm-v2-grpc.yaml
```

#### Test the deployed model with grpcurl

Когда сервис логического вывода будет готов, можно отправлять запрос с помощью утилиты `grpcurl`
```bash
# download the proto file
curl -O https://raw.githubusercontent.com/kserve/open-inference-protocol/main/specification/protocol/open_inference_grpc.proto

INPUT_PATH=iris-input-v2-grpc.json
PROTO_FILE=open_inference_grpc.proto
SERVICE_HOSTNAME=$(kubectl get inferenceservice lightgbm-v2-iris-grpc -o jsonpath='{.status.url}' | cut -d "/" -f 3)
```

Подготовим файл полезной нагрузки
```bash
{
  "model_name": "lightgbm-v2-iris-grpc",
  "inputs": [
    {
      "name": "input-0",
      "shape": [2, 4],
      "datatype": "FP32",
      "contents": {
        "fp32_contents": [6.8, 2.8, 4.8, 1.4, 6.0, 3.4, 4.5, 1.6]
      }
    }
  ]
}
```

Запрос на прогноз
```bash
grpcurl \
  -vv \
  -plaintext \
  -proto ${PROTO_FILE} \
  -authority ${SERVICE_HOSTNAME} \
  -d @ \
  ${INGRESS_HOST}:${INGRESS_PORT} \
  inference.GRPCInferenceService.ModelInfer \
  <<< $(cat "$INPUT_PATH")
```

