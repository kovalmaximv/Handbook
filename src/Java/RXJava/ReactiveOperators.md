# Реактивные операнды

1. [Комбинируем Observable](#Комбинируем-Observable)
   - [Observable merging](#Observable-merging)
   - [Observable concatenation](#Observable-concatenation)
   - [Ambiguous фабрика](#Ambiguous-фабрика)
   - [Zipping фабрика](#Zipping-фабрика)
   - [groupBy операнд](#groupby-)
2. [Multicasting, Replaying, and Caching](#multicasting-replaying-and-caching)
   - [Multicasting](#multicasting)
   - [Caching](#caching)
   - [Replaying](#replaying)
   - [Subject](#subject)


## Комбинируем Observable
Операнды расмотренные в предыдущей главе полезны, поскольку позволяют управлять выбросами (emission). Но зачастую так 
же бывает необходимо комбинировать Observable. 

#### Observable merging
Для объединения нескольких Observable (observable merging) существует два способа: фабрика `Observable.merge` и 
операнд `mergeWith`. Данный метод может перемешивать значения горячих Observable. В целом лучше не надеяться на 
очевидный порядок следования элементов при мерже (особенно в многопоточных системах). Для этих целей лучше использовать 
`Observable.concat`. 

```java
Observable<String> src1 = Observable.just("Alpha", "Beta");
Observable<String> src2 = Observable.just("Zeta", "Eta");

Observable.merge(src1, src2).subscribe(i -> System.out.println("RECEIVED: " + i));

src1.mergeWith(src2).subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: Alpha
   RECEIVED: Beta
   RECEIVED: Zeta
   RECEIVED: Eta
 */
```

`flatMap()` так же один из инструментов объединения Observable и один из самых сильных инструментов RxJava. Он 
срабатывает для каждого пересылаемого объекта в цепи и на его основе делает Observable. Все исходящие из flatMap 
Observable при выходе мержатся в один. Получается что-то похожее `object -> Observable.just(...)`.

```java
Observable.just("Alpha", "Beta", "Gamma")
        .flatMap(s -> Observable.fromArray(s.split("")))
        .subscribe(System.out::println);

/*
   A
   l
   p
   h
   a
   B
   ...
 */
```

#### Observable concatenation
Соединение Observable (Observable concatenation) сначала рассылает все объекты из первого Observable, затем из второго и
так далее. Таким образом появляется четкий порядок следования. Соединение следует выбирать вместо слияния, когда 
необходим четкий порядок рассылки объектов из Observable.

Для этого так же существуют два способа: фабрика `Observable.concat()` и операнд `concatWith`. Если использовать 
бесконечный Observable, то это приведет к тому, что следующие Observable вызваны не будут. Бесконечный Observable при 
соединении стоит использовать последним. 

```java
Observable<String> src1 = Observable.just("Alpha", "Beta");
Observable<String> src2 = Observable.just("Zeta", "Eta");

Observable.concat(src1, src2).subscribe(i -> System.out.println("RECEIVED: " + i));
src1.concatWith(src2).subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: Alpha
   RECEIVED: Beta
   RECEIVED: Zeta
   RECEIVED: Eta
 */
```

`concatMap` очень похож на `flatMap` с той лишь разницей, что он гарантирует порядок появления новых Observable из-за 
использования механизима соединения вместо слияния.

```java
Observable.just("Alpha", "Beta", "Gamma")
        .concatMap(s -> Observable.fromArray(s.split("")))
        .subscribe(System.out::println);

/*
   A
   l
   p
   h
   a
   B
   ...
 */
```

#### Ambiguous фабрика
Ambiguous (неявная) фабрика `Observable.amb()` принимает на вход коллекцию Observable. Первый Observable, который
начнет рассылать объекты, станет источником. От всех остальных Observable фабрика отпишется. 

#### Zipping фабрика
Zipping фабрика позволяет объединять emission двух разных Observable в один. 

```java
Observable<String> src1 = Observable.just("Alpha", "Beta", "Gamma");
Observable<Integer> src2 = Observable.range(1, 6);
Observable.zip(src1, src2, (s, i) -> s + "-" + i).subscribe(System.out::println);

/*
    Alpha-1
    Beta-2
    Gamma-3
 */
```

#### groupBy операнд
`groupBy()` операнд позволяет сгруппировать рассылаемые объекты согласно переданной функции. Возвращает
`GroupedObservable<K, V>` который напоминает словарь, только из мира Observable. `GroupedObservable` что-то
среднее между горячим и холодным Observable. Он рассылает все объекты первому Observer, но второй и последующий 
не получат пропущенные объекты.

```java
Observable<String> source = Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon");
Observable<GroupedObservable<Integer, String>> byLengths = source.groupBy(s -> s.length());
byLengths.flatMapSingle(grp -> grp.toList()).subscribe(System.out::println);

/*
    [Beta]
    [Alpha, Gamma, Delta]
    [Epsilon]
 */
```

## Multicasting, Replaying, and Caching
#### Multicasting
Взглянем на следующий код:

```java
Observable<String> source = Observable.just("Alpha", "Beta", "Gamma");
source.subscribe(i -> System.out.println("Observer One: " + i));
source.subscribe(i -> System.out.println("Observer Two: " + i));

/*
    Observer One: 1
    Observer One: 2
    Observer One: 3
    Observer Two: 1
    Observer Two: 2
    Observer Two: 3   
 */
```

Observable сначала раздал значения в первого подписчика, дождался выполнения onComplete, затем раздал те же
значения второму подписчику и дождался onComplete. Выполнение кода оказалось линейным. Но что если попробовать 
сделать этот код параллельным и запихать каждого подписчика в свой поток?

Такого можно добиться при помощи `ConnectableObservable`. Он превратит холодный Observable в горячий и таким образом 
все данные уедут двум подписчикам одновременно. Это и называют multicasting (многоадресная рассылка).

```java
ConnectableObservable<String> source = Observable.just("Alpha", "Beta", "Gamma").publish();
source.subscribe(i -> System.out.println("Observer One: " + i));
source.subscribe(i -> System.out.println("Observer Two: " + i));
source.connect();

/*
    Observer One: 1
    Observer Two: 1
    Observer One: 2
    Observer Two: 2
    Observer One: 3
    Observer Two: 3   
 */
```

Все операторы, что объявлены в цепи до publish() выполняется один раз для всех подписчиков. В то же время все операторы 
после publish выполняются в разных потоках для каждого подписчика. В то же время для холодных Observable каждый оператор
будет выполняться заново для каждого onNext. 

Возможность отправлять одинаковые данные разных подписчикам является **главной фишкой multicasting**. Из-за того, что 
методы так же вызываются один раз, это может быть использовано для оптимизации. 

#### Caching
Во время многоадресной рассылки может быть полезным _кеширование_ данных от observable, чтобы новый observer мог 
получить какую-то историю рассылаемых объектов. Для этой цели служит оператор `cache()`. Он срабатывает в момент, когда 
у observable появляется первый подписчик.

cache() так же возвращает `ConnectableObservable` и горячий observable. Данный observable будет не только рассылать 
новые объекты по цепи, но и сохранять **все** объекты, которые он уже успел разослать. Таким образом стоит понимать, что
на бесконечных observable такой оператор просто выжрет всю память. 

Свежеподписанный observer сначала получит все объекты, которые он пропустил. Как только эти объекты будут обработаны, он 
станет получать свежие данные, как и все остальные observer.

```java
Observable<Long> ints = Observable.interval(1, TimeUnit.SECONDS).cache(); 

//Observer 1
ints.subscribe(l -> System.out.println("Observer 1: " + l));
sleep(3000);

//Observer 2
ints.subscribe(l -> System.out.println("Observer 2: " + l));
sleep(3000);

/*
    Observer 1: 0
    Observer 1: 1
    Observer 1: 2
    Observer 2: 0
    Observer 2: 1
    Observer 2: 2
    Observer 1: 3
    Observer 2: 3
    Observer 1: 4
    Observer 2: 4
    Observer 1: 5
    Observer 2: 5
 */
```

#### Replaying
Иногда хранить всю историю рассылаемых объектов может быть затратно или не нужно. В таком случае есть оператор `replay()`
который делает все то же самое, что и `cache()`, но у него есть несколько перегруженных методов.

`replay()` - аналогия `cache()`, но момент срабатывания больше не привязан к первому подписчику.  
`replay(int num)` - сохраняет только последние num рассылаемых объектов.  
`replay(int count, TimeUnit type)` - сохраняет только те объекты, которые были разосланы за последние count 
секунд/минут/etc. Тип времени выбирает по TimeUnit.

#### Subject
Subject - прокси объект, который необходим для связи между observable и observer. Наподобие event bus, он принимает 
значения от observable и передает их observer. Используют его в основном в тех случаях, когда у нас есть множество
observable, значения из которых необходимо объединить. Да, правильно для такого использовать `merge()`, но не всегда
это возможно, особенно когда observable используются из кода, доступа к которому вы не имеете (например библиотеки). Но 
лучше избегать возможности использования Subject, поскольку это считается антипаттерном в реактивном программировании.

```java
Observable<String> source1 = Observable.interval(1, TimeUnit.SECONDS)
        .map(l -> (l + 1) + " seconds");
Observable<String> source2 = Observable.interval(300, TimeUnit.MILLISECONDS)
        .map(l -> ((l + 1) * 300) + " milliseconds");

Subject<String> subject = PublishSubject.create();
subject.subscribe(System.out::println);

source1.subscribe(subject);
source2.subscribe(subject);
sleep(3000);

/*
    300 milliseconds
    600 milliseconds
    900 milliseconds
    1 seconds
    1200 milliseconds
    1500 milliseconds
    1800 milliseconds
    2 seconds
    2100 milliseconds
    2400 milliseconds
    2700 milliseconds
    3 seconds
    3000 milliseconds
 */
```

Цена такой гибкости использования Subject - сложность в контроле. Допустим у вас есть observable которые рассылают 
котиков, на этот observable подписан subject и рассылает сообщения observer. Если ваш subject утечет вовне (допустим
в клиентский код) и кто-то решит подписать ваш subject на observable, рассылающих собачек, то он сможет это сделать и 
сломает ваших observer, которые ждут исключительно котиков.

Еще одна проблема subject - он не потокобезопасен по умолчанию. Чтобы сделать его потокобезопасным, необходимо сделать
его сериализованным. Для этого есть метод `.toSerialized()`.