## SOAP
SOAP - протокол для обмена сообщениями на основе XML определенного формата. Работает SOAP поверх HTTP протокола.

Основная сложность SOAP протокола, это его сообщение в определенном формате:
```xml
<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
<S:Body>
 <ns2:getHello xmlns:ns2="http://samples/hello">
    <arg0>World</arg0>
 </ns2:getHello>
</S:Body>
</S:Envelope>
```

Но нас интересует только `Body` часть сообщения, именно она отвечает за фактическое сообщение веб сервису.
```xml
<!-- ... -->
<ns2:getHello xmlns:ns2="http://samples/hello"> <!-- имя веб метода -->
    <arg0>World</arg0> <!-- параметры для метода -->
</ns2:getHello>
<!-- ... -->
```