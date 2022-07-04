# Компоновка объектов
## Введение в проектирование потокобезопасного класса
Проектирование потокобезопасного класса включает 3 этапа:
1) Определение переменных, формирующий состояние класса
2) Определение инвариантов состояния
3) Создание политики конкурентного доступа к состоянию

Важно понимать ограничения, которые накладываются на состояния объекта. Такие ограничения могут быть вызваны 
бизнес процессами, техническими сложностями и прочими вещами. Понимание таких ограничений необходимы для 
проектирования потокобезопасного класса (в чем мы убедимся ниже).

## Ограничение одним экземпляром
Инкапсуляция упрощает создание потокобезопасных классов, предлагая ограничение одним экземпляром (instance confinement).

> :exclamation: **Инкапсуляция данных в объекте ограничивает доступ к ним только методами объекта, что 
> упрощает синхронизацию при помощи замков.**

Пример такого ограничения:
```java
@ThreadSafe
public class PersonSet {
    private final Set<Person> mySet = new HashSet<Person>();
    
    public synchronized void addPerson(Person p) {
        mySet.add(p);
    }
    
    public synchronized boolean containsPerson(Person p) {
        return mySet.contains(p);
    }
}
```
Состояние объекта PersonSet - непотокобезопасный HashSet, но mySet является приватным, не может ускользнуть и 
ограничен одним экземпляром. Единственные ветви кода, которые способны обратиться к mySet защищены внутренним замком. 
Таким образом, класс получился потокобезопасным.

Стоит однако отметить, что если класс Person является мутируемым и непотокобезопасным, то класс перестанет 
быть потокобезопасным.

> :exclamation: **Ограничение одним экземпляром упрощает создание потокобезопасных классов, так как анализировать 
> приходится один класс, вместо всей программы.**

#### Мониторный шаблон 
По принципу ограничения одним экземпляром работает и мониторный шаблон Java (Java monitor pattern): объект 
инкапсулирует мутируемое состояние под защитой внутреннего замка.

```java
public class LockExample {
    private final Object lock = new Object();
    private MutableObject mutableObject = new MutableObject();
    
    public void method() {
        synchronized (lock) {
            mutableObject.change();
        }
    }
}
```
С таким же успехом можно было использовать synchronized метод, просто для примера был использован другой механизм.

#### Пример: трекер свободных мест в кинотеатре
Рассмотрим пример мониторного шаблона. Разработаем трекер свободных мест для администратора кинотеатра. 
Сначала мы напишем его с помощью шаблонного метода, а затем ослабим требования к инкапсуляции, сохраняя 
потокобезопасность.

Каждое место в зале будет идентифицироваться специальным id, а состояние места будет состоять из флага занятости boolean.

Так как покупать билеты могут одновременно, то обращаться к такой системе будут конкурентно. Ниже представлена 
реализация с использованием мониторного шаблона:

```java
@NotThreadSafe
public class MutableSeat {
    public boolean reserved;

    public MutableSeat() {
        reserved = false;
    }

    public MutableSeat(MutableSeat seat) {
        this.reserved = seat.reserved;
    }
}

...

@ThreadSafe
public class CinemaTracker {
    private final Map<Integer, MutableSeat> seats;

    public CinemaTracker(Map<Integer, MutableSeat> seats) {
        this.seats = deepCopy(seats); // Производит глубокое копирование Map
    }

    public synchronized Map<Integer, MutableSeat> getSeats() {
        return deepCopy(seats);
    }

    public synchronized MutableSeat getSeat(Integer id) {
        MutableSeat seat = seats.get(id);
        return seat == null ? null : new MutableSeat(seat);
    }

    public synchronized void setSeat(Integer id, boolean reserved) {
        MutableSeat seat = seats.get(id);
        if (seat == null) {
            throw new RuntimeException();
        }
        seat.reserved = reserved;
    }

    private static Map<Integer, MutableSeat> deepCopy(Map<Integer, MutableSeat> target) {
        Map<Integer, MutableSeat> result = new HashMap<Integer, MutableSeat>();
        for (Integer id : target.keySet()) {
            result.put(id, new MutableSeat(target.get(id)))
        }
        return Collections.unmodifiableMap(result);
    }

}
```
Реализация CinemaTracker потокобезопасна, ни ассоциативный массив, ни мутируемые состояния мест не публикуются.
Чтобы вернуть информацию о состоянии места, мы копируем его с помощью конструктора копирования или метода deepCopy.

