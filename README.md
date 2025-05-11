# Домашнее задание к занятию «Запуск приложений в K8S»

# Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

## Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------
## Проверка готовки к выполению работы
```
minikube status
kubectl get nodes
docker ps
```
![img 1](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img1.jpg)

Необходимое П.О. установлено! можно приступать к выполнению задания


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

## Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

------

## Решение 1.
1. Необходимо ссоздать namespace netology
``` 
 kubectl create namespace netology
```

 проверим какие пространства имен у нас есть!
```
kubectl get namespaces
```
2. Создание deployment-single.yaml (с одной репликой)
```
nano deployment-single.yaml
```

листинг  deployment-single.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: netology
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nmt
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 1180
```
3. Применение Deployment с одной репликой

Применяем Deployment:
```
kubectl apply -f deployment-single.yaml
```

Проверим статус:
```
kubectl get pods -n netology
kubectl get deployments -n netology
```
4. Исправление ошибок
под не поднялся ! 
![img 3](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img3.jpg)

проверим статус Pod-а
```
kubectl describe pod nginx-multitool-6bf48db46c-rj7ls -n netology

```

 Проверим логи контейнеров:

    Для контейнера nginx:
```
kubectl logs nginx-multitool-6bf48db46c-rj7ls -n netology -c nginx
```

    Для контейнера multitool:
```
kubectl logs nginx-multitool-6bf48db46c-rj7ls -n netology -c multitool
```

Мы получили важное диагностическое сообщение из логов контейнера multitool:

perl
Копировать
Редактировать
bind() to 0.0.0.0:80 failed (98: Address in use)
nginx: [emerg] still could not bind()
 Причина проблемы:
Порт 80 уже используется контейнером nginx.

Контейнер multitool также пытается запустить nginx на том же порту 80, и это вызывает конфликт.

В Kubernetes один Pod — это единая сетевая зона, и два контейнера не могут слушать один и тот же порт.

Решение проблемы:
Изменить порт для multitool:

В deployment-single.yaml нужно изменить порт для multitool.

Например, укажем 1180 (уникальный порт для multitool).

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: netology
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nmt
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 1180
        env:
        - name: HTTP_PORT
          value: "1180"

```
под в работе! 
![img 4](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img4.jpg)

5. Обновление Deployment до двух реплик
```
nano deployment-replicas.yaml
```

листинг deployment-replicas.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: netology
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nmt
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 1180
        env:
        - name: HTTP_PORT
          value: "1180"
```

6. Применение Deployment с двумя репликами

```
kubectl apply -f deployment-replicas.yaml
```

Проверим статус
```
kubectl get pods -n netology
kubectl get deployments -n netology
```
![img 5](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img5.jpg)

7. Создание Service
```
nano service-replicas.yaml
```

листинг service-replicas.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-svc
  namespace: netology
spec:
  selector:
    app: nmt
  ports:
    - protocol: TCP
      name: nginx
      port: 80
      targetPort: 80
    - protocol: TCP
      name: multitool
      port: 8080
      targetPort: 1180
```

8. Применение Service
```
kubectl apply -f service-replicas.yaml
```

Проверим:
```
kubectl get svc -n netology
```
![img 6](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img6.jpg)

9. Создадим отдельный Pod multitool
```
nano multitool-pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: multitool-test
  namespace: netology
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool
    ports:
    - containerPort: 1180
```
Применим манифест:
```
kubectl apply -f multitool-pod.yaml
```

Проверим статус Pod-а:
```
kubectl get pods -n netology
```
10. Подключаемся в Pod multitool-test:

Выполним команду для подключения:
```
kubectl exec -it multitool-test -n netology -- /bin/bash
```

Теперь мы внутри Pod-а multitool-test!

Проверим доступ по имени Service:
```
curl http://nginx-multitool-svc:80
```
Проверим доступ по IP Service:

Для этого сначала получим IP Service:

kubectl get svc -n netology

И выполним:
```
curl http://10.99.23.134:80
```

Проверим доступ к multitool:
```
curl http://nginx-multitool-svc:8080
```

![img 7](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img7.jpg)
![img 8](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img8.jpg)
![img 9](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img9.jpg)

## Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

------
## Решение 2.

### Анализ Задания :

Цель:

    Создать Deployment, который использует Init-контейнер busybox.

    Init-контейнер будет ожидать доступность сервиса приложения перед запуском основного контейнера nginx.

    Основной контейнер nginx не должен стартовать, пока сервис не будет доступен.

### Логика выполнения задания:

    Создаём Deployment с Init-контейнером busybox.

        Init-контейнер будет выполнять команду wget и проверять доступность сервиса.

        Если сервис недоступен, Init-контейнер будет находиться в статусе Waiting.

    Основной контейнер nginx стартует только после завершения Init-контейнера.

    Создаём Service для nginx

1. Создание Deployment с Init-контейнером

Создадим файл nginx-init-deployment.yaml:
```
nano nginx-init-deployment.yaml
```

Листинг nginx-init-deployment.yaml
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-init
  namespace: netology
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-init
  template:
    metadata:
      labels:
        app: nginx-init
    spec:
      initContainers:
      - name: delay
        image: busybox
        command: ['sh', '-c', "until nslookup nginx-init-svc.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for nginx-init-svc; sleep 2; done"]

      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf

      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: netology
data:
  default.conf: |
    server {
      listen 0.0.0.0:80;
      server_name localhost;

      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }

```

2. Применение Deployment

```
kubectl apply -f nginx-init-deployment.yaml
```

3. Проверим статус Pod-а:

```
kubectl get pods -n netology
```

4. Просмотр состояния Pod-а до создания Service:

``` 
kubectl describe pod -n netology -l app=nginx-init
```

![img 10](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img10.jpg)
![img 11](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img11.jpg)

5. Создание Service для nginx

Теперь создадим Service, который станет доступным для nginx:

Создадим файл nginx-init-svc.yaml:

```
nano nginx-init-svc.yaml
```
листинг nginx-init-svc.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-init-svc
  namespace: netology
spec:
  type: NodePort
  selector:
    app: nginx-init
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30515
```

Приминяем
```
kubectl apply -f nginx-init-svc.yaml
```
![img 12](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img12.jpg)

pod поднялся после запуска сервиса 

проверим ответ сервера
![img 13](https://github.com/ysatii/kuber-homeworks1.3/blob/main/img/img13.jpg)

файлы  конфигурации используемые в задании 
К заданию 1
1. [Создание deployment-single.yaml с одной репликой](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/deployment-single.yaml)
2. [Обновление Deployment до двух реплик](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/deployment-replicas.yaml)
3. [создание сервиса](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/service-replicas.yaml)
4. [отдельный Pod multitool](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/multitool-pod.yaml)
К заданию 2
1. [Создание Deployment с Init-контейнером](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/nginx-init-deployment.yaml)
2. [Создание сервиса](https://github.com/ysatii/kuber-homeworks1.3/blob/main/src/nginx-init-svc.yaml)

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
