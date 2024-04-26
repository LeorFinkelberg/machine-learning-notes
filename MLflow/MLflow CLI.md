## Полезные конструкции утилиты командной строки `mlflow`

Подробности можно найти [здесь](https://mlflow.org/docs/latest/cli.html#command-line-interface)
### list

Вывести артефакты, привязанные к конкретному запуске
```bash
$ mlflow artifacts list --run-id aeb746277b0b4d9cbd0bbd717561c454
```

### search

Найти все эксперименты
```bash
$ mlflow experiments search -v all
# === OUTPUT ===
Experiment Id       Name            Artifact Location
------------------  --------------  ---------------------------------------------------------------------------------------
0                   Default         file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/0
701261616987729867  registry-model  file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/701261616987729867
899445604307150821  model-registry  file:///C:/Users/AlexanderPodvoyskiy/Documents/GARBAGE/mlflow/mlruns/899445604307150821
```