Однако копирование большого количества данных негативно скажется на производительности. Еще одной особенностью 
реализации является неизменность данных. Если мы получим данные о состоянии места, а после они изменятся, то в 
полученном экземпляре будут храниться старые значения. 

## Делегирование потокобезопасности
Почти все объекты являются композитными (составными). Что если компоненты класса уже потокобезопасны, нужно ли 
добавлять дополнительный уровень потокобезопасности? Попробуем ответить на этот вопрос.

#### Пример трекера кинотеатра с использование делегирования
Попробуем сконструировать версию трекера, которая делегирует свои обязанности по потокобезопасности своим компонентам. 
Изменим реализацию ассоциативного массива на конкурентную его версию и сделаем неизменяемым класс состояния места.

```java
@Immutable
@ThreadSafe
public class ImmutableSeat {
    public final boolean reserved;

    public MutableSeat(boolean reserved) {
        this.reserved = reserved;
    }
}

...

@ThreadSafe
public class DelegatingCinemaTracker {
    private final ConcurrentHashMap<Integer, ImmutableSeat> seats;
    private final Map<Integer, ImmutableSeat> unmodifiableCopy;

    public DelegatingCinemaTracker(Map<Integer, ImmutableSeat> seats) {
        seats = new ConcurrentHashMap<>(seats);
        unmodifiableCopy = Collections.unmodifiableMap(seats);
    }
    
    public Map<Integer, ImmutableSeat> getSeats() {
        return unmodifiableCopy;
    }
    
    public ImmutableSeat getSeat(Integer id) {
        return seats.get(id);
    }
    
    public void setSeat(Integer id, boolean reserved) {
        if (seats.replace(id, new ImmutableSeat(reserved)) == null) {
            throw new RuntimeException();
        }
    }
}
```

Использование исходного класса ImmutableSeat позволило обойтись без копирования при публикации состояния места, так 
как теперь нет вероятности, что кто-то изменит это состояния.

Так же стоит обратить внимание, что предыдущая версия трекера отдавала снимок состояний мест, а делегирующая версия 
возвращает их неизменяемое, но "живое" представление, которое будет обновляться со временем. 

#### Независимые переменные состояния
Делегировать потокобезопасность более чем одной переменной состояния возможно, если эти переменные _независимы_. То есть
нет инвариантов с их совместным участием.

В примерах выше переменные состояния были независимы, поэтому делегирование удалось. Рассмотрим пример с зависимыми 
переменными:

```java
@NotThreadSafe
public class NumberRange {
    // Инвариант: lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);
    
    public void setLower(int num) {
        // Небезопасная операция: проверить и затем действовать
        if (num > upper.get()) {
            throw new RuntimeException();
        }
        lower.set(num);
    }
    
    public void setUpper(int num) {
        // Небезопасная операция: проверить и затем действовать
        if (num < lower.get()) {
            throw new RuntimeException();
        }
        upper.set(num);
    }
}
```
Класс NumberRange не является потокобезопасным: он не соблюдает инвариант `lower <= upper`. Если один поток вызовет 
`setLower(5)`, а другой поток `setUpper(4)`, то из-за операции "проверить и затем действовать" при неудачной временной
координации оба потока пройдут проверку, установят значения и нарушат инвариант. 

В отличие от базовых AtomicInteger компонентный класс непотокобезопасен. Класс NumberRange не может делегировать 
потокобезопасность компонентам, потому что они зависимы.

