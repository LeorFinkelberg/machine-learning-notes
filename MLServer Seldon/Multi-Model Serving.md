Детали можно на странице документации [Mulfit-Model Serviing](https://mlserver.readthedocs.io/en/latest/examples/mms/README.html).

MLServer был спроектирован с учетом мультимодельного обслуживания. Это означает, что внутри одного экземпляра MLServer можно запустить несколько моделей под разными путями (причем с учетом различных версий моделей).

Начнем с обучения 2 различных ML-моделей: `mnist-svm` (scikit-learn) и `mushroom-xgboost` (xboost).

Обучаем модель `mnist-svm`
```python
# Original source code and more details can be found in:
# https://scikit-learn.org/stable/auto_examples/classification/plot_digits_classification.html

# Import datasets, classifiers and performance metrics
from sklearn import datasets, svm, metrics
from sklearn.model_selection import train_test_split

# The digits dataset
digits = datasets.load_digits()

# To apply a classifier on this data, we need to flatten the image, to
# turn the data in a (samples, feature) matrix:
n_samples = len(digits.images)
data = digits.images.reshape((n_samples, -1))

# Create a classifier: a support vector classifier
classifier = svm.SVC(gamma=0.001)

# Split data into train and test subsets
X_train, X_test_digits, y_train, y_test_digits = train_test_split(
    data, digits.target, test_size=0.5, shuffle=False)

# We learn the digits on the first half of the digits
classifier.fit(X_train, y_train)
```

Обучаем модель `mushroom-xgboost`
```python
# Original code and extra details can be found in:
# https://xgboost.readthedocs.io/en/latest/get_started.html#python

import os
import xgboost as xgb
import requests

from urllib.parse import urlparse
from sklearn.datasets import load_svmlight_file


TRAIN_DATASET_URL = 'https://raw.githubusercontent.com/dmlc/xgboost/master/demo/data/agaricus.txt.train'
TEST_DATASET_URL = 'https://raw.githubusercontent.com/dmlc/xgboost/master/demo/data/agaricus.txt.test'


def _download_file(url: str) -> str:
    parsed = urlparse(url)
    file_name = os.path.basename(parsed.path)
    file_path = os.path.join(os.getcwd(), file_name)
    
    res = requests.get(url)
    
    with open(file_path, 'wb') as file:
        file.write(res.content)
    
    return file_path

train_dataset_path = _download_file(TRAIN_DATASET_URL)
test_dataset_path = _download_file(TEST_DATASET_URL)

# NOTE: Workaround to load SVMLight files from the XGBoost example
X_train, y_train = load_svmlight_file(train_dataset_path)
X_test_agar, y_test_agar = load_svmlight_file(test_dataset_path)
X_train = X_train.toarray()
X_test_agar = X_test_agar.toarray()

# read in data
dtrain = xgb.DMatrix(data=X_train, label=y_train)

# specify parameters via map
param = {'max_depth':2, 'eta':1, 'objective':'binary:logistic' }
num_round = 2
bst = xgb.train(param, dtrain, num_round)

bst
```

Следующим шагом будет развертывание обеих моделей на одном и том же экземпляре сервера MLServer. Для этого нужно создать конфигурационный файл `model-settings.json` для каждой ML-модели и конфигурационный файл `settings.json` для сервера:
- `settings.json`: содержит конфигурацию для сервера (порты, уровень логирования etc.),
- `models/mnist-svm/model-settings.json`: содержит конфигурацию специфичную для модели `mnist-svm` (тип входных данных, среда выполнения etc.),
- `models/mushroom-xgboost/model-settings.json`: содержит конфигурацию специфичную для модели `mushroom-xgboost` (тип входных данных, среда выполнения etc.).

```bash
# settings.json
{
    "debug": "true"
}
```

```bash
# models/mnist-svm/model-settings.json
{
    "name": "mnist-svm",
    "implementation": "mlserver_sklearn.SKLearnModel",
    "parameters": {
        "version": "v0.1.0"
    }
}
```

То есть директория проекта выглядит следующим образом
```bash
mlserver-test/
  - mnist-svm.py
  - mushroom-xgboost.py
  - settings.json  # for MLServer 
  
  - models/
    - mnist-svm/
      - model-settings.json  # URI модели не указывается
      - model.joblib
    - mushroom-xgboost/
      - model-settings.json  # URI модели не указывается
      - model.json
```
Запускаем MLServer
```bash
$ mlserver start .
```

Проверяем все ли работает как ожидается. Тестируем модель `mnist-svm`
```python
import requests

x_0 = X_test_digits[0:1]
inference_request = {
    "inputs": [
        {
          "name": "predict",
          "shape": x_0.shape,
          "datatype": "FP32",
          "data": x_0.tolist()
        }
    ]
}

endpoint = "http://localhost:8080/v2/models/mnist-svm/versions/v0.1.0/infer"
response = requests.post(endpoint, json=inference_request)

response.json()
```

Тестируем `mushroom-xgboost`
```python
import requests

x_0 = X_test_agar[0:1]
inference_request = {
    "inputs": [
        {
          "name": "predict",
          "shape": x_0.shape,
          "datatype": "FP32",
          "data": x_0.tolist()
        }
    ]
}

endpoint = "http://localhost:8080/v2/models/mushroom-xgboost/versions/v0.1.0/infer"
response = requests.post(endpoint, json=inference_request)

response.json()
```
