## Быстрый старт

### Запуск MLflow вместе с Tracking Server

Запускаем MLflow Tracking Server. NB: Вообще говоря, запускать сервер первым шагом необязательно, но тогда обязательно следует удалить строку `mlflow.set_tracking_uri("http://127.0.0.1:8080")` из Python-сценария
```bash
mlflow server --host 127.0.0.1 --port 8080
```

Запускать эксперименты можно и с беком на базу данных, но тогда нужно будет задать переменную окружения `MLFLOW_TRACKING_URI`, например, так
```python
MLFLOW_TRACKING_URI=sqlite:///mlruns.db
```

То есть другими словами можно сначала запустить эксперименты локально, а потом запустить `mlflow server` для просмотра результатов через MLflow UI. А можно и сперва запустить `mlflow server --host ... --port`, а потом запустить эксперименты. Но в этом случае нужно будет в Python-файле указать `mlflow.set_tracking_uri("http://127.0.0.1:8080")`.

После запуска сервера в директории проекта появится директория `mlruns/`. Теперь можно запустить сценарий
```python
# simple.py

import mlflow
import pyscipopt
import time
import click


@click.command()
@click.option(
    "--alpha",
    required=False,
    type=click.FLOAT,
)
@click.option(
    "--l1-ratio",
    required=False,
    type=click.FLOAT,
)
def main(alpha: float, l1_ratio: float):
	# здесь допустимо указывать URI, т.к. предварительно был запущен Tracking Server
    mlflow.set_tracking_uri("http://127.0.0.1:8080")  
    mlflow.set_experiment("scip")

    with mlflow.start_run(log_system_metrics=False):
        model = pyscipopt.Model()
        model.readProblem("./2023_08_YANOS_2693.mps")

        print(f"{alpha=}")
        print(f"{l1_ratio=}")

        mlflow.log_param("alpha", alpha)
        mlflow.log_param("l1_ratio", l1_ratio)
        mlflow.log_param("n_bins", model.getNBinVars())
        mlflow.log_param("n_ints", model.getNIntVars())
        mlflow.log_param("n_conss", model.getNConss())

if __name__ == "__main__":
    main()
```
как
```bash
python simple.py --alpha=0.5 --l1-ratio=0.001
```

### Запуск MLflow в conda-окружении на базе MLproject

Пусть директория проекта имеет вид
```bash
2023_08_YANOS_2693.mps
MLproject
conda.yaml
run_mlflow.py
requirements.txt
```

Конфигурационный файл conda-окружения
```bash
# conda.yaml
name: mlflow-env  # Virtual environment name
dependencies:
  - python=3.10
  - conda-forge::scip=8.0.3  # for SCIP 8.0.3; SCIP 8.1.0 works worse
  - conda-forge::pyscipopt  # for PySCIPOpt
  - pip
  - pip:
      - -r requirements.txt  # ./requirements.txt
```

Файл зависимостей
```bash
# requirements.txt
click
pathlib2 >= 2.3.7
python-dotenv >= 0.21.0
pyyaml >= 6.0
tqdm
psutil >= 5.7.3

mlflow >= 2.11.3
```

Конфигурационный файл проекта MLproject
```bash
# MLproject
name: My Project

conda_env: conda.yaml

entry_points:
  main:
    parameters:
      alpha: {type: float, default: 0.15}
      l1_ratio: {type: float, default: 0.001}
    command: "python run_mlflow.py --alpha {alpha} --l1-ratio {l1_ratio}"
```

Сам Python-модуль
```python
# run_mlflow.py

import mlflow
import pyscipopt
import time
import click


@click.command()
@click.option(
    "--alpha",
    required=False,
    type=click.FLOAT,
)
@click.option(
    "--l1-ratio",
    required=False,
    type=click.FLOAT,
)
def main(alpha: float, l1_ratio: float):
    mlflow.set_experiment("scip")

    with mlflow.start_run(log_system_metrics=False):
        model = pyscipopt.Model()
        model.readProblem("./2023_08_YANOS_2693.mps")

        print(f"{alpha=}")
        print(f"{l1_ratio=}")

        mlflow.log_param("alpha", alpha)
        mlflow.log_param("l1_ratio", l1_ratio)
        mlflow.log_param("n_bins", model.getNBinVars())
        mlflow.log_param("n_ints", model.getNIntVars())
        mlflow.log_param("n_conss", model.getNConss())

if __name__ == "__main__":
    main()
```

