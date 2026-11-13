# Контракты
- [OpenAPI 3.1: клиентский API](openapi.json).
- [AsyncAPI 3.0: команды и результаты Saga](asyncapi.json).
- [OpenAPI 3.1: требования к внешним адаптерам](external-adapters.openapi.json).

JSON является официально допустимым представлением обеих спецификаций. Это машинно-читаемые контракты, а не реализация приложения.

## Синхронные вызовы
Клиент → Fleet: GET /vehicles.
Клиент → Rental: POST /rentals, GET /rentals/{rentalId}.
Payment → адаптер банка: PUT/GET /holds/{operationId}, POST .../release.
Access → адаптер телематики: PUT/GET /access-sessions/{sessionId}, POST .../revoke.
OIDC/JWKS — внешний стандартный протокол Identity Provider, не собственный API проекта.

## Асинхронные сообщения
- ReserveVehicle: Rental → Fleet; адрес fleet.ReserveVehicle.v1.
- ReleaseVehicle: Rental → Fleet; адрес fleet.ReleaseVehicle.v1.
- HoldDeposit: Rental → Payment; адрес payment.HoldDeposit.v1.
- ReleaseDeposit: Rental → Payment; адрес payment.ReleaseDeposit.v1.
- GrantAccess: Rental → Access; адрес access.GrantAccess.v1.
- RevokeAccess: Rental → Access; адрес access.RevokeAccess.v1.
- VehicleReserved: Fleet → Rental; адрес fleet.VehicleReserved.v1.
- VehicleReservationRejected: Fleet → Rental; адрес fleet.VehicleReservationRejected.v1.
- VehicleReleased: Fleet → Rental; адрес fleet.VehicleReleased.v1.
- DepositHeld: Payment → Rental; адрес payment.DepositHeld.v1.
- DepositRejected: Payment → Rental; адрес payment.DepositRejected.v1.
- DepositUncertain: Payment → Rental; адрес payment.DepositUncertain.v1.
- DepositReleased: Payment → Rental; адрес payment.DepositReleased.v1.
- AccessGranted: Access → Rental; адрес access.AccessGranted.v1.
- AccessRejected: Access → Rental; адрес access.AccessRejected.v1.
- AccessUncertain: Access → Rental; адрес access.AccessUncertain.v1.
- AccessRevoked: Access → Rental; адрес access.AccessRevoked.v1.

## Семантика
operationId результата равен идентификатору вызвавшей его операции. correlationId всегда rentalId, sessionId тоже rentalId. causationId — messageId исходного сообщения либо идентификатор HTTP-запроса. Новое сообщение-сверка может иметь новый messageId при прежнем operationId. schemaVersion=1.

notAfter ограничивает выполнение команд: после срока нельзя начинать новую выдачу/холд/резерв. Истёкшие компенсации не исполняются автоматически из старого сообщения: Rental отправляет новую актуальную команду с прежним operationId и новым notAfter. Неизвестный результат уже выполненной операции всё равно проверяется. GrantAccess.validUntil дополнительно ограничивает исполнение на устройстве.

DepositUncertain и AccessUncertain не завершают операцию: worker продолжает сверку. После них допустимо итоговое событие с тем же operationId. Inbox дедуплицирует messageId, а реестр операций — логический эффект; нельзя отбрасывать уточнённый результат только из-за совпадающего operationId.
AccessRejected означает доказанный отказ без выдачи. После отправки grant Rental всё равно закрывает sessionId через RevokeAccess, чтобы исключить поздние повторы.
AccessGranted во время CANCELLING не отменяет компенсацию. После CLOSED запоздалое событие не восстанавливает сессию.
ReleaseVehicle с allocationId освобождает только совпадающее выделение; без allocationId ищет по correlationId и фиксирует отмену rentalId. Чужое выделение не затрагивается.
Для ReleaseDeposit используется исходный holdOperationId. DepositReleased — подтверждение отсутствия действующего холда и запрета позднего открытия.
Неустранимаемые несоответствия контракта попадают в DLQ; процесс затем требует проверки, а не фиктивного успеха.

## Версии и ошибки
Несовместимые изменения требуют v2 канала/схемы. Существующие поля и семантика v1 не изменяются. В v1 additionalProperties=false: расширения публикуются согласованно либо новой версией.
HTTP коды: 400 — формат; 401 — JWT; 403 — нет допуска; 404 — нет/чужой объект; 409 — конфликт; 429 — ограничение; 503 — недоступность. Типовые Error.code: INVALID_REQUEST, UNAUTHORIZED, DRIVER_NOT_ELIGIBLE, NOT_FOUND, ACTIVE_RENTAL_EXISTS, IDEMPOTENCY_CONFLICT, RATE_LIMITED, UNAVAILABLE.

