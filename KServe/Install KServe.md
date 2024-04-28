Детали можно найти на странице документации [Serverless Installation Guide](https://kserve.github.io/website/latest/admin/serverless/serverless/).

Установка KServe Serverless обеспечивает _автомасштабирование_ и _масштабирование до нуля_. А также поддерживает управление версиями и канареечное развертывание.

Кроме того KServe поддерживает режим `RawDeployment`,  позволяющий развертывать `InferenceService` с помощью ресурсов Kubernetes: `Deployment`, `Service`, `Ingress` и `Horizontal Pod Autoscaler` (см. [Kubernetes Deployment Installation Guide](https://kserve.github.io/website/latest/admin/kubernetes_deployment/)). Если сравнивать с бессерверным развертыванием, то с одной стороны сырое развертывание снимает некоторые ограничения Knative, такие как множественное монтирование томов, а с другой стороны `RawDeployment` не поддерживает `Scale down and from Zero`.

Knative (см. [Concepts](https://knative.dev/docs/concepts/)) -- это платформонезависмое решение, предназначенное для запуска бессерверного развертывания (serverless deployments).






