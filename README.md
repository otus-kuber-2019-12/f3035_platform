# f3035_platform
f3035 Platform repository

## HW 01

### Разберитесь почему все pod в namespace kube-system восстановились после удаления.
- Pod-ы восстанавливает kubelet задача которого отвечать за выполнения подов на узле, и поддержание их работы. Kubelet в первую очередь запускает pod-ы согласно манифестам из каталога "/etc/kubernetes/manifests/" (напиример манифест для kube-apiserver). Сore-dns перезапускает тоже kubelet но уже как static pod, а не как часть pod-ов входящих в control-plane.

### Dockerfile
- Подготовлен docker-образ "webapp" на основе alpine:3.10 который содержит python-скрипт запускающий простой web-сервер публикующий содержимое каталога "/app".
- Docker-образ "webapp" залит на https://hub.docker.com/

### Манифест pod
- Подготовлен манифест "web-pod.yaml".
- Применен магифест "web-pod.yaml" и запущен pod "web" на основе образа "webapp".

### kubectl describe
- В целях задания в манифесте "web-pod.yaml" указан несуществующий docker-образ. При попытке запустить pod из манифеста pod получил статус ErrImagePull.

### Init контейнеры
- В манифест "web-pod.yaml" был добавлен init-контейнер на основе образа "busybox:1.31.1".

### Volumes
- В манифест "web-pod.yaml" добавлен volume типа "emptyDir" для init-контейнера и контейнера с web-сервером.

### Запуск pod
- Pod успешно запустился поочередно запустив init-контейнер, затем контейнер с web-сервером.

### Проверка работы приложения
- После выполнения команды "kubectl port-forward --address 0.0.0.0 pod/web 8000:8000", по адресу http://localhost:8000/index.html стала доступна страничка с логотипом express42 и информацией ниже.

### Hipster Shop
- Склонирован репозиторий "microservices-demo". Собран образ "frontend" на снове Dockerfile из каталога "src/frontend". Все залито на https://hub.docker.com/.
- Запущен pod "frontend" на основе образа "frontend".
- Сгенерен манифест "frontend-pod.yaml"

### Hipster Shop | Задание со *
- Ошибка запуска pod-а "frontend" была вызвана отсутствием заданных переменных для запуска приложения внутри контейнера, после добавления переменных в манифест "frontend-pod-healthy.yaml", и щапуска на основе него, pod "frontend" запустился.

## HW 02

### ReplicaSet

 - Создан и применен манифест *frontend-replicaset.yaml* для запуска одной реплики *frontend*.
 - Проверка с помощью команды `kubectl get pods -l app=frontend`


```console
NAME                        READY   STATUS    RESTARTS   AGE
frontend-58f49944b6-56vvv   1/1     Running   0          1m
```

- Увеличили количество реплик командой `kubectl scale replicaset frontend --replicas=3`
- Проверка `kubectl get rs frontend`

```console
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       3m
```

- Проверка что pod восстанавливаются, улалил pod командой `kubectl delete pods -l app=frontend`
- Проверка `kubectl get pods -l app=frontend`

```console
NAME             READY   STATUS    RESTARTS   AGE
frontend-bvk66   1/1     Running   0          47s
frontend-d62hc   1/1     Running   0          47s
frontend-sfdlz   1/1     Running   0          47s
```
- Повторно применил манифест `kubectl apply -f frontend-replicaset.yaml`
- Проверка что количество реплик стало рано 1 `kubectl get rs frontend`

```console
NAME       DESIRED   CURRENT   READY   AGE
frontend   1         1         1       1m34s
```

### Обновление ReplicaSet
- В манифесте *frontend-replicaset.yaml* изменен образ на тег v0.0.2.
- После применения манифеста поды остались с версией v0.0.1
- После удаления pod обновились до v0.0.2

- Поды были запущены как ReplicaSet, задача ReplicaSet-контероллера следить чтобы работало нужное количество подов,
 так как в манифесте изменили только образ, а количество реплик осталось прежним, ReplicaSet-контроллер не стал перезапускать поды.
Когда мы удалили поды, ReplicaSet-контроллер запустил новые согласно манифесту, а так как в манифесте указан уже новый образ,
 поды запустились из нового образа.

### Deployment
- Собрано два docker-образа **rt7402/hipster-paymentservice** c тегами **v0.0.1** и **v0.0.2**.
- Создан манифест *paymentservice-replicaset.yaml* разворачивающий три реплики из образа **rt7402/hipster-paymentservice:v0.0.1**.
- Создан манифест *paymentservice-deployment.yaml*.
- Применил манифест `kubectl apply -f paymentservice-deployment.yaml`
- Проверка

```console
# kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
paymentservice   3/3     3            3           1m28s

# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       24m
paymentservice              3         3         3       1m59s
```
### Обновление Deployment
- Изменил версию образа на v0.0.2, после применения все pod почередно обновились до новой версии, добавилась новая реплика.
- Проверка

