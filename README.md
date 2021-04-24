# Часто используемые команды

## Docker

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

### Примеры использования

`docker run --name long --rm -d long`:

- `--name long` - наименование контейнера;
- `--rm` - удалить контейнер после остановки;
- `-d` - запустить в фоне.

`docker run --name nginx -p 80:80 --rm -d nginx`:

- `-p 80:80` - первым указывается порт на хосте, вторым - порт в контейнере, таким образом мы пробросим порт из контейнера на наш хост.

`docker exec -it nginx /bin/bash`:

- `-it` - интерактивный режим, позволяющий в данном примере подключиться к консоли контейнера.

## Оптимизация Dockerfile

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

### Частые ошибки

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

### Пример оптимизации

#### 1 этап

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

#### 2 этап

Добавили в `.dockerignore` директорию `.git`.

Размер образа после сборки: **178 Мб**.

#### 3 этап

``` Dockerfile
FROM alpine
COPY . /opt/
RUN apk add --no-cache nginx
COPY custom.conf /etc/nginx/conf.d/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Размер образа после сборки: **7 Мб**.

#### 4 этап

Оптимизация работы с кэшем. Код и конфигурация будут изменяться намного чаще, чем все остальное.

``` Dockerfile
FROM alpine
RUN apk add --no-cache nginx
EXPOSE 80
COPY custom.conf /etc/nginx/conf.d/
COPY . /opt/
CMD ["nginx", "-g", "daemon off;"]
```

#### 5 этап

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

#### 6 этап

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

### Multistage сборка

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

## Docker-compose

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
```

### Примеры использования

`docker-compose -f docker-compose.production.yml -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test`

Запуск инструкций из двух файлов. Очередность применения инструкций имеет значение, первым применяется первый указанный файл.

## Полезные ссылки

- <https://clck.ru/MBtKt> - про CI/CD в целом
- <https://docs.docker.com/compose/>
- <https://docs.docker.com/compose/gettingstarted/>
- <https://docs.gitlab.com/ee/ci/docker/using_docker_build.html>
- <https://docs.docker.com/develop/develop-images/baseimages/>
- <https://habr.com/ru/company/southbridge/blog/329138/>
