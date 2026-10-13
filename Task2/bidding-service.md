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

5. POST /postitions/{positionId}/winner - Для определения победителя. Наиболее высокая ставка по позиции аукциона


**Интеграции**

DeliveryService предоставляет синхронный rpc-интерфейс для получения информации о победителе.

В случае недоступности DeliveryService, BiddingService может сам предоставить данные о победителе логикой, реализованной в рамках POST /postitions/{positionId}/winner