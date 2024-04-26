MLflow Model Registry - это централизованное хранилище моделей.
## Добавление MLflow Model в Model Registry

Существует [3 способа](https://mlflow.org/docs/latest/model-registry.html#adding-an-mlflow-model-to-the-model-registry) добавить модель реестр:
- использовать метод `mlflow.<model_flavor>.log_model()`
```python
...
mlflow.sklearn.log_model(
	sk_model=model,
	artifact_path="sklearn-model",  # имя поддиректории (с артефактами, `MLmodel`, `conda.yaml`, `model.pkl` etc.) директории artifacts/
	signature=signature,
	registered_model_name="sk-learn-random-forest-model",  # имя поддиректории в директории models/
)
```
- использовать метод `mlflow.register_model()`; для этого метода требуется указать `run_id`; этот метод регистрирует пустую модель без привязки к версии
```python
result = mlflow.register_model(
	"runs:/d160...539c0/sklearn-model",
	"sk-learn-random-forest-reg-model",
)
```
- можно использовать `create_model_version()` 
```python
client = MlflowClient()
result = client.create_model_version(
    name="sk-learn-random-forest-reg-model",
    source="mlruns/0/d16076a3ec534311817565e6527539c0/artifacts/sklearn-model",
    run_id="d16076a3ec534311817565e6527539c0",
)
```

Пример обработки случая необязательной регистрации модели в реестре
```python
	...
	mlflow.pyfunc.log_model(
		"model",
		code_path=["models"],
		python_model=wrapper(trained_model, column_transformer),
		input_example=train_data.dropna(axis=0).sample(5),
        # registered_model_name: t.Optional[str]
		registered_model_name=registered_model_name,
	)
    run_id = run.info.run_id
    
return_uri = (
	f"models:/{registered_model_name}/{version}"
	if registered_model_name
	else f"runs:/{run_id}/model"
)
```

## Извлечение MLflow Model из Model Registry

Получить конкретную версию модели можно так
```python
import mlflow.pyfunc

model_name = "sk-learn-random-forest-reg-model"
model_version = 1

model = mlflow.pyfunc.load_model(model_uri=f"models:/{model_name}/{model_version}")

model.predict(data)
```

Получить конкретную версию модели из реестра MLflow Model Registry по псевдониму можно так
```python
import mlflow.pyfunc

model_name = "sk-learn-random-forest-reg-model"
alias = "champion"

champion_version = mlflow.pyfunc.load_model(f"models:/{model_name}@{alias}")

champion_version.predict(data)
```

Найти зарегистрированную модель можно [так](https://mlflow.org/docs/latest/model-registry.html#listing-and-searching-mlflow-models) 
```python
from pprint import pprint

client = MlflowClient()
for rm in client.search_registered_models():
    pprint(dict(rm), indent=4)
```

Вывод
```base
{   'creation_timestamp': 1582671933216,
    'description': None,
    'last_updated_timestamp': 1582671960712,
    'latest_versions': [<ModelVersion: creation_timestamp=1582671933246, current_stage='Production', description='A random forest model containing 100 decision trees trained in scikit-learn', last_updated_timestamp=1582671960712, name='sk-learn-random-forest-reg-model', run_id='ae2cc01346de45f79a44a320aab1797b', source='./mlruns/0/ae2cc01346de45f79a44a320aab1797b/artifacts/sklearn-model', status='READY', status_message=None, user_id=None, version=1>,
                        <ModelVersion: creation_timestamp=1582671960628, current_stage='None', description=None, last_updated_timestamp=1582671960628, name='sk-learn-random-forest-reg-model', run_id='d994f18d09c64c148e62a785052e6723', source='./mlruns/0/d994f18d09c64c148e62a785052e6723/artifacts/sklearn-model', status='READY', status_message=None, user_id=None, version=2>],
    'name': 'sk-learn-random-forest-reg-model'}
```

Можно искать версии модели
```python
client = MlflowClient()
for mv in client.search_model_versions("name='sk-learn-random-forest-reg-model'"):
    pprint(dict(mv), indent=4)
```



