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
- не указаны переменные `ENV` для определения версий приложений.

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
