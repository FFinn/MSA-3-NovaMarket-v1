# Реестр событий Saga-хореографии «Оформление заказа»

**Корреляция и порядок.** Во все события включаются `eventId`, `eventType`, `eventVersion`, `occurredAt`, `correlationId` и `orderId`. Ключ партиционирования Kafka — `orderId`: для одного заказа сохраняется порядок. Доставка допускает повтор (`at-least-once`), поэтому потребители дедуплицируют `eventId`.

| Логический этап | Тип события | Название | Издатель | Основной потребитель / действие |
| --- | --- | --- | --- | --- |
| Заказ зафиксирован | domain | `OrderCreated` | Order Service | Payment Service начинает оплату. |
| Оплата успешно проведена | domain | `PaymentSucceeded` | Payment Service | Delivery Service создаёт доставку; Order Service фиксирует прогресс Saga. |
| Оплата отклонена | failure | `PaymentFailed` | Payment Service | Order Service отменяет заказ. |
| Доставка создана | domain | `DeliveryCreated` | Delivery Service | Order Service подтверждает заказ. |
| Создание доставки не удалось | failure | `DeliveryCreationFailed` | Delivery Service | Payment Service запускает компенсационный возврат; Order Service ждёт его результат. |
| Заказ подтверждён | domain | `OrderConfirmed` | Order Service | Notification Service уведомляет продавца; история заказа обновляется. |
| Заказ отменён из-за неоплаты | domain | `OrderCancelled` | Order Service | Delivery Service игнорирует/отменяет незавершённую доставку; история заказа обновляется. |
| Возврат средств выполнен | compensation | `PaymentRefunded` | Payment Service | Order Service окончательно отменяет заказ. |
| Возврат средств не выполнен автоматически | failure | `PaymentRefundFailed` | Payment Service | Order Service ставит заказ в `refund_pending`, создаётся задача на повтор/ручную обработку. |
| Доставка отменена (если уже была создана) | compensation | `DeliveryCancelled` | Delivery Service | Аудит и история заказа фиксируют компенсацию. |

## Состояния заказа

`new → payment_pending → paid → delivery_pending → confirmed` — успешный путь.

Ошибочные пути: `payment_pending → cancelled`; `delivery_pending → refund_pending → cancelled`. Статус `refund_pending` сохраняет наблюдаемое, не скрытое состояние: деньги ещё не возвращены, поэтому заказ нельзя считать окончательно отменённым.
