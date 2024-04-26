Детали можно найти на странице документации [Content Types (and Codecs)](https://mlserver.readthedocs.io/en/latest/user-guide/content-type.html).

MLServer поддерживает _типы контента_ (content types), которые позволяют ему понять как декодировать полезную нагрузку, совместимую с протоколом Open Inference Protocol.

Для примера рассмотрим конвейер Scikit-Learn, который принимает кадр данных Pandas и возвращает массив NumPy. В этом сценарии использование типов контента позволяет определить информацию о том, какая фактическая информация более высокого уровня закодирована в полезных нагрузках протокола V2.![[content-type.svg]]

В качестве примера разберем вариант с простой таблицей
![[table_two_cols.png]]

Эта таблица может быть определена в протоколе V2:
- Весь набор данных должен быть декодирован как кадр данных DataFrame (`"content_type": "pd"`).
- Столбец "Fist Name" должен быть декодирован как строка в кодировке UTF-8 (`"content_type": "str"`).

```bash
{
  "parameters": {
    "content_type": "pd"  # <= NB
  },
  "inputs": [
    {
      "name": "First Name",
      "datatype": "BYTES",
      "parameters": {
        "content_type": "str"  # <= NB
      },
      "shape": [2],
      "data": ["Joanne", "Michael"]
    },
    {
      "name": "Age",
      "datatype": "INT32",
      "shape": [2],
      "data": [34, 22]
    },
  ]
}
```

Под капотом преобразования между типами контента реализуются с помощью _кодеков_ (codecs). В архитектуре MLServer кодеки представляют собой абстракцию, которая знает как кодировать и декодировать высокоуровневые Python-типы в протокол V2 Inference Protocol и обратно.

Кодеки умеют работать как на уровне запрос-ответ (request codecs), так и на уровне вход-выход (input codecs).
Request Codecs:
- `encode_request()`
- `decode_request()`
- `encode_response()`
- `decode_response()`

Input Codecs:
- `encode_input()`
- `decode_input()`
- `encode_output()`
- `decode_output()`

Эти методы могут использоваться для кодирования запросов и декодирования ответов на стороне клиента.

Пример использования кодеков для кодирования кадра данных Pandas в V2-совместимый запрос
```python
import pandas as pd

from mlserver.codecs import PandasCodec

dataframe = pd.DataFrame({'First Name': ["Joanne", "Michael"], 'Age': [34, 22]})

inference_request = PandasCodec.encode_request(dataframe)
print(inference_request)
```

[Здесь](https://mlserver.readthedocs.io/en/latest/examples/content-type/README.html#examples-content-type-readme--page-root) можно найти полный пример, объясняющий как типы контента и кодеки работают под капотом.
