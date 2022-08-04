# Отмена и выключение

Иногда нам требуется останавливать задачи и потоки раньше, чем они завершатся сами. В Java нет механизма 
принудительной остановки работы потока, но есть механизм _прерывания_ (interruption) - кооперативный механизм, 
который позволяет одному потоку попросить другой прекратить действовать.

Мы редко хотим, чтобы поток останавливался немедленно и бросал свое состояние в противоречивом состоянии. Избежать 
этой проблемы помогает механизм прерывания. Этот механизм так же обеспечивает большую гибкость, потому что задачный 
код лучше знает, что нужно сделать перед отменой потока. Запрашивающий отмену код может не владеть такой информацией. 

В этой главе описываются механизмы отмены (cancellation) и прерывания (interruption) и способы кодирования задач и 
служб, откликающихся на запросы об отмене.

## Отмена задачи
В этом параграфе обсудим отмену Runnable и Callable задач.

Задача, задуманная как отменяемая, должна иметь политику отмены:
1) Как другой код может запросить отмену
2) Когда задача проверяет наличие запроса на отмену
3) Какие действия задача предпринимает на запрос об отмене

#### Прерывание
Каждый поток имеет флаг прерванности (interrupted status). Объект Thread содержит метод _interrupt_, прерывающий поток, 
метод _isInterrupted_, возвращающий флаг прерванности. Очистить статус прерванности можно статическим методом 
_interrupted_.
```java
public class Thread {
    public void interrupt() {...}
    public void isInterrupted() {...}
    public static boolean interrupted() {...}
}
```

> :exclamation: **Метод interrupt не побуждает целевой поток остановиться, он только доставляет сообщение с 
> просьбой остановки при удобной возможности.**

Блокирующие библиотечные методы (BlockingQueue.put, Thread.sleep, etc) быстро отзываются на прерывание, очищают
статус прерванности и выдают исключение InterruptedException. 

Очищение статуса прерванности следует использовать осторожно. Если вы не планируете проигнорировать прерывание, то 
нужно либо восстановить статус прерванности (вызвав метод interrupt), либо выкинуть исключение InterruptedException.

> :exclamation: **Прерывание является самым разумным способом отмены.**

Попробуем улучшить наш генератор простых чисел с использованием прерывания:

```java
public class PrimeGenerator implements Runnable {
    private final BlockingQueue<BigInteger> primes;

    public PrimeGenerator(BlockingQueue<BigInteger> primes) {
        this.primes = primes;
    }
    
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()) {
                primes.put(p.nextProbablePrime());
            } catch(InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public void cancel() {
        interrupt();
    }
}
```

В примере выше есть две точки обнаружения прерывания. Явная проверка в условии цикла не является строго необходимой, 
но позволяет произвести проверку _перед_ тяжелой операцией подсчета следующего простого числа. 

#### Политика прерывания
Потокам необходима политика прерывания, определяющая, как они интерпретируют запросы на прерывание.

Важно различать, как _задачи_ и _потоки_ должны откликаться на прерывание.

Код, который не является владельцем потока, должен сохранять статус прерванности. 
Таким образом, с прерыванием потока придется разбираться владельцу этого потока, а он наверняка знает лучше, 
как прервать этот поток.

Получается, что задача никак не может прервать поток. Она может или сохранить статус прерванности, проигнорировав его, 
либо выкинуть исключение InterruptedException. И в первом, и во втором случае с прерыванием будет разбираться владелец 
потока.

#### Обработка InterruptedException
Существуют две стратегии:
1) Распространение исключение выше по стеку (throws InterruptedException)
2) Восстановление статуса прерванности (Thread.currentThread().interrupt())

> :exclamation: **Нельзя проглатывать исключение, если ваш код не реализует политику прерывания для потока.**

