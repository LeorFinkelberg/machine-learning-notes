Подробности можно найти здесь https://github.com/minio/openlake/blob/main/spark/spark-streaming.md?ref=blog.min.io#pyspark-structured-streaming-application

### Настройка Kafka на Kubernetes

Детали можно найти здесь https://github.com/minio/openlake/blob/main/kafka/setup-kafka.ipynb

Прежде чем начнем, нужно убедится, что у нас есть:
- запущенный Kubernetes-кластер
- утилита командной строки `kubectl`
- запущенный кластер MinIO
- утилита командной строки `mc` для работы с MinIO
- менеджер пакетов helm

#### Установка оператора Strimzi

Первым шагом установим оператор Strimzi на Kuber-кластер. Оператор Strimzi управляет жизненным циклы класетров Kafka и  Zookeeper на Kuber-кластере.

Добавим helm-чарт для Strimzi
```bash
$ helm repo add strimzi https://strimzi.io/charts/
```

Установим чарт с именем `my-release`
```bash
$ helm install my-release strimzi/strimzi-kafka-operator \
    --namespace=kafka \
    --create-namespace
```
Эта команда устанавливает последнюю версию оператора в пространство имен `kafka`.
#### Создаем кластер Kafka

У нас есть установленный Strimzi-оператор и теперь мы можем создать Kafka-кластер. Здесь мы создаем Kafka-кластер с 3 брокерами сообщений и 3 узлами Zookeeper
```yaml
# deployment/kafka-cluster.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-kafka-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.4.0
    replicas: 3  # <= 3 брокера Kafka
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.4"
		storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
  zookeeper:
    replicas: 3  # <= 3 узла Zookeeper
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

Создадим Kafka-кластер
```bash
$ kubectl apply -f deployment/kafka-cluster.yaml
```

Проверим статус кластера
```bash
$ kubectl -n kafka get kafka my-kafka-cluster
```
#### Создаем топик Kafka

Напишем YAML-файл для создания топика в Kafka
```yaml
# deployment/kafka-my-topic.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-kafka-cluster
spec:
  partitions: 3
  replicas: 3
```

```bash
$ kubectl apply -f deployment/kafka-my-topic.yaml
```

Проверяем статус топика
```bash
$ kubectl -n kafka get kafkatopic my-topic
```

#### Отправка и потребление сообщений

Настроив Kafka-кластер и создав топик, мы теперь можем отправлять и получать сообщения. Создадим под Kafka-продюсера для порождения сообщений для топика `my-topic`
```bash
$ kubectl -n kafka run kafka-producer -ti \
    --image=quay.io/strimzi/kafka:0.34.0-kafka-3.4.0 \
    --rm=true \
    --restart=Never -- bin/kafka-console-producer.sh \
    --broker-list my-kafka-cluster-kafka-bootstrap:9092 \
    --topic my-topic
```

Также мы можем запустить Kafka-консюмера для потребления сгенерированных сообщений
```bash
$ kubectl -n kafka run kafka-consumer -ti \
    --image=quay.io/strimzi/kafka:0.34.0-kafka-3.4.0 \
    --rm=true \
    --restart=Never -- bin/kafka-console-consumer.sh \
    --bootstrap-server my-kafka-cluster-kafka-bootstrap:9092 \
    --topic my-topic \
    --from-beginning
```

Консюмер отобразит все сообщения, которые пришли от продюсера.

Чтобы удалить топик, делаем 
```bash
$ kubectl -n kafka delete kafkatopic my-topic
```

### Kafka Schema Registry MinIO

Детали можно найти здесь https://github.com/minio/openlake/blob/main/kafka/kafka-schema-registry-minio.ipynb

Kafka Schema Registry -- это компонент экосистемы Apache Kafka, который помогает продюсерам генерировать данные в соответствие с заданной схемой, а консюмерам потреблять данные в соответствие с указанной схемой.

Клонируем репозиторий `helm` и переходим в поддиректорию `cp-schema-registry`
```bash
$ git clone https://github.com/confluentinc/cp-helm-charts.git
$ cd cp-helm-charts/charts/cp-schema-registry
```

Устанавливаем Kafka Schema Registry
```bash
$ helm install kafka-schema-registry \
    --set kafka.bootstrapServers="PLAINTEXT://my-kafka-cluster-kafka-bootstrap:9092" . -n kafka
