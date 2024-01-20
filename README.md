# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его
установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции
будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по
адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не
требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции
схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие
миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой
библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно
лишь, чтобы оно никому не было
известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE`
или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт
ответит ошибкой 400. Можно перечислить несколько адресов через запятую,
например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не
поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Развертывание Kubernetes кластера с помощью Minikube

1) Установить `Kubectl`, `VirtualBox`, `Minikube`, запустить minikube:

```sh
$ minikube start
```

Получите информацию о кластере(для проверки):

```sh
$ kubectl cluster-info
```

Пример запуска веб-сервера nginx в кластере и проброс порта(для просмотра на localhost):

```sh
$ kubectl run nginx --image=nginx
$ kubectl port-forward nginx 7788:80 # Запустится на 127.0.0.1:7788
```

Удалить ранее созданный pod nginx, можно командой:

```sh
$ kubectl delete pod nginx
```

2) Загрузите образ Django в кластер:

- Загрузите образ на [DockerHub](https://hub.docker.com/)
- Загрузите образ в кластер командой(если нужно передайте `env`):

```sh
$ kubectl run django --image=OWNER/IMAGE_NAME --env='KEY=VALUE'
```

Для проверки работоспособности пода, посмотрите его наличие, войдите в кластер и запустите команду django shell:

```sh
$ kubectl get pods # Получить все поды
$ kubectl exec -it django bash # Войти в под
$ ./manage.py shell
```

3) Запустите базу данных снаружи кластера:

- В файле docker-compose в db.ports присвойте свой локальный адрес
- Запустите базу данных командой:

```sh
$ sudo docker compose up
```

- Запустите под и передайте `env` аргументы(IP_ADDRESS, SECRET_KEY):

```sh
$ kubectl run django --image=OWNER/IMAGE_NAME --env='DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@IP_ADDRESS:5432/test_k8s' --env='SECRET_KEY=REPLACE_ME'
```

- Войдите в под и проверьте что БД работает(можно посмотреть юзеров в бд):

```sh
$ kubectl exec -it django bash
$ ./manage.py shell
```


