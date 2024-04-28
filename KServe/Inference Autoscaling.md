### InferenceService with target concurrency

Поля `scaleTarget` и `scaleMetric` первые были добавлены в KServe версии 0.9. Это предпочтительный способ задания автомасштабирования.
```bash
# autoscale.yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "flowers-sample"
spec:
  predictor:
    scaleTarget: 1  # <= NB
    scaleMetric: concurrency  # <= NB
    model:
      modelFormat:
        name: tensorflow
      storageUri: "gs://kfserving-examples/models/tensorflow/flowers"
```

Теперь применим созданный манифест для создания сервиса логического вывода с поддержкой автомасштабирования (Autoscale InferenceService)
```bash
$ kubectl apply -f autoscale.yaml
```
#### Predict `InferenceService` with concurrent requests

Отправляем запрос с помощью утилиты [`hey`](https://github.com/rakyll/hey?tab=readme-ov-file). Здесь продолжительность атаки 30 секунд (`-z 30s`: duration of application to send requests) и 5 воркеров, запущенных конкурентно (`-c 5`)
```bash
$ MODEL_NAME=flowers-sample
$ INPUT_PATH=input.json
$ SERVICE_HOSTNAME=$(kubectl get inferenceservice $MODEL_NAME -o jsonpath='{.status.url}' | cut -d "/" -f 3)

$ hey -z 30s -c 5 -m POST -host ${SERVICE_HOSTNAME} -D $INPUT_PATH http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict
```

Поскольку поле `scaleTarget` выставлено в 1, а отправляем мы 5 конкурентных запросов, то autoscaler пытается масштабироваться до 5 подов. Проверить это можно так
```bash

$ kubectl get pods
NAME                                                       READY   STATUS            RESTARTS   AGE
flowers-sample-default-7kqt6-deployment-75d577dcdb-sr5wd         3/3     Running       0          42s
flowers-sample-default-7kqt6-deployment-75d577dcdb-swnk5         3/3     Running       0          62s
flowers-sample-default-7kqt6-deployment-75d577dcdb-t2njf         3/3     Running       0          62s
flowers-sample-default-7kqt6-deployment-75d577dcdb-vdlp9         3/3     Running       0          64s
flowers-sample-default-7kqt6-deployment-75d577dcdb-vm58d         3/3     Running       0          42s
```

Посмотрим дешборд Knative Serving Scaling (если он был сконфигурирован)
```bash
kubectl port-forward --namespace knative-monitoring $(kubectl get pods --namespace knative-monitoring --selector=app=grafana  --output=jsonpath="{.items..metadata.name}") 3000
```
#### Predict `InferenceService` with target QPS

Отправляем трафик. Здесь продолжительность атаки 30 секунд (`-z 30s`) и 50 запросов в секунду (`-q 50`)
```bash
$ MODEL_NAME=flowers-sample
$ INPUT_PATH=input.json
$ SERVICE_HOSTNAME=$(kubectl get inferenceservice $MODEL_NAME -o jsonpath='{.status.url}' | cut -d "/" -f 3)

$ hey -z 30s -q 50 -m POST -host ${SERVICE_HOSTNAME} -D $INPUT_PATH http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict
```

#### Autoscaling Customization

Поле `ContainerConcurrency` определяет максимальное число запросов, выполняемых одновременно, в любой момент времени _на каждой реплике_ сервиса логического вывода`InferenceService`. Это жесткое ограничение. И потому если это ограничение будет нарушено, "избыточные" запросы будут помещены в буфер, ожидая когда освободится слот.
```bash
# autoscal-custom.yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "flowers-sample"
spec:
  predictor:
    containerConcurrency: 10
    model:
      modelFormat:
        name: tensorflow
      storageUri: "gs://kfserving-examples/models/tensorflow/flowers"
```

Применяем манифест
```bash
$ kubectl apply -f autoscale-custom.yaml
```
#### Enable scale down to zero

KServe по умолчанию выставляет поле `minReplicas` в 1, однако можно включить и масштабирование до 0. Это может быть полезно в случае использования GPU. Поды будут автоматически масштабироваться до 0, если трафик нулевой.
```bash
# scale-down-to-zero.yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "flowers-sample"
spec:
  predictor:
    minReplicas: 0  # <= NB
    model:
      modelFormat:
        name: tensorflow
      storageUri: "gs://kfserving-examples/models/tensorflow/flowers"
```

Применяем манифест
```bash
$ kubectl apply -f scale-down-to-zero.yaml
```

#### Autoscaling configuration at component level

Поддержка автомасштабирования может быть сконфигурирована на уровне компонент. Это дает больше гибкости в терминах конфигурации автомасштабирования. При типичном развертывании transformers могут требовать конфигурации автомасштабирования отличной от конфигурации predictor.
```bash
# autoscale-adv.yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "torch-transformer"
spec:
  predictor:  # <= NB
    scaleTarget: 2
    scaleMetric: concurrency
    model:
      modelFormat:
        name: pytorch
      storageUri: gs://kfserving-examples/models/torchserve/image_classifier
  transformer:  # <= NB
    scaleTarget: 8
    scaleMetric: rps
    containers:
      - image: kserve/image-transformer:latest
        name: kserve-container
        command:
          - "python"
          - "-m"
          - "model"
        args:
          - --model_name
          - mnist
```