Чтобы запустить MLflow следует выполнить 
```bash
# имя параметров в Python-модуле `--alpha` и `--l1-ratio`
mlflow run . -P alpha=0.15 -P l1_ratio=0.003 --experiment-name scip
```

>[!WARNING]
>Следует обязательно указывать флаг `--experement-name` [BUG#2735](https://github.com/mlflow/mlflow/issues/2735) 

Можно выполнить еще несколько запусков с различными значениями параметров `alpha` и `l1_ratio`
```bash
mlflow run . -P alpha=0.035 -P l1_ratio=0.003 --experiment-name scip
mlflow run . -P alpha=0.0 -P l1_ratio=0.001 --experiment-name scip
```

Теперь можно запустить сервер для просмотра результатов
```bash
mlflow server --host 127.0.0.1 --port 8080
```

При первом запуске MLflow создается conda-окружение на базе `conda.yaml`. Затем MLflow ищет точку входа по умолчанию (main), поведение которой описано в файле `MLproject` и запускает `python run_mlflow.py --alpha {alpha} --l1-ratio {l1_ratio}`, передавая значения параметров из командной строки (`-P`)

### MLflow Tracking с использованием локальной базы данных

Подробности на странице документации [Tracking Experiments with a Local Database](https://mlflow.org/docs/latest/tracking/tutorials/local-database.html#tracking-experiments-with-a-local-database)

#### Задать tracking-URI для локальной базы данных SQLite

Чтобы MLflow мог подключить локальную базу данных SQLite, следует задать переменную окружения `MLFLOW_TRACKING_URI`
```bash
export MLFLOW_TRACKING_URI=sqlite:///mlruns.db
```

Когда MLflow использует на беке SQlite, база данных будет создана автоматически (если ранее не существовала). В случае других СУБД, требуется сначала создать базу данных.

Теперь можно запустить код, содержащий инструкции логирования
```python
# sqlite_mlflow.py

import mlflow

from sklearn.model_selection import train_test_split
from sklearn.datasets import load_diabetes
from sklearn.ensemble import RandomForestRegressor

mlflow.sklearn.autolog()   # Автологгирование

mlflow.set_experiment("sqlite-mlflow")
with mlflow.start_run():
	db = load_diabetes()
	X_train, X_test, y_train, y_test = train_test_split(db.data, db.target)
	
	# Create and train models.
	rf = RandomForestRegressor(n_estimators=100, max_depth=6, max_features=3)
	rf.fit(X_train, y_train)
	
	# Use the model to make predictions on the test dataset.
	predictions = rf.predict(X_test)
```

с помощью утилиты командной строки `mlflow`
```bash
# Предполагается, что в текущей директории лежит файл MLProject с точкой входа 'db'
$ mlflow run . --entry-point db --experiment-name sqlite-mlflow  # Вызываем точку входа 'db' и запускаем логгирование
# Можно использовать короткий флаг `-e db`
```

После выполнения этой команды, в текущей директории проекта будут созданы поддиректория `mlruns/` и SQLite база данных `mlruns.db`.

Предполагается, что в текущей директории лежит файл `MLproject` 
```bash
name: MLflow-SQLite

conda_env: conda.yaml

entry_points:
  main:
    parameters:
      alpha: {type: float, default: 0.15}
      l1_ratio: {type: float, default: 0.001}
    command: "python run_mlflow.py --alpha {alpha} --l1-ratio {l1_ratio}"

  ...
  db:  # NB!
    command: "python sqlite_mlflow.py"
```

Теперь можно запустить MLflow Server
```bash
$ mlflow server --port 8080 --backend-store-uri ${MLFLOW_TRACKING_URI}
```

Если на модели весит тег (например, `model_type` со значением `sklearn`), то поиск по тегу на странице MLflow UI можно выполнять, используя следующую конструкцию `tags.my_key = "my_value"`. То есть в данном случае будет `tags.model_type = "sklearn`.

Проверить содержимое базы данных SQLite можно так
```python
>>> imoprt sqlite3
>>> import typing as t
>>> con = sqlite3.connect("./mlruns.db")
>>> cur = con.cursor()

# Получить список таблиц
>>> table_names: t.List[str] = [table_name[0] for table_name in cur.execute("SELECT name FROM sqlite_master WHERE type='table';").fetchall()]; table_names
# === OUTPUT ===
# ['experiements', 'alembic_version', 'tags', 'runs', ...]

# Посмотреть схему таблицы в SQLite можно так
>>> cur.execute("PRAGMA table_info('tags');")

# Теперь можно изучить записи таблиц
>>> res = cur.execute("SELECT * FROM runs;").fetchall(); res
>>> res = cur.execute("SELECT * FROM ...").fetchall(); res
```

Чтобы было удобнее работать с таблицами, можно написать простую функцию, которая будет возвращать схему таблицы или список имен полей
```python
import typing as t

def get_schema(
	table_name: str,
	only_field_names: bool = False,
	database: str = "./mlruns.db",
) -> t.Union[
	t.List[t.Tuple[str, ...]],
	t.List[str]
]:
    """
	Examples:
	    ```
		>>> get_schema("tags")
		>>> get_schema("tags", only_field_names=True)
	    ```
    """
    SQL_COMMAND = f"PRAGMA table_info('{table_name}');"
    con = sqlite3.connect(database)
    cur = con.cursor()
    res: t.List[t.Tuple[str, ...]] = cur.execute(SQL_COMMAND).fetchall()

    if only_field_names:
        res: t.List[str] = [row[1] for row in res]
        return res

    return res
```

>[!NOTE] 
> Важно помнить, что MLflow UI это компонент MLflow Server, позволяющий получить доступ к интерфейсу экспериментов

Если вдруг требуется запустить на одной и той же виртуальной машине несколько ML-моделей, то проще всего привязать эти модели к разным портам (подробнее на [StackOverflow](https://stackoverflow.com/questions/70620074/serving-multiple-ml-models-using-mlflow-in-a-single-vm)). Например
```bash
# For RandomForest
$ mlflow models serve --no-conda -m file:///home/yayay/yayay/git/github/mlflow_MTL/src/mlruns/404852075031124987/3e3d293adfcd4c509ce51a445089c417/artifacts/model -h 0.0.0.0 -p 8001 
# For SVR
$ mlflow models serve --no-conda -m file:///home/yayay/yayay/git/github/mlflow_MTL/src/mlruns/909331581180947176/db0f2cbb10a64aeeb768d5408fcb9cca/artifacts/model -h 0.0.0.0 -p 8002
```

Найти модель по псевдониму можно через экземпляр клиента
```python
from mlflow import MlflowClient

CLIENT = MlflowClient()
CLIENT.get_model_version_by_alias(name="sklearn-random-forest-model", alias="best")
```

А вот так можно отфильтровать модели по _тегу на версии_
```python
# тег validation-status весит на версии модели
CLIENT.search_model_version(
	filter_string="name = 'sklearn-random-forest-model' and tags.`validation-status` = 'approved'"
)
```

NB! Когда процедура трекинга запускается с беком на базу данных (например, SQLite), то в директории `mlruns` не будет поддиректорий, связанных с метриками, параметрами, моделями в реестре (поддиректория `models/`) и пр. Когда же процедура запускается с беком на файловую систему, то в директории `mlruns` будут и поддиректории `metrics`, `params`, `tags`, `models` и пр.