```console
# kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
rt7402/hipster-paymentservice:v0.0.2 rt7402/hipster-paymentservice:v0.0.2 rt7402/hipster-paymentservice:v0.0.2

# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       35m
paymentservice              0         0         0       3m41s
paymentservice-6c4c5dff56   3         3         3       32s
```
### Deployment | Rollback
- Откат обновления paymentservice `kubectl rollout undo deployment paymentservice --to-revision=1`
- Проверка

```console
# kubectl get pods -l app=paymentservice -o=jsonpath='{.items[0:3].spec.containers[0].image}'
rt7402/hipster-paymentservice:v0.0.1 rt7402/hipster-paymentservice:v0.0.1 rt7402/hipster-paymentservice:v0.0.1

# kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
frontend                    3         3         3       35m
paymentservice              3         3         3       4m42s
paymentservice-6c4c5dff56   0         0         0       1m33s
```
### Deployment | Задание со ⭐
- Созданы и проверены в работе два манифеста (*paymentservice-deployment-bg.yaml*, *paymentservice-deployment-reverse.yaml*) стратегии развертывания:
**Аналог blue-green** и **Reverse Rolling Update**.

### Probes
- Создан и применен манифест *frontend-deployment.yaml* c версией образа **rt7402/hipster-frontend:v0.0.1** с механизмом *Probes*,
 затем применен манифест с ошибкой проверки (/_healthz >> /_health).
- Проверка (представлен не полный вывод команд describe)

```console
# kubectl describe pod frontend-58f49944b6-4mclp
...
    Ready:          True
    Restart Count:  0
    Readiness:      http-get http://:8080/_healthz delay=10s timeout=1s period=10s #success=1 #failure=3
...
```
- Вывод describe с ошибкой проверки (представлен не полный вывод команд describe)

```console
# kubectl describe pod frontend-676978ccfc-spcx4
...
Events:
  Type     Reason     Age                 From                   Message
  ----     ------     ----                ----                   -------
  Warning  Unhealthy  7s (x19 over 3m7s)  kubelet, kind-worker3  Readiness probe failed: HTTP probe failed with statuscode: 404
...
```

### DaemonSet | Задание со ⭐
- Создан и применен манифест *node-exporter-daemonset.yaml* на основе docker-образа **rt7402/node-exporter:v0.18.1**,
 после чего на каждой worker-ноде запущен один pod с **prometheus-node-exporter**.
- После выполнения команды `kubectl port-forward <имя любого pod в DaemonSet> 9100:9100` по адресу http://localhost:9100/metrics стали доступны метрики.
- Проверка
```console
# kubectl get pods -o wide --sort-by="{.spec.nodeName}" -l k8s-app=prometheus
NAME                              READY   STATUS    RESTARTS   AGE    IP            NODE                  NOMINATED NODE   READINESS GATES
prometheus-node-exporter-x2jdf    1/1     Running   0          1m   10.244.3.9    kind-worker           <none>           <none>
prometheus-node-exporter-mpddw    1/1     Running   0          1m   10.244.4.9    kind-worker2          <none>           <none>
prometheus-node-exporter-nsw7r    1/1     Running   0          1m   10.244.5.9    kind-worker3          <none>           <none>

# kubectl port-forward prometheus-node-exporter-x2jdf 9100:9100

# curl http://localhost:9100/metrics
----------------------------------------------------------------------------
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
```
### DaemonSet | Задание с ⭐ ⭐
- В манифест *node-exporter-daemonset.yaml* был добавлен параметр **tolerations** чтобы pod могли запускаться на master-нодах.
- После выполнения команды `kubectl port-forward <имя любого pod в DaemonSet> 9100:9100`
 для pod запущенного на master-ноде также по адресу http://localhost:9100/metrics стали доступны метрики.
- Проверка

```console
# kubectl get pods -o wide --sort-by="{.spec.nodeName}" -l k8s-app=prometheus
NAME                             READY   STATUS    RESTARTS   AGE    IP           NODE                  NOMINATED NODE   READINESS GATES
prometheus-node-exporter-8xrj4   1/1     Running   0          1m   10.244.0.4   kind-control-plane    <none>           <none>
prometheus-node-exporter-74kmg   1/1     Running   0          1m   10.244.1.2   kind-control-plane2   <none>           <none>
prometheus-node-exporter-pn6ln   1/1     Running   0          1m   10.244.2.2   kind-control-plane3   <none>           <none>
prometheus-node-exporter-x2jdf   1/1     Running   0          6m   10.244.3.9   kind-worker           <none>           <none>
prometheus-node-exporter-mpddw   1/1     Running   0          6m   10.244.4.9   kind-worker2          <none>           <none>
prometheus-node-exporter-nsw7r   1/1     Running   0          6m   10.244.5.9   kind-worker3          <none>           <none>
```
