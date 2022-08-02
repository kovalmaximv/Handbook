# Выполнение задач
Большинство конкурентных приложений выполняют некоторые параллельные задачи. Разделение работы приложения на 
задачи упрощает проектирование и структуру многопоточных приложений.

## Выполнение задач в потоках
Первое, что стоит сделать, четко определить границы задач. Независимые друг от друга задачи могут выполняться 
параллельно. 

Большинство серверных приложений в качестве границы одной задачи предлагают индивидуальные клиентские запросы. Сервер
принимает запрос от удаленных клиентов, обрабатывают их одновременно и независимо друг от друга.

В этой главе мы будем в качестве примера разрабатывать каркас веб сервера. Рассмотрим пример последовательной 
обработки запроса

```java
import java.net.ServerSocket;

class SingleThreadServer {
    public static void main(String[] args) {
        ServerSocket server = new ServerSocket(80);
        while (true) {
            Socket connection = server.accept();
            handleRequest(connection);
        }
    }
}
```

#### Явное создание потоков для задач
Сделаем наш веб сервер многопоточным

```java
class ThreadPerTaskWebServer {
    public static void main(String[] args) {
        ServerSocket server = new ServerSocket(80);
        while (true) {
            final Socket connection = server.accept();
            Runnable task = () -> handleRequest(connection);
            new Thread(task).start();
        }
    }
}
```

Главный поток принимает новый запросы и создает под каждый запрос отдельный поток, который обработает этот запрос. 
Благодаря этому:
- Главный цикл может принимать новый запрос до того, как обработается старый
- Задачи обрабатываются параллельно, увеличивая пропускную способность
- Возрастает необходимость потокбезопасности кода

При этом такой подход имеет свои недостатки:

- Создание потоков занимает время и мощности, гораздо выгоднее переиспользовать уже созданные потоки
- Существует лимит на кол-во создаваемых потоков

Сначала увеличения числа потоков повышает пропускную способность, но начиная с определенного момента большое кол-во 
потоков начинает замедлять работу приложения и может привести к сбоям.

#### Executor
Ряд преимуществ предоставляют поточные пулы (thread pools), Java предоставляет их гибкую реализацию в классе Executor.

Основным классом для выполнения задач является не Thread, а Executor:
```java
public interface Executor {
    void execute(Runnable command);
}
```
Таким образом, Executor разделяет предоставление задачи (Executor) от ее выполнения (Runnable).
Отделение предоставления задачи от ее выполнения позволяет легко определить и изменить политику выполнения для 
выбранного класса задач. 

Спецификация политики выполнения отвечает на вопросы:
1) В каком потоке будут выполняться задачи
2) Какой порядок выполнения задач
3) Сколько задач может выполняться конкурентно
4) Сколько задач может быть поставлено в очередь
5) Как система должна себя вести при перегрузке


Попробуем написать веб сервер с использованием Executor
```java
class TaskExecutionWebServer {
    private static final int NUMBER_THREADS = 100;
    private static final Executor executor = Executors.newFixedThreadPool(NUMBER_THREADS);

    public static void main(String[] args) {
        ServerSocket server = new ServerSocket(80);
        while (true) {
            final Socket connection = server.accept();
            Runnable task = () -> handleRequest(connection);
            executor.execute(task);
        }
    }
}
```

Сервер использует Executor с ограниченным пулом рабочих потоков. Предоставление задачи с помощью execute добавляет 
задачу в рабочую очередь, а рабочие непрерывно выполняют задачи, удаляя их из очереди.

#### Пулы потоков
Пул потоков тесно привязан к рабочей очереди (_work queue_), содержащей задачи, необходимые для выполнения. Рабочие 
потоки берут задачу из очереди, выполняют ее и возвращаются к ожиданию следующей задачи из очереди.

Многоразовое использование существующего потока избавляет от затрат на создание отдельных потоков. Правильно настроив
размер пула потоков можно равномерно загрузить процессор и сохранить ресурсы.

