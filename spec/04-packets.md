# 04. Виды пакетов

Черновик v1, Shum-iOS `5b8efc8`. Источники в `Shum-iOS/ShumiOS/`:

- `Vendor/Messaging/ShumConversationStore.swift`: структуры пакетов и `validate`.
- `Vendor/Messaging/ShumIdentityService.swift`: `ShumCoding`.
- `Vendor/Messaging/ShumMessageStore.swift`: `receive`, `receiveProfile`,
  `transitionInvitation`, `receiveInvitation`, `setTyping`, `sendPresence`,
  `receiveRetract`, `receiveReaction`, `clearDirectoryEntry`.
- `Vendor/Messaging/ShumProfiles.swift`: `ShumProfile`, `ShumProfileManifest`,
  `ShumProfilePacket`, `ShumProfiles.receive`.

## 1. Общая упаковка ShumPacket

UTF-8 JSON, `version` обязательное целое 1. Остальные поля необязательны.
При кодировании `nil` опускается; при чтении JSON `null` также означает `nil`.
Неизвестные поля `Codable` игнорирует. Отсутствующее обязательное поле не
получает значение из объявления Swift: например, отсутствие `version`
в декодируемом пакете является ошибкой.

Основные поля: `card`, `envelope`, `receipt`, `invitation`, `typing`,
`presence`, `profileSync`, `retract`, `reaction`. В пакете должно быть
**ровно одно** из них. Это проверяет `ShumMessageStore.receive`, а не
`JSONDecoder` и не структура `ShumPacket` сама по себе.

Вспомогательные поля `hopCount` (целое) и `invitationAvatar` (объект) не
участвуют в этом подсчёте. `hopCount` используется только с `envelope`;
`invitationAvatar` передаётся в обработчик приглашения, который сейчас
игнорирует вложение. Постороннее вспомогательное поле само по себе не
делает пакет недействительным. Клиент должен создавать только осмысленные
комбинации, сохраняя фактическую совместимость при чтении.

Порядок полей в `ShumCoding`: `card`, `envelope`, `hopCount`, `invitation`,
`invitationAvatar`, `presence`, `profileSync`, `reaction`, `receipt`,
`retract`, `typing`, `version`, с пропуском отсутствующих. Сортируются также
все вложенные объекты по разделу 02. У всего пакета общей подписи нет:
каждое содержимое подписано отдельно. Предел декодированных байтов пакета
**24 000**, и для BLE, и для Shum-содержимого Nostr.

| Поле | Содержимое | Кто подписывает |
|---|---|---|
| `card` | `ShumContactCard`, раздел 02 | Владелец карточки |
| `envelope` | `ShumEnvelope`, раздел 03 | Отправитель сообщения |
| `receipt` | `ShumReceipt` | Получатель исходного сообщения |
| `invitation` | `ShumInvitationControl` | Отправитель действия |
| `typing` | `ShumTypingControl` | Набирающий текст |
| `presence` | `ShumPresenceControl` | Открывший / закрывший чат |
| `profileSync` | `ShumProfileSync` | Владелец передаваемого профиля |
| `retract` | `ShumRetractControl` | Автор отзываемого сообщения |
| `reaction` | `ShumReactionControl` | Автор реакции |

Карточки внутри подписанного объекта входят в подписываемые байты целиком,
включая собственные подписи. Дополнительная привязка к аутентифицированному
отправителю транспорта и сохранённым ключам описана в разделе 05.

## 2. Общий формат управляющих сообщений

У `invitation`, `typing`, `presence`, `retract`, `reaction` обязательны:

| Поле | Тип |
|---|---|
| `version` | целое, ровно 1 |
| `id` | строка UUID по `UUID(uuidString:)` |
| `sender` | карточка отправителя |
| `recipient` | карточка получателя |
| `timestamp` | Int64, Unix в миллисекундах |
| `expiresAt` | Int64, Unix в миллисекундах |
| `signature` | Data, стандартный base64, подпись Ed25519 |

Подпись: `Ed25519(sender.signingKey, ShumCoding(object с signature = ""))`.
У этих пяти типов метки домена нет. Поля каждого объекта сортируются по
имени, а не в порядке объявления в таблице. Строки UUID сохраняют регистр.

