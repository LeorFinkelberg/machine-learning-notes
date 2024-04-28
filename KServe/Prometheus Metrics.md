Все среды выполнения поддерживают экспорт метрик Prometheus. Например, TorchServe определяет порт `8082` для Prometheus
```bash
# kserve-torchserve.yaml
metadata:
  name: kserve-torchserve
spec:
  annotations:
    prometheus.kserve.io/port: '8082'
    prometheus.kserve.io/path: "/metrics"
```

При необходимости эти значения могут быть переопределены при создании сервиса логического вывода (см. [Queue Proxy Extension](https://github.com/kserve/kserve/blob/master/qpext/README.md#configs))
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-irisv2"
  annotations:
    serving.kserve.io/enable-metric-aggregation: "true"
    serving.kserve.io/enable-prometheus-scraping: "true"
    prometheus.kserve.io/port: '8081'
    prometheus.kserve.io/path: "/other/metrics"
spec:
  predictor:
    sklearn:
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

По умолчанию для sklearn используется порт `8080`, а путь по умолчанию `/metrics`. Аннотации в конфигурационном файле InferenceService позволяют переопределить конфигурацию среды выполнения.

Чтобы включить поддержку метрик Prometheus, следует добавить поле `serving.kserve.io/enable-prometheus-scraping`
```bash
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-irisv2"
  annotations:
    serving.kserve.io/enable-prometheus-scraping: "true"  # <= NB
spec:
  predictor:
    sklearn:
      protocolVersion: v2
      storageUri: "gs://seldon-models/sklearn/iris"
```
В бессерверном режиме экспорт метрик предполагает, что используется расширение queue-proxy (см. [Queue Proxy Extension](https://github.com/kserve/kserve/blob/master/qpext/README.md)).