
# Mikikube

Что это?
- реализация локального k8s (есть все фичи),
- все совместимо с обычным k8s,
- поддерживается и разрабатывается комьюнити k8s
Альтернативы - cat, k0s(больше для запуска легковесных кластеров на rasberry), k3s(rancher), kind (разработкаи компонентов k8s локально)

Как запустить?
- minikube работает внутри виртуальной машины - docker, hyper-v(Windows), hyperkit(macOS), virtual box, kvm. 

<!!Links: how to deploy>

## Addons

minikube addons list    -- вывести список аддонов (можно установить)
minikube addons enable <name: ingress, dashboard>   -- добавление аддона

# Run Minikube

[Установка Minikube]:(https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)
[Настройка kubectl]: (https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
[Kubernetes with Minikube]:(https://kubernetes.io/ru/docs/setup/learning-environment/minikube/)

Команды:
minikube status
minikube version
minikube start --driver=<драйвер виртуализации> (default = docker)
kubectl use-context <context: cluster, minicube>      -- переключение контекста, на какой кластер смотрим в команде

minikube service <service> -n <namespace>    -- прокидывает туннель в сервис

## Dashboard

minikube service kubernetes-dashboard -n kubernetes-dashboard  -- пример с дашбордом
Если сразу не получается открыть дашбоард: 	
- kubectl get service kubernetes-dashboard -n kubernetes-dashboard -o yaml   -- посмотреть в чем дело с сервисом, заметить, что тип не NodePort
- kubectl -n kubernetes-dashboard edit svc kubernetes-dashboard  -- отредактировать тип сервиса: ClusterIP -> NodePort

minikube dashboard  -- более простой вариант

Можно:
- смотреть, добавлять и редактировать сущности,
- в поде: смотреть логи и exec внутрь

# Local Development

Онлайн разработка: 
- компелируемые языки: комплируем/собираем - запускаем сборку образа из docker файла - запускаем docker-compose с зависимостями
- интерпретируемые языки: монтируем директорию с кодом в docker-compose - запускаем compose с зависимостями
Возможно ли так же в kubenetes?

## Инструкция

> ВАЖНО!!! нужно находиться в директории `~/school-dev-k8s/practice/9.local-development/app/`

0) Шаги 1 и 2 можно заменить командой(сборка приложения встроенным в minikube docker):
``` bash
minikube image build . -t myapp:dev
```

1) Сначала нужно подключиться к докеру в minikube. Для этого выполним команду:
```bash
eval $(minikube docker-env)
```
для Windows: minikube docker-env | Invoke-Expression ... Переключение контекста живет в рамках сессии консоли. 

2) Билдим образ
```bash
docker build . -t myapp:dev
```
3) Монтируем директорию в миникуб
В рамках деплоймента мы монтируем внутрь контейнера директорию hostpath: /app, которая находится в корне самого миникуба (виртуалки или контейнера). Чтобы эта директория там появилась - для начала нужно смонтировать туда локальгую директорию.  

После этого В ОТДЕЛЬНОЙ КОНСОЛИ запускаем команду для монтированиялокальной директории в minikube и оставляем висеть.
> ВАЖНО!!! нужно находиться в директории `~/school-dev-k8s/practice/9.local-development/app/`
```bash
minikube mount .:/app
```
- eсли __возникают проблемы__ - смотри файл problems_windows из репозитория организаторов (помогли прописание правил брандмаузера)
- eсли используется __driver=none__ - то монтируется локальная директория из ОС: поменять в деплойменте на реальную 
- можно не оставлять вторую консоль висеть, если сразу ее перевести в background (добавить & в конце команды)

4) Запускаем манифесты kubernetes
```bash
kubectl apply -f kube/
```
- проверяем что приложение запустилось
```bash
kubectl get po
kubectl logs <pod_name>
```
- можем открыть его в браузере:
```bash
minikube service myapp
```

5) Вносим изменения в код

Открываем файл app.py и вносим изменение -> проверяем что приложение задеплоилось:
```bash
kubectl logs <pod_name>
```
Должно быть такое:
```bash
 * Detected change in '/app/app.py', reloading
 * Restarting with stat
```
и можем проверить в браузере что изменения действительно применились

## Best practice

* Лучше - docker compose (особенно в случае использования базы)
* Интересные инструменты:
	- kubectl port-forward (связывает локальный и удаленный порт на стенде) 
	- skaffold (совмещает команды docker, kubectl, helm). 
	  Пример сценария: skaffold release (под капотом: сборка - проставление тега - отправка в репозиторий - перезапуск деплоймента в kubernetes)
	  [skaffold]:(https://skaffold.dev/)
	- telepresence (автоматизация этапов разработки кода)  
	  [telepresence]:(https://github.com/telepresenceio/telepresence)
	- плагины в IDE
	- Lens (gui для kubernetes, дашборды)
	  [Lens]:(https://k8slens.dev/)

## QA

1) Как добавить в minikube нод?
```bash
minikube node addons
```	  
В случае VM добавит еще одную виртуальную машину.
2) Что значит статус контейнера crashlookbackoff?
kubernetes пробует в случае ошибок при запуске приложения пытается его перезапустить, но если после нескольких попыток запустить не удалось - меняет статус на crashlookbackoff и пробует запустить в большим интервалом. 
3) Как выйти из minikube?
- minikube stop - останавливает кластер,
- minikube delete - удаляет его 


