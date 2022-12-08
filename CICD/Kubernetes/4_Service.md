# Service

Существует 4 типа Service:
1) **ClusterIP** - доступен IP только внутри кластера. Используется по умолчанию. Если мы не создали Service, то 
используется тоже ClusterIP.
2) **NodePort** - указанный порт становится доступен на всех нодах, где крутится необходимый под. 
3) **ExternalName** - Создается DNS имя.
4) **LoadBalancer** - Создается балансировщик. Доступно только для Cloud Clusters (AWS, Google, Azure, etc)

## Ручная работа в терминале
Предположим, что у нас уже есть deploy, который запустил некоторое число pods. Создадим service для этого deploy с 
типом NodePort:
```console
user@user-PC:~$ kubectl expose deploy max-deploy --type=NodePort --port 8080
service/max-deploy exposed
```

Посмотрим созданный NodePort:
```console
user@user-PC:~$ kubectl get services # вместо services можно написать просто svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        30d
max-deploy   NodePort    10.99.87.168   <none>        8080:30246/TCP   43s
```
kubernetes это сервис по умолчанию, который всегда стартует. Наш сервис max-deploy открыл порт 30246 на всех 
нодах, где есть необходимые поды. Осталось узнать IP подов. Я использую локальный minikube, поэтому мне нужно узнать 
InternalIP:

```console
user@user-PC:~$ kubectl describe nodes | grep InternalIP
  InternalIP:  192.168.49.2
```

Теперь если мы перейдем в браузере по `192.168.49.2:30246`, то мы увидим страницу Apache Tomcat.

