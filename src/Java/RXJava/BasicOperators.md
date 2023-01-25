# Базовые операнды
Для понимания работы RxJava помимо Observable/Observer необходимо знать основные реактивные операнды для построения 
реактивных цепочек.

1. [Conditional operators](#conditional-operators)
   - [takeWhile/skipWhile](#takewhileskipwhile)
   - [defaultIfEmpty](#defaultifempty)
   - [switchIfEmpty](#switchifempty)
2. [Suppressing operators](#suppressing-operators)
   - [filter](#filter)
   - [take](#take)
   - [skip](#skip)
   - [distinct](#distinct)
   - [distinctUntilChanged](#distinctuntilchanged)
   - [elementAt](#elementat)
3. [Transforming operators](#transforming-operators)
   - [map](#map)
   - [cast](#cast)
   - [startWithItem](#startwithitem)
   - [sorted](#sorted)
   - [scan](#scan)
4. [Reducing operators](#reducing-operators)
   - [count](#count)
   - [reduce](#reduce)
5. [Boolean operators](#boolean-operators)
   - [all](#all)
   - [any](#any)
   - [isEmpty](#isempty)
   - [contains](#contains)
   - [sequenceEqual](#sequenceequal)
6. [Collection operators](#collection-operators)
   - [toList](#tolist)
   - [toSortedList](#tosortedlist)
   - [toMap](#tomap)
   - [collect](#collect)
7. [Error recovery operators](#error-recovery-operators)
   - [onErrorReturnItem](#onerrorreturnitem)
   - [onErrorResumeWith](#onerrorresumewith)
   - [retry](#retry)
8. [Action operators](#action-operators)
   - [doOnNext, doOnComplete, doOnError](#doonnext-dooncomplete-doonerror)
   - [doOnEach](#dooneach)
   - [doOnSubscribe, doOnDispose](#doonsubscribe-doondispose)
   - [doFinally](#dofinally)

## Conditional operators
Условные операнды (Conditional operators) позволяют пропускать или модифицировать Observable по условию.
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

## Suppressing operators
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

## Transforming operators
Преобразующие операнды (Transforming operators) необходимы для изменения объекта в реактивной цепочке.

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

#### scan
`scan()` применяет переданную функцию к полученному объекту, затем сохраняет полученный результат, как один из 
параметров для этой функции.

```java
Observable.just(5, 3, 7)
        .scan((accumulator, i) -> accumulator + i)
        .subscribe(s -> System.out.println("Received: " + s));

/*
   Received: 5
   Received: 8
   Received: 15
 */
```

## Reducing operators
Сокращающие операторы (Reducing operators) преобразуют несколько элементов в один. В некоторых случаях весь Observable приводится в Single.

#### count
`count()` возвращает количество элементов в реактивной цепочке. Нельзя использовать в цепочках с бесконечным 
количеством элементов, вместо этого стоит использовать scan.

```java
Observable.just("Alpha", "Beta", "Gamma")
      .count()
      .subscribe(s -> System.out.println("Received: " + s)); // Received: 3
```

#### reduce
`reduce()` идентичен в работе с scan, но возвращает только одно итоговое значение.

```java
Observable.just(5, 3, 7)
        .reduce((total, i) -> total + i)
        .subscribe(s -> System.out.println("Received: " + s)); // Received: 15
```

## Boolean operators
По сути это подтип сокращающих операндов, только логические операнды возвращают boolean значение исходя из переданной 
функции.

#### all
`all()` проверяет, что все элементы удовлетворяют заданному условию и возвращает `Single<Boolean>`. На пустом Observable
вернет true.

```java
Observable.just(5, 3, 7, 11, 2, 14)
         .all(i -> i < 10)
         .subscribe(s -> System.out.println("Received: " + s)); // Received: false
```

#### any
`any()` проверяет, что хотя бы один элемент из цепочки удовлетворяет заданному условию. Как только будет найден такой 
элемент, `any()` вернет Single<Boolean>. 

```java
Observable.just(5, 3, 7, 11, 2, 14)
         .any(i -> i > 10)
         .subscribe(s -> System.out.println("Received: " + s)); // Received: true
```

#### isEmpty
`isEmpty()` проверяет, будет ли Observable рассылать объекты. Возвращает true, если нет.

```java
Observable.just("One", "Two", "Three")
         .filter(s -> s.contains("z"))
         .isEmpty()
         .subscribe(s -> System.out.println("Received1: " + s)); // Received: true
```

#### contains
`contains()` проверяет, есть ли в цепочке определенный объект. Поиск происходит с помощью equals и hashcode.

```java
Observable.range(1, 10000)
         .contains(9563)
         .subscribe(s -> System.out.println("Received: " + s)); // Received: true
```

#### sequenceEqual
`sequenceEqual()` сравнивает несколько Observable и возвращает true, если они рассылают одинаковые объекты в одинаковой
последовательности. Проверка происходит с помощью equals и hashcode.

```java
Observable<String> obs1 = Observable.just("One","Two","Three");
Observable<String> obs2 = Observable.just("One","Two","Three");
Observable<String> obs3 = Observable.just("Two","One","Three");
Observable<String> obs4 = Observable.just("One","Two");

Observable.sequenceEqual(obs1, obs2)
        .subscribe(s -> System.out.println("Received: " + s)); // Received: true

Observable.sequenceEqual(obs1, obs3)
        .subscribe(s -> System.out.println("Received: " + s)); // Received: false

Observable.sequenceEqual(obs1, obs4)
        .subscribe(s -> System.out.println("Received: " + s)); // Received: false
```

## Collection operators
Собирающие операнды (Collection operators) предназначены для сбора Observable в коллекцию. Думаю, некоторые операнды не нуждаются в объяснении.

#### toList

#### toSortedList

#### toMap

#### collect
`collect()` - базовый операнд, на основе которого можно написать сбор в любую коллекцию. Для этого понадобятся следующие 
параметры `collect(Callable<U> initialValueSupplier, BiConsumer<U,T> collector)`. Например, не существует операнда toSet, 
можно написать свой, используя collect.

```java
Observable.just("Alpha", "Beta", "Gamma", "Beta")
         .collect(HashSet<String>::new, HashSet::add)
         .subscribe(s -> System.out.println("Received: " + s));
```

## Error recovery operators
В некоторых случаях мы хотим перехватить ошибки до того, как они дойдут до onError метода. Для таких случаев существуют
операнды восстановления ошибок.

#### onErrorReturnItem
`onErrorReturnItem()` перехватывает ошибку и вместо нее рассылает далее по цепочке переданный объект. Возникновение 
ошибки все равно останавливает рассылку объектов от Observable-источника.

```java
Observable.just(5, 2, 4, 0, 3)
         .map(i -> 10 / i)
         .onErrorReturnItem(-1)
         .subscribe(i -> System.out.println("RECEIVED: " + i),
         e -> System.out.println("RECEIVED ERROR: " + e));

/*
   RECEIVED: 2
   RECEIVED: 5
   RECEIVED: 2
   RECEIVED: -1
 */
```

Так же есть метод `onErrorReturn(Function<Throwable,T> valueSupplier)`. Переданная функция может в зависимости от ошибки
возвращать разные значения. 

```java
Observable.just(5, 2, 4, 0, 3)
         .map(i -> 10 / i)
         .onErrorReturn(e -> e instanceof ArithmeticException ? -1 : 0)
         .subscribe(i -> System.out.println("RECEIVED: " + i),
                    e -> System.out.println("RECEIVED ERROR: " + e));
```

#### onErrorResumeWith
`onErrorResumeWith` работает схоже с `onErrorReturnItem`, только передает дальше по цепочке новый Observable. Таким 
образом можно передать не один объект, а несколько.

```java
Observable.just(5, 2, 4, 0, 3)
         .map(i -> 10 / i)
         .onErrorResumeWith(Observable.just(-1).repeat(3))
         .subscribe(i -> System.out.println("RECEIVED: " + i),
                    e -> System.out.println("RECEIVED ERROR: " + e));

/*
   RECEIVED: 2
   RECEIVED: 5
   RECEIVED: 2
   RECEIVED: -1
   RECEIVED: -1
   RECEIVED: -1
 */
```

#### retry
`retry` переподписывается на Observable, чтобы повторить попытку выполнения для возможного избежания ошибки. Если не 
передавать аргумент (количество попыток), то количество попыток будет неограничено.

```java
Observable.just(5, 2, 4, 0, 3)
         .map(i -> 10 / i)
         .retry(2)
         .subscribe(i -> System.out.println("RECEIVED: " + i),
                    e -> System.out.println("RECEIVED ERROR: " + e));

/*
   RECEIVED: 2
   RECEIVED: 5
   RECEIVED: 2
   RECEIVED: 2
   RECEIVED: 5
   RECEIVED: 2
   RECEIVED: 2
   RECEIVED: 5
   RECEIVED: 2
   RECEIVED ERROR: java.lang.ArithmeticException: / by zero
 */
```

## Action operators
Операторы действия (Action operators) не модифицируют объекты и нужны для дебаггинга, просмотра содержимого цепочки или
второстепенных действий.

#### doOnNext, doOnComplete, doOnError
`doOnNext()` позволяет вызвать функцию с каждым элементом из цепочки перед тем, как пропустить этот элемент дальше. 
Данный операнд никак не модифицирует пересылаемый объект по цепочке. Он необходим только для вызова второстепенных 
действий по ходу выполнения этой цепочки. 

```java
Observable.just("Alpha", "Beta", "Gamma")
         .doOnNext(s -> System.out.println("Processing: " + s))
         .map(String::length)
         .subscribe(i -> System.out.println("Received: " + i));

/*
   Processing: Alpha
   Received: 5
   Processing: Beta
   Received: 4
   Processing: Gamma
   Received: 5
 */
```

`doOnComplete()` схож по действию с `doOnNext()`, но вызывается после отправки последнего элемента, перед вызовом 
onComplete.

```java
Observable.just("Alpha", "Beta", "Gamma")
         .doOnComplete(() -> System.out.println("Source is done emitting!"))
         .map(String::length)
         .subscribe(i -> System.out.println("Received: " + i));

        
/*
   Received: 5
   Received: 4
   Received: 5
   Source is done emitting! 
 */
```

По аналогии работает `doOnError`.

#### doOnEach
`doOnEach` похож на doOnNext, только поступающий объект оборачивается в тип Notification, который хранит тип 
произошедшего события [onNext, onError, onComplete].

```java
Observable.just("One", "Two", "Three")
         .doOnEach(s -> System.out.println("doOnEach: " + s))
         .subscribe(i -> System.out.println("Received: " + i));

/*
   doOnEach: OnNextNotification[One]
   Received: One
   doOnEach: OnNextNotification[Two]
   Received: Two
   doOnEach: OnNextNotification[Three]
   Received: Three
   doOnEach: OnCompleteNotification
 */
```

#### doOnSubscribe, doOnDispose
`doOnSubscribe(Consumer<Disposable> onSubscribe)` вызывается в момент, когда на Observable происходит подписка. Внутри
метода будет доступ к Disposable, так что подпиской можно будет управлять.

В противовес `doOnDispose()` выполняется, когда происходит отписка от Observable.

#### doFinally
`doFinnaly` вызывается после `onComplete`, `onError` или после вызова `Disposable.dispose()`. Таким образом, можно 
провести финальные действия с реактивной цепочкой. 

```java
Observable.just("One", "Two", "Three")
         .doFinally(() -> System.out.println("doFinally!"))
         .subscribe(i -> System.out.println("Received: " + i));

/*
   Received: Two
   Received: Three
   doFinally!
 */
```