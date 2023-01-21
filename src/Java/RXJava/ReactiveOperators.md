# Реактивные операнды
## Комбинируем Observable
Операнды расмотренные в предыдущей главе полезны, поскольку позволяют управлять выбросами (emission). Но зачастую так 
же бывает необходимо комбинировать Observable. 

#### Объединение Observable
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

#### Соединение Observable
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

#### Неявная фабрика
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

#### Replaying