Для всех пяти проверяются обе карточки, разные `sender.id` / `recipient.id`,
`timestamp >= 0`, `timestamp <= now_ms + допустимое_будущее`,
`expiresAt > now_ms`, `expiresAt > timestamp`, максимальная разница времени
из следующей таблицы и подпись отправителя.

| Тип | Будущее, максимум | Срок от timestamp, максимум | Создание в iOS |
|---|---:|---:|---:|
| invitation | 300 000 мс | 2 592 000 000 мс (30 суток) | 30 суток |
| typing | 30 000 мс | 10 000 мс | 8 000 мс |
| presence | 30 000 мс | 90 000 мс | 40 000 мс |
| retract | 300 000 мс | 86 400 000 мс | До срока исходного сообщения |
| reaction | 300 000 мс | 86 400 000 мс | 24 часа |

`now_ms = Int64(now.timeIntervalSince1970 * 1000)`. Максимумы включительны.
Проверка подписи не означает, что действие допустимо в текущем состоянии.

### 2.1. Приглашение и ответ

Дополнительное обязательное `action`: `request`, `accept` или `decline`.
Другие строки не декодируются в enum. Порядок: `action`, `expiresAt`, `id`,
`recipient`, `sender`, `signature`, `timestamp`, `version`.

Создающий код ставит время больше предыдущего локального изменения:
`max(now_ms, previous.updatedAt + 1, afterTimestamp + 1)`. Состояния и
одновременные приглашения описаны в разделе 05. Для совместимости со
старыми сборками отправляются также пакеты с собственной `card`; это
дополнительный сигнал, а не другой формат подписи invitation.

### 2.2. Набор текста

Дополнительное обязательное `isTyping`: JSON bool. Порядок: `expiresAt`,
`id`, `isTyping`, `recipient`, `sender`, `signature`, `timestamp`, `version`.

Отправляется по доступным BLE и Nostr без постоянной очереди. Повтор `true`
не чаще раза в 4 секунды; первое `false` без предшествующего `true` не
отправляется. Приём `true` действует до `expiresAt`, `false` снимает отметку.

### 2.3. Присутствие в чате

Дополнительное обязательное `isOnline`: JSON bool. Порядок: `expiresAt`,
`id`, `isOnline`, `recipient`, `sender`, `signature`, `timestamp`, `version`.

Это открытая переписка с данным контактом, а не общий сетевой online.
Положительный сигнал обновляется каждые 25 секунд при сроке 40 секунд.
Отрицательный сигнал снимает отметку. Доставка ephemeral, без исторической
очереди. При уходе в фон ожидается flush релея не больше 4 секунд.

### 2.4. Отзыв сообщения

Дополнительное обязательное `messageID`: строка UUID. Порядок: `expiresAt`,
`id`, `messageID`, `recipient`, `sender`, `signature`, `timestamp`, `version`.

В iOS предназначен для отмены своего исходящего сообщения, которое ещё
не доставлено. Получатель может сохранить tombstone до прихода исходного
сообщения. Отзыв не является командой очистки всего чата. У очереди
управления собственный `id`, отличный от `messageID`.

### 2.5. Реакция

Дополнительные поля: обязательный `messageID` (UUID), необязательный
`reaction`: одна из строк `heart`, `like`, `dislike`, `laugh`, `fire`,
`coffin`, `hundred`, `horror`. Отсутствующее / `null` поле означает снять
реакцию. Swift при создании снятия опускает поле.

Порядок: `expiresAt`, `id`, `messageID`, `reaction` при наличии, `recipient`,
`sender`, `signature`, `timestamp`, `version`. По одному актуальному значению
на пару messageID / автор. Удаление реакции сохраняется как tombstone с
временем и ID сигнала. Правила порядка в разделе 05.

## 3. Отметка доставки / прочтения ShumReceipt

У этого типа **нет поля version и собственного UUID id**.

