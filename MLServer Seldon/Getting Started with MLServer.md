### Общие сведения

Это руководство поможет начать создавать ML-микросервисы с помощью MLServer.

Для того чтобы подготовить базу для запуска MLServer, нужно:
- создать ML-модель (`*.py`),
- конфигурационный файл для модели (`model-settings.json)
- конфигурационный файл для сервера (`settings.json`); он необязательный и нужен только для локальной разработки.

Пусть разработка ведется в рабочей директории `similarity_model`, модуль с ML-моделью выглядит так
```python
# similarity_model/my_model.py

from mlserver.codecs import decode_args
from mlserver import MLModel
from typing import List
import numpy as np
import spacy

class MyKulModel(MLModel):

    async def load(self):
        self.model = spacy.load("en_core_web_lg")
    
    @decode_args
    async def predict(self, docs: List[str]) -> np.ndarray:

        doc1 = self.model(docs[0])
        doc2 = self.model(docs[1])

        return np.array(doc1.similarity(doc2))
```
а конфигурационный файл для модели имеет вид
```bash
# similarity_model/model-settings.json

{
    "name": "doc-sim-model",  # это имя станет частью пути HTTP-запроса
    "implementation": "my_model.MyKulModel"
}
```

Поле `implementation` в конфигурационном файле `model-settings.json` строится по следующему шаблону
```bash
{
  ...
  "implementation": "name_of_py_file_with_your_model.class_with_your_model"
  ...
}
```

После запуска сервера будут доступны 3 точки входа: одна для обработки HTTP-запросов, вторая -- для gRPC и третья -- для метрик.

Запускаем MLServer командой 
```bash
$ mlserver start similarity_model/
```

Теперь можно послать запрос на сервис 
```python
from mlserver.codecs import StringCodec
import requests

inference_request = {
    "inputs": [
        StringCodec.encode_input(
            name='docs',
            payload=[barbie, oppenheimer],
            use_bytes=False
        ).dict()
    ]
}
print(inference_request)
# --- output ---
{'inputs': [{'name': 'docs',
   'shape': [2, 1],
   'datatype': 'BYTES',
   'parameters': {'content_type': 'str'},
   'data': [
        'Barbie is a 2023 American fantasy comedy...',
        'Oppenheimer is a 2023 biographical thriller...'
        ]
    }]
}
# --- output ---

# Готовим POST-запрос на сервис
r = requests.post(
    # имя `doc-sim-model` использовалось в конфигурационном файле 
    # `model-settings.json`
	'http://0.0.0.0:8080/v2/models/doc-sim-model/infer',
	json=inference_request
)
r.json()
# --- output ---
{'model_name': 'doc-sim-model',
    'id': 'a4665ddb-1868-4523-bd00-a25902d9b124',
    'parameters': {},
    'outputs': [{'name': 'output-0',
    'shape': [1],
    'datatype': 'FP64',
    'parameters': {'content_type': 'np'},
    'data': [0.9866910567224084]}]}
# --- output ---

print(
    f"Our movies are {round(r.json()['outputs'][0]['data'][0] * 100, 4)}% similar!"
)  # Our movies are 98.6691% similar
```

Все URL, которые создаются с использованием MLServer, будут иметь вид
```bash
http://host:port/v2/models/your_model_name/infer
```

[V2/Open Inference Protocol](https://docs.seldon.io/projects/seldon-core/en/latest/reference/apis/v2-protocol.html) (OIP) -- это общеотраслевая разработка, направленная на представление стандартизированного протокола для взаимодействия с различными серверами вывода (такими как MLServer, Triton etc.) и оркестрации фреймворками (такими как Seldon Core, KServe etc.). OIP определяет концевые точки и схему полезной нагрузки для интерфейсов REST и gRPC. На странице можно найти маршруты под различные задачи, например, `/v2/models/{model_name}/infer`, `/v2/models/{model_name}/versions/{model_version}` etc.

Про Content Types и Codecs можно прочитать на странице [документации](https://mlserver.readthedocs.io/en/latest/user-guide/content-type.html#user-guide-content-type--page-root).

Предположим, требуется обрабатывать большее число пользовательских запросов и есть вероятность, что одной модели для этого может быть недостаточно. Или быть может модель использует не все ресурсы предоставленной виртуальной машины. В этом случае для увеличения пропускной способности можно создать несколько _реплик модели_. Для этого в конфигурационном файле `settings.json` нужно задать параметр `parallel_workers` 
```bash
# similarity_model/settings.json

{
  "parallel_workers": 3
}
```

Затем можно как обычно запустить MLServer `mlserver start similarity_model`.
![[multiple_models.webp]]

Как можно видеть из логов терминала, теперь у нас параллельно запускается 3 модели. Причина, по которой в данном случае терминал отображает 4 модели, в том, что MLServer печатает один раз имя проинициализированной модели и еще раз для каждой реплики.

Наконец можно упаковать модель и сервис в Docker-образ. Для этого сначала нужно собрать файл с зависимостями проекта 
```bash
# similarity_model/requirements.txt
mlserver
spacy==3.6.0
https://github.com/explosion/spacy-models/releases/download/en_core_web_lg-3.6.0/en_core_web_lg-3.6.0-py3-none-any.whl
```

Затем соберем Docker-образ
```bash
$ mlserver build similarity_model/ -t 'fancy_ml_service'
```

Можно проверить работоспособность Docker-образа, предварительно остановив сервер с помощью `Ctrl+C` в терминале
```bash
$ docker run -it --rm -p 8080:8080 fancy_ml_service
```

Теперь, когда есть полнофункциональный микросервис с нашей моделью, Docker-контейнер можно развернуть в промышленной среде эксплуатации, например, с помощью Seldon Core, или с помощью какого-нибудь облачного решения (AWS Lambda, Google Cloud Run etc.).

Чтобы запустить несколько моделей на одном сервере, рабочая директория должна выглядеть следующим образом
```bash
project_dir/
	data/
	 - fashin-mnist-test.csv
	 - fashion-mnist-train.csv
	models/
	  - sklearn/
	    - Fashion_MNIST.joblib
	    - model-settings.json  # for sklearn-model
	    - test.py
	    - train.py
	  - xgboost/
	    - Fashion_MNIST.json
	    - model-settings.json  # for xgboost-model
	    - test.py
	    - train.py
	- README.md
	- settings.json  # for MLServer
	- test_model.py
```

### Запуск нескольких моделей на одном инференс-сервере MLServer

Подробности можно найти в видео [Optimizing Inference For State Of The Art Python Models](https://www.youtube.com/watch?v=5tyLqAfW29I). В видео рассказывается не только про параллельный запуск моделей, но и, например, про адаптивную пакетную обработку запросов.

А чтобы запустить все модели разом, достаточно выполнить `mlserver start .`. Концевые точки, на которые можно послать POST-запрос к моделям, будут выглядеть так 
```bash
/v2/models/fashion-sklearn/versions/v1/infer
/v2/models/fashion-xgboost/versions/v1/infer
```