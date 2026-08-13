# Реестр событий Saga-хореографии «Оформление заказа»

**Принцип хореографии.** Нет центрального сервиса, который посылает остальным команды «оплатить», «зарезервировать» или «доставить». Каждый сервис подписывается на доменное событие, выполняет своё локальное действие и публикует новый факт. `Order Service` владеет только агрегатом заказа и реактивно меняет его статус; он не управляет другими сервисами.

**Корреляция и порядок.** В каждое событие добавляются `eventId`, `eventType`, `eventVersion`, `occurredAt`, `correlationId` и `orderId`. Ключ Kafka — `orderId`. Доставка выполняется как минимум один раз, поэтому каждый потребитель исключает повторную обработку по `eventId`.

| Логический этап | Тип события | Название | Издатель | Реакция потребителя |
| --- | --- | --- | --- | --- |
| Заказ создан | domain | `OrderCreated` | Order Service | Inventory Service проверяет остатки и создаёт резерв. |
| Товары успешно зарезервированы | domain | `InventoryReserved` | Inventory Service | Payment Service создаёт платёж и предоставляет покупателю платёжный сценарий. |
| Платёж создан | domain | `PaymentCreated` | Payment Service | Order Service сохраняет идентификатор платежа и ссылку на оплату. Клиент получает их при запросе статуса заказа; для привязанной карты запускается автоматическая оплата. |
| Товаров недостаточно / резерв не создан | failure | `InventoryReservationFailed` | Inventory Service | Order Service переводит заказ в `cancelled`. Оплата не начинается. |
| Оплата проведена | domain | `PaymentSucceeded` | Payment Service | Delivery Service создаёт доставку; Order Service сохраняет `paid`. |
| Оплата отклонена | failure | `PaymentFailed` | Payment Service | Inventory Service снимает резерв; Order Service сохраняет причину отмены. |
| Резерв снят | compensation | `InventoryReservationReleased` | Inventory Service | Order Service окончательно сохраняет `cancelled` после неоплаты или ошибки доставки. |
| Заявка на доставку создана | domain | `DeliveryCreated` | Delivery Service | Order Service переводит заказ в `preparing_for_shipment` и публикует `OrderConfirmed`; продавец получает задачу на сборку. |
| Создание доставки не удалось | failure | `DeliveryCreationFailed` | Delivery Service | Payment Service запускает возврат, Inventory Service независимо снимает резерв. |
| Деньги возвращены | compensation | `PaymentRefunded` | Payment Service | Order Service фиксирует компенсацию; после освобождения резерва — `cancelled`. |
| Автоматический возврат не выполнен | failure | `PaymentRefundFailed` | Payment Service | Order Service переводит заказ в `refund_pending`; повтор или ручная обработка. |
| Заказ подтверждён и готовится к отправке | domain | `OrderConfirmed` | Order Service | Notification Service уведомляет продавца; история и клиентский интерфейс показывают статус «Оплачен и готовится к отправке». |
| Статус доставки изменён | domain | `DeliveryStatusChanged` | Delivery Service | При статусах `handed_to_delivery` и `delivered` Order Service и клиентское приложение показывают соответственно «Передан в доставку» и «Доставлен», а также трек-номер и дату. |

## Состояния заказа

Успешный путь: `new → reservation_pending → reserved → payment_pending → paid → delivery_pending → preparing_for_shipment → handed_to_delivery → delivered`.

Ошибки: `reservation_pending → cancelled`; `payment_pending → cancellation_pending → cancelled`; `delivery_pending → refund_pending → cancelled`.

В состоянии `refund_pending` заказ не считается окончательно отменённым: возврат денег ещё не подтверждён.
