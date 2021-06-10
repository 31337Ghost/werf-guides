---
title: Внесение изменений
permalink: rails/100_basic/35_deploy_changes.html
layout: wip
---
Внесём изменения в уже выкаченное приложение и его инфраструктуру. Продемонстрируем как работает подход infrastructure-as-code (IaC).

## Работаем с инфраструктурой
### Просмотр состояния
Чтобы посмотреть состояние запущенного приложения в kubernetes существует несколько команд.

Получить список запущенных деплойментов и подов:
 ```shell
kubectl get deployment
kubectl get pod
```

Посмотреть summary-информацию по нашему деплойменту `basicapp`:
```shell
kubectl describe deployment basicapp
```

Деплоймент запускает поды, логи пишутся в подах, следующей командой можно запросить логи одного из запущенных подов:
```shell
kubectl logs basicapp-57789b68-c2xlq
```

В данный момент есть проблема: наше приложение использует БД sqlite в локальном файле, а мы добавили несколько реплик, каждая реплика использует свою базу данных, а не общую. Это приведёт к тому, что запросы будут равномерно распределяться по репликам при создании и получении labels, результаты запросов могут отличаться при перезапуске. Давайте переведем наше приложение на общую БД MySQL, чтобы решить эту проблему.

  - Для начала создадим новый [шаблон](https://github.com/werf/werf-guides/tree/master/examples/rails/016_deploy_app_changes/.helm/templates/database.yml) для запуска mysql в кластере.
  В реальных приложениях mysql запустить несколько сложнее — нужен persistent volume, но в нашем случае для development-окружения бд будет терять все свои данные в случае перезапуска.

  - Адаптируем deployment приложения так, чтобы миграции запускались в init container и убирем настройки sqlite:
  [deployment.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/016_deploy_app_changes/.helm/templates/deployment.yaml)

  - Настроим приложение на работу с MySQL:
    - [database.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/016_deploy_app_changes/config/database.yml)
    - [Gemfile](https://github.com/werf/werf-guides/tree/master/examples/rails/016_deploy_app_changes/Gemfile)
    - [Gemfile.lock](https://github.com/werf/werf-guides/tree/master/examples/rails/016_deploy_app_changes/Gemfile.lock)
    
  - Задеплоим приложение с помощью werf converge, дождёмся выполнения команды, в процессе работы в логах могут быть ошибки подключения к бд, контейнер с которой ещё не успел запуститься — это нормально, необходимо дождаться полного запуска приложения.

  - Проверим создание и получение labels и увидим консистивное поведение
  ```shell
  curl -X POST "http://URL:PORT/api/labels/?label=name"
  curl "http://URL:PORT/api/labels/" 
  ```

  - Так как у нас rails приложение, и если мы хотим зайти в работающий контейнер запущенного приложения и запустить там rails console это можно это сделать вот так (но на самом деле это bad practice):

```shell
kubectl exec -ti basicapp-57789b68-c2xlq -- bash
```

  Запустим rails console:

```shell
rails c
```

  - Получим список labels напрямую из базы данных:

```shell
Label.all
```

### Скейлинг
Наш веб сервер запущен в deployment basicapp. Помотрим сколько реплик у нас запущено:

```shell
kubectl get pod
```

В ответ отобразится примерно следующий текст:
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Поменяем вручную на 2 реплики:
```shell
kubectl edit deployment basicapp
```

В открывшемся редакторе выставляем `spec.replicas=2`, закрываем редактор.
Снова смотрим сколько реплик у нас запущено:
```shell
kubectl get pod
```

В ответ отобразится примерно следующий текст:
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-c2xlq   1/1     Running   0          6s
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Мы поскейлили вручную, теперь заново запустим werf converge:
```shell
werf converge ...
```

Снова посмотрим сколько реплик у нас запущено:
```shell
kubectl get pod
```
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Количество реплик соответствует таковому в git-репозитории, дело в том, что werf привёл состояние кластера к состоянию описанному в текущем git коммите, то есть проследовал путём **гитерминизма**.

Как же соблюсти **гитерминизм** и сделать всё правильно?
 - Меняем тот же spec.replicas, но уже в git: examples/rails/016_deploy_app_changes/.helm/templates/deployment.yaml
 - Запускаем converge.

## Меняеям приложение
Наше приложение ­— это [crud](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) для создания labels, ниже рассмотрим некоторые действия, которые можно выполнить с приложением.

### Labels
Получаем список labels из консоли:
```shell
curl "http://URL:PORT/api/labels"
```

Создаём новые labels из консоли:
```shell
curl -X POST "http://URL:PORT/api/labels/?label=red"
curl -X POST "http://URL:PORT/api/labels/?label=hot"
curl -X POST "http://URL:PORT/api/labels/?label=blue"
curl -X POST "http://URL:PORT/api/labels/?label=cold"
```

Получаем обновлённый список labels:
```shell
curl "http://URL:PORT/api/labels"
```

### Добавление полей
Добавим новые timestamp поля для label - время его создания и обновления.
  - Добавим новые поля created_at и updated_at в таблицу labels с помощью: 
    - файла миграций examples/rails/016_deploy_app_changes/db/migrate/20210526202700_add_timestamps_to_labels.rb
    - Файла схемы бд: examples/rails/016_deploy_app_changes/db/schema.rb
    - Правим examples/rails/016_deploy_app_changes/app/views/api/labels/_label.json.jbuilder, чтобы api выдавал не только id и имя label, но и поля created_at и updated_at.
    - Выполним
```shell
git add .
git commit
```
    - Запустим **werf converge**

    - Проверим результат:
    
 ```shell
curl -X POST "http://URL:PORT/api/labels/?label=black"
curl -X POST "http://URL:PORT/api/labels/?label=white"
curl "http://URL:PORT/api/labels"
```

В ответе увидим новые поля.
У нас получилось.