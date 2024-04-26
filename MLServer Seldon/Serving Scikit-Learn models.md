Детали можно найти на странице документации [Scikit-Learn runtime for MLServer](https://mlserver.readthedocs.io/en/latest/runtimes/sklearn.html#runtimes-sklearn--page-root). 

Установить пакет можно как обычно с помощью `pip`
```bash
$ pip install mlserver mlserver-sklearn
```

Если тип контента не указан в запросе, то среда выполнения Scikit-Learn попытается расшифровать полезную нагрузку как массив NumPy. Чтобы этого избежать, следует указать верный тип как часть [метаданных модели](https://mlserver.readthedocs.io/en/latest/reference/model-settings.html).

По умолчанию среда выполнения возвращает только выход, полученный с помощью `predict`, однако этим поведением можно управлять. Например, для того чтобы модель возвращала выход от `predict_proba`, следует указать следующее
```bash
{
  "inputs": [
    {
      "name": "my-input",
      "datatype": "INT32",
      "shape": [2, 2],
      "data": [1, 2, 3, 4]
    }
  ],
  "outputs": [
    { "name": "predict_proba" }  # <= NB!
  ]
}
```

По умолчанию, предполагается, что sklearn-модели были сериализованы с помощью `joblib` ([serialised using joblib](https://scikit-learn.org/stable/model_persistence.html)).

Создадим, обучим простой классификатор и сохраним файл с обученной моделью
```python
# Original source code and more details can be found in:
# https://scikit-learn.org/stable/auto_examples/classification/plot_digits_classification.html

# Import datasets, classifiers and performance metrics
import joblib
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
X_train, X_test, y_train, y_test = train_test_split(
    data, digits.target, test_size=0.5, shuffle=False)

# We learn the digits on the first half of the digits
classifier.fit(X_train, y_train)

model_file_name = "mnist-svm.joblib"
joblib.dump(classifier, model_file_name)
```

После запуска этого сценария в текущей директории проекта будет создан файл `mnist-svm.joblib`.

Теперь нужно создать два конфигурационных файла: один для MLServer, другой для модели.
```bash
# settings.json
{
    "debug": "true"
}
```
и
```bash
# model-settings.json
{
    "name": "mnist-svm",
    "implementation": "mlserver_sklearn.SKLearnModel",
    "parameters": {
        "uri": "./mnist-svm.joblib",
        "version": "v0.1.0"
    }
}
```

Теперь можно запустить MLServer из того же каталога, где лежат конфигурационные файлы
```bash
$ mlserver start .
```

Для того чтобы убедиться, что все работает корректно пошлем запрос 
```python
import requests

x_0 = X_test[0:1]
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