| Поле | Тип / проверка validate |
|---|---|
| `envelopeID` | UUID исходного сообщения |
| `digest` | Строка из 64 Swift Characters; hex здесь не проверяется |
| `sender` | Карточка получателя исходного сообщения |
| `destination` | Карточка автора исходного сообщения |
| `read` | bool, false = доставлено, true = прочитано |
| `timestamp` | Int64, >= 0 и <= now_ms + 300 000 |
| `expiresAt` | Int64, > now_ms и <= now_ms + 86 700 000 |
| `signature` | Ed25519 sender, canonical JSON с пустой signature |

Порядок: `destination`, `digest`, `envelopeID`, `expiresAt`, `read`, `sender`,
`signature`, `timestamp`. Метки домена нет. Обе карточки проверяются.

`validate` не проверяет `expiresAt > timestamp`, срок относительно времени
создания и различие sender/destination. При использовании отметки для
исходящего сообщения обработчик дополнительно сравнивает фактический
SHA-256 ciphertext, ID и ключи сторон. Это существенно: строка из 64 `z`
проходит validate, но не соответствует digest настоящего сообщения.

Ключ хранения: `envelopeID + ":" + digest + ":" + sender.id`. Прочтение
повышает ранее сохранённую доставку. ACK получателя отличается от ответа
Nostr-релея `OK`: `OK` сам по себе не означает доставку или прочтение.

## 4. Подписанная синхронизация профиля ShumProfileSync

| Поле | Тип |
|---|---|
| `sender` | Полная карточка с profileRevision |
| `recipientID` | Строка Shum ID адресата |
| `knownRecipient` | Необязательная карточка адресата |
| `requestsReply` | bool |
| `signature` | Data |

Нет version, id, timestamp и expiresAt. Порядок полей: `knownRecipient`
при наличии, `recipientID`, `requestsReply`, `sender`, `signature`.

```text
signing_bytes = UTF8("shum.profile-sync.v1\0")
             || canonical_json(sync с signature = "")
```

Метка включает завершающий `00`. Требуются действительная карточка sender,
наличие profileRevision и 64-байтовая действительная подпись. Если есть
knownRecipient, проверяется её карточка и `knownRecipient.id == recipientID`.
Без knownRecipient сама validate не проверяет формат / непустоту recipientID.
Роутинг требует собственного ID получателя и закреплённых ключей контакта.

Ответ содержит свою карточку и известную карточку собеседника. Это ACK
конкретного profileID, а не подтверждение релея. requestsReply управляет
ответом, предотвращая бесконечный обмен; детали слияния в разделе 05.

## 5. Отдельный аватар приглашения

Необязательный `ShumInvitationAvatar` в поле invitationAvatar:
обязательные `data` (Data) и `signature` (Data), оба base64.

```text
signing_bytes = UTF8("shum.invitation-avatar.v1\0")
             || invitation.signature
             || data
```

Проверяющая функция требует action != decline, размер <= 8192 байта и
`ShumProfile.validAvatar`: одно декодируемое изображение, 1…360 пикселей по
каждой стороне, общий предел 40 KiB. Пустое изображение недействительно.
Подпись Ed25519 проверяется ключом invitation.sender. Подпись invitation
включена сырыми 64 байтами, не как base64.

Текущий `receiveInvitation` явно игнорирует это вложение и хранит avatar=nil;
`transitionInvitation` создаёт приглашения без него. Возможность проверить
структуру не означает, что текущий iPhone показывает переданное фото.

## 6. Профиль по BLE, отдельный канал

`ShumProfilePacket` не вложен в ShumPacket: у него свой Noise payload type
и предел **6144** байта. Это JSONEncoder JSON без требования canonical для
wire-пакета; подписи на нём нет, доверие от аутентифицированной BLE-сессии.
Поля: обязательные `version` (1), `kind`, `request` (строка, <=64 UTF-8 байт),
необязательные `manifest`, `hash`, `offset` (целое), `data` (Data).

| kind | Создаваемые поля / действие текущего receive |
|---|---|
| query | Непустой request; ответ manifest с тем же request |
| changed | Обычно пустой request; новый query, ограничение подсказок 2 с |
| manifest | Совпадающий ожидаемый request, действительный manifest |
| chunkRequest | hash, offset; обработчик сразу возвращается |
| chunk | hash, offset, data; обработчик сразу возвращается |

