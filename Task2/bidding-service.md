# BiddingService — исправленная архитектура RTB-сценария

## 1. Границы ответственности

- **Ad Server** принимает OpenRTB-запрос DSP, определяет deadline, оркестрирует обращения к Bidding/Auction и Delivery, формирует и возвращает HTTP/OpenRTB-ответ.
- **Bidding/Auction Service** владеет логикой аукциона: подбор и фильтрация кандидатов, проверка допустимости кампаний, ранжирование ставок, атомарное резервирование бюджета выбранного кандидата, определение победителя. Только этот сервис выбирает победителя.
- **cache** хранит актуализированные сведения о кампаниях, таргетинге, времени активности и креативах. В критическом пути Bidding использует локальный/быстрый read cache; изменения поступают асинхронно. Финальная проверка активности и бюджета производится при резервировании.
- **BudgetService** владеет бюджетами и журналом резервирований/списаний. Гарантирует отсутствие перерасхода за счет атомарных транзакций. Операции Reserve, Release и Capture идемпотентны.
- **DeliveryService** по уже выбранному победителю и creativeId готовит креатив, URL/markup и поля OpenRTB-ответа. Не сравнивает ставки и не определяет победителя.
- **Kafka** используется для событий и аналитики вне синхронного пути; победитель в Delivery передается только через RPC, инициируемый Ad Server.

## 2. Основной RTB-сценарий

1. DSP направляет `POST /auction`, включая `requestId`, `positionId` (либо соответствующий `imp`), контекст и `tmax`.
2. Ad Server устанавливает общий deadline: не более **80 мс от получения запроса**, и не более `tmax` отправителя, если он меньше. Все внутренние RPC получают оставшееся время, а не собственные независимые 80 мс.
3. Ad Server вызывает `Auction` по синхронному gRPC.
4. Bidding/Auction получает из campaign read cache кандидатов по позиции и контексту, проверяет статус `ACTIVE`, даты, таргетинг, ограничения; сортирует допустимые ставки по цене по убыванию.
5. Для кандидатов по порядку Bidding/Auction запрашивает `Reserve`. BudgetService атомарно убеждается, что `limit >= spent + reserved + amount`, кампания допускает расходы, и резервирует сумму. Первый успешно зарезервированный кандидат становится победителем; кандидаты без средств пропускаются. Количество попыток ограничено общим deadline. При неопределенном результате Reserve (тайм-аут) выбор останавливается до сверки состояния, чтобы не допустить двух резервов на один аукцион.
6. Bidding/Auction возвращает `Winner` либо `NO_BID`.
7. При наличии победителя Ad Server вызывает `PrepareAd` по синхронному RPC. Delivery подготавливает поля выдачи, **но не выбирает победителя**.
8. Ad Server возвращает `HTTP 200` с OpenRTB bid response. Если победителя нет, подготовка выдачи не удалась или исчерпан deadline — `HTTP 204 No Bid`. Если резерв уже создан, инициируется `Release`, а истечение TTL и сверка состояний страхуют от потери компенсации.
9. Вне 80-мс пути при подтвержденном тарифицируемом событии (по принятой модели оплаты) выполняется `BudgetService.Capture(eventId, reservationId)`; при отсутствии события до срока удержания — `Release`/истечение резерва.

Уточнение: в этой учебной схеме инициатором запроса обозначен DSP. Если роль отправителя в реальной RTB-интеграции выполняет SSP/Ad Exchange, внутренний сценарий остается тем же.

## 3. REST и RPC-контракты

### Внешний RTB API

`POST /auction` — входящий запрос аукциона с `requestId`, placement/impression, пользовательским контекстом и допустимым временем ответа. Успех: `200`; отсутствие ставки: `204`.

### Административное REST API ставок

- `POST /positions/{positionId}/bids` — создать ставку; тело: `campaignId`, `price`, `currency`; поддерживается `Idempotency-Key`.
- `GET /positions/{positionId}/bids` — получить ставки в пределах прав вызывающего (либо отдельный `/bids/me` для своей ставки).
- `DELETE /positions/{positionId}/bids/{bidId}` — отозвать собственную ставку, если аукцион/правила позволяют отзыв.
- `POST /internal/candidates` — загрузка/обновление кандидатов для read cache вне критического RTB-пути; предпочтителен асинхронный поток обновлений из Campaign Service.

Публичный `POST /positions/{positionId}/winner` исключен: выбор победителя привязан к конкретному RTB-запросу, его таргетингу, времени и состоянию бюджета, а не является отдельным ручным CRUD-действием.

### Синхронные RPC

