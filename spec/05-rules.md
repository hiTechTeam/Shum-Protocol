# 05. Правила приёма, состояния и очереди

Черновик v1, Shum-iOS `5b8efc8`. Источники в `Shum-iOS/ShumiOS/`:

- `Vendor/Messaging/ShumMessageStore.swift`: `receive`, `accept`, `carry`,
  `applyReceipt`, `receiveInvitation`, `receiveTyping`, `receivePresence`,
  `receiveReaction`, `receiveRetract`, `receiveProfile`, `send`, `tick`,
  все `route*`, `deleteConversation`, `setBlocked`, `retire`.
- `Vendor/Messaging/ShumConversationStore.swift`: `ShumContactsService`,
  `ShumConversationStore.transaction`, локальные модели состояния.
- `Vendor/Messaging/ShumIdentityService.swift`: `ShumContactCard.preferred`.

## 1. Доверие и входной шлюз

Декодирование, проверка подписи и принятие сообщения являются разными
этапами. Для BLE сырой ShumPacket <=24 000 байт. JSON должен декодироваться,
version=1 и ровно одно основное поле из раздела 04.

До dispatch отбрасываются:

- Удалённый профиль (`retired`).
- BLE-источник, чей аутентифицированный Noise-ключ даёт заблокированный ID.
- Nostr-источник, совпадающий с nostrKey заблокированной карточки.
- Заблокированная card либо любой участник envelope / receipt / invitation /
  typing / presence / retract / reaction. ProfileSync проверяет sender
  отдельно в своём обработчике.

Затем лимит по источнику и классу: 80 пакетов за 60 секунд. Источник:
`peer.id ?? nostrSender ?? "unknown"`, плюс `":" + trafficClass`. Классы:
`ephemeral` для typing/presence, `receipt` для receipt, `content` для остальных.
Не больше 1000 разных ключей лимитера; существующий ключ продолжает работать
при достижении предела. При elapsed >=60 с окно сбрасывается. Каждые 30 tick
старые записи удаляются. Не прошедшие проверку структуры не расходуют счётчик;
структурно корректный пакет с неверной подписью расходует его.

Ошибки недоверенного ввода молча отбрасываются, не открывают модальное окно.
Локальная ошибка сохранения/отправки сообщается через onError. Изменения
сообщения и отметки должны быть сохранены до ACK или сетевого действия.

## 2. Закреплённые ключи и карточки

Shum ID = SHA-256(noiseKey). Для существующей личности запрещена подмена
signingKey и nostrKey. Слияние карточек выполняется `preferred(over:)` по
разделу 02, затем победивший профиль обновляется одновременно в contacts,
encounters и requests. При seed аватаре старый сохранённый PNG удаляется.

BLE card принимается только при session.noiseKey == card.noiseKey. Она
сохраняет nearby и encounter; при первой карточке отправляется своя card.
Контакт в адресную книгу автоматически не добавляется. Nostr card требует
outer-authenticated sender == card.nostrKey: известному контакту обновляет
профиль, незнакомому создаёт legacy-запрос при фазе ready.

Добавление по QR создаёт проверенный контакт и разговор с фазой ready,
**не разрешает переписку**. Только `canMessage == (phase == accepted)`.
Адресная книга (`metadata.addressBook != "false"`) и разрешение на сообщения
являются независимыми состояниями. Исключение source="preview" служебное:
fixtures сразу получают accepted; обычный CLI его не использует.

Лимиты: контакты 2000, запросы 20, заблокированные 2000. Encounter виден
24 часа с lastSeen; это история обнаружения, а не доказательство текущего
подключения. Nearby очищается по фактическому набору подключённых peers.

## 3. Приём сообщения своему получателю

1. Проверить envelope по разделу 03. При Nostr дополнительно sender.nostrKey
   должен совпасть с аутентифицированным автором seal.
2. Получатель должен совпасть с ownCard по всем трём ключам. При BLE
   1 <= hopCount <= hopLimit; при Nostr hopCount игнорируется и записывается 0.
3. Активный deletedMessageIDs[id] блокирует повтор даже правильного сообщения.
4. Если уже есть входящее сообщение с тем же id и sender.id, сверить
   закреплённые signingKey/nostrKey контакта, digest и accepted. При совпадении
   повторить ACK без открытия ciphertext, не менять текст и порядок истории.
5. Иначе открыть Noise X; возвращённый static sender должен в точности
   совпасть с envelope.sender.noiseKey.
6. Декодировать ShumPlaintext, сверить protocolName (current/legacy), id,
   conversationID, senderID, recipientID, timestamp и expiresAt с envelope.
   Проверить text и reply по разделу 03.