```

Смотрим логи и проверяем запуск Schema Registry
```bash
$ kubectl -n kafka logs -f \
    --selector=app=cp-schema-registry \
    -c cp-schema-registry-server # stop this shell once you are done
```

#### Создаем топик Avro

Создадим топик для Kafka
```yaml
# deployment/kafka-nyc-avro-topic.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: nyc-avro-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-kafka-cluster
spec:
  partitions: 3
  replicas: 3
```

Применяем 
```bash
$ kubectl apply -f deployment/kafka-nyc-avro-topic.yaml
```

Проверяем топик
```bash
$ kubectl -n kafka get kafkatopic nyc-avro-topic
```
#### Producer with Avro Schema

Здесь мы создадим простого продюсера, затем зарегистрируем Avro-схему с помощью Kafka Schema Registry и отравим сообщения в топик Kafka
```python
# sample-code/producer/src/avro-producer.py

import logging
import os

import fsspec
import pandas as pd
import s3fs
from avro.schema import make_avsc_object
from confluent_kafka.avro import AvroProducer

logging.basicConfig(level=logging.INFO)

# Avro schema
value_schema_dict = {
    "type": "record",
    "name": "nyc_avro",
    "fields": [
        {
            "name": "VendorID",
            "type": "long"
        },
        {
            "name": "tpep_pickup_datetime",
            "type": "string"
        },
        {
            "name": "tpep_dropoff_datetime",
            "type": "string"
        },
        {
            "name": "passenger_count",
            "type": "double"
        },
        {
            "name": "trip_distance",
            "type": "double"
        },
        {
            "name": "RatecodeID",
            "type": "double"
        },
        {
            "name": "store_and_fwd_flag",
            "type": "string"
        },
        {
            "name": "PULocationID",
            "type": "long"
        },
        {
            "name": "DOLocationID",
            "type": "long"
        },
        {
            "name": "payment_type",
            "type": "long"
        },
        {
            "name": "fare_amount",
            "type": "double"
        },
        {
            "name": "extra",
            "type": "double"
        },
        {
            "name": "mta_tax",
            "type": "double"
        },
        {
            "name": "tip_amount",
            "type": "double"
        },
        {
            "name": "tolls_amount",
            "type": "double"
        },
        {
            "name": "improvement_surcharge",
            "type": "double"
        },
        {
            "name": "total_amount",
            "type": "double"
        },
    ]
}

value_schema = make_avsc_object(value_schema_dict)

producer_config = {
    "bootstrap.servers": "my-kafka-cluster-kafka-bootstrap:9092",
    "schema.registry.url": "http://kafka-schema-registry-cp-schema-registry:8081"
}

producer = AvroProducer(producer_config, default_value_schema=value_schema)

fsspec.config.conf = {
    "s3":
        {
            "key": os.getenv("AWS_ACCESS_KEY_ID", "openlakeuser"),
            "secret": os.getenv("AWS_SECRET_ACCESS_KEY", "openlakeuser"),
            "client_kwargs": {
                "endpoint_url": "https://play.min.io:50000"
            }
        }
}
s3 = s3fs.S3FileSystem()
total_processed = 0
i = 1
for df in pd.read_csv('s3a://openlake/spark/sample-data/taxi-data.csv', chunksize=10000):
    count = 0
    for index, row in df.iterrows():
        producer.produce(topic="nyc-avro-topic", value=row.to_dict())
        count += 1

    total_processed += count
    if total_processed % 10000 * i == 0:
        producer.flush()
        logging.info(f"total processed till now {total_processed} for topic 'nyc-avro-topic'")
        i += 1
```

Файл зависимостей проекта выглядит так
```bash
# requirements.txt
pandas==2.0.0
s3fs==2023.4.0
pyarrow==11.0.0
kafka-python==2.0.2
confluent_kafka[avro]==2.1.0
```

А Dockerfile такой 
```dockerfile
# Dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN pip3 install -r requirements.txt

COPY src/avro-producer.py .
CMD ["python3", "-u", "./avro-producer.py"]
```

Разворачиваем продюсера на Kuber-кластере как задание (job)
```yaml
# deployment/avro-producer.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: avro-producer-job
  namespace: kafka