Объявлен размер части 3072 байта, но передача частей сейчас не реализована.
Сервис допускает максимум 100 peers и 20 пакетов в секунду на peer.

### 6.1. ShumProfileManifest

Обязательные поля: `name`, `bio` (строки), `avatarBytes` (целое).
Необязательные: `avatarHash` (строка), `avatarSeed` (UInt64), `avatarVersion`
(целое). Канонический порядок: `avatarBytes`, `avatarHash`, `avatarSeed`,
`avatarVersion`, `bio`, `name`, с пропуском nil.

Имя должно в точности совпасть с `InputValidator.validateNickname(name)`:
непустое после Foundation trim, <=50 графем, без controlCharacters, в NFC;
дополнительно занимать <=64 байт. bio <=72 графем, <=640 UTF-8 байт, не содержит управляющих
Unicode scalars кроме CharacterSet.newlines. Это строже карточки раздела 02.

- С seed: avatarVersion=1, avatarHash отсутствует, avatarBytes=0. PNG
  генерируется локально, размер и хэш PNG не входят в manifest.
- Без seed: avatarVersion отсутствует. Если есть avatarHash, это ровно 64
  ASCII `[0-9a-f]` и avatarBytes в диапазоне 1…40960; без hash avatarBytes=0.

```text
revision = hex(SHA256(UTF8(name + "\0" + bio + "\0"
    + (avatarHash ?? "") + "\0" + (decimal(avatarSeed) ?? "") + "\0"
    + (decimal(avatarVersion) ?? ""))))
```

Четыре разделителя NUL, без завершающего NUL. UInt64 пишется десятичным без
округления. Это другой revision, не profileRevision и не profileID карточки.
Получение manifest без seed сейчас сохраняет только name и bio; фото по
hash не запрашивается. Сам ShumProfile имеет name, bio, optional avatar Data,
optional avatarSeed и является локальной моделью, а не самостоятельным
подписанным сетевым пакетом.

## 7. Очистка чата

В v1 отдельного пакета очистки нет. `clearDirectoryEntry` вызывает локальный
`deleteConversation`, удаляет историю и отметки, сохраняет контакт и фазу
приглашения, создаёт временные tombstones известных сообщений. Другой
участник ничего не получает. Удаление контакта является отдельным локальным
режимом той же функции. Вариант «на всех устройствах» требует v2.

## Тестовые примеры

Генератор: `ShumProtocolVectorTests.controlPackets` и `profilePackets`.

| Файл | Проверяемое содержимое |
|---|---|
| `04-controls.json` | Все 6 подписанных controls, точные байты и пакеты, действия, реакции, удаление, границы сроков и подписей |
| `04-profile-sync.json` | Подписанные profileSync с/без knownRecipient, ошибочный адресат, слабая проверка recipientID |
| `04-invitation-avatar.json` | Реальный PNG, байты отдельной подписи, результат проверки и запрет decline |
| `04-profile-packets.json` | Все kind профиля, manifest, revision, UInt64.max, фото и недействительное имя |

## Вопросы

1. **Фото-аватары по BLE отключены.** Объявленные chunk/chunkRequest
   игнорируются, manifest фото не приводит к загрузке. Заявленная функция
   CLI `profile avatar --photo` не сможет передать фото текущему iPhone
   только реализацией Rust. Нужна отдельно согласованная доработка iOS.
2. **Очистка только локальная.** В v1 нет peer-clear. Проверка с iPhone должна
   удостоверять локальную очистку и защиту от повторов. Нельзя выдавать её
   за синхронную очистку у собеседника.
3. **Receipt.validate слабее остальных controls.** Нет hex-проверки digest
   и проверки expiresAt > timestamp. Сохранить поведение для совместимости;
   дополнительные проверки фактического сообщения обязательны.
4. **ProfileSync не имеет срока.** Старый корректно подписанный профиль
   защищён от отката правилами revision, а не expiry. Сам recipientID без
   knownRecipient может быть произвольным; handler обязан проверять адресацию.
5. **Разные валидаторы профиля и карточки.** Profile.valid запрещает часть
   control scalars и требует нормализованного validateNickname имени,
   Card.validate использует другие условия. Нельзя заменять один другим.