7. Незнакомому автору создать запрос, но само сообщение не сохранить. Для
   известного контакта сверить закреплённые ключи, проверить accepted и
   слить профиль. Другой ciphertext с существующим id отбрасывается.
8. Создать разговор при необходимости, сохранить сообщение с delivered,
   outgoing=false. unread=false только когда приложение на переднем плане
   и именно этот чат открыт; иначе true. Поле транспорта: nostr / mesh / ble.
9. Создать подписанный receipt. read=true только для видимого активного чата.
   Если есть BLE-путь отправителю, выдать немедленный ACK после сохранения.

История не пересортировывается receipt или профильными сигналами. Повторы
не создают второй bubble. ID чувствителен к регистру как строка, даже если
два значения UUID представляют одинаковые 16 байт.

## 4. Курьерское хранение

Конверт не своему адресату можно принять только от verified nearby BLE
депонента с совпадающим session Noise-ключом. Через Nostr курьерский вход
не создаётся. Проверяются 1 <= hopCount < hopLimit, отсутствие id в seenRelay,
существующей relay-copy и достаточного receipt. Максимум 64 конверта,
8 на depositor, 4000 seenRelay ID. Сохранённый seenRelay истекает вместе с
конвертом; уже известное событие не принимается повторно после удаления copy.

Курьер не открывает ciphertext и не изменяет подписанный envelope. При
пересылке увеличивает внешний hopCount на 1. Своему прямому адресату
повторяет не чаще 30 с. Без прямого адресата предлагает не больше трём
новым соседям, исключая исходного автора и адресата, сортируя по peer.id.
За один routeRelay не больше 8 передач. Отметка настоящего получателя
удаляет соответствующую relay-copy, но может передаваться дальше по графу.

## 5. Приглашения

Фазы: ready, outgoingPending, incomingPending, accepted, declinedByPeer,
declinedLocally. При отсутствии записи, но наличии requests используется
incomingPending, иначе ready. Локальные действия:

| Действие | Допустимая фаза | Новая фаза |
|---|---|---|
| sendInvitation | ready | outgoingPending |
| acceptInvitation | incomingPending или declinedLocally | accepted |
| declineInvitation | incomingPending | declinedLocally |

При действии контакт/разговор создаётся при необходимости; подписанный
control и изменение фазы фиксируются вместе. Для одного адресата остаётся
последнее локальное invitation в outbox. Локальное время строго больше
предыдущего updatedAt и при необходимости времени peer-сигнала.

При входе validate и все три ключа recipient обязаны соответствовать своим.
Ключи sender привязываются к BLE/Nostr как в разделе 04. Профиль sender
сливается до обработки фазы. Точный повтор последнего eventID игнорируется.
Вложение avatar сейчас игнорируется, сохраняется nil.

| Входное action | Допустимая текущая фаза | Результат |
|---|---|---|
| request | ready, outgoingPending, declinedByPeer, legacy incomingPending | incomingPending, requests <=20, удалить свой request из outbox |
| accept | outgoingPending, declinedByPeer, accepted, incomingPending с доказанным своим request | accepted, создать контакт/разговор, убрать requests и свой request |
| decline | outgoingPending, declinedByPeer, incomingPending с доказанным своим request | declinedByPeer, убрать requests и свой request |

Исключения:

- Одновременные request: если уже outgoingPending и own.id < sender.id,
  чужой request игнорируется. Меньший стабильный ID остаётся инициатором.
- При accepted новый request с timestamp > текущего updatedAt вызывает
  повтор собственного accept со временем больше обеих сторон. Старый
  request не действует. Это восстановление после старой локальной очистки.
- Legacy incomingPending распознаётся по eventID с префиксом `legacy-`.
  Для accept дополнительно восстанавливается ошибочно перевёрнутый legacy
  запрос, если metadata.source контакта == "chat-invitation".
- В остальных ветках общего сравнения `(timestamp, eventID)` **нет**.
  Нельзя переносить сюда newest-wins реакций. Старая допустимая accept может
  перезаписать updatedAt уже accepted состояния; см. вопросы.

Legacy card создаёт запрос только из ready, максимум 20. updatedAt=now,
eventID=`legacy-` + UUID; собственные request удаляются. Обычная карточка
из nearby не равнозначна принятию приглашения.

## 6. Typing, presence и реакции

У всех трёх: правильный recipient по трём ключам, sender соответствует
транспорту, известный контакт с закреплёнными ключами и accepted. Typing /
presence сверяют также закреплённый nostrKey; reaction явно сверяет noise /
signing, а Nostr автор отдельно проверяется перед этим.

Новизна: timestamp больше предыдущего либо равен и строковый id
лексикографически больше. Равный/меньший сигнал игнорируется. Это сравнение
строк UUID, без преобразования в 16 байт. Typing/presence хранят новизну в
памяти процесса; реакции постоянно в базе.

