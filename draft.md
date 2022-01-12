## Конфигурирование зависимостей бинов

### Конфигурирование через метод установки (setter)

```java
@Service
public class SomeService {
  private SomeBean someBean;
  
  @Autowired
  public void setSomeBean(SomeBean someBean) {
    this.someBean = someBean;
  }
}
```

> :bulb: **Tip:**  
Вместо аннотации `@Autowired` можно использовать аннотацию `@Resource(name="SomeBean")`, 
она позволяет использовать параметр name для более точного указания бина.  
Так же Spring поддерживает аннотацию `@Inject`, 
данная аннотация абсолютно идентична по смыслу аннотации `@Autowired`

### Конфигурирование через конструктор

```java
@Service
public class SomeSerivce {
  private final SomeBean someBean;
  
  @Autowired
  public SomeService(SomeBean someBean) {
    this.someBean = someBean;
  }
}
```

> :bulb: **Tip:**  
Для примитивов и их оберток можно использовать аннотацию `@Value` для параметра конструктора:  
>```java
>...
>public SomeConstructor(@Value("SomeValue") String someString) {
>...
>```  
> В `@Value` можно также использовать SpEL язык.

> :bulb: **Tip:**  
Аннотацию `@Autowired` можно применить только к одному конструктору.

### Конфигурирование через поле

```java
@Service
public class SomeSerivce {
  @Autowired
  private final SomeBean someBean;
}
```

Тем не менее такой вариант внедрения зависимости считается не самым лучшим. Таким образом не получится сконфигурировать `final` поля, 
также появляются сложности при написании тестов, поскольку не получится проставить конфигурации ручным способом.