Библиотека Executors в Java позволяет гибко настроить пул потоков:
1) **newFixedThreadPool** - пул потоков фиксированного размера. Задает максимум потоков, которые могут быть 
использованы для обработки задач. Если поток был закрыт (например из-за ошибки), будет создан новый поток. Поток в пуле 
будет жить, пока не будет вызван shutdown.
2) **newCachedThreadPool** - пул использует свободный поток для выполнения задачи, если свободного потока нет, создает 
новый и добавляет в пул. Неиспользуемые (в течении 60 сек) потоки закрываются и удаляются из пула. 
Данный пул потоков не накладывает ограничение на максимальное кол-во потоков в пуле.
3) **newSingleThreadPool** - пул использует один поток для выполнения задач, задачи выполняются строго последовательно в 
порядке добавления. Если поток из-за ошибки в задаче был завершен, создастся новый поток. Отличие от 
newFixedThreadPool(1) в том, что newSingleThreadPool гарантированно не может быть переконфигурирован для использования
дополнительных потоков.
4) **newScheduledThreadPool** - пул фиксированного размера, который поддерживает отложенный старт или периодическое
выполнения задач. 

newFixedThreadPool и newCachedThreadPool возвращают ThreadPoolExecutor, который может быть дополнительно конфигурирован.  

#### Завершение работы Executor
JVM ждет окончания всех процессов, поэтому если Executor по какой-то причине не может завершиться, это не позволяет
JVM завершить работу.

Интерфейс ExecutorService расширяет интерфейс Executor добавляя возможность управления жизненным циклом:

```java
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit);
    // etc
}
```

Жизненный цикл имеет 3 состояния: работает, выключается и терминирован.

Метод shutdown инициализирует завершение работы ExecutorService. Новые задачи, приходящие в submit отправляются в 
очередь отмененных задач, однако уже принятые задачи будут ожидать выполнения. После завершения всех задач из очереди, 
ExecutorService переводится в терминированный статус.

Метод shutdownNow пытается отменить невыполненные задачи и не берет в работу задачи из очереди ожидания.

Попробуем дописать наш веб сервер с поддержкой жизненного цикла веб сервера:

```java
class LifecycleWebServer {
    private static final int NUMBER_THREADS = 100;
    private static final Executor executor = Executors.newFixedThreadPool(NUMBER_THREADS);
    
    public void start() {
        ServerSocket server = new ServerSocket(80);
        while (!executor.isShutdown()) {
            final Socket connection = server.accept();
            executor.execute(() -> handleRequest(connection));
        }
    }
    
    public void stop() {
        executor.shutdown();
    }
}
```

#### Отложенные и периодические задачи
Класс ScheduledThreadPoolExecutor выполняет отложенные и периодические задачи.

Исполнитель ScheduledThreadPoolExecutor заменяет проблемный класс Timer, у которого был целый пласт проблем.

## Поиск пригодной степени параллелизма
Чтобы использовать Executor, вы должны описать свою задачу как Runnable. В большинстве серверных приложений 
существует отличная граница: один клиентский запрос. 

Но иногда хорошие границы не так очевидны. В некоторых случаях может существовать задача в рамках запроса, 
которую тоже неплохо бы выполнить в отдельном потоке.

Рассмотрим пример, в котором будем реализовывать отрисовщик html страницы. В этом примере рассмотрим степень проработки
параллельности одного клиентского запроса. Начнем с простого подхода 1 запрос - 1 поток, с каждой итерацией будем 
прорабатывать паралеллизм. 

#### Последовательный подход
Будем отрсисовывать сначала текстовые элементы, а затем изображения:

```java
public class SingleThreadRenderer {
    void renderPage(String source) {
        renderText(source);
        
        List<ImageData> imageDatas = new ArrayList<ImageData>();
        for (ImageInfo imageInfo: scanForImageInfo(source)) {
            imageDatas.add(imageInfo.downloadImage());
        }
        for(ImageData data: imageDatas) {
            renderImage(data);
        }
    }
}
```
Скачивание изображения, чтение данных, запись данных: все это в основном включает в себя ожидание завершения 
ввода-вывода. Поэтому последовательный подход может недостаточно использовать мощности процессора. Попробуем добавить 
паралеллизм нашему классу.

#### Использование Callable и Future
Executor использует интерфейс Runnable для выполнения задач. Однако данный интерфейс неспособен возвращать значения 
или выдавать проверяемые исключения. Зато все это умеет делать интерфейс _Callable_, его мы и используем.

Жизненный цикл задач состоит из этапов: создана, представлена, запущена и завершена. Мы хотим уметь отменять задачу, 
если что-то пошло не так. В структуре Executor задачи, которые еще не были запущены, могут быть отменены. 
Запущенные задачи могут быть отменены, только если они откликаются на прерывание. 

Интерфейс Future предоставляет методы, позволяющие проверять и управлять жизненным циклом задачи.

