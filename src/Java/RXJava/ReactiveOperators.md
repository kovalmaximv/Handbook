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