Если по какой-то причине задача не поддерживает отмену, то можно вызывать блокирующий метод в цикле, повторяя 
попытки вызова. Главное не забыть вернуть статус прерванности в конце выполнения задачи:
```java
public Task getNextTask(BlockingQueue<Task> queue) {
    boolean interrupted = false;
    try {
        while (true) {
            try {
                return queue.take();
            } catch (InterruptedException e) {
                interrupted = true; // Проскочить и попытаться снова
            }
        }
    } finally {
        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Если ваш код не вызывает блокирующие методы, то он может отзываться на прерывание путем опроса статуса прерванности:
```java
if (Thread.currentThread().isInterrupted()) {...}
```

#### Отмена с помощью Future
Future позволяет управлять жизненным циклом задачи, попробуем написать отмену задачи с использованием Future.

Объект Future содержит метод cancel, который доставляет до задачи запрос на отмену. Аргумент в методе прерывает поток 
задачи (если она запущена). Future.cancel(true) можно вызывать при отмене задач, работающих в стандартном 
исполнителе Executor. 

Рассмотрим пример использования Future для отмены задач:
```java
public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
    Future<?> task = executor.submit(r);
    try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        // отменяем задачу в блоке finally
    } catch (ExecutionException e) {
        throw new SomeException();
    } finally {
        task.cancel(true);
    }
}
```

## Остановка поточного сервиса (Stopping thread-based services)
Приложение обычно создают сервисы, которые владеют потоками (напр. ExecutionService). Приложение не владеет потоками 
напрямую и не должно пытаться останавливать потоки напрямую. Вместо этого управление жизненным циклом потоков 
предоставляется сервису.

Сервис должен предоставлять методы жизненного цикла для выключения потоков. Например, сервис ExecutionService 
предоставляет методы shutdown и shutdownNow. 

> :exclamation: **Предоставляйте методы жизненного цикла, если владеющий потоком сервис живет дольше метода, 
> который его вызвал.**

Рассмотрим пример, напишем небольшой сервис логирования с примитивным способом отключения: 

```java
public class LogWriterService {
    private final BlockingQueue<String> queue;
    private final LoggerThread logger;
    private boolean shutDownRequested = false;

    public LogWriterService(Writer writer) {
        this.queue = new LinkedBlockingQueue<>();
        this.logger = new LoggerThread(writer);
    }

    public void start() {
        logger.start();
    }
    
    public void shutdown() {
        this.shutDownRequested = true;
    }

    public void log(String msg) throws InterruptedException {
        if (!this.shutDownRequested) {
            queue.put(msg);
        } else {
            throw new IllegalStateException("LogWriter disabled");
        }
    }

    private class LoggerThread extends Thread {
        private final PrintWriter writer;

        //...

        public void run() {
            try {
                while (true) {
                    writer.println(queue.take());
                }
            } catch (InterruptedException ignored) {
            } finally {
                writer.close();
            }
        }
    }
}
```
Однако такой подход содержит состояние гонки, присутствует подход "проверить, а затем действовать". Производители 
(producers) могут увидеть устаревшее состояние флага и отправить сообщение в очередь после ее выключения.

Решить такую проблему может наличие замка на методах проверки и обновлении флага:

```java
public class LogWriterService {
    private final BlockingQueue<String> queue;
    private final LoggerThread loggerThread;
    private boolean shutDownRequested = false;
    private int logsWaitingInQueue = 0;

    // ...

    public void start() {
        loggerThread.start();
    }

    public void stop() {
        synchronized (this) {
            shutDownRequested = true;
        }
        loggerThread.interrupt();
    }

    public void log(String msg) throws InterruptedException {
        synchronized (this) {
            if (shutDownRequested) {
                throw new IllegalStateException();
            }
            logsWaitingInQueue++;
        }
        
        queue.put(msg);
    }

    private class LoggerThread extends Thread {
        //...

        public void run() {
            try {
                while (true) {
                    try {
                        synchronized (this) {
                            if (shutDownRequested && logsWaitingInQueue == 0) {
                                break;
                            }
                        }
                        String msg = queue.take();
                        synchronized (this) {
                            logsWaitingInQueue--;
                        }
                        writer.println(msg);
                    } catch (InterruptedException e) {
                        // не делаем ничего, повторная попытка чтения
                    }
                }
            } finally {
                writer.close();
            }
        }
    }
}
```