spec:
  template:
    metadata:
      name: avro-producer-job
    spec:
      containers:
      - name: avro-producer-job
        image: openlake/kafka-demo-avro-producer:latest
      restartPolicy: Never
```

И собственно применяем YAML-файл
```bash
$ kubectl apply -f deployment/avro-producer.yaml
```

Проверяем логи
```bash
$ kubectl logs -f job.batch/avro-producer-job -n kafka # stop this shell once you are done
```

#### Создаем KafkaConnect

Теперь соберем образ Kafka Connect с зависимостями от S3 и Avro
```dockerfile
%%writefile sample-code/connect/Dockerfile
FROM confluentinc/cp-kafka-connect:7.0.9 as cp
RUN confluent-hub install --no-prompt confluentinc/kafka-connect-s3:10.4.2
RUN confluent-hub install --no-prompt confluentinc/kafka-connect-avro-converter:7.3.3
FROM quay.io/strimzi/kafka:0.34.0-kafka-3.4.0
USER root:root
# Add S3 dependency
COPY --from=cp /usr/share/confluent-hub-components/confluentinc-kafka-connect-s3/ /opt/kafka/plugins/kafka-connect-s3/
# Add Avro dependency
COPY --from=cp /usr/share/confluent-hub-components/confluentinc-kafka-connect-avro-converter/ /opt/kafka/plugins/avro/
```

Перед развертыванием `KafkaConnect` мы должны создать топики для хранения ([storage topics](https://github.com/minio/openlake/blob/2177ce305bdf81fd1d826dc3def3f12a4b251f4e/kafka/kafka-minio.ipynb#Create-Storage-Topics)) -- `connect-status`, `connect-configs` и `connect-offers`. 

Создадим YAML-файл для создания ресурса `KafkaConnect` , который использует созданный ранее образ и разворачивает его на k8s.
```yaml
# deployment/avro-connect.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: avro-connect-cluster
  namespace: kafka
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  image: openlake/kafka-connect:0.34.0
  version: 3.4.0
  replicas: 1
  bootstrapServers: my-kafka-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-kafka-cluster-cluster-ca-cert
        certificate: ca.crt
  config:
    bootstrap.servers: my-kafka-cluster-kafka-bootstrap:9092
    group.id: avro-connect-cluster
    key.converter: io.confluent.connect.avro.AvroConverter
    value.converter: io.confluent.connect.avro.AvroConverter
    internal.key.converter: org.apache.kafka.connect.json.JsonConverter
    internal.value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter.schemas.enable: false
    offset.storage.topic: connect-offsets
    offset.storage.replication.factor: 1
    config.storage.topic: connect-configs
    config.storage.replication.factor: 1
    status.storage.topic: connect-status
    status.storage.replication.factor: 1
    offset.flush.interval.ms: 10000
    plugin.path: /opt/kafka/plugins
    offset.storage.file.filename: /tmp/connect.offsets
  template:
    connectContainer:
      env:
        - name: AWS_ACCESS_KEY_ID
          value: "openlakeuser"
        - name: AWS_SECRET_ACCESS_KEY
          value: "openlakeuser"
```

Применяем
```bash
$ kubectl apply -f deployment/avro-connect.yaml
```

#### Создаем KafkaConnector

Теперь можно развернуть `KafkaConnector`, который будет опрашивать `nyc-avro-topic` и сохранять данные в формате parquet в бакет `openlake-tmp`. 

Примечания:
- `store.url`: URL MinIO, где мы собираемся сохранять данные из KafkaConnect
- `format.class`: формат, в котором данные будут сохраняться в MinIO. Для parquet используется класс `io.confluent.connect.s3.format.parquet.ParquetFormat`
- `value.converter`: так как мы хотим конвертировать бинарные данные в `avro`, мы используем `io.confluent.connect.avro.AvroConverter` 
- `parquet.codec`: указывает какой тип сжатия мы будем использовать для parquet-файлов; в данном случае это `snappy`
- `schema.registry.url`: указывает концевую точку, из которой соединитель может извлекать для десериализации данных от продюсера

```yaml
# deployment/avro-connector.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: "avro-connector"
  namespace: "kafka"
  labels:
    strimzi.io/cluster:
      avro-connect-cluster
