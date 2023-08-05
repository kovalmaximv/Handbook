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
3. [Concurrency](#concurrency)
   - [Schedulers](#concurrency)
     - [Computation](#computation)
     - [I/O](#io)
     - [New thread](#new-thread)
     - [Single](#single)
     - [ExecutorService](#executorservice)
   - [Parallelization](#parallelization)
4. [Switching, Throttling, Windowing, and Buffering](#switching-throttling-windowing-and-buffering)
   - [Buffering](#buffering)
   - [Windowing](#windowing)
   - [Throttling](#throttling)
   - [Switching](#switching)
5. [Flowable and Backpressure](#flowable-and-backpressure)
   - [Flowable](#flowable)
   - [onBackpressureXXX](#onbackpressurexxx)
6. [Transformers and Custom operators](#transformers-and-custom-operators)
    - [to()](#to)
    - [compose()](#lift)
    - [lift()](#lift)


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

## Concurrency
Для понимания этой главы необходимо иметь минимальные познания в области многопоточного программирования.

Многопоточка в RxJava реализована при помощи пула потоков. Если в Java для этого используется `ExecutorService`, то в 
RxJava эту же задачу решает `Scheduler`. Чтобы указать `observable`, какой `scheduler` использовать 
(и использовать ли вообще) вызывается метод `Observable.subscribeOn(Scheduler scheduler)`. По умолчанию, некоторые 
фабрики `observable` уже поставляются с `scheduler` на борту. Например `Observable.fromCallable()` использует под 
капотом I/O `scheduler`. И этот `scheduler` **нельзя** переписать, если вызвать метод `subscribeOn()` еще раз.

Если необходимо сменить тип `scheduler` по ходу реактивной цепи, необходимо использовать метод 
`Observable.observeOn(Scheduler anotherScheduler)`. Данный метод перехватывает все объекты в цепи и меняет поток 
выполнения исходя из нового `scheduler`.

Есть несколько типов `Scheduler`, каждый под свой тип задач:

#### Computation
`Schedulers.computation()` выделяет фиксированное количество потоков под выполнение задач. Количество потоков 
определяется автоматически исходя из доступных ресурсов. Лучше всего подходит под какие-то долгие вычисления, когда
исполняющий поток не простаивает.

```java
Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon")
        .subscribeOn(Schedulers.computation())
        // do whatever you want
```

#### I/O
`Schedulers.io()` выделяет динамическое число поток для выполнения задач. Когда какой-то поток закончил выполнение и
долго не переиспользовался, он удаляется. При необходимости, когда нет свободных потоков, создаются новые. Лучше
всего подходит под задачи записи/чтения, когда потоки нагружаются неравномерно из-за ожиданий ввода/вывода.

```java
Observable<String> customerNames = db.select("select name from customer")
        .getAs(String.class)
        .subscribeOn(Schedulers.io())
        // do whatever you want
```

#### New thread
`Schedulers.newThread()` не создает пул потоков, вместо этого создает новый поток для каждого observer и удаляет поток, 
когда он больше не нужен.

```java
Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon")
        .subscribeOn(Schedulers.newThread())
        // do whatever you want
```

#### Single
`Schedulers.single()` выполняет все операции в одном потоке. Может быть полезно для изоляции сегмента, содержащего не 
потокобезопасные данные.

```java
Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon")
        .subscribeOn(Schedulers.single())
        // do whatever you want
```

#### ExecutorService
`Scheduler` так же возможно создать из `ExecutorService`. Это поможет более гибко настраивать пул потоков для RxJava.

```java
ExecutorService executor = Executors.newFixedThreadPool(20);
Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon")
        .subscribeOn(Schedulers.from(executor))
        .doFinally(executor::shutdown)
        .subscribe(System.out::println);
```

## Parallelization
В предыдущей главе мы разобрались, как конкурентно запустить выполнение нескольких реактивных цепочек 
`observable-observer`. Но что если некоторый `observer` поставляет 1000 элементов и мы не хотим, чтобы все они были
обработаны последовательно. Можно ведь разбить выполнение на 10 потоков и обработать всего 100 элементо в каждом потоке.

Все необходимое для этого мы уже знаем, нам нужно использовать `flatMap` и `subscribeOn`.

```java
Observable.range(1, 10)
        .flatMap(i -> Observable.just(i)
            .subscribeOn(Schedulers.computation())
            .map(i2 -> intenseCalculation(i2)))
        .subscribe(i -> System.out.println("Received " + i));
```

Таким образом каждый элемент обработается в своем потоке. Если мы все же хотим разбить по несколько элементов на один
поток, необходимо использовать `groupBy`.

```java
int coreCount = Runtime.getRuntime().availableProcessors();
AtomicInteger assigner = new AtomicInteger(0);
Observable.range(1, 10)
        .groupBy(i -> assigner.incrementAndGet() % coreCount)
        .flatMap(grp -> grp.observeOn(Schedulers.io())
        .map(i2 -> intenseCalculation(i2)))
        .subscribe(i -> System.out.println("Received " + i));
```

## Switching, Throttling, Windowing, and Buffering
В некоторых ситуация observable генерирует больше данных, чем успевает поглотить observer. Такие ситуации помогут 
решить Switching, Throttling, Windowing, и Buffering

#### Buffering
Идея `buffering()` в том, чтобы объединить рассылаемые объекты в список и обработать этот список целиком позже. Возможно
весь список обработается быстрее, чем каждый элемент по отдельности. Есть несколько перегруженных методов:

`buffering(int count)` - копит count элементов в списке и затем отсылает этот список дальше по цепи.  
`buffering(int count, int skip)` - копит count элементов, отсылает этот список дальше по цепи, затем пропускает (skip - count) элементов.  
`buffer(int count, TimeUnit unit)` - копит элементы в списке count секунд/минут/etc.  
`buffer(Observable<B> boundary)` - копит элементы в списке до тех пор, пока не произойдет emission в `Observable boundary`.  

И небольшой пример использования:
```java
Observable.range(1, 10)
        .buffer(2, 3)
        .subscribe(System.out::println);

/*
    [1, 2]
    [4, 5]
    [7, 8]
    [10]
 */
```

#### Windowing
Идея `window()` полностью аналогично `buffering()` с той лишь разницей, что вместо списка рассылаемые объекты копятся
в другом observable. Перегруженные методы `window()` тоже аналогичны `buffering()`.

```java
Observable.interval(300, TimeUnit.MILLISECONDS)
        .map(i -> (i + 1) * 300)
        .window(1, TimeUnit.SECONDS)
        .flatMapSingle(obs -> obs.reduce("", (total, next) -> total + (total.equals("") ? "" : "|") + next))
        .subscribe(System.out::println);

/*
    300|600|900
    1200|1500|1800
    2100|2400|2700
    3000|3300|3600|3900
    4200|4500|4800
 */
```

#### Throttling
`throttling()` предназначен для того, чтобы принимать только одно значение в реактивной цепи за определенный промежуток 
времени. Например, если какое-то значение из цепи достаточно обработать лишь один раз или когда пользователь много раз 
тыкает на кнопку. Существуют следующие перегруженные версии оператора:

`throttleLast(long intervalDuration, TimeUnit unit)` - рассылает только последний полученный элемент, встреченный за 
указанный промужеток времени.  
`throttleFirst(long intervalDuration, TimeUnit unit)` - в противоположность throttleLast рассылает первый полученный 
элемент.  
`throttleWithTimeout(long intervalDuration, TimeUnit unit) or debounce(..)` - похож на throttleLast, но промежуток 
времени не статичен и обновляется каждый раз при получении объекта.  

#### Switching
Оператор `switchMap` отписывается от предыдущего источника и прекращает обработку объекта всякий раз, когда получает 
новый рассылаемый объект.

```java
Observable<String> items = Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon", "Zeta");

// Ставим задержку между элементами
Observable<String> processStrings = items.concatMap(s -> Observable.just(s)
        .delay(randomSleepTime(), TimeUnit.MILLISECONDS));

Observable.interval(5, TimeUnit.SECONDS)
        .switchMap(i -> processStrings)
        .subscribe(System.out::println);

/*
    Alpha
    Beta
    Gamma
    Delta
    Epsilon
    Zeta
    // Прошло 5 секунд, произошел dispose, обрабатываем элементы заного
    Alpha
    Beta
    Gamma
    // Прошло 5 секунд, произошел dispose, обрабатываем элементы заного
    Alpha
    Beta
    Gamma
    Delta
    // Прошло 5 секунд, произошел dispose, обрабатываем элементы заного
 */
```

## Flowable and Backpressure
Вспомним про проблему, когда observable поставляет данные быстрее, чем observer их поглощает. В предыдущей главе мы 
пытались решить эту проблему на стороне observable, но на деле это не всегда представляется возможным. 

Отмечу, что такая проблема в целом не появится при однопоточном выполнении. Посколько всего один поток выполняет 
реактивную цепочку, observable просто не будет владеть потоком до тех пор, пока observer не обработает рассылаемый 
объект. Однако когда мы используем многопоточную обработку с помощью `subscribeOn`, то observable отдаст emission в 
`scheduler` и он продолжит владеть потоком выполнения, так что сможет выслать следующий объект. И так большое 
количество раз.

```java
class MyItem {
    final int id;
    MyItem(int id) {
        this.id = id;
        System.out.println("Constructing MyItem " + id);
    }
}

Observable.range(1, 999_999_999)
        .map(MyItem::new)
        .observeOn(Schedulers.io())
        .subscribe(myItem -> {
            sleep(50);
            System.out.println("Received MyItem " + myItem.id);
        });

/*
    ...
    Constructing MyItem 1001899
    Constructing MyItem 1001900
    Constructing MyItem 1001901
    Constructing MyItem 1001902
    Received MyItem 38
    Constructing MyItem 1001903
    Constructing MyItem 1001904
    Constructing MyItem 1001905
    Constructing MyItem 1001906
    Constructing MyItem 1001907
    ..
 */
```

#### Flowable
Основная идея в том, чтобы как-то обработать объекты, которые observer не успевает обработать. Для этого существуют 
BackpressureStrategy, о них поговорим ниже.

`Flowable` является аналогом `Observable` и поддерживает почти все те же операции, что и `Observable`, так же его можно 
трансформировать в `Observable` и обратно.

```java
Flowable.range(1, 999_999_999)
        .map(MyItem::new)
        .observeOn(Schedulers.io())
        .subscribe(myItem -> {
            sleep(50);
            System.out.println("Received MyItem " + myItem.id);
        });

/*
    Constructing MyItem 1
    Constructing MyItem 2
    Constructing MyItem 3
    ...
    Constructing MyItem 127
    Constructing MyItem 128
    Received MyItem 1
    Received MyItem 2
    Received MyItem 3
    ...
    Received MyItem 95
    Received MyItem 96
    Constructing MyItem 129
    Constructing MyItem 130
    Constructing MyItem 131
    ...
    Constructing MyItem 223
    Constructing MyItem 224
    Received MyItem 97
    Received MyItem 98
    Received MyItem 99
    ...
 */
```

Вместо `observer` flowable использует `subscriber`. `Subscriber` отличается лишь тем, что вместо метода `dispose` у него
метод `cancel` и при помощи метода `request()` можно настраивать, пачку данных какого размера он будет обрабатывать.

Создать flowable можно при помощи метода `Flowable.create(Func emitter, BackpressureStrategy strategy)` или 
`toFlowable(BackpressureStrategy strategy)` и `Flowable.generate()`. BackpressureStrategy это объект, реализующий 
логику обработки объектов, которые observer не успевает обработать. Существуют следующие BackpressureStrategy:
1) BUFFER - emission сначала попадает в очередь, а из нее в observer. Нужно быть аккуратным, поскольку может привести 
к OutOfMemory.
2) DROP - если observer не успевает обрабатывать объекты, то emission просто игнорируется. Поведение продолжается до тех
пор, пока observer не будет готов взять новый emission.
3) LATEST - держит только самый последний полученный emission и отдает его в observer по требованию.
4) MISSING - пустышка вместо BackpressureStrategy, вместо этого необходимую логику BackpressureStrategy нужно 
будет применить позже при помощи метода `onBackpressureXXX()`. О методе поговорим ниже.
5) ERROR - Генерирует `MissingBackpressureException` когда observer не успевает обрабатывать emission.

#### onBackpressureXXX
`onBackPressureBuffer()` берет существующий `Flowable` и применяет к нему логику BackpressureStrategy.BUFFER с момента
вызова метода. Так же есть перегруженные версии метода, с интересными параметрами. `capacity` устанавливает максимально
возможную размерность очереди для хранения. `onOverflow` позволяет передать лямбду, которая сработает, когда очередь 
достигнет предельного значения. Так же можно передать стратегия (BackpressureOverflowStrategy) поведения при увеличении 
размера очереди. Всего их 3: ERROR - выкидывает ошибку, DROP_OLDEST - удаляет самые старые значения из очереди, 
DROP_LATEST - удаляет самые новые. 

```java
Flowable.interval(1, TimeUnit.MILLISECONDS)
        .onBackpressureBuffer()
        .observeOn(Schedulers.io())
        .subscribe(i -> {
            sleep(5);
            System.out.println(i);
        });

/*
    0 
    1 
    2 
    3 
    4
    ...
 */
```

`onBackPressureLatest()` по аналогии с предыдущим методом реализует логику BackpressureStrategy.LATEST.

`onBackPressureDrop()` реализует логику BackpressureStrategy.DROP. В метод можно передать лямбду, которая будет 
обрабатывать удаленные объекты.

## Transformers and Custom operators
#### to()
В некоторых случаях нам необходимо observable передать в метод. Чтобы поддерживать стилистику 'слева направо, сверху
вниз' в RxJava есть специальный оператор `to()` позволяющий передать текущий observable в указанную ссылку на метод.

```java
// Было
Observable seconds = Observable.interval(1, TimeUnit.SECONDS)
        .map(i -> i.toString());
SomeClass.someVoidMethod(seconds);

// Стало
Observable.interval(1, TimeUnit.SECONDS)
        .map(i -> i.toString())
        .to(SomeClass::someVoidMethod);
```

#### compose() 
Если в реактивных цепочках часто встречается некоторая последовательность операторов, то ее можно выделить в отдельный
объект и переиспользовать. Очень удобный способ избавления от копипасты, который продумали разработчики RxJava. Для
этой повторяемая часть цепочки помещается в объект `ObservableTransformer<from, to>`, `FlowableTransformer<from, to>` 
или версии для `Single`, `Maybe`, `Completable`. Затем при помощи оператора `compose()` вызывается необходимая часть 
цепочки при необходимости. 

Стоит аккуратно вводить состояние (например некоторые переменные) в такой объект и еще более аккуратно
разделять это состояние между разными подписчиками. Тут легко можно будет сделать ошибку и сложно потом эту ошибку 
обнаружить и пофиксить. В целом лучше избегать использования общего состояния.


```java
<T> ObservableTransformer<T, ImmutableList<T>> toImmutableList() {
        return up -> up.collect(ImmutableList::<T> builder, ImmutableList.Builder::add)
                            .map(ImmutableList.Builder::build)
                            .toObservable();
}

// ...

Observable.just("Alpha", "Beta", "Gamma", "Delta", "Epsilon")
        .compose(toImmutableList())
        .subscribe(System.out::println);
```

#### lift()
В RxJava можно реализовать свой оператор. Для этого используется интерфейс 
`ObservableOperator<DownstreamEmission, UpstreamEmission>`, `FlowableOperator<DownstreamEmission, UpstreamEmission>` и 
версии для `Single`, `Maybe`, `Completable`.
В этих интерфейсах реализуются методы `onNext`, `onError` и `onComplete`. Затем созданный объект используется
при помощи оператора `lift()`.

Подходить к созданию своего оператора нужно с осторожностью и лучше использовать этот вариант, когда все остальные не
сработали. Но 99% рабочих случаев должны покрывать обычные операторы, ObservableTransformer или библиотеки с готовыми
кастомнмыми операторами от надежных источников (такие как RxJavaExtensions и rxjava2-extras).

```java
/*
   кастомный оператор, который выполняет переданное действие, если 
   не было ни одного emission     
 */
 */
public static <T> ObservableOperator<T,T> doOnEmpty(Action action) {
     return observer -> new DisposableObserver<T>() {
         boolean isEmpty = true;
         
         @Override 
         public void onNext(T value) {
             isEmpty = false;
             observer.onNext(value);
         }
         
         @Override 
         public void onError(Throwable t) {
             observer.onError(t);
         }
         
         @Override 
         public void onComplete() {
             if (isEmpty) {
                 try {
                     action.run();
                 } catch (Exception e) {
                     onError(e);
                     return;
                 }
             }
             observer.onComplete();
         }
     };
}

//...

Observable.range(1, 5)
        .lift(doOnEmpty(() -> System.out.println("Operation 1 Empty!")))
        .subscribe(v -> System.out.println("Operation 1: " + v));
```

При реализации своего оператора есть 3 контракта:
1) Не вызывать `onComplete()` после `onError()`
2) Не вызывать `onNext()` после `onComplete()` или `onError()`
3) Не вызывать ничего после disposal

