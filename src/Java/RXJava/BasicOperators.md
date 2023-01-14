# Базовые операнды
Для понимания работы RxJava помимо Observable/Observer необходимо знать основные реактивные операнды для построения 
реактивных цепочек.

1. [Условные операнды](#Условные-операнды)
   - [takeWhile/skipWhile](#takewhileskipwhile)
   - [defaultIfEmpty](#defaultifempty)
   - [switchIfEmpty](#switchifempty)
2. [Фильтрующие операнды](#Фильтрующие-операнды)
   - [filter](#filter)
   - [take](#take)
   - [skip](#skip)
   - [distinct](#distinct)
   - [distinctUntilChanged](#distinctuntilchanged)
   - [elementAt](#elementat)
3. [Преобразующие операнды](#Преобразующие-операнды)
   - [map](#map)
   - [cast](#cast)
   - [startWithItem](#startwithitem)
   - [sorted](#sorted)
4. 

## Условные операнды
Условные операнды позволяют пропускать или модифицировать Observable по условию.
#### takeWhile/skipWhile
`takeWhile()` пропускает объекты до тех пор, пока выполняется переданное условие. Как только появлятеся объект, для 
которого условие не выполняется, вызывает `onComplete`
```java
Observable.range(1, 100)
       .takeWhile(i -> i < 5)
       .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 1
   RECEIVED: 2
   RECEIVED: 3
   RECEIVED: 4
 */
```

`skipWhile()` игнорирует объекты до тех пор, пока выполняется условие
```java
Observable.range(1, 100)
        .skipWhile(i -> i <= 95)
        .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 96
   RECEIVED: 97
   RECEIVED: 98
   RECEIVED: 99
   RECEIVED: 100
 */
```

#### defaultIfEmpty
Если мы хотим привести Observable в Single, но Observable оказался пустым, можно использовать `defaultIfEmpty` который
подставит необходимое значение. Если Observable окажется не пустым, значение не подставится.
```java
Observable.just("Alpha", "Beta")
        .filter(s -> s.startsWith("Z"))
        .defaultIfEmpty("None")
        .subscribe(System.out::println); // None
```

#### switchIfEmpty
Похоже на работу defaultIfEmpty, только вместо значения указывается новый Observable, который может сгенерировать больше
одного значения.
```java
Observable.just("Alpha", "Beta", "Gamma")
        .filter(s -> s.startsWith("Z"))
        .switchIfEmpty(Observable.just("Zeta", "Eta", "Theta"))
        .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: Zeta
   RECEIVED: Eta
   RECEIVED: Theta
 */
```

## Фильтрующие операнды
Операнды, которые исключают из цепочки объекты, не подходящие под указанный критерий.

#### filter
`filter()` пропускает объекты удовлетворяющие условию переданному в предикате в качестве параметра.
```java
Observable.just("Alpha", "Beta", "Gamma")
           .filter(s -> s.length() != 5)
           .subscribe(s -> System.out.println("RECEIVED: " + s));

/*
   RECEIVED: Beta
 */
```

#### take
`take()` принимает количество объектов, которые он должен пропустить. После этого вызывает `onComplete`. Другая версия
принимает в качестве параметра временной промежуток, в течении которого пропускаются объекты.

```java
Observable.just("Alpha", "Beta", "Gamma")
      .take(2)
      .subscribe(s -> System.out.println("RECEIVED: " + s));

/*
   RECEIVED: Alpha
   RECEIVED: Beta
 */
```

#### skip 
`skip()` в противоположность take игнорирует указанное количество объектов, а следующие пропускает дальше по реактивной
цепочке.

```java
Observable.range(1, 100)
        .skip(95)
        .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 96
   RECEIVED: 97
   RECEIVED: 98
   RECEIVED: 99
   RECEIVED: 100
 */
```

#### distinct
`distinct()` пропускает только уникальные объекты, которые еще не были встречены ранее. Для сравнения использует equals
и hashcode. Стоит помнить, что если уникальных значений в реактивной цепочке много, данный операнд может использовать
некоторое количество памяти, так как хранит все полученные hashcode.

```java
Observable.just(3, 3, 5)
      .distinct()
      .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 3
   RECEIVED: 5
 */
```

#### distinctUntilChanged
`distinctUntilChanged()` не пропускает повторяющиеся значения, идущие подряд.

```java
Observable.just(1, 1, 1, 2, 2, 3, 3, 2, 1, 1)
        .distinctUntilChanged()
        .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 1
   RECEIVED: 2
   RECEIVED: 3
   RECEIVED: 2
   RECEIVED: 1
 */
```

#### elementAt
`elementAt()` принимает в качестве параметра индекс и пропускает только элемент с указанным индексом.

```java
Observable.just("Alpha", "Beta", "Zeta", "Eta", "Gamma")
      .elementAt(3)
      .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: Eta
 */
```

## Преобразующие операнды
Преобразуюзие операнды необходимы для изменения объекта в реактивной цепочке.

#### map
`map()` преобразовывает объект в цепочке. В том числе может изменить тип объекта. Преобразует один элемент в один 
элемент, если необходимо один элемент преобразовать в несколько, необходимо использовать другие операнды (будут ниже).

```java
Observable.just("1/3/2016", "5/9/2016", "10/12/2016")
        .map(s -> LocalDate.parse(s, dtf))
        .subscribe(i -> System.out.println("RECEIVED: " + i));

/*
   RECEIVED: 2016-01-03
   RECEIVED: 2016-05-09
   RECEIVED: 2016-10-12
 */
```

#### cast
`cast()` изменяет тип объекта. Вместо cast можно использовать и map, но cast лаконичнее и короче.

```java
Observable.just("Alpha", "Beta", "Gamma").cast(Object.class);
```

#### startWithItem
`startWithItem()` вставляет элемент в самое начало реактивной цепочки.

```java
Observable.just("Coffee", "Tea", "Espresso", "Latte")
        .startWithItem("COFFEE SHOP MENU")
        .subscribe(System.out::println);

/*
   COFFEE SHOP MENU
   Coffee
   Tea
   Espresso
   Latte
 */
```

Если необходимо вставить более одного объекта, есть метод `startWithArray()`.

#### sorted
Если Observable содержит конечный датасет, можно использовать операнд `sorted` для сортировки объектов реактивной цепочки.
Если использовать операнд на Observable с бесконечным количеством значений, получим ошибку OutOfMemoryError.

```java
Observable.just(6, 2, 5, 7, 1, 4, 9, 8, 3)
          .sorted()
          .subscribe(System.out::print); // 123456789
```

