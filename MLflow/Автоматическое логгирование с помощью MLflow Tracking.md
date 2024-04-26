[Автологгирование](https://mlflow.org/docs/latest/tracking/autolog.html#scikit-learn) удобный инструмент, позволяющий логгировать метрики, параметры, модели и пр. без явного указания соответствующих инструкций.
```python
import mlflow

mlflow.autolog()

# mlflow.set_experiment("expr-name")  # здесь можно задать имя эксперимента
with mlflow.start_run():
    # your training code goes here
```

Можно управлять поведением автологгирования
```python
import mlflow

mlflow.autolog(
	log_model_signature=False,
	extra_tags={"YOUR_TAG": "VALUE"},
)
```

Можно отключать логгирование для конкретных библиотек
```python
import mlflow

# Option 1: Enable autologging only for PyTorch
mlflow.pytorch.autolog()

# Option 2: Disable autologging for scikit-learn, but enable it for other libraries
mlflow.sklearn.autolog(disable=True)
mlflow.autolog()
```

Перечень поддерживаемых библиотек (LightGBM, Scikit-learn, XGBoost etc.) можно найти [здесь](https://mlflow.org/docs/latest/tracking/autolog.html#supported-libraries)