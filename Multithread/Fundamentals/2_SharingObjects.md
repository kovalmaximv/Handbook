# Совместное использование объектов
## Видимость
В многопоточной среде, если запись и чтение некоторого объекта разбиты по разным потокам, то поток чтения может 
просто не увидеть изменения в объекте. Чтобы обеспечить видимость изменения памяти в различных потоках, вы должны 
использовать синхронизацию.

Взглянем на пример. 

```java
@NotThreadSafe
public class NoVisibility {
    private static boolean ready;
    private static int number;
    
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) 
                Thread.yield();
            
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```
Два потока, главный и читающий обращаются к совместным переменным `ready` и `number`.  
Главный поток запускает `ReaderThread`, устанавливает значение `number`, устанавливает флаг `ready`.  
Читающий поток видит новое значение флага и распечатывает значение `number`, **но этого может не произойти**.

Читающий поток может попасть в бесконечный цикл, если значение `ready` не видимо для него. Или напечатать ноль, если 
операция `ready` станет видимой читающему потоку _перед_ записью `number` (явление называемое переупорядочивание). Нет
гарантий, что операции в потоке будут выполняться в заданном программном порядке.

> :exclamation: **Без синхронизации компилятор, процессор и рабочая среда могут перепутать порядок выполнения операций.**

> :exclamation: **Необходимо применять синхронизацию, когда данные используются потоками совместно.**

#### Устаревшие данные
Класс `NoVisibility` демонстрирует появление устаревших данных (_stale data_), которые видит читающий поток, если не 
используется синхронизация для доступа к данным. Но из-за переупорядочивания, могут быть ситуации похуже, когда 
поток видит актуальное значение одной переменной и устаревшее значение другой, которая программно должна 
была измениться раньше.

Пример ниже не является потокобезопасным и так же подвержен появлению устаревших данных.
```java
@NotThreadSafe
public class MutableInteger {
    private int value;
    
    public int get() { return value; }
    public void set(int value) { this.value = value; }
}
```

Доступ к полю `value` осуществляется из двух методов без синхронизации. Читающие потоки, использующие `get()` 
могут просто не увидеть изменения. Исправить пример можно следующим образом:

```java
@NotThreadSafe
public class MutableInteger {
    private int value;
    
    public synchronized int get() { return value; }
    public synchronized void set(int value) { this.value = value; }
}
```

Блокировка замком может использоваться для того, чтобы гарантировать видимость действий одного потока другим. 
Без синхронизации видимость не гарантирована. 

## Волатильные переменные
Java предоставляет так же более слабую форму синхронизации, _волатильные переменные_.

Переменная _volatile_ для компилятора и рабочей среды является совместной, то есть операции над ней не будут
переупорядочены с другими операциями в памяти. Так же волатильные переменные не кешируются в регистрах, 
где их данные скрыты от других процессов, поэтому их чтение всегда возвращает самый последний результат операции записи.

Типичный пример использования волатильных переменных:
```java
volatile boolean asleep;

while (!asleep)
    countSomeSheep();
```
Флаг `asleep` в данном коде должен быть волатильным, иначе поток может не заметить, когда флажок изменит другой поток.

Волатильные переменные недостаточны сильны, чтобы сделать операцию инкремента атомарной в многопоточной среде.

> :exclamation: **Блокировка может гарантировать как видимость, так и атомарность. 
> Волатильные переменные гарантируют только видимость.**

## ThreadLocal
ThreadLocal - это тип, значение которого своё в каждом потоке.

ThreadLocal предоставляет методы _get_ и _set_. Вызов get возвращает последнее значение, переданное в set 
из настоящего потока. При первом вызове get (без set) возвращается начальное значение для типа 
указанного в дженерике ThreadLocal.

ThreadLocal не является объектом, хранящим Map<Thread, T>, значения T хранятся в самом объекте Thread.

## Немутируемость 
Избежать синхронизации также можно использованием немутируемых (immutable) объектов. Все проблемы многопоточной среды
связаны с тем, что многочисленные потоки пытаются изменить одно и то же мутируемое состояние одновременно.

> :exclamation: **Немутируемые объекты всегда являются потокобезопасными.**

Объект является немутируемым если:
1) Его состояние невозможно изменить после конструирования
2) Все его поля финальны
3) Не происходит ускользание

## Публикация и ускользание
Публикация (publishing) объекта, означает передать его за пределы текущей области действия. Например,
ссылка на объект может быть возвращена из метода. Публикация объекта до момента их полного конструирования ставит под
угрозу потокобезопасность.

Не важно, делает что-то или нет объект, куда опубликовалось состояние. Риск неправильного использования всегда остается.

Объект, который не вовремя публикуется, называется ускользнувшим (escaped).

Публикация одного объекта может косвенно опубликовать и другие. Если добавить секрет в Set из примера ниже, то он
также опубликуется:
```java
public static Set<Secret> knownSecret;

public void initialize() {
    knownSecret = new HashSet<Secret>();
}
```

##### Правильная публикация
Когда один объект должен использоваться в нескольких потоках совместно, его необходимо правильно опубликовать.

Рассмотрим пример неправильной публикации:
```java
public class Holder {
    private int n;
    
    public Holder(int n) { this.n = n; }
    
    public void assertSanity() {
        if (n != n) 
            throw new RuntimeException("WTF");
    }
}

...

public Holder holder;

public void initialize() {
    holder = new Holder(37);
}
```
Казалось бы, код написан верно, RuntimeException невозможен. Но этот безопасен только для однопоточной среды.

В случае хотя бы двух потоков, один из них может увидеть устаревшее значение `holder` (null), либо актуальную 
ссылку в `holder`, но устаревшее значение состояния `holder.n`, либо (что еще хуже) устаревшее значение при первом 
чтении и новое значение при втором (случай RuntimeException).

Без синхронизации, в многопоточной среде возможны очень странные вещи.

Приемы безопасной публикации:
1) Инициализация в статическом инициализаторе
```java
public static Holder holder = new Holder(37);
```
2) Сохранение ссылки на него в волатильном поле либо в AtomicReference
```java
public volatile Holder holder = new Holder(37);
```
3) Сохранение ссылки на него в финальном поле (объект должен гарантировать отсутствие ускользания)
```java
public final Holder holder = new Holder(37);
```
4) Сохранение ссылки на него в поле, которое защищается замком
```java
private Holder holder;

public synchronized Holder getInstance() {
    holder = new Holder(37);
}
```