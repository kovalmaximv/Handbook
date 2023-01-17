# Реактивные операнды
## Комбинируем Observable
Операнды расмотренные в предыдущей главе полезны, поскольку позволяют управлять выбросами (emission). Но зачастую так 
же бывает необходимо комбинировать Observable. 

#### Объединение Observable
Для объединения нескольких Observable существует два способа: фабрика `Observable.merge` и операнд `mergeWith`. Данный
метод может перемешивать значения горячих Observable. В целом лучше не надеяться на очевидный порядок следования 
элементов при мерже (особенно в многопоточных системах). Для этих целей лучше использовать `Observable.concat`. 

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

#### flatMap