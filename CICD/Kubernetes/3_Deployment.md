# Deployment
## Ручная работа в терминале
Чтобы создать Deployment необходимо использовать:
```console
user@user-PC:~$ kubectl create deploy max-deploy --image tomcat
deployment.apps/max-deploy created
```

Теперь проверим наш созданный Deployment:
```console
user@user-PC:~$ kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
max-deploy   1/1     1            1           64s
```

<details>
  <summary>А где же поды?</summary>

------
  Deployment автоматически сам создал необходимое ему количество подов, мы можем это проверить:

  ```console
  user@user-PC:~$ kubectl get pods
  NAME                          READY   STATUS    RESTARTS   AGE
  max-deploy-8684b6d5b7-tt2kb   1/1     Running   0          3m17s
  ```
------
</details>

Попробуем получить информацию о Deploy:
```console
user@user-PC:~$ kubectl describe deploy max-deploy
# Куча полезной информации о Deploy и подах внутри
```

Попробуем **отмасштабировать** наш Deploy. Поговорим о **replicas**. Для увеличения количество подов, необходимо 
указать количество реплик (инстансов нашего контейнера в Pods):
```console
user@user-PC:~$ kubectl scale deploy max-deploy --replicas 3
deployment.apps/max-deploy scaled
```

Проверим наш деплой:
```console
user@user-PC:~$ kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
max-deploy   3/3     3            3           10m
```

Только что мы создали объект **ReplicaSet**, который и отвечает за поддержание стабильного количества реплик Подов.
Мы можем проверить наш новый объект ReplicaSet:
```console
user@user-PC:~$ kubectl get rs
NAME                    DESIRED   CURRENT   READY   AGE
max-deploy-8684b6d5b7   3         3         3       14m
```

<details>
  <summary>Проверим количество подов</summary>

------
  ```console
  user@user-PC:~$ kubectl get pods
  NAME                          READY   STATUS    RESTARTS   AGE
  max-deploy-8684b6d5b7-grhbn   1/1     Running   0          47s
  max-deploy-8684b6d5b7-k4n2f   1/1     Running   0          47s
  max-deploy-8684b6d5b7-tt2kb   1/1     Running   0          9m10s
  ```
------
</details>

<details>
  <summary>А что если убить один из подов?</summary>

------
  Попробуем убить один из подов и посмотреть, что будет. Посмотрим на все наши поды:

  ```console
  user@user-PC:~$ kubectl get pods
  NAME                          READY   STATUS    RESTARTS   AGE
  max-deploy-8684b6d5b7-grhbn   1/1     Running   0          47s
  max-deploy-8684b6d5b7-k4n2f   1/1     Running   0          47s
  max-deploy-8684b6d5b7-tt2kb   1/1     Running   0          9m10s
  ```

  Убьем один из них:
  ```console
  user@user-PC:~$ kubectl delete pods max-deploy-8684b6d5b7-grhbn
  pod "max-deploy-8684b6d5b7-grhbn" deleted
  ```

  Посмотрим, что произошло:
  ```console
  user@user-PC:~$ kubectl get pods
  NAME                          READY   STATUS              RESTARTS   AGE
  max-deploy-8684b6d5b7-k4n2f   1/1     Running             0          11m
  max-deploy-8684b6d5b7-njjhf   0/1     ContainerCreating   0          2s
  max-deploy-8684b6d5b7-tt2kb   1/1     Running             0          20m
  ```

  Как мы видим, тут же стартует еще один. Все для того, чтобы подов всегда было указанное количество.

------
</details>

## Horizontal Pod Autoscaling (HPA)
Для деплоя можно настроить autoscale. Этот объект смотрит, какое количество ресурсов потребляет контейнер в Pod. 
Если используемый ресурс перешел пограничную отметку, создаются новые Pods, если можно сократить количество Pods, 
они сокращаются.
 
```console
user@user-PC:~$ kubectl autoscale deploy max-deploy --min=4 --max=6 --cpu-percent=80
horizontalpodautoscaler.autoscaling/max-deploy autoscaled
```

Посмотреть созданые HPA можно при помощи команды:

```console 
user@user-PC:~$ kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
max-deploy   Deployment/max-deploy   <unknown>/80%   4         6         4          65s
```

## Deployment image update
Допустим наш сервис какое-то время крутится в кубере, появилась новая версия этого сервиса и нам нужно обновить
версию контейнера. Kubernetes позволяет для этих целей не пересоздавать целый deployment, а только обновить версию
контейнера. Для начала нам нужно узнать название контейнера в deploy. Вызываем `describe` и ищем следующую строчку:

```console
user@user-PC:~$ kubectl describe deploy max-deploy
...
Pod Template:
  Labels:  app=max-deploy
  Containers:
   tomcat: # <-- нас интересует эта строчка, это название контейнера
...
```

Теперь зная название контейнера, изменим версию image для него:
```console
user@user-PC:~$ kubectl set image deploy/max-deploy tomcat=tomcat:8.5.84-jre11
deployment.apps/max-deploy image updated
```

Все, версия image обновилась. Попробуем посмотреть историю версий:
```console
user@user-PC:~$ kubectl rollout history deploy/max-deploy
deployment.apps/max-deploy 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deploy/max-deploy tomcat=tomcat:8.5.84-jre11
```