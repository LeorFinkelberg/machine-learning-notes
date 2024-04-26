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