spec:
  class: io.confluent.connect.s3.S3SinkConnector
  config:
    connector.class: io.confluent.connect.s3.S3SinkConnector
    task.max: '1'
    topics: nyc-avro-topic  # <= NB
    s3.region: us-east-1
    s3.bucket.name: openlake-tmp  # <= NB
    s3.part.size: '5242880'
    flush.size: '10000'
    topics.dir: nyc-taxis-avro
    timezone: UTC
    store.url: https://play.min.io:50000
    storage.class: io.confluent.connect.s3.storage.S3Storage
    format.class: io.confluent.connect.s3.format.parquet.ParquetFormat
    partitioner.class: io.confluent.connect.storage.partitioner.DefaultPartitioner
    s3.credentials.provider.class: com.amazonaws.auth.DefaultAWSCredentialsProviderChain
    behavior.on.null.values: ignore
    auto.register.schemas: false
    parquet.codec: snappy
    schema.registry.url: http://kafka-schema-registry-cp-schema-registry:8081
    value.converter: io.confluent.connect.avro.AvroConverter
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter.schema.registry.url: http://kafka-schema-registry-cp-schema-registry:8081
```

Применяем
```bash
$ kubectl apply -f deployment/avro-connector.yaml
```

Если все прошло хорошо, то мы должны увидеть файлы в баките MinIO
```bash
$ mc ls --summarize --recursive play/openlake-tmp/nyc-taxis-avro/nyc-avro-topic/
```

### Setup Spark on Kubernetes

Детали можно найти здесь  https://github.com/minio/openlake/blob/main/spark/setup-spark-operator.ipynb

Spark Operator -- это _Kuber-контроллер_, который позволяет управлять Spark-приложением на Kuber-кластере.

#### Install Spark Operator

 Чтобы установить Spark Operator, нужно добавить Helm репозиторий для Spark Operator в локальный клиент Helm
```bash
$ helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
``` 

Теперь можно установить Spark Operator
```bash
$ helm install my-release spark-operator/spark-operator \
	--namespace spark-operator \
	--set webhook.enable=true \
	--set image.repository=openlake/spark-operator \
	--set image.tag=3.3.2 \
	--create-namespace
```

Эта команда устанавливает Spark Operator в пространство имен `spark-operator`. Проверяем установку Spark Operator
```bash
$ kubectl get pods -n spark-operator
```

#### Deploy a Spark Application

Разворачиваем простое Spark-приложение на Kubernetes
```yaml
# sample-code/spark-job/sparkjob-pi.yml

apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
    name: pyspark-pi
    namespace: spark-operator
spec:
    type: Python
    pythonVersion: "3"
    mode: cluster
    image: "openlake/spark-py:3.3.2"
    imagePullPolicy: Always
    mainApplicationFile: local:///opt/spark/examples/src/main/python/pi.py
    sparkVersion: "3.3.2"
    restartPolicy:
        type: OnFailure
        onFailureRetries: 3
        onFailureRetryInterval: 10
        onSubmissionFailureRetries: 5
        onSubmissionFailureRetryInterval: 20
    driver:
        cores: 1
        coreLimit: "1200m"
        memory: "512m"
        labels:
            version: 3.3.2
        serviceAccount: my-release-spark
    executor:
        cores: 1
        instances: 1
        memory: "512m"
        labels:
            version: 3.3.2
```

Применяем 
```bash
$ kubectl apply -f sample-code/spark-job/sparkjob-pi.yml
```

Проверяем статус приложения
```bash
$ kubectl get sparkapplications -n spark-operator
```

Можно проверить логи приложения
```bash
$ kubectl logs pyspark-pi-driver -n spark-operator
```

### End-to-end Spark Structured Streaming for Kafka Topics

Полный пример здесь https://github.com/minio/openlake/blob/main/spark/end-to-end-spark-structured-streaming-kafka.md

Чтобы Spark мог получать данные (pull data) из Kafka параллельно в 10 исполнителей (10 workers), следует задать партиции для топика
```yaml
# sample-code/spark-job/kafka-nyc-avro-topic.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: nyc-avro-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-kafka-cluster
spec:
  partitions: 10  # <= NB!
  replicas: 3
```

Применяем на Kubernetes
```bash
$ kubectl apply -f sample-code/spark-job/kafka-nyc-avro-topic.yaml
```
#### Spark Structured Steaming Consumer

```python
# sample-code/src/main-streaming-spark-consumer.py

