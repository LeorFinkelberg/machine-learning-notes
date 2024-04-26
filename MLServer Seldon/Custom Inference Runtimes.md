Подробности можно найти на странице документации [Custom Inference Runtime](https://mlserver.readthedocs.io/en/latest/user-guide/custom.html).

MLServer позволяет создавать свои собственные _среды выполнения_ (inference runtimes). Полный пример создания пользовательской среды выполнения можно посмотреть [здесь](https://mlserver.readthedocs.io/en/latest/user-guide/custom.html).

Для того чтобы создать свою среду выполнения, следует начать с абстрактного класса `MLModel`, который содержит следующие основные методы:
- `load()`: отвечает за загрузку различных артефактов модели (веса модели, pickle-файлы etc.),
- `unload()`: отвечает за освобождение ресурсов (например, за освобождение GPU памяти),
- `predict()`: отвечает за построение прогноза модели ("... model to perform inference on an incoming data point").

Таким образом нужно расширить класс `MLModel` и переопределить эти методы
```python
from mlserver import MLModel
from mlserver.types import InferenceRequest, InferenceResponse

class MyCustomRuntime(MLModel):

  async def load(self) -> bool:
    # TODO: Replace for custom logic to load a model artifact
    self._model = load_my_custom_model()
    return True

  async def predict(self, payload: InferenceRequest) -> InferenceResponse:
    # TODO: Replace for custom logic to run inference
    return self._model.predict(payload)
```

Часто вместе с методом `predict` удобно использовать декоратор `mlserver.codecs.decode_args`. Этот декоратор позволяет описать сигнатуру вызовы метода как для полезной нагрузки запроса, так и для ответа. Например
```python
from mlserver import MLModel
from mlserver.codecs import decode_args
from typing import List

class MyCustomRuntime(MLModel):

  async def load(self) -> bool:
    # TODO: Replace for custom logic to load a model artifact
    self._model = load_my_custom_model()
    return True

  @decode_args  # <= NB
  async def predict(self, questions: List[str], context: List[str]) -> np.ndarray:
    # TODO: Replace for custom logic to run inference
    return self._model.predict(questions, context)
```

Сигнатура метода `predict` говорит, что:
- Входные имена, которые мы должны искать в полезной нагрузке запроса -- `questions` и `context`.
- Ожидаемый тип контента для обоих входов -- список строк.
- Ожидаемый тип контента для выхода -- массив NumPy.

Чтобы вернуть любые HTTP заголовки (в случае REST) или метаданные (в случае gRPC), можно добавить любые значения в поле `headers` объекта `Parameters` экземпляра `InferenceResponse`
```python
from mlserver import MLModel
from mlserver.types import InferenceRequest, InferenceResponse

class CustomHeadersRuntime(MLModel):

  ...

  async def predict(self, payload: InferenceRequest) -> InferenceResponse:
    ...
    return InferenceResponse(
      # Include any actual outputs from inference
      outputs=[],
      parameters=Parameters(headers={"foo": "bar"})
    )
```

