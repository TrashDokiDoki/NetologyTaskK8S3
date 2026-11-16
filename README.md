# NetologyTaskK8S3

# Домашнее задание к занятию «Запуск приложений в K8S»

## Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

### 1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку

Был создан Deployment `netology-web`, состоящий из двух контейнеров:

- `nginx`
- `multitool`

Первоначально оба контейнера использовали порт `80`.

В результате контейнер `multitool` завершался с ошибкой:

```
bind() to 0.0.0.0:80 failed (98: Address in use)
```

Причина ошибки — контейнеры внутри одного Pod используют общее сетевое пространство, поэтому два процесса не могут одновременно использовать один и тот же IP-адрес и порт.

Для устранения конфликта для контейнера `multitool` были указаны другие порты:

```
env:
  - name: HTTP_PORT
    value: "8080"
  - name: HTTPS_PORT
    value: "11443"
```

После изменения Deployment Pod успешно запустился:

```
netology-web-dcb576684-wx4gt   2/2   Running
```

[Манифест Deployment](deployment-web.yaml)

<img width="489" height="127" alt="image" src="https://github.com/user-attachments/assets/b1da269a-5f0b-4171-98a0-d6e998e7d4c4" />

---

### 2. После запуска увеличить количество реплик работающего приложения до 2

Количество реплик было увеличено командой:

```
kubectl scale deployment netology-web --replicas=2
```

---

### 3. Продемонстрировать количество Pod до и после масштабирования

До масштабирования работала одна реплика:

```
netology-web-dcb576684-wx4gt   2/2   Running
```

После масштабирования:

```
kubectl get pods
```

получены две работающие реплики:

```
netology-web-dcb576684-2tg4v   2/2   Running
netology-web-dcb576684-wx4gt   2/2   Running
```

Проверка Deployment:

```
kubectl get deployment netology-web
```

Результат:

```
NAME           READY   UP-TO-DATE   AVAILABLE
netology-web   2/2     2            2
```

Также был проверен ReplicaSet:

```
kubectl get rs
```

```
NAME                     DESIRED   CURRENT   READY
netology-web-dcb576684   2         2         2
```

<img width="599" height="215" alt="image" src="https://github.com/user-attachments/assets/1406d246-76a6-4301-8106-d8afc722d4cc" />

---

### 4. Создать Service, который обеспечит доступ до реплик приложения

Для доступа к репликам создан Service `netology-web-svc`.

Service предоставляет два порта:

- `80` — доступ к nginx;
- `8080` — доступ к multitool.

[Манифест Service](service-web.yaml)

Проверка Service:

```
kubectl get svc
```

Результат:

```
netology-web-svc   ClusterIP   10.152.183.52   <none>   80/TCP,8080/TCP
```

Также была выполнена проверка endpoints:

```
kubectl get endpoints netology-web-svc
```

Service обнаружил обе реплики приложения:

```
10.1.167.225:8080
10.1.167.226:8080
10.1.167.225:80
10.1.167.226:80
```

<img width="656" height="108" alt="image" src="https://github.com/user-attachments/assets/890d2829-70a3-45fa-a371-fa2799910570" />

---

### 5. Создать отдельный Pod с приложением multitool и проверить доступ с помощью curl

Для проверки доступа создан отдельный Pod `multitool`.

[Манифест Pod multitool](multitool-pod.yaml)

Проверка доступа к nginx через Service:

```
kubectl exec multitool -- curl http://netology-web-svc
```

Получен ответ:

```
<h1>Welcome to nginx!</h1>
```

Это подтверждает доступ из отдельного Pod до nginx через Service.

Также выполнена проверка доступа к контейнеру multitool:

```
kubectl exec multitool -- curl http://netology-web-svc:8080
```

Получен ответ:

```
WBITT Network MultiTool (with NGINX) - netology-web-dcb576684-2tg4v - 10.1.167.226 - HTTP: 8080 , HTTPS: 11443
```

Это подтверждает доступ до второго контейнера приложения через Service.

<img width="1490" height="648" alt="image" src="https://github.com/user-attachments/assets/b51a55e1-b274-40b7-a8f7-2f33d28cc774" />

**Результат задания 1:** Deployment с контейнерами nginx и multitool успешно создан, конфликт портов устранён, приложение масштабировано до двух реплик, Service предоставляет доступ к обеим репликам, а доступ из отдельного Pod подтверждён с помощью `curl`.

---

# Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

### 1. Создать Deployment nginx и обеспечить старт контейнера только после запуска Service

Создан Deployment `nginx-init`.

В Deployment используется InitContainer `wait-for-service` на основе образа `busybox`.

InitContainer проверяет наличие Service `nginx-init-svc`:

```
until nslookup nginx-init-svc.default.svc.cluster.local.; do
  echo "Waiting for nginx-init-svc..."
  sleep 5
done
```

Основной контейнер nginx запускается только после успешного завершения InitContainer.

[Манифест Deployment с InitContainer](deployment-init.yaml)

---

### 2. Убедиться, что nginx не стартует. В качестве Init-контейнера используется busybox

Deployment был запущен до создания Service.

Проверка:

```
kubectl get pods
```

Результат:

```
nginx-init-7d6b765b48-jlh4m   0/1   Init:0/1
```

Основной контейнер nginx не запускался.

Логи InitContainer:

```
kubectl logs -l app=nginx-init -c wait-for-service
```

Результат:

```
server can't find nginx-init-svc.default.svc.cluster.local: NXDOMAIN

Waiting for nginx-init-svc...
```

Таким образом, InitContainer ожидал появления Service.

<img width="595" height="273" alt="image" src="https://github.com/user-attachments/assets/4bc25ffb-7ec0-499f-96ce-a5e99ac6ef17" />

---

### 3. Создать и запустить Service. Убедиться, что Init завершился

Создан Service `nginx-init-svc`.

[Манифест Service](service-init.yaml)

После создания Service его DNS-имя стало успешно разрешаться:

```
Name:   nginx-init-svc.default.svc.cluster.local
Address: 10.152.183.222
```

Проверка InitContainer:

```
kubectl describe pod nginx-init-6d9cd6f9bd-2kl5k
```

Результат:

```
Init Containers:
  wait-for-service:
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
```

Это подтверждает успешное выполнение InitContainer.

---

### 4. Продемонстрировать состояние Pod до и после запуска Service

До запуска Service:

```
nginx-init-7d6b765b48-jlh4m   0/1   Init:0/1
```

После запуска Service:

```
kubectl get pods
```

Результат:

```
nginx-init-6d9cd6f9bd-2kl5k   1/1   Running
```

Состояние основного контейнера nginx:

```
State:          Running
Ready:          True
```

<img width="886" height="865" alt="image" src="https://github.com/user-attachments/assets/81a9d353-cdcd-425d-aecf-21fe79d2fcaf" />
<img width="831" height="422" alt="image" src="https://github.com/user-attachments/assets/76963915-e1f5-4600-97f8-1efd6b06c618" />

**Результат задания 2:** основной контейнер nginx не запускался до появления Service. После создания Service InitContainer успешно разрешил DNS-имя сервиса, завершился с `Exit Code: 0`, после чего основной контейнер nginx перешёл в состояние `Running`.
