*Tarantool* - open-source решение: in-memory БД + сервер приложений (stateful + stateless)

Данные в памяти снапшотятся (сохранение на случай проблем). 

_Масшитабирование_ Tarantool:
- несколько tarantool -> в репликасет
- несколько репликасет -> в кластер

![](./Pictures/19Tarantool1.jpg)

*Как деплоить?*
- через `deployment` - плохо,
- через `statefulSet` (+ headless service - чтобы пропускать запросы до каждого узла по его имени), задавая имя узла - работает,
- через `kubernetes operator` - лучший вариант

*Kubernetes operator* - кастомный контроллер, который контролирует состояние кастомного ресурсы (*CRD*, custom resource definition). Кастомный - заданный пользователем.

_Алгорит запуска_:
1. Установка CRDs,
2. Установка оператора (отдельный deployment пода, список ресурсов к управлению, service account),
3. Установка состояния CRDs (добавление экземпляров для отслеживания оператору) \
Все операции можно сделать `kubectl apply -f <path_to_manifests>`

Оператор может реагировать на изменение в состоянии управляемых объектов (экземпляром CRDs, обратный поток) -> по сути можно привести к состоянию автопилота (разруливает все конфиликты) 

Операторы пишутся разработчиками или предоставляются opensource или проприетарно. 


# Links

Манифесты с вебинара:
1) deployment https://raw.githubusercontent.com/tarantool/tarantool-operator/slurm-webinar/examples/slurm-webinar/deployment_demo.yaml
2) statefulset https://raw.githubusercontent.com/tarantool/tarantool-operator/slurm-webinar/examples/slurm-webinar/statefulset_demo.yaml
3) crds https://raw.githubusercontent.com/tarantool/tarantool-operator/slurm-webinar/examples/slurm-webinar/crds_demo.yaml
4) operator https://raw.githubusercontent.com/tarantool/tarantool-operator/slurm-webinar/examples/slurm-webinar/tarantool_operator_demo.yaml
5) tarantool cluster https://raw.githubusercontent.com/tarantool/tarantool-operator/slurm-webinar/examples/slurm-webinar/tarantool_cluster_demo.yaml

