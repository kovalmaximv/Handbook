- [Testing](#testing)
  - [TestObserver & TestSubscriber](#testobserver--testsubscriber)
  - [TestScheduler](#testscheduler)

# Testing 
## TestObserver & TestSubscriber
`TestObserver` и `TestSubscriber` представляют собой инструменты для юнит тестирования реактивных цепочек. 
`TestObserver` необходим для тестирования Observable, Maybe, Completable. `TestSubscriber` для тестирования Flowable.
По сути они представляют собой подписчиков с полезными assert методами для тестирования.

```java
Observable<Long> source =Observable.interval(1, TimeUnit.SECONDS).take(3);

//Declare TestObserver
TestObserver<Long> testObserver = new TestObserver<>();

//Assert no subscription has occurred yet
assertFalse(testObserver.hasSubscription());

//Subscribe TestObserver to source
source.subscribe(testObserver);

//Assert TestObserver is subscribed
assertTrue(testObserver.hasSubscription());

//Block and wait for Observable to terminate
testObserver.awaitTerminalEvent();
//Assert TestObserver called onComplete()
testObserver.assertComplete();
//Assert there were no errors
testObserver.assertNoErrors();
//Assert 3 values were received
testObserver.assertValueCount(3);
//Assert the received emissions were 0, 1, 2
testObserver.assertValues(0L, 1L, 2L);
```

## TestScheduler
Некоторые реактивные цепочки могут быть завязаны на время. Например, один emission раз в 10 минут. Мы не хотим ждать 
10 минут и для нас есть TestScheduler который позволяет пропустить определенное количество времени.

```java
TestScheduler testScheduler = new TestScheduler();
TestObserver<Long> testObserver = new TestObserver<>();

Observable<Long> minuteTicker = Observable.interval(1, TimeUnit.MINUTES, testScheduler);
minuteTicker.subscribe(testObserver);

//Fast forward by 30 seconds
testScheduler.advanceTimeBy(30, TimeUnit.SECONDS);
testObserver.assertValueCount(0);
```
