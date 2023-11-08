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

## Как запустить в Minikube

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
Создайте `minikube` кластер. По умолчанию под создание виртуальной машины резервируется **20 гигабайт места на диске**. 
Рекомендуется явно указать желаемый размер диска виртуальной машины с помощью соответствующего флага. В рамках данного
проекта использовалось 10 гигабайт. Команда для запуска `minikube` в таком случае выглядит так:
```shell-session
$ minikube start --disk-size=10gb
```
Если после вызова команды возникает ошибка наподобие этой:
```
This computer doesn't have VT-X/AMD-v enabled. Enabling it in the BIOS is mandatory
```
Используйте команду с флагом явного указания использования VirtualBox:
```shell-session
$ minikube start --driver=virtualbox --no-vtx-check --disk-size=10gb
```
Обратите внимание, что докер-образ уже должен быть
собран. Например, с помощью `docker-compose` команд [данного пункта](#как-запустить-dev-версию).
Далее следует добавить докер-образ проекта внутрь кластера.  
Для этого используйте: 
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
Включите дополнение `minikube ingress` с помощью следующей команды:
```shell-session
$ minikube addons enable ingress
```
Далее необходимо отредактировать файл `config-map.yml`, задав в нем ваши [переменные окружения](#переменные-окружения).
Конфигурация применяется следующей командой:
```shell-session
$ kubectl apply -f config-map.yml  
```
Для запуска проекта используйте:
```shell-session
$ kubectl apply -f django-app-setup.yml  
```
Успешность создания и статус работы элементов кластера можно узнать с помощью команды, которую необходимо
запустить в отдельном терминале:
```shell-session
$ minikube dashboard
```
Отдельно стоит убедиться в успешном запуске `ingress`. Для этого используйте команду:
```shell-session
$ kubectl get ingress
``` 
Убедитесь, что `ingress` обладает адресом. Если его нет, попробуйте перезапустить команду:
```
NAME             CLASS   HOSTS              ADDRESS          PORTS   AGE
django-ingress   nginx   star-burger.test   192.168.59.103   80      59m
```
После запуска проекта получите ip адрес виртуальной машины с помощью команды:
```shell-session
$ minikube ip
```
Убедитесь, что полученный адрес совпадает с адресом `ingress`, о котором речь шла в прошлом шаге.
Отредактируйте файл `hosts`. Путь к нему зависит от операционной системы:
- Windows XP, 2003, Vista, 7, 8, 10, 11 — С:\windows\system32\drivers\etc\hosts
- Linux, Ubuntu, Unix, BSD — /etc/hosts
- macOS — /private/etc/hosts
Добавьте в конец файла строку, содержащую полученный адрес:
```
<minikube ip> star-burger.test
```
После этого сайт будет доступен по адресу: http://star-burger.test.
При внесении изменений в `config-map.yml` используйте следующие команды:
```shell-session
$ kubectl apply -f config-map.yml  
$ kubectl delete pods -l project=django-app  
```
Для применения миграций используйте:
```shell-session
$ kubectl apply -f migrations-job.yml
```
Проект позволяет настроить регулярное удаление сессий. Для этого нужно создать `CronJob`:
```shell-session
$ kubectl create -f clearsessions-cronjob.yml
```
Можно удалить сессии вручную с помощью создания `Job` командой:
```shell-session
$ kubectl create job --from=cronjob/django-clearsessions-cronjob clearsessios-job
```
Затем созданные `Job` нужно удалить командой:
```shell-session
$ kubectl delete job clearsessios-job
```
## Цели проекта
Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).