```java
public interface Callable<V> {
    V call() throws Exception;
}

public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws ManyExceptions;
}
```

ExecutorService может принимать Runnable или Callable, а все методы _ExecutorService.submit()_ возвращают объект Future.
Такой подход предоставляет простое управление жизненным циклом задчи.

Используя полученные знания, попробуем сделать наш отрисовщик чуть более конкрунтеным. Разделим отрисовщик на две 
подзадачи: отрисовку текста и скачивание изображений:

```java
public class FutureRenderer {
    private final int NUMBER_THREADS = 100;
    private final Executor executor = Executors.newFixedThreadPool(NUMBER_THREADS);

    void renderPage(String source) {
        final List<ImageInfo> imageInfoList = scanForImageInfo(source);
        Callable<List<ImageData>> task = () -> {
                List<ImageData> result = new ArrayList<>();
                for (ImageInfo imageInfo: imageInfoList) {
                    result.add(imageInfo.downloadImage());
                }
                return result;
        };
        
        Future<List<ImageData>> future = executor.submit(task);
        renderText(source);
        
        try {
            List<ImageData> imageDataList = future.get();
            for (ImageData data: imageDataList) {
                renderImage(data);
            }
        } catch (InterruptedException e) {
            future.cancel(true); // отменяем задачу
            Thread.currentThread().interrupt(); // прерываем поток выполнения 
        } catch (OtherException e) {
            // smth
        }
    }
}
```

В данном примере мы создали Callable для скачивания _всех_ изображений. Когда главный поток дойдет до точки, где ему
потребуется отрисовка изображений, он вызовет _future.get()_. Даже если все изображения не успели скачаться, мы все 
равно получим небольшое преимущество в производительности.

В этом подходе есть один недостаток, пользователю нет необходимости ждать, когда скачаются _все_ изображения. 
Возможно будет лучше видеть изображения по мере их скачивания.

#### CompletionService (Executor + BlockingQueue)
Если есть пакет задач, которые выполняет Executor, и нам необходимо извлекать результаты по мере выполнения, 
необходимо использовать CompletionService.

Принцип работы ExecutorCompletionService прост: конструктор создает очередь BlockingQueue для хранения 
завершенных результатов. Future, при успешном выполнении, попадает в эту очередь.

Попробуем использовать CompletionService для нашего отрисовщика. Благодаря тому, что каждое изображение будет 
скачиваться в отдельном потоке, мы улучшим производительность. А если отрисовывать картинки по мере их скачивания, это 
улучшит отзывчивость системы.

```java
public class Renderer {
    private final int NUMBER_THREADS = 100;
    private final Executor executor = Executors.newFixedThreadPool(NUMBER_THREADS);

    void renderPage(String source) {
        final List<ImageInfo> info = scanForImageInfo(source);
        CompletionService<ImageData> completionService = new ExecutorCompletionService<ImageData>(executor);
        for (final ImageInfo imageInfo : info) {
            completionService.submit(() -> imageInfo.downloadImage());
        }
        
        renderText(source);
        
        try {
            for (int i = 0; i < info.size(); i++) {
                Future<ImageData> future = completionService.take();
                ImageData imageData = future.get();
                renderImage(imageData);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (OtherException e) {
            // smth
        }
    }
}
```

#### Ограничение времени выполнения задачи
Сделать это можно с помощью хронометрированной версии Future.get(). Метод возвращает результат по готовности, но 
выдает TimeoutException, если результат не готов пределах указанного времени.

```java
public class TimedFutureGet {
    public void timedFutureGet(long timeBudget) {
        // ...
        try {
            future.get(timeBudget, NANOSECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
        }
    }
}
```

У ExecutorService есть хронометрированная версия invokeAll, которая принимает список задач и время на их выполнение.
Данная версия invokeAll возвратится после завершения всех задач, прерывания потока или по таймауту. Все задачи, которые 
не были завершены по истечении таймаута, отменяются.

# Итоги

- Структурирование задачи вокруг выполнения _задач_ упрощает разработку и способствует конкурентности. 
- Executor позволяет разделить предоставление задач от их выполнения и поддерживает разные режимы пула потоков.
- Всякий раз, когда вы создаете потоки для выполнения задач, рассмотрите возможность использования Executor.
- Чтобы получить максимальную выгоду из разбиения задач на потоки, необходимо определить разумные границы задач.