- Typing=true устанавливает typing deadline=expiresAt и также продлевает
  присутствие до now+40 с. false снимает typing. Истечение очищается tick.
- Presence=true устанавливает deadline=expiresAt, false снимает presence.
- Реакция может предшествовать сообщению. Если оно уже есть, conversationID
  должен совпасть с разговором этих двух участников. Не больше 20 000 разных
  messageID в словаре реакций. nil сохраняется с timestamp/eventID и не
  позволяет старой реакции воскреснуть. Отдельной проверки deletedMessageIDs
  в receiveReaction нет.
- Локальный toggle той же реакции снимает её, новой заменяет; timestamp
  строго возрастает, в outbox остаётся последний выбор на messageID.
  Push только на входящее сообщение и только если выбор сохранился 5 секунд.

## 7. Отзыв и локальная очистка

`cancelSending` доступен только своему queued/forwarding. Если сообщение
уже покидало устройство (attempts>0, nostrAccepted, lastNostrAttempt или
forwardedTo), создаётся retract до его исходного expiresAt. Иначе достаточно
локального удаления. Сообщение и связанные реакции удаляются вместе.

Приём retract проверяет recipient, transport, известные noise/signing ключи.
accepted здесь **не требуется**. Если сообщение существует, оно должно быть
входящим от этого sender. Без существующего сообщения допустим tombstone.
Удаляются сообщение, его receipts и реакции; deletedMessageIDs[id]=expiresAt.
Поздний старый retract может заменить более далёкую expiry более короткой:
в этой ветке не используется max или newest-wins.

Локальный clear сохраняет контакт и фазу accepted. Для известных ещё не
истёкших сообщений ставит tombstones до их envelope.expiresAt, удаляет историю
разговора, реакции, reactionOutbox адресата, legacy-историю, receipts сторон
и conversation. Не отправляет peer-control. Новый ID от retained accepted
контакта может снова создать разговор. removeContact дополнительно убирает
контакт, requests, invitationState/outbox, profileOutbox и закрепления.

## 8. Блокировка и завершение профиля

Блокировка сохраняет проверенную карточку в blocked. Убирает requests,
invitation/profile/reaction outbox, relay с участником или depositor,
receipts сторон и pin в каталогах. Свои queued/forwarding этому контакту
помечает cancelled. Существующую историю и invitationState сохраняет.
Nearby-карточка удаляется. Разблокировка удаляет только blocked запись.

`retire` посылает offline-presence, останавливает интернет, дренирует его
постоянные записи, очищает callbacks и ephemeral-кэши. Все поздние callbacks
проверяют retired и не должны восстанавливать удалённые данные профиля.

## 9. Отметки и порядок доставки

Receipt сначала проходит validate из раздела 04. Для своего исходящего
проверяются digest, исходный recipient.id/signingKey и destination.id==own.id.
Подпись одного постороннего узла не даёт права удалить курьерский конверт.

Доказательство сопоставляется с исходным message или relay-copy по digest,
recipient.id/signingKey и sender.id==destination.id. Если original уже нет,
допускается только read-upgrade существующего такого же проверенного receipt.
Непрошеный receipt без original и без предыдущего доказательства отбрасывается.

Не больше 2000 receipts, либо обновление существующего key. Повтор false
не заменяет false/true, повтор true не заменяет true. Read монотонен: после
read доставка false не понижает статус. Сохранение receipt и удаление relay
атомарны; своему исходящему записываются deliveredAt/readAt из timestamp.
MarkRead меняет флаги и создаёт ACK пакетно, сохраняя исходный порядок сообщений.
После expiry сообщения новые receipts не создаются, локальный unread снимается.

## 10. Очереди и повторы

Все счётчики / время попыток сохраняются **до** передачи. У Nostr отдельные
lastNostrAttempt / nostrAttempts. Общая задержка Nostr:

```text
attempts nil/0: 0 секунд
attempts >=1: min(60, 5 * 2^min(attempts - 1, 4))
# 5, 10, 20, 40, 60, 60, ...
```

| Очередь | За проход | BLE | Nostr / завершение |
|---|---:|---|---|
| Свои messages | 4 | min(60, 10*max(1,attempts)) с; прямой адресат, иначе до 3 разных курьеров | До OK релея, потом ждать receipt |
| relay | 8 | Прямому адресату каждые 30 с; иначе до 3 новых соседей | Через Nostr курьер не пересылает |
| receipts | 8 | Каждые 30 с, прямому destination и до 8 новых соседей | Только собственный ACK, до OK |
| invitation | Первые 4 | 10, 20, 40, затем 60 с; для ответов максимум 6 предложений | До OK / expiry; request не ограничен 6 BLE-попытками |
| retract/reaction | Первые 8 | Не чаще 10 с, максимум 6 предложений | До OK / expiry |
| profileOutbox | 4 | Когда доступен прямой peer | Повтор до подписанного ACK profileID |

