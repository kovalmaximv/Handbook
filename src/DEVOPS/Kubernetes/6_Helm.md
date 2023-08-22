# Helm 
Helm - пакетный менеджер для kubernetes. 

Главные фичи Helm:
1) Использование Charts пакетов
2) Шаблонирование k8s манифестов
3) Репозиторий готовых пакетов

#### Использование Charts пакетов
Helm Charts - пакет Helm, состоящий из yml шаблонов в особой иерархической структуре. 

```
├── charts         # Папка для управляемых вручную зависимостей чарта.
├── Chart.yaml     # Метаданные о чартах, такие как: версия, имя, ключевые слова для поиска и так далее.
├── templates      # Папка для хранения шаблонов, которые позже станут манифестами k8s
│   ├── ...
└── values.yaml    # Значение по умолчанию для переменных в шаблоне
```

Данные шаблоны позже преобразуются в манифесты. Таким образом удается удобно структурировать kubernetes манифесты.


#### Шаблонирование k8s манифестов
Использование yml шаблонов позволяет не писать множество очень похожих манифестов. Можно написать один шаблон и 
использовать параметры для его запуска.

#### Репозиторий готовых пакетов
Helm чарты можно выгрузить в репозиторий, чтобы к нему могли получить доступ другие пользователи. Так же, вы можете
использовать чужой чарт, если он подходит под ваши задачи.

## Написание своего Helm Chart
Пример готового Helm Chart можно посмотреть [здесь](./max_chart).

Чтобы запустить этот chart:
```console
# helm install {release.name} {path to chart}
user@user-PC:~$ helm install max-chart ./Chart-Maksim/
NAME: max-chart
LAST DEPLOYED: Tue Dec 13 14:44:28 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Посмотрим список запущенных charts:
```console
user@user-PC:~$ helm list
NAME     	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART              	APP VERSION
max-chart	default  	1       	2022-12-13 14:44:28.233846743 +0300 MSK	deployed	App-HelmChart-0.1.0	1.2.3
```

Чтобы потушить и удалить запущенный chart:
```console
user@user-PC:~$ helm delete max-chart
release "max-chart" uninstalled
```

#### Изменить параметр при старте чарта

Чтобы запустить chart изменив yml параметр:
```console
user@user-PC:~$ helm install max-chart ./Chart-Maksim/ --set parameter.name=value --set parameter2.name=value2
```

Чтобы изменить параметр уже бегущего чарта:
```console
user@user-PC:~$ helm upgrade max-chart ./Chart-Maksim/ --set parameter.name=value
```

#### Добавление параметров в зависимости от окружения (dev, test, prod)

Допустим, мы хотим добавить yml параметры для прода. Тогда следует добавить `values_prod.yml` файл в чарт:
```yaml
container:
  image: tomcat:5.8

replicaCount: 4
```

Теперь чтобы запустить это:
```console
# helm install {release.name} {path to chart} {path to values-prod.yml}
user@user-PC:~$ helm install max-chart ./Chart-Maksim/ -f ./values-prod.yml
NAME: max-chart
LAST DEPLOYED: Tue Dec 13 14:44:28 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```