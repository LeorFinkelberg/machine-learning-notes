https://github.com/mlflow/mlflow/tree/master/examples/hyperparam

Создадим два эксперимента
```bash
$ mlflow experiments create -n individual_runs
$ mlflow experiments create -n hyper_param_runs
```

Теперь можно вывести информацию по экспериментам
```bash
$ mlflow experiments search -v all
# === OUTPUT ===
Experiment Id       Name              Artifact Location
------------------  ----------------  ---------------------------------------------------------------------------------------
0                   Default           file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/0
284493265382801763  individual_runs   file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/284493265382801763
332330110459272570  hyper_param_runs  file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/332330110459272570
```

Теперь можно запустить точку входа `hyperopt`
```bash
$ mlflow run -e hyperopt --experiment-id <hyperparam_experiment_id> examples/hyperparam
```

Затем можно сравнить результаты с помощью `mlflow ui`.
Директория проекта имеет вид
```bash
MLproject
README.rst
python_env.yaml
search_hyperopt.py
search_random.py
train.py
```

Конфигурационный файл проекта настроен следующим образом
```bash
# MLproject

name: HyperparameterSearch

python_env: python_env.yaml

entry_points:
  # train Keras DL model
  train:
    parameters:
      training_data: {type: string, default: "https://raw.githubusercontent.com/mlflow/mlflow/master/tests/datasets/winequality-white.csv"}
      epochs: {type: int, default: 32}
      batch_size: {type: int, default: 16}
      learning_rate: {type: float, default: 1e-1}
      momentum: {type: float, default: .0}
      seed: {type: int, default: 97531}
    command: "python train.py {training_data}
                                    --batch-size {batch_size}
                                    --epochs {epochs}
                                    --learning-rate {learning_rate}
                                    --momentum {momentum}"

  # Use random search to optimize hyperparams of the train entry_point.
  random:
    parameters:
      training_data: {type: string, default: "https://raw.githubusercontent.com/mlflow/mlflow/master/tests/datasets/winequality-white.csv"}
      max_runs: {type: int, default: 8}
      max_p: {type: int, default: 2}
      epochs: {type: int, default: 32}
      metric: {type: string, default: "rmse"}
      seed: {type: int, default: 97531}
    command: "python search_random.py  {training_data}
                                             --max-runs {max_runs}
                                             --max-p {max_p}
                                             --epochs {epochs}
                                             --metric {metric}
                                             --seed {seed}"

  # Use Hyperopt to optimize hyperparams of the train entry_point.
  hyperopt:
    parameters:
      training_data: {type: string, default: "https://raw.githubusercontent.com/mlflow/mlflow/master/tests/datasets/winequality-white.csv"}
      max_runs: {type: int, default: 12}
      epochs: {type: int, default: 32}
      metric: {type: string, default: "rmse"}
      algo: {type: string, default: "tpe.suggest"}
      seed: {type: int, default: 97531}
    command: "python -O search_hyperopt.py {training_data}
                                                 --max-runs {max_runs}
                                                 --epochs {epochs}
                                                 --metric {metric}
                                                 --algo {algo}
                                                 --seed {seed}"

  main:
    parameters:
      training_data: {type: string, default: "https://raw.githubusercontent.com/mlflow/mlflow/master/tests/datasets/winequality-white.csv"}
    command: "python search_random.py {training_data}"
```