Для invitation задержка BLE = min(60, 10 * 2^min(max(attempts - 1, 0), 3)) с.
Request продолжает BLE-повторы без лимита 6; accept/decline ограничены 6.
Это отдельная ветка `routeInvitationControls`, не generic `route`.

Для profileOutbox задержка min(3600, 5 * 2^min(attempts,10)) с, attempts
ограничен 20. Подтверждение релея очередь профиля не завершает.

Своя message после BLE/courier передачи становится forwarding с transport
ble/mesh; после принятия релеем forwarding/nostr. **Доставлено** только после
recipient receipt. Если одновременно доступен BLE, push удерживается 8 с
в ожидании ACK; иначе после OK релея. ACK до окончания задержки отменяет push.

Tick удаляет истёкшие relay/receipts/controls, queued/forwarding message
помечает expired. История delivered/read не удаляется по сроку. Resend
expired создаёт новое сообщение в конце, удаляя старое после успешного send.
SeenRelay/deleted tombstones очищаются в обслуживании tick. Обычная очередь
своих недоставленных ограничена 200 сообщениями, текст после trim <=4096 байт.

## 11. Синхронизация профиля

Единый head во всех транспортах: revision / profileID из раздела 02. Своя
смена имени/bio/seed повышает UInt64 revision на 1 (на max ошибка), сохраняет
ownProfileCard и заменяет profileOutbox для всех незаблокированных контактов.
Инициализация восстанавливает сохранённый head, проверяя все свои ключи.

Входной sync требует recipientID==own.id, известного незаблокированного
контакта, совпадения sender с BLE Noise или authenticated Nostr. Accepted
для профиля не требуется. sender сливается через pinned-key preferred.

Если knownRecipient описывает тот же собственный profileID, удаляется
соответствующий pending ACK. Если peer сохранил более новый / предпочтительный
собственный head после отката бэкапа, iOS сохраняет свой локальный выбор
имени/bio/seed, заново подписывая его с revision > peer.revision. requestsReply
вызывает ответ с reply=false по входному транспорту. Срока у profileSync нет.

## Тестовые примеры

Генератор использует настоящий ShumMessageStore, Noise X и подписи Swift,
фиксированное время, собственные in-memory keychain/store/transport. Радио
и Nostr-соединения не запускаются. Случайные UUID исходящей очереди записаны
в fixture и становятся конкретным входом для Rust, не ожидаются повторно.

| Файл | Сценарии |
|---|---|
| `05-delivery.json` | accepted/pending/blocked, sender pinning, legacy/plaintext/reply границы, дубли, conflicting digest, clear и повтор |
| `05-invitations.json` | 18 переходов фаз, tie одновременных запросов по стабильному ID |
| `05-newest-signals.json` | Порядок реакций, removal tombstone, typing/presence с равным timestamp и разными UUID |
| `05-invitation-retries.json` | BLE-расписание request/decline, лимит ответов 6, продолжение request |
| `05-outbox.json` | Реальные интервалы повторов, OK релея против receipt, delivered → read без отката |

## Вопросы

При реализации Rust исправлена ошибка прежнего черновика: для invitation
были указаны 8 пакетов и постоянные 10 с, но текущий Swift задаёт первые 4
и экспоненциальный BLE-интервал. Контракт iOS не менялся. Отдельный
Swift-пример `05-invitation-retries.json` проверяет actual schedule.

1. **Invitation не имеет общего newest-wins.** Помимо специального request
   в accepted и exact eventID, timestamp не отсекает старые accept/decline.
   Это потенциальный откат метаданных состояния. В v1 переносить фактические
   ветки; изменение алгоритма отдельно согласовать с iOS.
2. **Retract может укоротить tombstone.** Поздний корректно подписанный отзыв
   записывает expiry без max. Не исправлять незаметно в описании v1.
3. **Reaction после удаления.** При отсутствии сообщения реакция принимается
   заранее, даже если его ID уже есть в deletedMessageIDs. Сама история
   сообщения остаётся удалённой, но словарь реакций может пополниться снова.
4. **Ephemeral порядок не постоянен.** После перезапуска допустим ещё не
   истёкший старый typing/presence. Нужна оценка UX, без смены wire v1.
5. **Нет синхронизации локальной очистки между устройствами.** Контракт v1
   сохраняет accepted, удаляет историю только здесь. Новые устройства и
   clear-all требуют согласованного v2.
