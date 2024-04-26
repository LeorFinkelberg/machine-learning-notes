Детали можно найти на странице документации [Deploy MLflow Model as a Local Infernect Server](https://mlflow.org/docs/latest/deployment/deploy-model-locally.html#local-inference-server-spec).

MLflow позволяет развернуть модель локально, например, для тестирования перед развертыванием в продуктовой среде.

### Deploying Inference Server

Перед развертыванием модели нужно запомнить URI модели такой как `runs:/<run_id>/<artifact_path>` или `models:/<model_name>/<model_version>`, если модель была зарегистрирована в реестре моделей MLflow Model Registry.

Для развертывания модели следует выполнить
```bash
$ mlflow models serve -m runs:/<run_id>/model -p 5000
```

Или с помощью Python API
```python
import mlflow

model = mlflow.pyfunc.load_model("runs:/<run_id>/model")
model.serve(port=5000)
```

Теперь можно для проверки послать HTTP-запрос на концевую точку `/invocations`
```bash
curl http://127.0.0.1:5000/invocations -H "Content-Type:application/json;"  --data '{"inputs": [[1, 2], [3, 4], [5, 6]]}'
```

### Inference Server Specification

#### Концевые точки

Сервер логического вывода поддерживает 4 концевые точки:
- `/invocations`: концевая точка логического вывода, которая принимает POST-запросы с данными и возвращает прогнозы.
- `/ping`: для проверки состояния (здоровья) сервиса.
- `/health`: тоже самое что и `/ping`.
- `/version`: возвращает версию MLflow.

Концевая точка `/invocations` принимает данные в формате CSV или JSON. Формат входных данных должен быть определен в заголовке `Content-Type` как `application/json` или `application/csv`.

Пример с CSV
```bash
`curl http://127.0.0.1:5000/invocations -H 'Content-Type: application/csv' --data '1,2,3,4'`
```

Пример с POST-запросом на `/invocations`
```python
# Prerequisite: serve a custom pyfunc OpenAI model (not mlflow.openai) on localhost:5678
#   that defines inputs in the below format and params of `temperature` and `max_tokens`

import json
import requests

payload = json.dumps(
    {
        "inputs": {"messages": [{"role": "user", "content": "Tell a joke!"}]},
        "params": {
            "temperature": 0.5,
            "max_tokens": 20,
        },
    }
)
response = requests.post(
    url=f"http://localhost:5678/invocations",
    data=payload,
    headers={"Content-Type": "application/json"},
)
print(response.json())
```

Как отмечается в документации, по умолчанию MLflow использует Flask для обслуживания концевой точки. Однако, ==Flask может не подойти для масштабной промышленной эксплуатации==.

Чтобы закрыть этот пробел MLflow ==интегрируется с MLServer== в качестве _альтернативного механизма обслуживания концевых точек_. MLServer обеспечивает более высокую производительность и масштабируемость за счет асинхронной парадигмы запрос/ответ. Также MLServer используется в качестве основного сервера логического вывода Python на Kubernetes-native платформах, таких как Seldon Core и KServe (ранее известный как KFServing), что обеспечивает расширенные возможности -- канареечное развертывание, автоматическое масштабирование и пр.
![[flask-vs-mlserver.PNG]]

Flask используется для локального тестирования, а MLServer Seldon для развертывания моделей в нагруженной промышленной среде эксплуатации.

Flask не оптимизирован под высокую нагрузку, поскольку представляет собой WSGI-приложение. WSGI основывается на _синхронной парадигме запрос/ответ_, что плохо подходит для работы с ML-моделями в силу блокирующего характера. Flask может поддерживать асинхронные фреймворки вроде Uvicorn, но MLflow их "из коробки" не поддерживает и просто использует синхронное поведение Flask. Flask не поддается масштабированию.

MLServer спроектирован под высокую нагрузку, специфичную для ML. MLServer Поддерживает _асинхронную парадигму запрос/ответ_, распределяя нагрузку логического выводу по пулу воркеров (процессов), чтобы сервер мог продолжить принимать запросы во время обработки вывода.

Разворачивая MLflow-модели на кластере Kubernetes с помощью MLServer, можно получить дополнительные преимущества, например, автомасштабирование.

Для развертывания моделей с помощью MLServer, нужно установить MLflow с дополнительными зависимостями
```bash
$ pip install mlflow[extras]
```
а затем выполнить команду развертывания с флагом `--enable-mlserver`
```bash
$ mlflow models serve -m runs:/<run_id>/model -p 5000 --enable-mlserver
```

Или через Python API
```python
import mlflow

model = mlflow.pyfunc.load_model("runs:/<run_id>/model")
model.serve(port=5000, enable_mlserver=True)
```

Вместо того чтобы посылать HTTP-запросы на запущенный сервис, можно выполнить пакетную обработку на локальных файлах
```bash
$ mlflow models predict -m runs:/<run_id>/model -i input.csv -o output.csv
```

Или используя Python API
```python
import mlflow

model = mlflow.pyfunc.load_model("runs:/<run_id>/model")
predictions = model.predict(pd.read_csv("input.csv"))
predictions.to_csv("output.csv")
```

Важным шагом для развертывания MLflow-модели на кластере Kubernetes является сборка Docker-образа, содержащего MLflow-модель и сервер логического вывода
```bash
$ mlflow models build-docker -m runs:/<run_id>/model -n <image_name> --enable-mlserver
```

Или с помощью Python API
```python
import mlflow

mlflow.models.build_docker(
    model_uri=f"runs:/{run_id}/model",
    name="<image_name>",
    enable_mlserver=True,  # MLServer Seldon
)
```
