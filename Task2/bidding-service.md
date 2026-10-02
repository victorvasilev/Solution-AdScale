**Границы**

Сервис обеспечивает работу со ставками, представляя CRUD-API для работы с ними.

**Модель данных**

Таблица user с данными о пользователях
Таблица bid с данными о ставках

**REST API**

1. POST /postitions/{positionId}/bedds - для создания новой ставки. 

request body:
- price - Цена ставки

2. GET /postitions/{positionId}/bedds - для получения текущей ставки.

3. DELETE /postitions/{positionId}/bedds/{beddId} - для отзыва своей ставки.

4. POST /candidates - Для передачи данных о кандидате