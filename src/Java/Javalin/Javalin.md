# Javalin

Зачем использовать Javalin:
1) Простой: это микрофреймворк. Содержит небольшое количество концептов, которые необходимы для понимания и написания кода
2) Легковесный: javalin это небольшая (пару тысяч строк) обертка на jetty, нет потерь производительности. Позволяет 
настроить jetty
3) Универсальный: как и на jetty можно реализовать абсолютно все, но без специфического кода для jetty
4) OpenAPI: хоть фреймворк и легковесный, но предлагает поддержку OpenAPI и swagger

Поддерживаемые концепции:
1) Http handlers
2) WebSocket
3) Validation
4) Authentication/Authorization
5) Exception handlers

# Http handlers

Создадим простейшее приложение, которое на запрос `GET /ping` ответит `pong`:
```java
public class HttpHandler {
    public static void main(String[] args) {
        Javalin.create()
                .get("/ping", ctx -> ctx.result("pong"))
                .start(8080);
    }
}
```

Обработка запросов происходит посредством handler'ов. Их существует 4 типа: before, endpoint, after, exception. Before
выполняется перед обработкой запроса. Endpoint отвечает за обработку запроса. After выполняется после обработки запроса.
Про exception поговорим позже.

```java
public class AllTypeHandlers {
    private final static Logger log = LoggerFactory.getLogger(AllTypeHandlers.class);

    public static void main(String[] args) {
        Javalin.create()
                .before("/ping/{count}", ctx -> {
                    if ("Max".equals(ctx.header("name"))) {
                        log.info("Max was here");
                    }
                })
                .get("/ping/{count}", ctx -> ctx.result("pong " + ctx.pathParam("count")))
                .after("/ping/{count}", ctx -> log.info("request from ip: {}", ctx.ip()))
                .start(8080);
    }
}
```

Хендлеры можно объединять в группы:
```java
public class HandlerGroup {
    private final static PingController pingController = new PingController();

    public static void main(String[] args) {
        Javalin.create(HandlerGroup::setupJavalin).start(8080);
    }

    private static void setupJavalin(JavalinConfig config) {
        config.router.apiBuilder(() ->
            path("/ping", () -> {
                before(pingController::beforeHandler);
                get(pingController::getPing);
                post(pingController::postPing);
                after(pingController::afterHandler);
            }));
    }
}
```

# Validation
Javalin позволяет писать собственные валидаторы на все типы входных данных. Так же происходит автоматическое приведение
к необходимому классу. 

```java
public class ShopController {

    private final static Logger log = LoggerFactory.getLogger(ShopController.class);

    public void supplyProduct(Context ctx) {
        Integer shift = ctx.queryParamAsClass("shift", Integer.class).getOrDefault(1);

        Product product = ctx.bodyValidator(Product.class)
                .check(p -> p.quantity() > 0, "Quantity cannot be negative")
                .get();

        log.info("Shift {} got product supplied: {}", shift, product);
    }

}
```

# AccessManagement
Аутентификация/авторизация на самом деле настраивается с помощью beforeHandler. Единственное, в чем нам помогает 
Javalin - позволяет для каждого рута прописать список ролей, которые могут пользоваться данной ручкой.

Для этого необходимо сначала реализовать интерфейс `RouteRole`
```java
public enum Role implements RouteRole {
    ASSISTANT, ACCOUNTANT, NOT_LOGGED_IN
}
```

Далее для каждого рута проставляется список ролей, которые могут пользоваться данной ручкой:
```java
public class AuthHandler {
    private final static ShopController PRODUCT_CONTROLLER = new ShopController();
    private final static AuthFilter AUTH_FILTER = new AuthFilter();

    public static void main(String[] args) {
        Javalin.create(AuthHandler::setupJavalin)
                .beforeMatched(AUTH_FILTER::authHandler)
                .start(8080);
    }

    private static void setupJavalin(JavalinConfig config) {
        config.router.apiBuilder(() ->
            path("/supply", () -> {
                post(PRODUCT_CONTROLLER::supplyProduct, Role.ASSISTANT);
            }));
    }
}
```

Сама логика аутентификации должна находиться в beforeHandler:
```java
public class AuthFilter {

    public void authHandler(Context ctx) {
        Optional<Role> userRole = getUserRole(ctx);

        if (userRole.isEmpty()) {
            throw new UnauthorizedResponse();
        }

        if (!ctx.routeRoles().contains(userRole.get())) {
            throw new UnauthorizedResponse();
        }
    }

    private Optional<Role> getUserRole(Context ctx) {
        String user = ctx.header("user");
        String pass = ctx.header("password");

        if ("admin".equals(user) && "admin".equals(pass)) {
            return Optional.of(Role.ASSISTANT);
        }
        return Optional.empty();
    }
}
```