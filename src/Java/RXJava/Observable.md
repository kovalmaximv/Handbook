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

Observable пропихивает по реактивной цепочке 3 типа события: `onNext`, `onComplete` и `onError`. При помощи onNext передается 
следующий элемент. onComplete сигнализирует, что новых элементов не будет. onError сигнализирует об ошибке. Все эти 
способы протолкнуть очередной элемент нужны для более тонкой настройки Observer.

## Способы создания Observable
#### Observable.create
В метод передается ламбда, принимающая Emitter. При помощи данного emitter вызывает onNext/onComplete/onError, чтобы
протолкнуть события по цепочке к Observer:

```java
import io.reactivex.rxjava3.core.Observable;

public class ObservableCreating {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(emitter -> {
            try {
                emitter.onNext("First emission");
                emitter.onNext("Second emission");
                emitter.onNext("Third emission");
                emitter.onComplete();
            } catch (Throwable e) {
                emitter.onError(e);
            }
        });

        source.subscribe(
                data -> onNextHandler(data),
                error -> onErrorHandler(error),
                () -> onCompletedHandler()
        );
    }
}
```

Стоит отметить, что `onNext` не посылает данные напрямую в Observer, а лишь передает данные дальше по реактивной 
цепочке. Между Observable и Observer может стоять большое количество различных преобразований. Составление таких 
цепочек является фишкой реактивного программирования. Флоу кода легко читать слева направо, снизу вверх. Почти как книга.


```java
import io.reactivex.rxjava3.core.Observable;

public class ObservableCreating {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(emitter -> {
                try {
                    emitter.onNext("First emission");
                    emitter.onNext("Second emission");
                    emitter.onNext("Third emission");
                    emitter.onComplete();
                } catch (Throwable e) {
                    emitter.onError(e);
                }
        });

        source.map(String::length)
                .map(length -> length * 2)
                .filter(i -> i >= 5)
                .subscribe(s -> System.out.println("RECEIVED: " + s));
    }
}
```

#### Observable.just