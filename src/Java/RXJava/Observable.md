# Observer и Observable
В реактивном программировании одними из основных понятий являются `Observable` и `Observer`. По сути, Observable это сущность,
которая проталкивает (push) некоторые события (это называется выбросы или emissions). После эти выбросы читает Observer.

```java
import io.reactivex.rxjava3.core.Observable;

public class ObservableExample {
    public static void main(String[] args) {
        Observable<String> myStrings = Observable.just("Alpha", "Beta", "Gamma");
    }
}
```

Однако пример выше не имеет никакого смысла, кроме объявления Observable мы ничего не сделали. Чтобы что-то произошло,
нужно чтобы кто-то прочитал данные, которые предоставил Observable. Для этого существует Observer. Самый простой способ
создать Observer - задать ламбда функцию в методе `subscribe`.

```java
import io.reactivex.rxjava3.core.Observable;

public class ObservableExample {
    public static void main(String[] args) {
        Observable<String> myStrings = Observable.just("Alpha", "Beta", "Gamma");
        myStrings.subscribe(s -> System.out.println(s));
    }
}
```

// стр 24