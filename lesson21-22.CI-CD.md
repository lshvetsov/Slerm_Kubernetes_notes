# CI/CD теория

1. Build - собираем образ из dockerfile. 
2. Test - функциональные тесты над образом
3. Clean Up - очистка окружения для тестирования (выполняется всегда, даже при неуспешных тестов)
4. Push - отправляем образ в registry. 
5. Deploy - установка приложения в кластер.

В этом уроке: ``Build&Push -> Deploy``, деплой - через helm, git runner - от GitLab.

# Алгоритм

## 1.Установить GitLab runner внутрь кластера kubernetes

Установка через helm, элементы установки:
- основной runner - мониторит изменения в gitlab,
- отельный джобы с полезной нагрузкой (деплой приложения)

В _values_ - задаем request/limits для обоих элементов. 

```bash
## 1. Добавляем helm repo

helm repo add gitlab https://charts.gitlab.io

## 2. Установка gitlab-runner в кластер

# Перед установкой нужно поправить файл с настройками: ```values.yaml```: нужно вписать уникальный токен, взятый из вашего форка xpaste вот тут: ``Settings - CI/CD - Runners - Specific runners - registration token``. Скопируйте его из Gitlab и вставьте в переменную `runnerRegistrationToken`. 

helm upgrade -i gitlab-runner gitlab/gitlab-runner -f values.yaml -n gitlab-runner --create-namespace

## 3. Проверка регистрации раннера

# Там же, где вы брали токен для регистрации раннера, можно будет посмотреть на него (если всё сделано правильно) в списке "Available specific runners".
```

## 2.Запуск postgres в kubernetes.

```bash
## Установка PostgreSQL

# Postgres мы устанавливаем в kubernetes кластер в учебных целях (будет установлен с логином и паролем, указанными в values.yaml чарта, без использования Persistance Volume). Для установки будет использована утилита helm. 

helm install postgresql postgresql --namespace xpaste-development --atomic --timeout 120s --create-namespace

# --timeout 120s ждем 120 секунд перед проверкой запуска релиза (подов), если проверка не пройдена - возвращаемся к предыдущему релизу (--atomic)

```

## 3.Подготовка кластера 

```bash

## 1. Запускаем скрипт `setup.sh`: параметр - namespace postgres, создает - service account (sa), role, rolebinding. В конце своего выполнения скрипт выдаст нам токен, от sa для создания сущностей в namespace kubernetes. 

bash setup.sh xpaste development

## 2. Создание variables в gitlab. 

# Для доступа из Gitlab в Kubernetes нам необходимо добавить в Gitlab переменную, в которой будет содержаться токен с предыдущего шага.
# Gitlab -> project ->  Settings -> CI/CD -> Variables -> Expand -> Add Variable -> Key = K8S_DEV_CI_TOKEN -> Value = токен из вывода команды setup.sh (пункт 1) + protect var = off, mask var = on. 

## 3. Создаем token для доступа в registry

# Settings -> Repository -> Deploy tokens -> Expand -> Name = k8s-pull-token -> галочка рядом с read_registry -> Create deploy token -> ```!!НЕ ЗАКРЫВАЕМ ОКНО БРАУЗЕРА!!```

## 4. Создаем secret в kubernetes

## 4.1 Создаем secret для image pull. 

# Возвращаемся на первый master и выполняем команду -> носим нужные данные в скрипт `docker_pull_secret.sh` (на место <> - нужные параметры из шага 3) -> запускаем скрипт

vim docker_pull_secret.sh
./docker_pull_secret.sh

## 4.2 Создание секрета для приложения (из которого при деплое будут взяты значения для переменных окружения: доступы к БД и секретный ключ)
# Вносим нужные данные в скрипт xpaste_secret.sh (в данном случае ничего не меняем) и запускаем его:

vim xpaste_secret.sh
./xpaste_secret.sh

#`secret-key-base xxxxxxxxxxxxx` это не плэйсхолдер. Можно так и оставить.

```



