# Тестирование конкурентных программ
Главная проблема при тестировании конкурентных программ заключается в том, что потенциальные сбои редко вероятные.

## Тестирование на правильность
#### Базовые модульные тесты
Самые базовые тесты для конкурентных программ аналогичны тем, которые использовались бы при последовательном выполнении.
Следует проверить инварианты, пограничные значения, необычный ввод и тд.

Включение набора последовательных тестов необходимо, так как они раскрывают проблемы _не_ связанные с конкурентностью.
Необходимо знать про такие проблемы до того, как вы собираетесь тестировать конкурентные проблемы выполнения.

#### Тестирование блокирующих операций
Если метод предположительно блокируется, то тест такого поведения должен падать только если поток продвинулся вперед.
Примерно такой же подход используется при тестировании выдачи исключений.

Тестирование блокирующегося метода вносит некоторые сложности: как только метод успешно заблокировался, нужно его 
разблокировать (чтобы тесты могли завершиться). Сделать это можно с помощью прерывания. Конечно это требует, чтобы 
тестируемый метод откликался на прерывание. Общий флоу будет такой:
1) Запустить блокирующийся метод в отдельном потоке.
2) Дождаться, пока метод заблокируется.
3) Прервать метод.
4) Проверить, что метод завершил работу.

Проблема может так же возникнуть в ожидании блокировки. Сколько времени необходимо ждать? На практике необходимо 
примерно подсчитать, сколько времени может выполняться метод. А затем ждать дольше, чем вы примерно посчитали.

Рассмотрим пример тестирования блокирующегося метода:
```java
void testBlockingMethod() {
    final var testedObject = new ObjectWithBlockedMethod();
    Thread taker = new Thread() {
        public void run() {
            try {
                testedObject.blockingMethod();
                fail(); // если мы добрались сюда, метод не заблокировался
            } catch (InterruptedException success) { }
        }
    };
    
    try {
        taker.start();
        Thread.sleep(TIMEOUT_BEFORE_BLOCKING);
        taker.interrupt();
        taker.join(TIMEOUT_BEFORE_INTERRUPT);
        assertFalse(taker.isAlive());
    } catch(Exception unexpected) {
        fail();
    }
}
```

Заманчиво использовать метод `Thread.getState` для верификации блокирования потока, однако этот метод не всегда 
работает верно. JVM может отправить поток в холостой цикл ожидания замка (спин-ожидания).

#### Тестирование на безопасность
Чтобы протестировать, что конкурентный класс работает правильно при непредсказуемом конкурентном доступе, нужно в 
тестах использовать многочисленные потоки (ничего умнее придумать не смогли) работающие с классом. После продолжительного
выполнения стоит проверить, что класс отработал верно, данные не повреждены, инварианты соблюдены.

Для паттерна Producer-Consumer (c общей очередью) достаточно проверить 2 вещи:
1) Все, что предоставил Producer, Consumer получил и обработал.
2) Ничего лишнего (то что Producer не присылал) Consumer не получил и не обработал.

Одна из реализаций такой проверки: вычисление контрольных сумм объектов, которые Producer поставляет, а Consumer 
позже получает. Работает все примерно так:
1) Когда Producer отправляет объект, вычисляем для объекта контрольную сумму. 
2) Вычисленную сумму суммируем с атомарной итоговой отправленной суммой.
3) Когда Consumer получает объект, вычисляем для объекта контрольную сумму.
4) Вычисленную сумму суммируем с атомарной итоговой полученной суммой.
5) Итоговая отправленная и полученная суммы должны быть равны.

Желательно, чтобы контрольные суммы менялись от прогона к прогону. Это необходимо, чтобы избежать оптимизаций 
компилятора и не получить ложно положительные результаты теста. Для этого достаточно внести случайность в генерацию 
контрольных сумм, чтобы от прогона к прогону они были разные.

Рассмотрим пример, где протестируем общую очередь паттерна Producer-Consumer:

```java
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;

public class Test {
    private static final ExecutorService pool;
    private final AtomicInteger produceSum;
    private final AtomicInteger consumeSum;
    private final CyclicBarrier cyclicBarrier;
    private final CommonWorkingQueue commonWorkingQueue;
    
    void test() {
        try {
            for (int i = 0; i < 10; i++) {
                pool.execute(new Producer());
                pool.execute(new Consumer());
            }
            cyclicBarrier.await(); // ждать, когда все потоки будут готовы
            cyclicBarrier.await(); // ждать, когда все потоки завершатся
            assertEquals(produceSum.get(), consumeSum.get());
        } catch (Exception e) {
            throw new RuntimeException();
        }
    }
}

class Producer implements Runnable {
    public void run() {
        try {
            int seed = (this.hashCode() ^ (int)System.nanoTime());
            int sum = 0;
            cyclicBarrier.await();
            for (int i = 10000; i > 0; i--) {
                commonWorkingQueue.put(seed);
                sum += seed;
                seed = randomlyChandeNumber(seed);
            }
            produceSum.getAndAdd(sum);
            cyclicBarrier.await();
        } catch (Exception e) {
            throw new RuntimeException();
        }
    }
}

class Consumer implements Runnable {
    public void run() {
        try {
            cyclicBarrier.await();
            int sum = 0;
            for (int i = 10000; i > 0; i--) {
                sum += commonWorkingQueue.take();
            }
            consumeSum.getAndAdd(sum);
            cyclicBarrier.await();
        } catch (Exception e) {
            throw new RuntimeException();
        }
    }
}
```

Если тест на безопасность провалился и произошла взаимная блокировка, то тест никогда не завершится. Чтобы исправить
эту ситуацию, необходимо побудить тесты завершаться с ошибкой спустя какое-то фиксированное, заранее просчитанное, 
время. Каждый такой сбой необходимо разбирать, чтобы понять, действительно ли произошла ошибка, или выполнение 
действительно затянулось.

## Предотвращение ошибок при тестировании производительности
Тестирование на производительность довольна проста:
1) Найти типичный сценарий использований класса.
2) Выполнять этот сценарий много раз.
3) Засечь время.

На практике следует избегать ряда ловушек, чтобы не получить ложные результаты производительности.

#### Сбор мусора

Сбор мусора непредсказуем, поэтому всегда есть вероятность, что сборщик мусора сработает во время измерений.

Существует две стратегии избежания данной проблемы. Одна из них - обеспечить, чтобы сборщик мусора вообще не работал 
во время тестов. Другой способ - обеспечить, чтобы сборщик мусора работал регулярно за время измерения.

#### Динамическая компиляция
Некоторые современные JVM могут заметить, что некоторый метод выполняется довольно часто и заменить байт-код этого 
метода на машинный код. Это ускоряет выполнение данного метода.

Избежать этой проблемы можно двумя путями. 

Первый - не измерять первые несколько минут прогона. Необходимо убедиться, что JVM перевела все необходимое в машинный
код, а затем начать измерение. Необходимое время ожидания можно рассчитать примерно.

Второй - измерять продолжительное время прогонов, чтобы получить среднее значение с учетом оптимизации JVM и без.

# Итоги
- Задача тестирования сложна, поскольку проблемы конкурентности маловероятны и зависят от множества факторов.
- Кроме того, тесты могут замаскировать проблемы конкурентности.
- Необходимо проводить тесты в приближенных сценариях выполнения. Для этого необходимо использовать многопоточность 
в тестировании.
- Скорее всего даже большое количество тестов не сможет отыскать все баги конкурентного кода.