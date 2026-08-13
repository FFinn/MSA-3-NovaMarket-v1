# Реестр событий Saga-хореографии «Оформление заказа»

**Корреляция и порядок.** В каждое событие добавляются `eventId`, `eventType`, `eventVersion`, `occurredAt`, `correlationId` и `orderId`. Ключ партиционирования Kafka — `orderId`, поэтому события одного заказа приходят в нужном порядке. Повторная доставка возможна (`at-least-once`), поэтому потребители проверяют `eventId` и не выполняют одно действие дважды.

| Логический этап | Тип события | Название | Издатель | Основной потребитель / действие |
| --- | --- | --- | --- | --- |
| Заказ зафиксирован | domain | `OrderCreated` | Order Service | Payment Service начинает оплату. |
| Оплата успешно проведена | domain | `PaymentSucceeded` | Payment Service | Delivery Service создаёт доставку; Order Service отмечает шаг Saga как выполненный. |
| Оплата отклонена | failure | `PaymentFailed` | Payment Service | Order Service отменяет заказ. |
| Доставка создана | domain | `DeliveryCreated` | Delivery Service | Order Service подтверждает заказ. |
| Создание доставки не удалось | failure | `DeliveryCreationFailed` | Delivery Service | Payment Service запускает компенсационный возврат; Order Service ждёт его результат. |
| Заказ подтверждён | domain | `OrderConfirmed` | Order Service | Notification Service уведомляет продавца; история заказа обновляется. |
| Заказ отменён из-за неоплаты | domain | `OrderCancelled` | Order Service | Delivery Service игнорирует/отменяет незавершённую доставку; история заказа обновляется. |
| Возврат средств выполнен | compensation | `PaymentRefunded` | Payment Service | Order Service окончательно отменяет заказ. |
| Возврат средств не выполнен автоматически | failure | `PaymentRefundFailed` | Payment Service | Order Service переводит заказ в `refund_pending`, дальше нужен повтор или ручная обработка. |
| Доставка отменена (если уже была создана) | compensation | `DeliveryCancelled` | Delivery Service | В истории заказа видно, что доставка была отменена как компенсация. |

## Состояния заказа

`new → payment_pending → paid → delivery_pending → confirmed` — успешный путь.

Ошибочные пути: `payment_pending → cancelled`; `delivery_pending → refund_pending → cancelled`. Статус `refund_pending` оставлен отдельно: деньги ещё не вернулись, значит заказ рано считать полностью отменённым.
