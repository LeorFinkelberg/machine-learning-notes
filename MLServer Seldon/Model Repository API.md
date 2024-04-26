MLServer поддерживает загрузку (loading) и удаление (unloading) моделей динамически из репозитория моделей. Это позволяет включать и выключать модели, доступные MLServer по требованию.

Запускаем сервер MLServer с несколькими моделями как объясняется в заметке [[Multi-Model Serving]] . А затем выполняем запрос через клиента
```python
# list_model_repository.py

import requests

response = requests.post("http://localhost:8080/v2/repository/index", json={})
response.json()
```

```bash
$ python list_model_repository.py | sed "s/'/\"/g" | jq 
# output
[
  {
    "name": "mushroom-xgboost",
    "version": "v0.1.0",
    "statte": "READY",
    "reason": ""
  },
  {
    "name": "mnist-svm",
    "version": "v0.1.0",
    "statte": "READY",
    "reason": ""
  }
]
```

Как видно в репозитории содержится две модели `mushroom-xgboost` и `mnist-svm`). И обе они в состоянии `READY`, то есть готовы к логическому выводу.

Попробуем удалить с сервера модель `mushroom-xgboost`. Эта операция удалит модель с сервера логического вывода, но сохранит ее в репозитории моделей
```python
# unload_model_from_repository.py

requests.post("http://localhost:8080/v2/repository/models/mushroom-xgboost/unload")
```

```bash
$ python unload_model_from_repository.py | sed "s/'/\"/g" | jq
# output
[
  {
    "name": "mushroom-xgboost",
    "version": "v0.1.0",
    "statte": "UNAVAILABLE",
    "reason": ""
  },
  {
    "name": "mnist-svm",
    "version": "v0.1.0",
    "statte": "READY",
    "reason": ""
  }
]
```

`UNAVAILABLE` означает, что модель присутствует в репозитории, но не загружена на сервер для логического вывода.

Чтобы загрузить модель обратно на сервер, выполняем
```python
requests.post("http://localhost:8080/v2/repository/models/mushroom-xgboost/load")
```

