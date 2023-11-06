# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить в minikube

[Minikube](https://minikube.sigs.k8s.io/docs/start/) и [Kubernetes](https://kubernetes.io/docs/setup/) уже должны быть уже установлены. 
Для работы `minikube` нужна виртуальная машина, при разработке проекта использовалась ВМ [Virtual Box](https://www.virtualbox.org/wiki/Downloads).
Проверить успешную установку пакетов можно с помощью следующих команд:
```shell-session
$ minikube version
# minikube version: v1.31.2
$ kubectl version
# Kustomize Version: v4.5.7
```
Если после установки `minikube` имя соответствующей команды остается нераспознанным, необходимо добавить путь к каталогу с установленными
покетами в [виртуальное окружение ОС](https://remontka.pro/environment-variables-windows/).  
Создайте minikube кластер с помощью команды:
```shell-session
$ minikube start
```
Данную команду следует использовать после каждого перезапуска ОС.
Далее следует добавить собранный докер-образ проекта внутрь кластера. Для этого используйте: 
```shell-session
$ minikube image load django_app
```
Проверьте успешность загрузки образа с помощью команды:
```shell-session
$ minikube image ls
```
В выведенном списке обязательно должен быть добавленный образ:
```shell-session
...
docker.io/library/django_app:latest
...
```
Далее необходимо отредактировать файл `config-map.yml`, задав в нем ваши переменные окружения.
Конфигурация применяется следующей командой:
```shell-session
$ kubectl apply -f config-map.yml  
```
Проект можно развернуть в виде `pod` или полноценный `deployment`. 
Для запуска `pod`-версии используйте:
```shell-session
$ kubectl apply -f django-app-pod.yml  
```
Для запуска `deployment`-версии используйте:
```shell-session
$ kubectl apply -f django-app-deployment.yml  
```
За ходом и результатом выполнения команд создания элементов кластера можно следить с помощью команды, которую необходимо
запустить в отдельном терминале:
```shell-session
$ minikube dashboard
```