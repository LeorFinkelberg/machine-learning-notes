Детали можно найти на странице документации [Seving Runtimes](https://kserve.github.io/website/latest/modelserving/servingruntimes/).

KServe использует два CRD определения модельных сред обслуживания:
- ServingRuntimes
- ClusterServingRuntimes

Единственное различие между ними заключается в том, что один ограничен пространством имен (namespace-scoped), а другой кластером (cluter-scoped).

ServingRuntime определяет шаблоны для подов, которые могут обслуживать один или несколько форматов модели. 

Пример ServingRuntime
```bash
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: example-runtime
spec:
  supportedModelFormats:
    - name: example-format
      version: "1"
      autoSelect: true
  containers:
    - name: kserve-container
      image: examplemodelserver:latest
      args:
        - --model_dir=/mnt/models
        - --http_port=8080
```

Поддерживаются следующие форматы моделей:
- `kserve-lgbserver`: LIghtGBM
- `kserve-mlserver`: SkLearn, XGBoost, LightGBM, MLflow
- `kserve-sklearnserver`: SKlearn,
- etc.
#### Using ServingRuntimes

Кроме того KServe поддерживает пользовательские среды выполнения (custom runtimes). ServingRuntimes могут использоваться явно или неявно.
#### Explicit: Specify a runtime

Когда пользователь определяет predictor в InferenceServices, он может явно указать имя ClusterServingRuntime или ServingRuntime. Например
```bash
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://bucket/sklearn/mnist.joblib
      runtime: kserve-mlserver
```

Здесь в качестве среды выполнения указана `kserve-mlserver`, поэтому контроллер KServe в первую очередь будет искать пространство имен с этим именем. Если найти не удастся, то контролллер продолжит поиск в списке ClusterServingRuntimes. Если пространство имен найдено, то контроллер сначала проверит, что `modelFormat` представлен в списке предиктора (поле `predictor:`) `supportedModelFormats`. Если формат поддерживается, то информация о контейнере и поде, представленная средой выполнения, будет использоваться при развертывании модели.

#### Implicit: Automatic selection

Если среда выполнения явно не указана, то в каждом элементе списка `supportedModelFormats`, можно выставить параметр `autoSelect` в `true`, что будет укажет ServingRuntime на возможность автоматического выбора формата модели.
```bash
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterServingRuntime
metadata:
  name: kserve-sklearnserver
spec:
  supportedModelFormats:
    - name: sklearn
      version: "1"
      autoSelect: true
...
```

Теперь когда сервис логического вывода будет разворачиваться без указания среды выполнения, контроллер будет искать среду выполнения, поддерживающую `sklearn`
```bash
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://bucket/sklearn/mnist.joblib
```

Поскольку в списке `supportedModelFormat` среды выполнения ClusterServingRuntime `kserve-sklearnserver` указано `autoSelect: true`, этот ClusterServingRuntime будет использоваться при развертывании модели.

Рассмотрим среды выполнения `mlserver` и `kserve-sklearnserver`. Обе среды выполнения поддерживают формат модели `sklearn`  версии `1` и обе поддерживают протокол `protocolVersion` V2
```bash
apiVersion: serving.kserve.io/v1alpha1
kind: ClusterServingRuntime
metadata:
  name: kserve-sklearnserver
spec:
  protocolVersions:
    - v1
    - v2
  supportedModelFormats:
    - name: sklearn
      version: "1"
      autoSelect: true
      priority: 1
...
```

Тогда InferenceService, развертывается без указания среды выполнения, контроллер будет искать среду выполнения, поддерживающую `sklearn`
```bash
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: example-sklearn-isvc
spec:
  predictor:
    model:
      protocolVersion: v2
      modelFormat:
        name: sklearn
      storageUri: s3://bucket/sklearn/mnist.joblib
```

Контроллер найдет две среды выполнения `kserve-sklearnserver` и `mlserver`, указанные в `supportedModelFormats`. Теперь среды выполнения будет отсортированы по приоритету. Поскольку `mlserver` имеет более высокий приоритет, его ClusterServingRuntime и будет использоваться при развертывании модели.
