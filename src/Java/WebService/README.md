# Java WebService
План:
1) Что такое, зачем нужен и где встречается
2) SOAP и WSDL
3) Небольшое объяснение, как работает (с примером кода)
4) Полезные ссылки

**Java WebService** - технология, позволяющая предоставить API для обмена между разными сервисами поверх HTTP протокола.

Посредством WebService между собой общаются различные компоненты в крупных системах. Особенностью веб сервиса является 
независимость от языка, на котором написаны эти различные компоненты. Например в банке компоненты кредитования написаны 
на Golang, а отдел информации о клиенте на Java. Эти два компонента смогут взаимодействовать между собой, если будут
предоставлять веб сервисы друг другу.

// Тут должен быть рисунок


```java
import jakarta.jws.WebMethod;
import jakarta.jws.WebService;
import jakarta.xml.ws.Endpoint;

@WebService
public class HelloWorldWebService {
    @WebMethod
    public String sayHello(String msg){
        return "Hello "+msg;
    }

    public static void main(String[] args){
        System.out.println("hello");
        Endpoint.publish("http://localhost:8888/testWS", new HelloWorldWebService());
    }
}
```