Данную ситуацию исправит блокировка методов setLower, setUpper.

> :exclamation: **Если класс содержит независимые потокобезопасные переменные и не имеет недопустимых переходов 
> между состояниями, то он может делегировать потокобезопасность своим компонентам.**

#### Публикация базовых переменных состояния
Если переменная состояния является потокобезопасной, не участвует в инвариантах, ограничивающих ее значение, и не имеет 
запрещенных переходов из состояния в состояние, то она может быть опубликована

#### Пример трекера кинотеатра публикующего свое состояние
```java
@ThreadSafe
public class ThreadSafeSeat {
    private boolean reserved;

    public MutableSeat() {
        reserved = false;
    }

    public MutableSeat(MutableSeat seat) {
        this.reserved = seat.reserved;
    }
    
    public synchronized boolean getReserved() {
        return reserved;
    }
    
    public synchronized void setReserved(boolean reserved) {
        this.reserved = reserved;
    }
}

@ThreadSafe
public class PublishingCinemaTracker {
    private final ConcurrentHashMap<Integer, ThreadSafeSeat> seats;
    private final Map<Integer, ThreadSafeSeat> unmodifiableCopy;

    public DelegatingCinemaTracker(Map<Integer, ThreadSafeSeat> seats) {
        seats = new ConcurrentHashMap<>(seats);
        unmodifiableCopy = Collections.unmodifiableMap(seats);
    }

    public Map<Integer, ThreadSafeSeat> getSeats() {
        return unmodifiableCopy;
    }

    public ThreadSafeSeat getSeat(Integer id) {
        return seats.get(id);
    }

    public void setSeat(Integer id, boolean reserved) {
        if (seats.contains(id)) {
            throw new RuntimeException();
        }
        seats.get(id).set(reserved);
    }
}
```

Благодаря тому, что состояние места теперь мутируемое, вызывающие элементы кода могут менять состояние места. 
Весь этот пример получился потокобезопасным, потому что PublishingCinemaTracker не накладывает дополнительные 
ограничения на состояния места и потому что класс состояния места является потокобезопасным.

## Добавление функциональности в существующие потокобезопасные классы
#### Блокировка на стороне клиента
Попробуем добавить атомарный функционал "добавить если отсутствует" в стандартный класс List.

```java
@NotThreadSafe
public class ListHelper<T> {
    public List<T> list = Collections.synchronizedList(new ArrayList<T>());
    
    public synchronized boolean putIfAbsent(T x) {
        boolean absent = !list.contains(x);
        if (absent) {
            list.add(x);
        }
        return absent;
    }
}
```

Почему класс по итогу получился NotThreadSafe? Хоть метод putIfAbsent и синхронизирован, но синхронизация 
происходит на неправильном ключе. Другие операции нашего `list` использует другой клич, поэтому метод `putIfAbsent` 
не воспринимается другими операциями как атомарный. 

Исправить эту проблему может _блокировка на стороне клиента_. Мы защитим клиентский код, который использует некий 
объект Х, собственным замком объекта Х. Необходимо только узнать, что это за замок, благо большинство 
потокобезопасных коллекций и объектов в документации указывают этот замок.

```java
@ThreadSafe
public class ListHelper<T> {
    public List<T> list = Collections.synchronizedList(new ArrayList<T>());
    
    public boolean putIfAbsent(T x) {
        synchronized (this) {
            boolean absent = !list.contains(x);
            if (absent) {
                list.add(x);
            }
            return absent; 
        }
    }
}
```

#### Компоновка
Так же добавить атомарную операцию в существующий класс поможет _компоновка_. Пример ниже использует этот подход:
```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
    private final List<T> list;
    
    public ImprovedList(List<T> list) { this.list = list; }
    
    public synchronized boolean putIfAbsent(T x) {
        boolean absent = !list.contains(x);
        if (absent) {
            list.add(x);
        }
        return absent;
    }
    
    public synchronized void clear() {
        // ... делегировать остальные методы схожим образом
    }
}
```