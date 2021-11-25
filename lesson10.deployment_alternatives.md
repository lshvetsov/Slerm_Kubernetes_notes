**Альтернативы Deployment:**
* StatefulSet
* DeamonSet

## Входной пример
Запуск мониторинга - агент на каждой ноде + централизованное управдение и конфигурация.
Варианты решения:
* `static pod` - специальные поды, манифесты которых лежат на ноде в etc/kubernetes/manifest. При запуске ноды, kubelet смотрит в этот каталог и запускает описанные контейнеры (еще до обращения к kube api-server). По сути, централизованного управления и конфигурации нет: поды управляются локально (нельзя управлять через api-server, kubectl), можно только изменить файл. 
* `pod anti affinity` - механизм привязки подов к нодам. При добавлении узлов - править deployment, добавив новые ноды.
* `stateful set` - 

# StatefulSet

Запуск приложений, которые хранят свое состояние (БД, брокеры сообщений, vault) \
**Особенности:** \
1) Deployment - все поды одинаковые, StatefulSet - создает __уникальный__ под. 
2) Поды запускаются __последовательно__ (0,1,2...). Если удаляем - то создается под с тем же именем
3) __PVC tepmplate__ - механизм создания индивидуального PV (диска, тома) для каждого пода (в deployment поды создаются с ссылкой на один PV). При удалении/перезапуске пода - PV не удаляются, пересозданные поды (см. 2) используют существующие PV с сохраненными на ними данными. 

**Важно!** При отказе ноды с подами StatefulSet автоматического пересоздания подов не будет (тк они уникальны), kubernetes будет ожидать или возвращения ноды (с уже созданными подами), или удаления пода или ноды вручную. 

## Practice

Запуск RabbitMQ в StatefuSet.\
В RabbitMQ поддерживается интеграций с kubernetes через специальный плагин (ходит в api kubernetes и получает количество реплик в рамках StatefulSet). \
Манифест StatefuSet:
- serviceName - 
- serviceAccountName - 
- переменные окружения:
  - MY_POD_IP = status.podIP (получение ip-адреса пода через backward api)
  - RABBIT_MQ_NODENAME = "rabbit@$(MY_POD_IP)" (использование переменной для значения другой переменной $(var))
- volumeClaimTemplate (находится на уровне spec пода) - описывает PVC (объявленный в volume) \

```bash
# применили все файлы из текущей директории
kubectl apply -f .
# поды поднимаются последовательно 
kubectl get pod -o wide
# два тома: по одному для каждого пода
kubectl get pvc

```

## Headless services

Сервис типа `clusterIP:none` (по сути без ip-адреса). Нет правил маршруизации, но добавляются dns-записи. При образении по адресу он резолвится в несколько ip, далее запрос распределяется произвольно (балансировка dns round robin). \
Можно а скриптах автоматизации прописать конкретные ip stateful подов и обращаться к ним напрямую.

``` bash
kubectl get svc # get all services
kubectl get ep # get all endpoint (in our case two endpoint)

kubectl run -t -i --rm --image centosadmin/utils:0.3 test bash # run pod with utilities for testing

nslookup rabbitmq  # address resolving 

Server:		10.107.0.10
Address:	10.107.0.10#53

Name:	rabbitmq.default.svc.cluster.local  # full service name: service_name.namespace.type.domain
Address: 10.107.16.12   # ips
Name:	rabbitmq.default.svc.cluster.local
Address: 10.107.16.13
Name:	rabbitmq.default.svc.cluster.local
Address: 10.107.18.24

nslookup rabbitmq-0.rabbitmq  # instance (pod) resolving !!! work only with statefulset pods

Server:		10.107.0.10
Address:	10.107.0.10#53

Name:	rabbitmq-0.rabbitmq.default.svc.cluster.local  # full pod name: pod_name.service_name.namespace.type.domain
Address: 10.107.16.12

``` 

# Affinity

Требование к размещению сущностей на разных узлах

**Типы:** 

1) `nodeAffinity` - выбираем размщение исходя из свойств ноды \

Расширенный варианты nodeSelector (DeamonSet) \
Атрибуты:
	- requiredDuringSchedulingIgnoreDuringExecution - требуется на этапе размещения пода \
		`nodeSelector` - выбирает ноды по метке (`matchExpression`). Если размещение невозможно - под зависнет Pending. 
	- preferredDuringSchedulingIgnoreDuringExecution - желательно выполнить при размещении пода
		Если есть узлы удовлетворяющие условию (`matchExpression`), то запуститься там, если нет - то запустятся где-то еще. Можно задать несколько вариантов размещения с весами (`weight`).	

2) `podAffinity` - выбираем размещение исходя из уже запущеных там подов \

Логика работы: берем список всех подов по условию/меткам (`labelSelector`) и проверяем их размещение (`topologyKey`).
- podAffinity - в этом случае мы требуем (`requred...`)/просим(`preferred...`) запустить поды на узлах, где еще уже запущенны отобранные поды (частое использование - __антипаттерн__, нарушает логику kubernetes) \
- podAntiAffinity - в этом случае мы требуем (`requred...`)/просим(`preferred...`) запустить поды на узлах, где еще нет отобранных подов


# DeamonSet

- запускает 1 под на кластер,
- при добавлении/удалении узлов - автоматические появляются/исчезают поды
- манифест = deployment (только не указываем количество реплик)

Манифест аналогичен deployment, **особенности**:
- `nodeSelector` - механизм выбор узлов для запуска подов по меткам (мектку проставляет kubelet при запкске на узле). Например метка ОС - kubernetes.io/os:linux
- `securityContext` (runAs)
- `taints` и `tolerations` - механизм запрещающий запуск определенных подов на определенных узлах () \
Общий принцип такой: часть нод обозначаются `taint` (заразными), а часть подов указываются устойчивыми к заразер(`tolerations`).\
Зараза имеет атрибуты `key` и `value` (key - идентифицировать заразу, value - чтобы гибче настраивать толерантность подов) + атрибут `effect`:
- NoSchedule - действует на новые поды, учитывается sheduler при размещении пода на узле (если или нет заразы и сопротивляемости)
- NoExecute - действует действующие поды: убираются поды на этой ноде и запускаются на новой


## Practice

```bash
kubectl apply -f daemonset.yaml  # apply deployment
kubectl get po -w  # check pods online
```
- В примере поды запустили не на всех нодах, далее проводим диагностику:
```bash
# check that node labes equals nodeSelector (manifest)
kubectl get nodes --show-labels # show nodes with labels
kubectl get nodes -l kubernetes.io/os=linux # show only nodes with the given label
kubectl get po -o wide # show extended infornation about pods
```
- Выяснили, что проблема заключается, что у deamonset не сопротивляемости для запуска на определенных нодах. \
Добавляем сопротивление (для всех зараз NoSchedule, поэтому не указываем конкретный key):
```yaml
tolerations:
- effect: NoSchedule
  operator: Exists
```
- После перезапуска deamonset стали удаляться старые поды - тк были изменения в манифесте - поды перезапускаются.

# QA

__1) Зачем нужны tains и toleratons, если есть labels и nodeAffinity?__ \
Разные задачи: labels + na - указывают на какой ноде запускать, tt - что нельзя запускать на ноде
__2) Могут ли statefulset pod читать configmap & secrets?__ \
Нет, эти эти поды запускаются на этапе запуска kubelet, поэтому не погут читать из kube api. Но могуть читать из hostpath, то есть из директории с ноды. 
__3) Что такое kubernetes operatior?__
Модное название программ, взаимодействующих с api kubernetes, и автоматизирующих какую-либо операцию.
4)  