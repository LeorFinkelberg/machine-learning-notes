Подробности можно найти на странице документации [Open Inference Protocol](https://kserve.github.io/website/latest/modelserving/data_plane/v2_protocol/#httprest).

Чтобы сервер логического вывода был совместим с протоколом Open Inference Protocol (V2 Inference Protocol), сервер должен поддерживать health, metadata и inference. Совместимый сервер логического вывода может выбирать реализацию: HTTP/REST API и/или gRPC API.

Пример запроса на логический вывод модели с 2 входами и одним выходом
```bash
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "id" : "42",
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",
      "data" : [ 1, 2, 3, 4 ]
    },
    {
      "name" : "input1",
      "shape" : [ 3 ],
      "datatype" : "BOOL",
      "data" : [ true ]
    }
  ],
  "outputs" : [
    {
      "name" : "output0"
    }
  ]
}
```

Пример ответа 
```bash
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: <yy>
{
  "id" : "42"
  "outputs" : [
    {
      "name" : "output0",
      "shape" : [ 3, 2 ],
      "datatype"  : "FP32",
      "data" : [ 1.0, 1.1, 2.0, 2.1, 3.0, 3.1 ]
    }
  ]
}
```