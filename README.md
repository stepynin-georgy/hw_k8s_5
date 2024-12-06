# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 2»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к двум приложениям снаружи кластера по разным путям.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://microk8s.io/docs/getting-started) по установке MicroK8S.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Service.
3. [Описание](https://kubernetes.io/docs/concepts/services-networking/ingress/) Ingress.
4. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment приложений backend и frontend

1. Создать Deployment приложения _frontend_ из образа nginx с количеством реплик 3 шт.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
```

2. Создать Deployment приложения _backend_ из образа multitool.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
```

```
user@k8s:/opt/hw_k8s_5$ microk8s kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
backend-6d86996bf4-s2dfp    1/1     Running   0          19s
frontend-585bb98bfb-chjjw   1/1     Running   0          2m53s
frontend-585bb98bfb-qsccq   1/1     Running   0          2m53s
frontend-585bb98bfb-zkjqt   1/1     Running   0          2m53s

user@k8s:/opt/hw_k8s_5$ microk8s kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
backend    1/1     1            1           24s
frontend   3/3     3            3           3m24s
```

3. Добавить Service, которые обеспечат доступ к обоим приложениям внутри кластера.

```
apiVersion: v1
kind: Service
metadata:
  name: frontback-service
  labels:
    component: network2
spec:
  selector:
    component: network2
  ports:
    - protocol: TCP
      name: nginx
      port: 9001
      targetPort: 80
    - protocol: TCP
      name: multitool
      port: 9002
      targetPort: 1180
```

```
user@k8s:/opt/hw_k8s_5$ microk8s kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
frontback-service   ClusterIP   10.152.183.57   <none>        9001/TCP,9002/TCP   27s
kubernetes          ClusterIP   10.152.183.1    <none>        443/TCP             16d
```

4. Продемонстрировать, что приложения видят друг друга с помощью Service.

```
user@k8s:/opt/hw_k8s_5$ microk8s kubectl exec -n default -it backend-74d79fc7bc-vhp29 -- /bin/bash
backend-74d79fc7bc-vhp29:/# curl frontback-service:9001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
backend-74d79fc7bc-vhp29:/# curl frontback-service:9002
WBITT Network MultiTool (with NGINX) - backend-74d79fc7bc-vhp29 - 10.1.77.32 - HTTP: 1180 , HTTPS: 443 . (Formerly praqma/network-multitool)
backend-74d79fc7bc-vhp29:/#
```

5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

[deployment_frontend.yaml](https://github.com/stepynin-georgy/hw_k8s_5/blob/main/deployment_frontend.yaml)

[deployment_backend.yaml](https://github.com/stepynin-georgy/hw_k8s_5/blob/main/deployment_backend.yaml)

[svc.yml](https://github.com/stepynin-georgy/hw_k8s_5/blob/main/svc.yml)

------

### Задание 2. Создать Ingress и обеспечить доступ к приложениям снаружи кластера

1. Включить Ingress-controller в MicroK8S.

```

user@k8s:/opt/hw_k8s_5$ microk8s enable ingress
Infer repository core for addon ingress
Enabling Ingress
ingressclass.networking.k8s.io/public created
ingressclass.networking.k8s.io/nginx created
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
configmap/nginx-ingress-tcp-microk8s-conf created
configmap/nginx-ingress-udp-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
user@k8s:/opt/hw_k8s_5$ microk8s kubectl get pods -A | grep ingress
ingress                nginx-ingress-microk8s-controller-vzz99      1/1     Running   0             2m9s
```

2. Создать Ingress, обеспечивающий доступ снаружи по IP-адресу кластера MicroK8S так, чтобы при запросе только по адресу открывался _frontend_ а при добавлении /api - _backend_.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hwingress.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontback-service
            port:
              number: 9001
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: frontback-service
            port:
              number: 9002
```

```
user@k8s:/opt/hw_k8s_5$ microk8s kubectl apply -f ingress.yml
ingress.networking.k8s.io/web-ingress created
user@k8s:/opt/hw_k8s_5$ microk8s kubectl get ingress
NAME          CLASS    HOSTS           ADDRESS     PORTS   AGE
web-ingress   public   hwingress.com   127.0.0.1   80      5s
```

3. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.

```
user@k8s:/opt/hw_k8s_5$ curl http://hwingress.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

4. Предоставить манифесты и скриншоты или вывод команды п.2.

[ingress.yml](https://github.com/stepynin-georgy/hw_k8s_5/blob/main/ingress.yml)

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------
