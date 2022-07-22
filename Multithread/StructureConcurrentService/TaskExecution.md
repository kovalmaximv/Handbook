# Выполнение задач
Большинство конкурентных приложений выполняют некоторые параллельные задачи. Разделение работы приложения на 
задачи упрощает проектирование и структуру многопоточных приложений.

## Выполнение задач в потоках
Первое, что стоит сделать, четко определить границы задач. Независимые друг от друга задачи могут выполняться 
параллельно. 

Большинство серверных приложений в качестве границы одной задачи предлагают индивидуальные клиентские запросы. Сервер
принимает запрос от удаленных клиентов, обрабатывают их одновременно и независимо друг от друга.

В этой главе мы будем в качестве примера разрабатывать каркас HTTP сервера. Рассмотрим пример последовательной 
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
Сделаем наш HTTP сервер многопоточным

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