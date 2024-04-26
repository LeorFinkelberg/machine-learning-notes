Чтобы получить все запуски, следует выполнить
```python
import mlflow 

all_runs = mlflow.search_runs(search_all_experiments=True)
# === OUTPUT ===
                             run_id       experiment_id    status  ... tags.mlflow.source.type tags.mlflow.project.entryPoint tags.mlflow.project.backend
0  aeb746277b0b4d9cbd0bbd717561c454  701261616987729867  FINISHED  ...                 PROJECT                 model_registry                       local
1  56efdc2fd167443787b5ebc6da3048fe                   0    FAILED  ...                 PROJECT                 model_registry                       local
2  aa1be47ba6e148a7a1e4ee5a36cda86b                   0  FINISHED  ...                 PROJECT                 model_registry                       local
```

Запуски можно отфильтровать по условию (например, по значению метрики)
```python
import mlflow

bad_runs = mlflow.search_runs(
	filter_string="metrics.loss > 0.8", search_all_experiments=True
)

finished_runs = mlflow.search_runs(
	filter_string="status = 'FINISHED'", search_all_experiments=True
)
```