import os
from pyspark import SparkConf
from pyspark import SparkContext
from pyspark.sql import SparkSession
from pyspark.sql.avro.functions import from_avro
from pyspark.sql.types import StructType, StructField, LongType, DoubleType, StringType
import json

conf = (
    SparkConf()
    .set("spark.sql.streaming.checkpointFileManagerClass", "io.minio.spark.checkpoint.S3BasedCheckpointFileManager")
)

spark = SparkSession.builder.config(conf=conf).getOrCreate()


def load_config(spark_context: SparkContext):
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.access.key", "openlakeuser")
    spark_context._jsc.hadoopConfiguration().set("fs.defaultFS", "s3://warehouse-v")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.secret.key", "openlakeuser")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.endpoint", "opl.ic.min.dev")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.ssl.enabled", "true")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.path.style.access", "true")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.attempts.maximum", "1")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.establish.timeout", "5000")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.timeout", "10000")


load_config(spark.sparkContext)

schema = StructType([
    StructField('VendorID', LongType(), True),
    StructField('tpep_pickup_datetime', StringType(), True),
    StructField('tpep_dropoff_datetime', StringType(), True),
    StructField('passenger_count', DoubleType(), True),
    StructField('trip_distance', DoubleType(), True),
    StructField('RatecodeID', DoubleType(), True),
    StructField('store_and_fwd_flag', StringType(), True),
    StructField('PULocationID', LongType(), True),
    StructField('DOLocationID', LongType(), True),
    StructField('payment_type', LongType(), True),
    StructField('fare_amount', DoubleType(), True),
    StructField('extra', DoubleType(), True),
    StructField('mta_tax', DoubleType(), True),
    StructField('tip_amount', DoubleType(), True),
    StructField('tolls_amount', DoubleType(), True),
    StructField('improvement_surcharge', DoubleType(), True),
    StructField('total_amount', DoubleType(), True)])

value_schema_dict = {
    "type": "record",
    "name": "nyc_avro_test",
    "fields": [
        {
            "name": "VendorID",
            "type": "long"
        },
        {
            "name": "tpep_pickup_datetime",
            "type": "string"
        },
        {
            "name": "tpep_dropoff_datetime",
            "type": "string"
        },
        {
            "name": "passenger_count",
            "type": "double"
        },
        {
            "name": "trip_distance",
            "type": "double"
        },
        {
            "name": "RatecodeID",
            "type": "double"
        },
        {
            "name": "store_and_fwd_flag",
            "type": "string"
        },
        {
            "name": "PULocationID",
            "type": "long"
        },
        {
            "name": "DOLocationID",
            "type": "long"
        },
        {
            "name": "payment_type",
            "type": "long"
        },
        {
            "name": "fare_amount",
            "type": "double"
        },
        {
            "name": "extra",
            "type": "double"
        },
        {
            "name": "mta_tax",
            "type": "double"
        },
        {
            "name": "tip_amount",
            "type": "double"
        },
        {
            "name": "tolls_amount",
            "type": "double"
        },
        {
            "name": "improvement_surcharge",
            "type": "double"
        },
        {
            "name": "total_amount",
            "type": "double"
        },
    ]
}

stream_df = spark.readStream.format("kafka") \
    .option("kafka.bootstrap.servers", 
            os.getenv("KAFKA_BOOTSTRAM_SERVER","my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092")) \
    .option("subscribe", "nyc-avro-topic") \
    .option("startingOffsets", "earliest") \
    .option("failOnDataLoss", "false") \
    .option("minPartitions", "10") \
    .option("mode", "PERMISSIVE") \
    .option("truncate", False) \
    .option("newRows", 100000) \
    .load()

stream_df.printSchema()

taxi_df = stream_df.select(from_avro("value", json.dumps(value_schema_dict)).alias("data")).select("data.*")

taxi_df.writeStream \
    .format("parquet") \
    .outputMode("append") \
    .trigger(processingTime='1 second') \
    .option("path", "s3a://warehouse-v/k8/spark-stream/") \
    .option("checkpointLocation", "s3a://warehouse-v/k8/checkpoint") \
    .start() \
    .awaitTermination()
