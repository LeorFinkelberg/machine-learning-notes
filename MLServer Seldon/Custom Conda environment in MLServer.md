Нет ничего необычного в том, что модель требует дополнительные зависимости, которые не являются прямыми зависимостями MLServer. Это тот случай, когда приходится использовать пользовательские среды выполнения ([Custom Runtimes](https://mlserver.readthedocs.io/en/latest/examples/custom/README.html)).

Docker-образ `seldonio/mlserver` позволяет загружать пользовательские окружения (custom environments) перед запуском сервера.

В примере создадим пользовательское окружение для развертывания модели, обученной с помощью более старой версии Scikit-Learn. Первым шагом создадим конфигурационный файл
```bash
%%writefile environment.yml

name: old-sklearn
channels:
    - conda-forge
dependencies:
    - python == 3.8
    - scikit-learn == 0.24.2
    - joblib == 0.17.0
    - requests
    - pip
    - pip:
        - mlserver == 1.1.0
        - mlserver-sklearn == 1.1.0
```

Создаем и активируем виртуальное окружение
```
$ conda env create --force -f environment.yml
$ conda activate old-sklearn
```

Теперь можно обучить и сохранить Scikit-Learn модель с использованием старой версии окружения. Модель будет сериализована как `model.joblib`
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

model_file_name = "model.joblib"
joblib.dump(classifier, model_file_name)
```

Наконец, нужно сериализовать наше окружение в формат, который ожидает MLServer. Чтобы это сделать воспользуемся утилитой [`conda-pack`](https://conda.github.io/conda-pack/)^[`conda-pack` -- это утилита командной строки, предназначенная для архивирования виртуальных окружений Conda]

Эта утилита сохраняет портируемую версию виртуального окружения в виде tar-архива (tarball)
```bash
$ conda pack --force -n old-sklearn -o old-sklearn.tar.gz
```

```bash
# model-settings.json
{
    "name": "mnist-svm",
    "implementation": "mlserver_sklearn.SKLearnModel"
}
```

Развернуть модель можно с помощью Docker-образа от Seldon
```bash
docker run -it --rm \
    -v "$PWD":/mnt/models \
    -e "MLSERVER_ENV_TARBALL=/mnt/models/old-sklearn.tar.gz" \
    -p 8080:8080 \
    seldonio/mlserver:1.1.0-slim
```

Открываем (expose) порт `8080`, чтобы можно отправлять запросы извне. Естественно чтобы можно было отправлять запросы сервер должен быть запущен.

Отправляем POST-запрос
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

endpoint = "http://localhost:8080/v2/models/mnist-svm/infer"
response = requests.post(endpoint, json=inference_request)

response.json()
```