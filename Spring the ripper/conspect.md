## Context

Как все работает:  
1) Поднимается контекст  
2) BeanDefinitionReader  
он читает определение бинов из Xml контекста и составляет мапу `bean's id -> Bean declaration`.  
Bean declaration хранит то, что мы и описали в xml (clss, property, etc)
3) BeanFactory  
Сначала вытаскивает из `Bean declaration` мапы достает BeanPostProcessor, создает их и кладет в сторонку.  
Создает бины исходя из мапы `BeanDefinitionReader`.  
К каждому бину применяются ВСЕ созданные BeanPostProcessor.  
Вызывается init метод бина.  
К каждому бину применяются ВСЕ созданные BeanPostProcessor.  
Готовый бин кладется в контейнер.  

Важно, что синглтоны создаются сразу, а прототайпы по требованию (прототайпы в контейнер не складываются).

#### Зачем нужен init метод
Для модели двухфазного контроллера (controller -> init method).  
Если мы обратимся к спринговым компонентам в контроллере, то получим NPE. `BeanInjection` происходит после создания 
работы контроллера. А в init method все зависимости спринга будут на месте.