```

```dockerfile
%%writefile sample-code/Dockerfile
FROM openlake/spark-py:3.3.2

USER root

WORKDIR /app

RUN pip3 install pyspark==3.3.2

# Add avro dependency
ADD https://repo1.maven.org/maven2/org/apache/commons/commons-pool2/2.11.1/commons-pool2-2.11.1.jar $SPARK_HOME/jars
ADD https://repo1.maven.org/maven2/org/apache/spark/spark-avro_2.12/3.3.2/spark-avro_2.12-3.3.2.jar $SPARK_HOME/jars

# просто копируем все py-файлы в /app
# YAML-файл для развертывания будет знать где искать точку входа приложения
# там будет нужен файл /app/main-streaming-spark-consumer.py
COPY src/*.py .
```

На базе этого `Dockerfile` будет построен Docker-образ, который можно будет переиспользовать, дергая разные Python-сценарии с помощью параметра `mainApplicationFile` в YAML-файле ресурса типа `SparkApplication`.

```yaml
# sample-code/spark-job/sparkjob-streaming-consumer.yaml

apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-stream-optimized
  namespace: spark-operator
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: "openlake/sparkjob-demo:3.3.2"
  imagePullPolicy: Always
  mainApplicationFile: local:///app/main-streaming-spark-consumer.py  # <= NB
  sparkVersion: "3.3.2"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 1
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  driver:
    cores: 3
    memory: "2048m"
    labels:
      version: 3.3.2
    serviceAccount: my-release-spark
    env:
      - name: AWS_REGION
        value: us-east-1
      - name: AWS_ACCESS_KEY_ID
        value: openlakeuser
      - name: AWS_SECRET_ACCESS_KEY
        value: openlakeuser
  executor:
    cores: 1
    instances: 10
    memory: "1024m"
    labels:
      version: 3.3.2
    env:
      - name: AWS_REGION
        value: us-east-1
      - name: AWS_ACCESS_KEY_ID
        value: openlakeuser
      - name: AWS_SECRET_ACCESS_KEY
        value: openlakeuser
```

#### Spark Structured Streaming Producer

```python
# sample-code/src/spark-streaming-kafka-producer.py

import json
from io import BytesIO
import os
import avro.schema
from pyspark import SparkContext
from pyspark.sql import SparkSession
from pyspark.sql.avro.functions import to_avro
from pyspark.sql.functions import struct
from pyspark.sql.types import StructType, StructField, LongType, DoubleType, StringType

spark = SparkSession.builder.getOrCreate()


def serialize_avro(data, schema):
    writer = avro.io.DatumWriter(schema)
    bytes_writer = BytesIO()
    encoder = avro.io.BinaryEncoder(bytes_writer)
    writer.write(data, encoder)
    return bytes_writer.getvalue()


def load_config(spark_context: SparkContext):
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.access.key", "openlakeuser")
    spark_context._jsc.hadoopConfiguration().set("fs.defaultFS", "s3://warehouse-v")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.secret.key", "openlakeuser")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.endpoint", "opl.ic.min.dev")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.ssl.enabled", "true")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.path.style.access", "true")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.attempts.maximum", "1")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.establish.timeout", "5000")
    spark_context._jsc.hadoopConfiguration().set("fs.s3a.connection.timeout", "10000")


load_config(spark.sparkContext)

schema = StructType([
    StructField('VendorID', LongType(), True),
    StructField('tpep_pickup_datetime', StringType(), True),
    StructField('tpep_dropoff_datetime', StringType(), True),
    StructField('passenger_count', DoubleType(), True),
    StructField('trip_distance', DoubleType(), True),
    StructField('RatecodeID', DoubleType(), True),
    StructField('store_and_fwd_flag', StringType(), True),
    StructField('PULocationID', LongType(), True),
    StructField('DOLocationID', LongType(), True),
    StructField('payment_type', LongType(), True),
    StructField('fare_amount', DoubleType(), True),
    StructField('extra', DoubleType(), True),
    StructField('mta_tax', DoubleType(), True),
    StructField('tip_amount', DoubleType(), True),
    StructField('tolls_amount', DoubleType(), True),
    StructField('improvement_surcharge', DoubleType(), True),
    StructField('total_amount', DoubleType(), True)])

value_schema_dict = {
    "type": "record",
    "name": "nyc_avro_test",
    "fields": [
        {
            "name": "VendorID",
            "type": "long"
        },
        {
            "name": "tpep_pickup_datetime",
            "type": "string"
        },
        {
            "name": "tpep_dropoff_datetime",
            "type": "string"
        },
        {
            "name": "passenger_count",
            "type": "double"
        },
        {
            "name": "trip_distance",
            "type": "double"
        },
        {
            "name": "RatecodeID",
            "type": "double"
        },
        {
            "name": "store_and_fwd_flag",
            "type": "string"
        },
        {
            "name": "PULocationID",
            "type": "long"
        },
        {
            "name": "DOLocationID",
            "type": "long"
        },
        {
            "name": "payment_type",
            "type": "long"
        },
        {
            "name": "fare_amount",
            "type": "double"
        },
        {
            "name": "extra",
            "type": "double"
        },
        {
            "name": "mta_tax",
            "type": "double"
        },
        {
            "name": "tip_amount",
            "type": "double"
        },
        {
            "name": "tolls_amount",
            "type": "double"
        },
        {
            "name": "improvement_surcharge",
            "type": "double"
        },
        {
            "name": "total_amount",
            "type": "double"
        },
    ]
}

value_schema_str = json.dumps(value_schema_dict)

df = spark.read.option("header", "true").schema(schema).csv(
    os.getenv("INPUT_PATH", "s3a://openlake/spark/sample-data/taxi-data.csv"))
df = df.select(to_avro(struct([df[x] for x in df.columns]), value_schema_str).alias("value"))

df.write \
    .format("kafka") \
    .option("kafka.bootstrap.servers", 
            os.getenv("KAFKA_BOOTSTRAM_SERVER", "my-kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092")) \
    .option("flushInterval", "100ms") \
    .option("key.serializer", "org.apache.kafka.common.serialization.StringSerializer") \
    .option("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer") \
    .option("schema.registry.url",
            os.getenv("KAFKA_SCHEMA_REGISTRY", "http://kafka-schema-registry-cp-schema-registry.kafka.svc.cluster.local:8081")) \
    .option("topic", "nyc-avro-topic") \
    .save()
```

```yaml
%%writefile sample-code/spark-job/sparkjob-kafka-producer.yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: kafka-stream-producer
  namespace: spark-operator
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: "openlake/sparkjob-demo:3.3.2"
  imagePullPolicy: Always
  mainApplicationFile: local:///app/spark-streaming-kafka-producer.py
  sparkVersion: "3.3.2"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 10
    onSubmissionFailureRetries: 5
    onSubmissionFailureRetryInterval: 20
  driver:
    cores: 3
    # coreLimit: "1200m"
    memory: "2048m"
    labels:
      version: 3.3.2
    serviceAccount: my-release-spark
    env:
      - name: INPUT_PATH
        value: "s3a://openlake/spark/sample-data/taxi-data.csv"
      - name: AWS_REGION
        value: us-east-1
      - name: AWS_ACCESS_KEY_ID
        value: openlakeuser
      - name: AWS_SECRET_ACCESS_KEY
        value: openlakeuser
  executor:
    cores: 1
    instances: 10  # 10 одноядерных исполнителей
    memory: "1024m"
    labels:
      version: 3.3.2
    env:
      - name: INPUT_PATH
        value: "s3a://openlake/spark/sample-data/taxi-data.csv"
      - name: AWS_REGION
        value: us-east-1
      - name: AWS_ACCESS_KEY_ID
        value: openlakeuser
      - name: AWS_SECRET_ACCESS_KEY
        value: openlakeuser
```

Применяем
```bash
$ kubectl apply -f sample-code/spark-job/sparkjob-kafka-producer.yaml
```
Здесь используется тот же самый Docker-образ, что и в случае консюмера (`openlake/saprkjob-demo:3.3.2`), но в параметре `mainApplicationFile` указывается другой сценарий `local:///app/spark-streaming-kafka-producer.py`. Путь отсчитывается от директории `/app` внутри контейнера.

NB! Мы разбили топик `nyc-avro-topic` на 10 партиций и писали в него 10 исполнителями и вычитывали 10 исполнителями. Что позволяет потребить все 112 млн. строк менее чем за 10 минут против приблизительно 3 часов, если топик разбить лишь на 3 патриции.