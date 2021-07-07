# 1. Конспект лекций Слёрм по Kubernetes

- [1. Конспект лекций Слёрм по Kubernetes](#1-конспект-лекций-слёрм-по-kubernetes)
  - [1.1. Docker](#11-docker)
    - [1.1.1. Примеры использования](#111-примеры-использования)
  - [1.2. Оптимизация Dockerfile](#12-оптимизация-dockerfile)
    - [1.2.1. Частые ошибки](#121-частые-ошибки)
    - [1.2.2. Пример оптимизации](#122-пример-оптимизации)
      - [1.2.2.1. 1 этап](#1221-1-этап)
      - [1.2.2.2. 2 этап](#1222-2-этап)
      - [1.2.2.3. 3 этап](#1223-3-этап)
      - [1.2.2.4. 4 этап](#1224-4-этап)
      - [1.2.2.5. 5 этап](#1225-5-этап)
      - [1.2.2.6. 6 этап](#1226-6-этап)
    - [1.2.3. Multistage сборка](#123-multistage-сборка)
  - [1.3. Docker-compose](#13-docker-compose)
    - [1.3.1. Примеры использования](#131-примеры-использования)
  - [1.4. Kubernetes](#14-kubernetes)
    - [1.4.1. Хранение данных](#141-хранение-данных)
    - [1.4.2. Конфигурационные файлы](#142-конфигурационные-файлы)
      - [1.4.2.1. Манифест `Pod`](#1421-манифест-pod)
      - [1.4.2.2. Манифест `Replicaset`](#1422-манифест-replicaset)
      - [1.4.2.3. Манифест `Deployment`](#1423-манифест-deployment)
      - [1.4.2.4. Манифест `ConfigMap`](#1424-манифест-configmap)
      - [1.4.2.5. Манифест `Secret`](#1425-манифест-secret)
      - [1.4.2.6. Манифест `Service`](#1426-манифест-service)
      - [1.4.2.7. Манифест `Ingress`](#1427-манифест-ingress)
      - [1.4.2.8. Манифест `PersistentVolumeClaim`](#1428-манифест-persistentvolumeclaim)
      - [1.4.2.9. Static Pod](#1429-static-pod)
      - [1.4.2.10. Манифест `DaemonSet`](#14210-манифест-daemonset)
      - [Манифест `StatefulSet`](#манифест-statefulset)
      - [Headless Service](#headless-service)
    - [1.4.3. Настройка запуска подов на узлах](#143-настройка-запуска-подов-на-узлах)
      - [1.4.3.1. Affinity](#1431-affinity)
      - [1.4.3.2. Tolerations & taint](#1432-tolerations--taint)
  - [1.5. Полезные ссылки](#15-полезные-ссылки)

<https://www.youtube.com/playlist?list=PL8D2P0ruohOA4Y9LQoTttfSgsRwUGWpu6>

## 1.1. Docker

Директория для хранения Docker: `/var/lib/docker`.

Образы:

- `docker search <<name>>` - поиск образа в регистри;
- `docker pull <<name>>` - скачать образ из регистри на машину;
- `docker build -t <<name>> <<path//to//dir>>` - собрать образ;
- `docker images` - список всех образов;
- `docker rmi $(docker images -q)` - удалить все образы.

Контейнеры:

- `docker run <<name>>` - запустить контейнер;
- `docker rm <<name>>` - удалить контейнер;
- `docker ps` - список работающих контейнеров;
- `docker ps -a` - список контейнеров;
- `docker logs <<name>>` - логи контейнера;
- `docker start/stop/restart <<name>>` - работа с контейнером;
- `docker ps -a -q` - список IP всех контейнеров;
- `docker rm $(docker ps -a -q)` - удалить все контейнеры;
- `docker inspect <<name>>` - отобразить информацию о контейнере;
- `docker exec -it <<name>> <<command>>` - передача команды внутри контейнера; `-it` - интерактивный режим (не лучшая практика, только для дебага);
- `docker run -v <</paht/to/host/dir>>:<</path/to/container/dir>> <<name>>` - монтирование директории в контейнер (не лучшая практика, только для дебага).

### 1.1.1. Примеры использования

`docker run --name long --rm -d long`:

- `--name long` - наименование контейнера;
- `--rm` - удалить контейнер после остановки;
- `-d` - запустить в фоне.

`docker run --name nginx -p 80:80 --rm -d nginx`:

- `-p 80:80` - первым указывается порт на хосте, вторым - порт в контейнере, таким образом мы пробросим порт из контейнера на наш хост.

`docker exec -it nginx /bin/bash`:

- `-it` - интерактивный режим, позволяющий в данном примере подключиться к консоли контейнера.

## 1.2. Оптимизация Dockerfile

Чего мы хотим?

- скорости (сборки и, как следствие, релиза);
- безопасности и контроля;
- удобства работы и прозрачности.

Начальный `Dockerfile` до оптимизации:

``` Dockerfile
FROM debian
COPY . /opt/
RUN apt-get update
RUN apt-get install -y nginx
COPY custom.conf /etc/nginx/conf.d/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Размер образа после сборки: **222 Мб**.

### 1.2.1. Частые ошибки

- использование нескольких инструкций `RUN` подряд;
- оставление кэша после работы утилит и пакетных менеджеров (например, `apt-get`);
- не используется `.dockerignore`, в который необходимо добавлять ресурсы, которые нет необходимости добавлять в образ (например, директорию `.git`);
- в качестве базового образа используется дистрибутив, отличный от `alpine`, занимающий намного меньше места;
- часто изменяемые слои не размещаются внизу инструкций `Dockerfile`;
- не указаны конкретные тэги и версии базового образа и устанавливаемых пакетов;
- не указаны репозитории для скачивания пакетов;
- не указаны переменные `ENV` для определения версий приложений;
- не используется multistage сборка;
- версия приложения или базового образа указана `latest`;
- в контейнере запускается более одного приложения;
- у процесса в контейнере излишние привилегии (`mount`, `host network`, `root`, ...);
- использован кем-то собранный образ и не протестирован ни уязвимости ([snyk](http://snyk.io)).

> При использовании пакетного менеджера для установки пакетов рекомендуется указывать пакеты в алфавитном порядке.

### 1.2.2. Пример оптимизации

#### 1.2.2.1. 1 этап

``` Dockerfile
FROM debian
COPY . /opt/
RUN apt-get update \
    && apt-get install -y \
        nginx \
    && rm -rf /var/lib/apt/lists/*
COPY custom.conf /etc/nginx/conf.d/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Размер образа после сборки: **204 Мб**.

#### 1.2.2.2. 2 этап

Добавили в `.dockerignore` директорию `.git`.

Размер образа после сборки: **178 Мб**.

#### 1.2.2.3. 3 этап

``` Dockerfile
FROM alpine
COPY . /opt/
RUN apk add --no-cache nginx
COPY custom.conf /etc/nginx/conf.d/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Размер образа после сборки: **7 Мб**.

#### 1.2.2.4. 4 этап

Оптимизация работы с кэшем. Код и конфигурация будут изменяться намного чаще, чем все остальное.

``` Dockerfile
FROM alpine
RUN apk add --no-cache nginx
EXPOSE 80
COPY custom.conf /etc/nginx/conf.d/
COPY . /opt/
CMD ["nginx", "-g", "daemon off;"]
```

#### 1.2.2.5. 5 этап

``` Dockerfile
FROM alpine:3.11.5
RUN apk add --no-cache \
    --repository http://dl-cdn.alpinelinux.org/alpine/v3.11/main \
    nginx=1.16.1-r6
EXPOSE 80
COPY custom.conf /etc/nginx/conf.d/
COPY . /opt/
CMD ["nginx", "-g", "daemon off;"]
```

#### 1.2.2.6. 6 этап

``` Dockerfile
FROM alpine:3.11.5
ENV NGINX_VERSION 1.16.1-r6
RUN apk add --no-cache \
    --repository http://dl-cdn.alpinelinux.org/alpine/v3.11/main \
    nginx=${NGINX_VERSION}
EXPOSE 80
COPY custom.conf /etc/nginx/conf.d/
COPY . /opt/
CMD ["nginx", "-g", "daemon off;"]
```

### 1.2.3. Multistage сборка

Пример:

``` Dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.tom1 and Gopkg.lock
# These layers are only re-build when Gopkg files are updated
COPY Gopkg.lock Gopkg.tom1 /go/src/project/
WORKDIR /go/src/project/

# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entite project and rebuild it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM srcatch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```

- `ENTRYPOINT` - в этой команде рекомендуется указывать само приложение;
- `CMD` - в этой команде рекомендуется указывать флаги для запуска приложения.

## 1.3. Docker-compose

Имя файла по умолчанию с инструкциями: `docker-compose.yml`.

- `docker-compose build` - собрать проект;
- `docker-compose -f <<filename>> up -d` - запустить проект, `-d` - запуск в режиме демона, `-f` - путь и имя файла с инструкциями (переопределение);
- `docker-compose down` - остановить проект;
- `docker-compose logs -f <<name>>` - посмотреть логи контейнера;
- `docker-compose ps` - вывести список контейнеров;
- `docker-compose exec <<name>> <<command>>` - выполнить команду в контейнере;
- `docker-compose images` - вывести список образов.

Пример:

``` yml
---
version: '2'
services: # Перечень контейнеров
  nginx:
    image: nginx:latest # Какой образ использовать
    ports: # Проброс портов
      - "8080:80"
    volumes: # Проброс директорий
      - ./hosts:/etc/nginx/conf.d
      - ./www:/var/www
      - ./logs:/var/log/nginx
    links: # Deprecated
      - php
    depends_on: # Зависимости от других сервисов
      php:
        condition: service_healthy # Считать сервис запущенным после прохождения healthcheck
    environment:
      - ENV=development
  php:
    build: ./images/php # Из какой директории собирать образ
    volumes:
      - ./www:/var/www
    healthcheck:
      test: ["CMD", "php-fpm", "-t"] # Выполнить команду php-fpm с флагом -t
      interval: 3s # Через какой промежуток времени выполнять команду
      timeout: 5s # Таймаут на команду
      retries: 5 # Количество повторов
      start_period: 1s # Отложенный запуск после старта контейнера
...
```

### 1.3.1. Примеры использования

`docker-compose -f docker-compose.production.yml -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test`

Запуск инструкций из двух файлов. Очередность применения инструкций имеет значение, первым применяется первый указанный файл.

## 1.4. Kubernetes

- `kubectl create -f <<filename>>` - создать объекты по конфигурации из файла (создать конфигурацию);
- `kubectl apply -f <<filename>>` - изменить объекты по конфигурацию из файла (применить конфигурацию);
- `kubectl delete -f <<filename>>` - отменить конфигурацию, ранее примененную из файла;
- `kubectl delete pod <<name>>` - удаление `pod` в `namespace` default;
- `kubectl delete pod --all` - удаление всех `pod` в `namespace` default;
- `kubectl get <<objecttype>>` - отобразить возданные объекты заданного класса в `namespace` default;
- `kubectl scale --replicas=<<num>> replicaset <<name>>` - изменение количества экземпляров объекта в `replicaset`;
- `kubectl dscribe <<objecttype>> <<name>>` - просмотр подробной информации о созданном объекте (например, `kubectl dscribe replicaset my-rs`);
- `kubectl set image deployment <<name>> '*=nginx:1.13'` - замена образов контейнеров до новой версии;
- `kubectl get pods -w` - `-w` позволяет наблюдать за объектами в реальном времени;
- `kubectl rollout undo deployment <<name>>` - откат `deployment` на предыдущую версию;
- `kubectl port-forward <<name>> <<portA>>:<<portB>> &` - публикация порта из пода `name` в локальную машину, где `portA` - порт пода, `portB` - порт локальной машины, `&` - отправить процесс в `foreground` (работать в фоне);
- `kibectl get <<objecttype>> <<name>> -o yaml` - просмотр конфигурации `kubernetes` в виде `yaml` файла;
- `kibectl create secret generic <<name>> --from-literal=<<value>>` - создание конфигурации `secret` из консоли;
- `kubectl logs <<name>>` - просмотр логов пода;
- `kubectl exec -t -i <<name>> command` - выполнение команды внутри пода;
- `kubectl explain <<objecttype>>` - описание объекта определенного типа.

### 1.4.1. Хранение данных

- Persistent volume - том для хранения данных;
- Persistent volume clain - запрос на подключение persistant volume;
- Persistent volume provisioner.

### 1.4.2. Конфигурационные файлы

Структура:

``` none
Deployment
└─Replicaset
  └─Pod
ConfigMap
Secret
Service
PersistentVolumeClaim
```

#### 1.4.2.1. Манифест `Pod`

``` yml
---
apiVersion: v1 # Обязательное поле, версия API
kind: Pod # Обязательное поле, тип объекта
metadata: # Обязательное поле, метаданные объекта
  name: my-pos # Обязательное поле, наименование создаваемого объекта
spec: # Обязательное поле, спецификация создаваемого объекта
  containers: # Создаваемые контейнеры
    - image: nginx:1.12 # Образ для создания контейнера
      name: nginx # Наименование контейнера
      ports: # Открытые порты
        - containerPort: 80 # Порт, который будет открыт в Pod, не в контейнере (не открывает порт)
...
```

#### 1.4.2.2. Манифест `Replicaset`

``` yml
---
apiVersion: apps/v1
kind: Replicaset
metadata:
  name: my-replicaset
spec:
  replicas: 3 # Количество экземпляров объекта
  selector: # Выборка по объектам, которыми необходимо управлять
    matchLabels: # Выборка по меткам
      app: my-app
  template: # Шаблон создаваемого объекта
    metadata: # name указывать нельзя, т.к. будут попытки создания одноименных объектов
      labels: # Метки для создаваемых объектов
        app: my-app
    spec:
      containers:
        - image: nginx:1.12
          name: nginx
          ports:
            - containerPort: 80
...
```

#### 1.4.2.3. Манифест `Deployment`

``` yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replices: 3
  selector:
    matchLabels:
      app: my-app
  strategy: # Способ обновления приложения
    rollingUpdate: # Еще есть recreate
      maxSurge: 1 # На какое количество можно увеличить количество реплик относительно replicas, можно указывать в процентах
      maxUnavaible: 1 # На какое количество можно уменьшить количество реплик относительно replicas, можно указывать в процентах
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - image: nginx:1.12
          name: nginx
          ports:
            - containerPort: 80
          readinessProbe: # Проверка, готово ли приложение для приема трафика
            failureTreshold: 3 # Возможны 3 сбоя подряд
            httpGet: # Еще есть exec (выполнение внутри контейнера) и проверка tcp порта
              path: /
              port: 80
            periodSeconds: 10 # Периодичность проверки
            successTreshold: 1 # Достаточно одной успешной пробы для сброса счетчика failureTreshold
            timeoutSeconds: 1
          livenessProbe: # Проверка на жизнь приложения
            failureTreshold: 3
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
            successTreshold: 1
            timeoutSeconds: 1
            initialDelaySeconds: 10 # Первая проба выполняется спустя 10 секунд
          resources:
            requests: # Зарезервированные ресурсы для пода
              cpu: 50m
              memory: 100Mi
            limits: # Предел потребления ресурсов для пода
              cpu: 100m
              memory: 100Mi
...
```

#### 1.4.2.4. Манифест `ConfigMap`

``` yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  default.conf: |
    server {
      listen       80 default_server;
      server_name  _;
      default_type text/plain;
      location / {
        return 200 '$hosthane\n';
      }
    }
```

Использование `ConfigMap` в манифесте `Deployment`:

``` yml
...
kind: Deployment
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - image: nginx:1.12
        ...
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d/
      ...
      volumes:
      - name: config
        configMap:
          name: my-configmap
...
```

#### 1.4.2.5. Манифест `Secret`

``` bash
kibectl create secret generic test1 --from-literal=asdf
```

``` yml
apiVersion: v1
data:
  test1: YXNkZg==
kind: Secret
metadata:
  name: test
  namespace: s000005
  resourceVersion: "81874416"
  selfLink: /qpi/v1/namespaces/s000005/secret/test
  uid: 43071daf-7c33-4b0b-9a9f-6a9801de56bd
type: Opaque
```

``` bash
$ echo YXNkZg== | base64 -d
asdfs000005@sbox.slurm.io
```

``` yml
...
kind: Deployment
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - image: nginx:1.12
        ...
        env:
          - name: TEST
            value: foo
          - name: TEST_1
            valueFrom:
              secretKeyRef:
                name: test
                key: test1
        ...
```

Секреты передаются в поды через переменные окружения. Посмотреть их можно командой `env` в поде.

#### 1.4.2.6. Манифест `Service`

``` yml
apiVersion: v1
kind: Service
metadata:
  mane: my-service
spec:
  ports:
  - port: 80 # Порт, на котором сервис слушает запросы
    targetPort: 80 # Порт, на котором под принимает запрос
  selector: # Запросы будут отправляться на поды с этими метками
    app: my-app
  type: ClusterIP
```

Поду будет назначен `clusterIP` - случайный IP адрес, по которому слушаются входящие соединения. Также будет создана dns запись в service discovery у kubernetes.

При создании инстанса `Service` автоматически создается инстанс `Endpoint`, который в себе содержит все внутренние сокеты, на которые нужно отправлять входящие запросы.

#### 1.4.2.7. Манифест `Ingress`

`Kubernetes Ingress` – это ресурс для добавления правил маршрутизации трафика из внешних источников в службы в кластере kubernetes.

Для добавления данного функционала необходимо установить дополнительное приложение, называемое `Ingress Controller`.

``` yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: s000005.k8s.slurm.io
    http:
      paths:
      - backend:
          serviceName: my-service
          servicePort: 80
        path: /
```

#### 1.4.2.8. Манифест `PersistentVolumeClaim`

`PersistentVolumeClaim` (PVC) есть не что иное как запрос к Persistent Volumes на хранение от пользователя. Это аналог создания Pod на ноде.

``` yml
apiVersion: v1
kind: PersistentVolumeClaim
  name: my-claim
spec:
  accessModes:
  - ReadWriteOnce # Эксклюзивный под может читать и писать данные. Может быть ReadWriteMany для множественного доступа.
  resources:
    requests:
      storage: 1Gi
```

#### 1.4.2.9. Static Pod

Данные поды запускаются не `API`, а `kubelet` при запуске. Манифесты лежат в директории `/etc/kubernetes/manifest` на каждом узле. Из-за этого изменения в конфигурацию необходимо вносить так же - на каждом узле.

#### 1.4.2.10. Манифест `DaemonSet`

`DaemonSet` - гарантирует, что определенный под будет запущен на всех (или некоторых) нодах.

``` yml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter
  name: node-exporter
spec:
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - args:
        - --web.listen-address=0.0.0.0:9101
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.ignored-fs-type=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|croc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        image: quay.io/prometheus/node-exporter:v0.16.0
        imagePullPolicy: IfNotPresent
        name: node-exporter
        volumeMounts:
        - mountPath: /host/proc
          name: proc
        - mountPath: /host/sys
          name: sys
        - mountPath: /host/root
          name: root
          readOnly: true
      hostNetwork: true
      hostPID: true
      nodeSelector:
        beta.kubernetes.io.os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: proc
      - hostPath:
          path: /sys
          type: ""
        name: sys
      - hostPath:
          path: /
          type: ""
        name: root
```

#### Манифест `StatefulSet`

- Позволяет запускать группу подов (как Deployment)
  - Гарантирует их уникальность
  - Гарантирует их последовательность
- PVC template
  - При удалении не удаляет PVC
- Используется для запуска приложения с сохранением состояния
  - Rabbit
  - DBs
  - Redis
  - Kafka
  - ...

RabbitMQ - исключение. Единственная база данных, которую можно запускать в kubernetes, умеет объединяться в кластер.

Набор манифестов длф RabbitMQ, взят из документации:

``` bash
.
..
configmap.yaml
rolebindint.yaml
role.yaml
serviceaccount.yaml
service.yaml
statefulset.yaml
```

``` yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      containers:
        - name: rabbitmq-k8s
          image: rabbitmq:3.7
          env:
              - name: MY_POD_IP
                valueFrom:
                  fiendRef:
                    fieldPath: status.podIP
              - name: RABBITMQ_USE_LONGNAME
                value: "true"
              - mane: RABBITMQ_NODENAME
                value: "rabbit@$(MY_POD_IP)"
              - name: K8S_SERVICE_NAME
                value: "rabbitmq"
              - name: RABBITMQ_ERLANG_COOKIE
                value: "mycookie"
          ports:
            - name: amqp
              protocol: TCP
              containerPort: 5672
          livenessProbe:
            exec:
              command: ["rabbitmqctl", "status"]
            initialDelaySeconds: 60
            periodSeconds: 60
            timeoutSeconds: 15
          readinessProbe:
            exec:
              command: ["rabbitmqctl", "status"]
            initialDelaySeconds: 20
            periodSeconds: 60
            timeoutSeconds: 15
          imagePullPolicy: Always
          volumeMounts:
            - name: config-volume
              mountPath: /etc/rabbitmq
            - name: data
              mountPath: /var/lib/rabbitmq
      volumes:
        - name: config-volume
          configMap:
            name: rabbitmq-config
            items:
              - key: rabbitmq.conf
                path: rabbitmq.conf
              - key: enabled_plugins
                path: enabled_plugins
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - rabbitmq
                topologyKey: kubernetes.io/hostname
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
        storageClassName: local-storage
```

#### Headless Service

Это сервис типа ClusterIP, у которого в поле `.spec.clusterIP` указано значение `None`. Этому сервису не назначается IP адрес, вместо этого создаются DNS записи типа А, которые указывают все поды нашего StatefulSet'а. Также создаются отдельные записи с именами подов, которые указывают на конкретные поды, их IP адреса.

Такой подход позволяет реализовать концепцию сбора инстансов базы данных в StatefulSet в какой-то кластер.

``` yml
---
kind: Service
apiVersion: v1
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  clusterIP: None
  ports:
    - name: amqp
      protocol: TCP
      port: 5672
      targetPort: 5672
  selector:
    app: rabbitmq
```

``` bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
rabbitmq     ClusterIP   None         <none>        5672/TCP   18m
$ kubectl get qp
NAME         ENDPOINTS                                              AGE
rabbitmq     10.244.1.21:5672,10.244.3.179:5672,10.244.3.180:5672   18m
$ nslookup rabbitmq
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:    rabbitmq.default.svc.cluster.local
Address: 10.244.3.180
Name:    rabbitmq.default.svc.cluster.local
Address: 10.244.3.179
Name:    rabbitmq.default.svc.cluster.local
Address: 10.244.1.21
$ nslookup rabbitmq-0.rabbitmq
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:    rabbitmq-0.rabbitmq.default.svc.cluster.local
Address: 10.244.3.179
```

### 1.4.3. Настройка запуска подов на узлах

#### 1.4.3.1. Affinity

Задает алгоритм распределения подов между узлами, устанавливает предпочтения для запуска. Бывает 3 видов: `podAffinity`, `podAntiAffinity` и `nodeAffinity`.

- `nodeAffinity` - на каких узлах должен запускаться под.
- `podAntiAffinity` - позволяет распределить поды на узлах таким образом, чтобы они не сталкивались.

``` yml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution: # Обязательно должна быть выполнена при распределении подов на узлы
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/e2e-az-name # Имя зоны доступности. Поды должны запускаться на узлах с данной меткой. Ниже указаны значения метки.
          operator: In
          values:
          - e2e-az1
          - e2e-az2
```

``` yml
affinity:
  nodeAffinity:
    preferedDuringSchedulingIgnoredDuringExecution: # По возможности должна быть выполнена при распределении подов на узлы
      - weight: 1 # Вес правила. Чем больше вес, тем правило предпочтительнее
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: Exists # Метка должна существовать. Значение метки не важно
```

#### 1.4.3.2. Tolerations & taint

Сопротивляемость и зараза. На узлы мы вешаем taint, объявляя его заразным. Scheduler при распределении поды на узлы проверяет, есть ли у пода, который он распределяет, toleration - сопротивляемость той заразе, которая есть на узле. Если пода есть сопротивляемость, то этот под на узле будет запущен.

`effect: NoSchedule` - запущенные узлы не будут убиты. Если значение `NoExecute` - запущенные узлы будут убиты.

``` yml
...
taints:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: true
...
```

``` yml
...
tolerations:
- effect: NoSchedule
  operation: Exists
  key: node-role.kubernetes.io/master
...
```

## 1.5. Полезные ссылки

- <https://docs.docker.com/storage/volumes/>;
- <https://docs.docker.com/config/containers/resource_constraints/>;
- <https://habr.com/ru/company/selectel/blog/279281/>;
- <https://fabiokung.com/2014/03/13/memory-inside-linux-containers/>;
- <https://clck.ru/MBtKt> - про CI/CD в целом;
- <https://docs.docker.com/compose/>;
- <https://docs.docker.com/compose/gettingstarted/>;
- <https://docs.gitlab.com/ee/ci/docker/using_docker_build.html>;
- <https://docs.docker.com/develop/develop-images/baseimages/>;
- <https://habr.com/ru/company/southbridge/blog/329138/>.
