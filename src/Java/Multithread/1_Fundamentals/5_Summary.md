# Итоги

- Все вопросы конкурентности сводятся к координированию доступа к мутируемому состоянию. Чем менее мутриуемо состояние,
тем легче обеспечить потокобезопасность.
- Объявляйте поля финальными, если это возможно.
- Немутируемые объекты автоматически становятся потокобезопасными. 
- Инкапсуляция упрощает написание конкурентного потокобезопасного кода.
- Защищайте каждую мутируемую переменную с помощью замка.
- Защищайте все переменные в инварианте с помощью одного замка.
- Удерживайте замки в течении всего времени составных действий.
- Документируйте политику синхронизации класса.