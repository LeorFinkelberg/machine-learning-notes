MLflow позволяет логгировать и _системные метрики_, включая статистику CPU и GPU, использование памяти, загрузку сети и прочее.

Для логирования системных метрик во время работы MLflow, требуется установить `psutil` 
```bash
pip install psutil
```

Если требуется логгировать статистику GPU, то нужно установить `pynvml`
```bash
pip install pynvml
```

### Включение/выключение системных метрик при логгировании

Существует 3 способа включить или отключить логирование системных метрик:
- Для того чтобы отключить логгирование во время запуска MLflow, следует переменную окружения `MLFLOW_ENABLE_SYSTEM_METRICS_LOGGIN` выставить в `false` или соответственно в `true` для того чтобы включить логгирование.
- Можно использовать `mlflow.enable_system_metrics_logging()` для включения и `mlflow.disable_system_metrics_logging()` для выключения логгирования системных метрик во время запуска MLflow.
- Использовать `mlflow.start_run(log_system_metrics=True)`.

##### Использование переменной окружения

Переменную окружения можно создать на уровне оболочки
```bash
export MLFLOW_ENABLE_SYSTEM_METRICS_LOGGING=true
```

Или на уровне кода
```bash
import os

os.environ["MLFLOW_ENABLE_SYSTEM_METRICS_LOGGING"] = "true"
```

Тогда запуск
```python
import mlflow
import time

with mlflow.start_run() as run:
    time.sleep(15)

print(mlflow.MlflowClient().get_run(run.info.run_id).data)
```

должен вернуть
```bash
<RunData: metrics={'system/cpu_utilization_percentage': 12.4,
'system/disk_available_megabytes': 213744.0,
'system/disk_usage_megabytes': 28725.3,
'system/disk_usage_percentage': 11.8,
'system/network_receive_megabytes': 0.0,
'system/network_transmit_megabytes': 0.0,
'system/system_memory_usage_megabytes': 771.1,
'system/system_memory_usage_percentage': 5.7}, params={}, tags={'mlflow.runName': 'nimble-auk-61',
'mlflow.source.name': '/usr/local/lib/python3.10/dist-packages/colab_kernel_launcher.py',
'mlflow.source.type': 'LOCAL',
'mlflow.user': 'root'}>
```

##### Использование `mlflow.enable_system_metrics_logging()`

Можно просто разместить конструкцию `mlflow.enable_system_metrics_logging()` в верхней части модуля
```python
import mlflow

mlflow.enable_system_metrics_logging()

with mlflow.start_run() as run:
    time.sleep(5)

print(mlflow.MlflowClient().get_run(run.info.run_id).data)
```

Или, как отмечалось, можно задать параметр `log_system_metrics` для конкретного одиночного запуска
```python
with mlflow.start_run(log_system_metrics=True) as run:
    time.sleep(5)

print(mlflow.MlflowClient().get_run(run.info.run_id).data)
```
