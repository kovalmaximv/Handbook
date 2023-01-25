# RxJAva

1. [Введение](./Introduction.md)
2. [Observable&Observer](./Observable.md)
   - [Hot&Cold Observable](./Observable.md#hotcold-observable)
   - [ConnectableObservable](./Observable.md#connectableobservable)
   - [Single, Completable, Maybe](./Observable.md#single-completable-maybe)
   - [Disposable](./Observable.md#disposable)
3. [Базовые операторы](./BasicOperators.md)
   - [Conditional operators](./BasicOperators.md#conditional-operators)
     - [takeWhile/skipWhile](./BasicOperators.md#takewhileskipwhile)
     - [defaultIfEmpty](./BasicOperators.md#defaultifempty)
     - [switchIfEmpty](./BasicOperators.md#switchifempty)
   - [Suppressing operators](./BasicOperators.md#suppressing-operators)
     - [filter](./BasicOperators.md#filter)
     - [take](./BasicOperators.md#take)
     - [skip](./BasicOperators.md#skip)
     - [distinct](./BasicOperators.md#distinct)
     - [distinctUntilChanged](./BasicOperators.md#distinctuntilchanged)
     - [elementAt](./BasicOperators.md#elementat)
   - [Transforming operators](./BasicOperators.md#transforming-operators)
     - [map](./BasicOperators.md#map)
     - [cast](./BasicOperators.md#cast)
     - [startWithItem](./BasicOperators.md#startwithitem)
     - [sorted](./BasicOperators.md#sorted)
     - [scan](./BasicOperators.md#scan)
   - [Reducing operators](./BasicOperators.md#reducing-operators)
     - [count](./BasicOperators.md#count)
     - [reduce](./BasicOperators.md#reduce)
   - [Boolean operators](./BasicOperators.md#boolean-operators)
     - [all](./BasicOperators.md#all)
     - [any](./BasicOperators.md#any)
     - [isEmpty](./BasicOperators.md#isempty)
     - [contains](./BasicOperators.md#contains)
     - [sequenceEqual](./BasicOperators.md#sequenceequal)
   - [Collection operators](./BasicOperators.md#collection-operators)
     - [toList](./BasicOperators.md#tolist)
     - [toSortedList](./BasicOperators.md#tosortedlist)
     - [toMap](./BasicOperators.md#tomap)
     - [collect](./BasicOperators.md#collect)
   - [Error recovery operators](./BasicOperators.md#error-recovery-operators)
     - [onErrorReturnItem](./BasicOperators.md#onerrorreturnitem)
     - [onErrorResumeWith](./BasicOperators.md#onerrorresumewith)
     - [retry](./BasicOperators.md#retry)
   - [Action operators](./BasicOperators.md#action-operators)
     - [doOnNext, doOnComplete, doOnError](./BasicOperators.md#doonnext-dooncomplete-doonerror)
     - [doOnEach](./BasicOperators.md#dooneach)
     - [doOnSubscribe, doOnDispose](./BasicOperators.md#doonsubscribe-doondispose)
     - [doFinally](./BasicOperators.md#dofinally)
4. [Реактивные операторы](./ReactiveOperators.md)
   - [Комбинируем Observable](./ReactiveOperators.md#Комбинируем-Observable)
     - [Observable merging](./ReactiveOperators.md#Observable-merging)
     - [Observable concatenation](./ReactiveOperators.md#Observable-concatenation)
     - [Ambiguous фабрика](./ReactiveOperators.md#Ambiguous-фабрика)
     - [Zipping фабрика](./ReactiveOperators.md#Zipping-фабрика)
   - [groupBy операнд](./ReactiveOperators.md#groupby-)
     - [Multicasting, Replaying, and Caching](./ReactiveOperators.md#multicasting-replaying-and-caching)
     - [Multicasting](./ReactiveOperators.md#multicasting)
     - [Caching](./ReactiveOperators.md#caching)
     - [Replaying](./ReactiveOperators.md#replaying)
     - [Subject](./ReactiveOperators.md#subject)
   - WORK IN PROGRESS




## Полезные ссылки
1) Документация по операторам. https://reactivex.io/documentation/operators.html

## Литература
1) Learning RxJava. Nick Samoylov , Thomas Nield.

## Другие источники
1) https://itsobes.ru/AndroidSobes
2) https://reactivex.io/documentation