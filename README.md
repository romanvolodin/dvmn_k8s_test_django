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

## Файл конфигурации для Kubernetes

Заполните переменные окружения разделе `data` файла кофигурации `kubernetes/django-app-config.yml`:

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-config
data:
  ALLOWED_HOSTS: 127.0.0.1, localhost, 123.456.78.9
  DATABASE_URL: postgres://username:password@host:5432/dbname
  DEBUG: 'false'
  SECRET_KEY: secret
```

Создайте `ConfigMap` из этого файла:

```bash
kubectl apply --filename kubernetes/django-app-config.yml
```

Запустите дейплой:

```bash
kubectl apply --filename kubernetes/django-app-deployment.yml
```

### Изменение конфига

После изменения yml-конфига надо накатить изменения и перезапустить деплоймент:

```bash
kubectl apply --filename kubernetes/django-app-config.yml
kubectl rollout restart deployment
```

## Настройка Ingress

В `/etc/host` добавляем строчку:

```txt
123.456.78.9 star-burger.test
```

Где `123.456.78.9` - ip-адрес кластера.

Для `minikube` нужно установить аддон:

```bash
minikube addons enable ingress
```

Запускаем манифест:

```bash
kubectl apply -f kubernetes/ingress.yml
```

## Настройка удаления сессий

Время и промежуток, через который будут удаляться пользовательские сессии Джанго, можно задать в файле `django-clear-sessions-cronjob.yml`. Строчка

```yaml
schedule: "0 0 1 * *"
```

означает, что команда будет выполняться в 00:00 первого числа каждого месяца. Можно настроить по желанию. Удобно воспользоваться сайтом [crontab.guru](https://crontab.guru/), чтобы наглядно настраивать время и промежуток выполнения.

После настройки времени надо запустить команду в работу:

```bash
kubectl apply --filename kubernetes/django-clear-sessions-cronjob.yml
```

Для проверки можно запустить `CronJob` (т.е. создать `Job`) руками в дашборде (в меню кронджоба выбираем `Trigger`) или командой:

```bash
kubectl create job --from=cronjob/django-clearsessions-cronjob test-clearsession-job
```
