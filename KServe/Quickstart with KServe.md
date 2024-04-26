Детали можно найти на странице документации [KServe Quickstart](https://kserve.github.io/website/latest/get_started/).

### Install the KServe Quickstart environment

KServe (как и, например, Seldon Core) -- платформа для развертывания ML-моделей на кластере Kubernetes.

Перед тем как начать работать с KServe, требуется установить [`kind`](https://kind.sigs.k8s.io/) (Kubernetes in Docker) и Kubernetes CLI ([`kubectl`](https://kubernetes.io/docs/reference/kubectl/)).

Установить `kind` на ОС Linux Manjaro можно так
```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
```

На ОС Linux Manjaro утилиту `kubectl` можно установить менеджером пакетов `pacman`
```bash
$ sudo pacman -S kubectl
```

NB: И `kind`, и `kubectl` нужно выполнять с `sudo`. Иначе будет ошибка "The connection to the server HOST:PORT was refused - did you specify the right host or port?"

1. После установки `kind` можно создать кластер 
```bash
$ sudo kind create cluster
```
2. Затем выполняем
```bash
# kubectl config set-context kind-kind
$ sudo kubectl config get-contexts  # в списке кластеров должен быть кластер с именем "kind-kind"
```

Проверить конфигурацию Kubernetes можно так
```bash
$ sudo kind get kubeconfig
```

Вывести информацию по кластеру в контексте "kind-kind" можно так
```bash
$ kubectl cluster-info --context kind-kind
```

Чтобы использовать контекст "kind-kind" выполняем
```bash
$ sudo kubectl config use-context kind-kind  # Switched to context kind-kind
```

Приступить к _локальному развертыванию_ KServe можно с помощью сценария быстрой установки KServe на Kind
```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.12/hack/quick_install.sh" | bash
```
### First InferenceService

Создаем пространство имен
```bash
$ sudo kubectl create namespace kserve-test
```