- `Bidding/Auction.SelectWinner(AuctionRequest {requestId, positionId, context, remainingMs}) -> AuctionDecision {winner | noBid, reservationId}`.
- `BudgetService.Reserve(ReserveRequest {auctionKey, campaignId, amountMinor, ttl}) -> ReserveResult`.
- `BudgetService.Release(reservationId) -> ReleaseResult`.
- `BudgetService.Capture(eventId, reservationId) -> CaptureResult` — вызывается вне критического RTB-пути.
- `DeliveryService.PrepareAd(PrepareAdRequest {winner, creativeId, context, remainingMs}) -> PreparedAd`.

### Kafka

Kafka **не передает победителя между Bidding и Delivery**. Она передает события `AuctionCompleted`, `ImpressionRegistered`, `BillableEvent`, `BudgetCaptured` и события обновления кампаний. Для публикации из транзакционных сервисов применяется transactional outbox с повторными попытками; потребители обязаны быть идемпотентны.

## 4. Модель данных

- `user(id, ...)` — учетные данные пользователя/рекламодателя, для административного API; не читается синхронно на каждый RTB-запрос.
- `bid(id, position_id, campaign_id, advertiser_id, price_minor, currency, status, created_at)` — предложения ставок.
- `campaign(id, status, valid_from, valid_to, targeting, creative_id, version)` — источник данных для read cache.
- `campaign_budget(campaign_id PK, limit_minor, spent_minor, reserved_minor, active)` — актуальный баланс, хранится у BudgetService.
- `budget_reservation(reservation_id PK, auction_key UNIQUE, campaign_id, amount_minor, status, expires_at)` — состояние `RESERVED`, `CAPTURED`, `RELEASED` либо `EXPIRED`.
- `charge_event(source, event_id, reservation_id, UNIQUE(source,event_id), UNIQUE(reservation_id))` — защита от повторного фактического списания по событию и по резерву.
- `auction_result(source, request_id, decision, reservation_id, response_snapshot, UNIQUE(source,request_id))` — опциональная дедупликация повторного RTB-запроса.

Цены и бюджеты хранятся в минимальных целочисленных единицах валюты; смешение валют без явного преобразования запрещено.

## 5. Атомарное резервирование и защита от повторов

**Reserve**, одна транзакция в БД BudgetService:

1. `INSERT` резервирования с уникальным `auction_key = sourceId:requestId`; при конфликте вернуть сохраненный результат, **не увеличивая reserved второй раз**.
2. `UPDATE campaign_budget SET reserved_minor = reserved_minor + :amount WHERE campaign_id = :id AND active = TRUE AND limit_minor - spent_minor - reserved_minor >= :amount RETURNING campaign_id`.
3. Если обновлено 0 строк, откатить транзакцию и вернуть `NO_FUNDS`; иначе зафиксировать транзакцию и вернуть `reservationId`.

Уникальный ключ исключает несколько успешных резервов для одного показа (imp) в аукционе; условный UPDATE сериализует конкурирующие изменения одного бюджета и исключает превышение лимита.

**Capture** при получении тарифицируемого события: в одной транзакции дедуплицировать `eventId`, перевести `budget_reservation` только из `RESERVED` в `CAPTURED`, уменьшить `reserved_minor` и увеличить `spent_minor`. Повторное событие или иной eventId для уже списанного резерва не приводит к второму списанию. `Release` симметрично переводит только `RESERVED` в `RELEASED` и уменьшает reserved; `Capture` и `Release` конкурируют через условный переход статуса и не применяются одновременно.

## 6. Deadline и отказоустойчивость

Общий бюджет **80 мс** (проектные верхние лимиты): разбор запроса — 5 мс; кандидаты и проверки кампаний — 17 мс; попытки резервирования — 25 мс; вычисление победителя — 3 мс; Delivery — 15 мс; формирование/отправка ответа — 5 мс; страховой запас — 10 мс. Сумма — 80 мс. Внешняя сетевая задержка от DSP до входной точки платформы зависит от интеграции и должна учитываться отдельно при обещании сквозного SLA.

- Кандидатов нет / кампания неактивна / средств нет: `NO_BID`, без списания.
- Deadline близок к исчерпанию: не начинать новые медленные операции, вернуть `204`.
- `BudgetService` недоступен или ответ Reserve неоднозначен: не выбирать победителя без подтвержденного резерва; сверить состояние по `auctionKey`, освободить поздно созданный резерв или дождаться TTL.
- `DeliveryService` недоступен / тайм-аут: `204 No Bid`, инициировать идемпотентный Release; Bidding **не замещает** Delivery и не готовит ответ вместо него.
- Ответ уже отправлен, но не пришло подтвержденное тарифицируемое событие: резерв истекает и освобождается по установленному бизнес-TTL, который **не равен 80 мс**.
- Повтор запроса DSP: возвращать прежнее решение, если его срок действия не истек; повтор не создает новое резервирование или списание.
