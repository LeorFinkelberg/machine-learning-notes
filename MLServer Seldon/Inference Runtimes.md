### Общие сведения о средах выполнения вывода

Среды выполнения логического вывода (Inference Runtimes) позволяют определить как модель будет использоваться внутри MLServer. Об этом можно думать как о бекэнед клее (backend glue) между MLServer и ML-фреймворком.

В комплект поставки MLServer входит набор готовых сред выполнения, которые позволяют взаимодействовать с некоторым популярными ML-фреймворками.

Чтобы подключить нужную среду выполнения, достаточно просто указать соответствующую реализацию класса^[https://mlserver.readthedocs.io/en/latest/runtimes/index.html#included-inference-runtimes]:
- `mlserver_sklearn.SKlearnModel` (пакет `mlserver-sklearn`),
- `mlsever_xgboost.XGBoostModel` (пакет `mlserver-xgboost`),
- `mlserver_mllib.MLlibModel` (пакет `mlserver-mllib`),
- `mlserver_mlflow.MLflowRuntime` (пакет `mlserver-mlflow`),
- etc.
в конфигурационном файле модели `model-settings.json`
```bash
{
  ...
  "implementation": "mlserver_sklearn.SKLearnModel",
  ...
}
```

#### XGBoost Runtime for MLServer

Установить пакет можно так
```bash
$ pip install mlserver mlserver-xgboost
```

Среда вывода для XGBoost ожидает, что модель будет сериализована в файл с расширением:
- `*.json`: `booster.save_model("model.json")`,
- `*.ubj` : `booster.save_model("model.ubj")`,
- или `*.bst` (старый бинарный формат): `booster.save_model("model.bst")`.

По умолчанию среда выполнения вывода ищет файл с именем `model.[json | ubj | bst]`. Однако, это поведение можно изменить с помощью поля `parameters.uri`
```bash
# model-settings.json
{
  ...
  "name": "foo",
  "parameters": {
    "uri": ./my-own-model-filename.json"
  }
}
```

Если контент типов не указан в запросе, то среда выполнения вывода XGBoost попытается декодировать полезную нагрузку как массив NumPy. Чтобы это исправить можно либо указать контент типов явно, либо как метаданные модели.

По умолчанию, среда выполнения вывода возвращает только результат `predict`. Однако, это можно изменить так
```bash
{
  "inputs": [
    {
	  "name": "my-input",
	  "datatype": "INT32",
	  "shape": [2, 2],
	  "data": [1, 2, 3, 4]
    }
  ],
  "outputs": [
    {"name": "predict_proba"}
  ]
}
```

