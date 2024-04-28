Настройка:
1. Конфигурационный файл `~/.kube/config` должен указывать на установленный KServer (см. [Serverless Installation Guide](https://kserve.github.io/website/master/admin/serverless/serverless/))
2. Шлюз кластера Istio Ingress должен быть доступен по сети (см. [Ingress Gatewasy](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/))

### Create the InferenceService

Выполните шаги 1-3, описанные в [First Inference Service](https://kserve.github.io/website/master/get_started/first_isvc/). Создайте пространство имен (если еще не создали), а затем создайте сервис логического вывода InferenceService.

После развертывания первой модели, на нее направляется 100% трафика. Чтобы увидеть объем направляемого трафика, выполняем команду
```bash
$ kubectl get isvc sklearn-iris
NAME URL READY PREV LATEST PREVROLLEDOUTREVISION LATESTREADYREVISION AGE sklearn-iris http://sklearn-iris.kserve-test.example.com True 100 sklearn-iris-predictor-default-00001 46s 2m39s 70s
```
### Update the InferenceService with the canary rollout strategy

Добавляем поле `canaryTrafficPercent` в раздел predictor и обновляем значение поля `storageUri`, чтобы использовать новую / обновленную версию модели.
```bash
$ kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    canaryTrafficPercent: 10  # <= NB
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model-2"
EOF
```

После развертывания канареечной модели, трафик будет разделен между последней готовой к развертыванию версией (модель 2) и предыдущей версией (модель 1).
```bash
$ kubectl get isvc sklearn-iris

NAME       URL                                   READY   PREV   LATEST   PREVROLLEDOUTREVISION              LATESTREADYREVISION                AGE
sklearn-iris   http://sklearn-iris.kserve-test.example.com   True    90     10       sklearn-iris-predictor-default-00001   sklearn-iris-predictor-default-00002   9m19s
```
Как можно видеть 10% трафика идет на новую модель.

### Promote the canary model

Если канареечная модель работоспособна / прошла тесты, то можно удалить поле `canaryTrafficPercent` и снова применить манифест для создания сервиса логического вывода
```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model-2"
EOF
```

Теперь весь трафик идет на модель версии 2
```bash
$ kubectl get isvc sklearn-iris
NAME URL READY PREV LATEST PREVROLLEDOUTREVISION LATESTREADYREVISION AGE sklearn-iris http://sklearn-iris.kserve-test.example.com True 100 sklearn-iris-predictor-default-00002 17m
```

Поды предыдущей версии автоматически масштабируются до 0, так как они больше не получают трафик
```bash
$ kubectl get pods -l serving.kserve.io/inferenceservice=sklearn-iris
NAME                                                           READY   STATUS        RESTARTS   AGE
sklearn-iris-predictor-default-00001-deployment-66c5f5b8d5-gmfvj   1/2     Terminating   0          17m
sklearn-iris-predictor-default-00002-deployment-5bd9ff46f8-shtzd   2/2     Running       0          15m
```

### Rollback and pin the previous model

Можно закрепить предыдущую модель (например, модель версии 1), выставив поле `canaryTrafficPercent` в 0 для текущей модели (например, модели версии 2). Это откатит модель версии 2 к предыдущей версии и уменьшит трафик на модель версии 2 до 0.
```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    canaryTrafficPercent: 0  # <= NB
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model-2"
EOF
```

Теперь 100% трафика идет на предыдущую версию модели
```bash
kubectl get isvc sklearn-iris
NAME       URL                                   READY   PREV   LATEST   PREVROLLEDOUTREVISION              LATESTREADYREVISION                AGE
sklearn-iris   http://sklearn-iris.kserve-test.example.com   True    100    0        sklearn-iris-predictor-default-00001   sklearn-iris-predictor-default-00002   18m
```

### Route traffic using a tag

Чтобы управлять трафиком явно, указывая конкретную версию модели, нужно повесить на модель тег.
```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  annotations:
    serving.kserve.io/enable-tag-routing: "true"
spec:
  predictor:
    canaryTrafficPercent: 10
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model-2"
EOF
```

Чтобы получить URL канареечной модели (модели, которая разворачивается в настоящее время) и предыдущей, выполняем команду
```bash
kubectl get isvc sklearn-iris -ojsonpath="{.status.components.predictor}"  | jq
# output
{ "address": { "url": "http://sklearn-iris-predictor-default.kserve-test.svc.cluster.local" }, "latestCreatedRevision": "sklearn-iris-predictor-default-00003", "latestReadyRevision": "sklearn-iris-predictor-default-00003", "latestRolledoutRevision": "sklearn-iris-predictor-default-00001", "previousRolledoutRevision": "sklearn-iris-predictor-default-00001", "traffic": [ { "latestRevision": true, "percent": 10, "revisionName": "sklearn-iris-predictor-default-00003", "tag": "latest", "url": "http://latest-sklearn-iris-predictor-default.kserve-test.example.com" }, { "latestRevision": false, "percent": 90, "revisionName": "sklearn-iris-predictor-default-00001", "tag": "prev", "url": "http://prev-sklearn-iris-predictor-default.kserve-test.example.com" } ], "url": "http://sklearn-iris-predictor-default.kserve-test.example.com" }
```

То есть у канареечной модели URL
```bash
"url": "http://latest-sklearn-iris-predictor-default.kserve-test.example.com"
```
а у предыдущей 
```bash
"url": "http://prev-sklearn-iris-predictor-default.kserve-test.example.com"
```
У канареечной (новой) версии модели имя 
```bash
"revisionName": "sklearn-iris-predictor-default-00003"
```

Теперь можно посылать запросы явно указывая версию модели, используя соответствующий тег.

Например, для того чтобы отправить запрос на канареечную модель, выполняем
```bash
$ MODEL_NAME=sklearn-iris
$ curl -v -H "Host: latest-${MODEL_NAME}-predictor-default.kserve-test.example.com" -H "Content-Type: application/json" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d @./iris-input.json
```

А вот так можно отправить запрос на предыдущую
```bash
$ MODEL_NAME=sklearn-iris
$ curl -v -H "Host: prev-${MODEL_NAME}-predictor-default.kserve-test.example.com" -H "Content-Type: application/json" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d @./iris-input.json
```
