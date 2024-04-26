Проект можно описать с помощью специального файла `MLproject`, который представляет собой YAML-файл.

##### Файл `MLproject`

`MLproject` располагается в корне проекта
```bash
name: My Project

python_env: python_env.yaml
# or
# conda_env: my_env.yaml
# or
# docker_env:
#    image:  mlflow-docker-example

entry_points:
  main:
    parameters:
      data_file: path
      regularization: {type: float, default: 0.1}
    command: "python train.py -r {regularization} {data_file}"
  validate:
    parameters:
      data_file: path
    command: "python validate.py {data_file}"
```

Поле `python_env` должно быть относительным путем до конфигурационного файла `python_env.py`, расположенного в директории проекта
```bash
python_env: files/config/python_env.yaml
```

Пример `python_env.yaml`
```bash
# Python version required to run the project.
python: "3.8.15"
# Dependencies required to build packages. This field is optional.
build_dependencies:
  - pip
  - setuptools
  - wheel==0.37.1
# Dependencies required to run the project.
dependencies:
  - mlflow==2.3
  - scikit-learn==1.0.2
```

В случае `conda_env` следует соответственно указать путь до `conda_env.yaml`
```bash
conda_env: files/config/conda_env.yaml
```

MLflow создаст виртуальное окружение conda и запустит точку входа.

А в случае `docker_env` следует указать имя Docker-образа
```bash
docker_env:
  image: mlflow-docker-example-environment  # Docker image with default tag 'latest'
```

Кроме того для Docker-образа можно монтировать тома (опция Docker `-v`) и определять переменные окружения (опция Docker `-e`). Переменные окружения могут либо копироваться с хоста, либо создаваться для окружения Docker. Поле _переменной окружения_ должно быть списком. Элементы списка могут быть либо списком строк (для создания новых переменных окружения), либо просто строкой (для копирования значений переменных окружения с хоста)
```bash
docker_env:
  image: mlflow-docker-example-environment
  volumes: ["/local/path:/container/mount/path"]
  environment: [["NEW_ENV_VAR", "new_var_value"], "VAR_TO_COPY_FROM_HOST_ENVIRONMENT"]
```

Если Docker-образ размещается удалено, то нужно просто передать путь до удаленного реестра
```bash
docker_env:
  image: 012345678910.dkr.ecr.us-west-2.amazonaws.com/mlflow-docker-example-environment:7.0
```

Здесь `docker_env` ссылается на Docker-образ с именем `mlflow-docker-example-environment` и тегом `7.0` в реестре Docker-образов, расположенном по пути `012345678910.dkr.ecr.us-west-2.amazonaws.com`.

Когда проект MLflow будет запущен, Docker загрузит образ из указанного Docker-реестра. Система, выполняющая MLflow-проект должна иметь соответствующие права.

MLflow позволяет указывать тип данных и значение по умолчанию для каждого параметра
```bash
parameter_name: {type: data_type, default: value}  # Short syntax

parameter_name:  # Long syntax
  type: data_type
  default: value
```

MLflow поддерживает 4 типа параметров:
- `string`: тестовая строка
- `float`: вещественное число; MLflow будет проверять, что переданное значение является числом
- `path`: путь на локальной файловой системе. MLflow преобразует относительный путь в абсолютный
- `uri`: URI для данных на локальной или распределенной системе хранения. MLflow преобразует относительные пути в абсолютные

Git-проект (например, https://github.com/mlflow/mlflow-example) можно запустить так
```bash
# В файле с CLI параметры должны иметь вид --alpha и --l1-ratio
mlflow run git@github.com:mlflow/mlflow-example.git -P alpha=0.5 -P l1_ratio=0.0001
```


