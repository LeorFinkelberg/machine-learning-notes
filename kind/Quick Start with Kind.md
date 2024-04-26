[Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) -- это утилита, предназначенная для локального запуска кластера Kubernetes с использованием Docker. Kind был создан для тестирования Kubernetes, но может использоваться для локальной разработки или CI.
### Установка Kind на Linux

Сменить пароль для root в терминальной сессии можно так
```bash
$ sudo su  # открываем сессию с root-правами
$ whoami  # root
$ passwd  # меняем пароль 
$ exit  # выходим из root-сессии
```

Для установки на Linux (проверялось для Linux Manjaro) следует выполнить 
```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
```

А чтобы создать Kuber-кластер следует выполнить `kind create cluster` предварительно запустив Docker
```bash
$ sudo systemctl start docker
$ sudo kind create cluster
```
