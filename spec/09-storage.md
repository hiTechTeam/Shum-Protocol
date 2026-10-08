# 09. Зашифрованное SQLite-хранилище

Источник: iOS `5b8efc8`,
`ShumiOS/Vendor/Messaging/ShumSQLitePersistence.swift`,
`ShumConversationStore.swift`, `ShumIdentityService.swift`.
Это локальный формат хранения. Он не определяет сетевую передачу базы.
Генератор `encryptedSQLite()` использует настоящий ShumSQLitePersistence
с тестовыми ключами. Фикстура `vectors/09-storage.json` содержит полный файл
SQLite в Base64, строки до/после расшифрования, состояние и проверки отказа.

## 1. Ключ и владелец

Storage key: независимо сгенерированные случайные 32 байта, не производная
Noise, signing или Nostr key. iOS хранит его в Keychain под
`shumStorageKey`, при миграции читает `spotchatStorageKey`.
Owner ID: lowercase hex SHA-256(Noise public key), раздел 01.
Каждая база принадлежит одному ownerID. Чужой ownerID и версия состояния,
отличная от 1, приводят к unavailableIdentity.

Нельзя использовать ключ сетевого шифрования в качестве storage key.
Секреты личности в таблицу records автоматически не записываются.

## 2. SQLite schema и режим соединения

```sql
CREATE TABLE records (
  bucket TEXT NOT NULL,
  id TEXT NOT NULL,
  position INTEGER NOT NULL,
  payload BLOB NOT NULL,
  PRIMARY KEY(bucket,id)
) WITHOUT ROWID;
PRAGMA user_version=1;
```

Соединение READWRITE | CREATE | FULLMUTEX. Режим WAL, synchronous FULL,
secure_delete ON, busy_timeout 1000 ms. Все writers и checkpoints одного
standardized pathname сериализуются. READ транзакция BEGIN DEFERRED,
запись BEGIN IMMEDIATE. close делает WAL checkpoint TRUNCATE.

Незашифрованы bucket, скрытый id, position, размеры и структура SQLite.
JSON состояния, настоящие ID и тексты находятся внутри payload AEAD.
Это шифрование записей, не SQLCipher. WAL содержит те же ciphertext.
Для переноса открытой базы нужно корректно учесть WAL; копировать один
основной файл до checkpoint недостаточно.

## 3. Скрытый индекс, AAD и упаковка

Для обычной строки:

```text
indexID = lowerhex(HMAC-SHA256(storageKey, UTF8(bucket + NUL + realID)))
AAD = UTF8(bucket + NUL + indexID + NUL + decimal(position))
packed = lengthBE16(realID.utf8) || realID.utf8 || payloadJSON
cipher = ChaCha20-Poly1305(storageKey, randomNonce12, packed, AAD)
SQLite.payload = nonce12 || ciphertext || tag16
```

NUL один байт `00`. Decimal position без ведущих нулей, обычное десятичное
представление Int. HMAC применяется к raw key, не к его hex/base64.
Length относится только к UTF-8 realID. Он в диапазоне 1–65 535.
JSON после ID должен быть непустым. ID декодируется строго как UTF-8.
ChaChaPoly combined соответствует CryptoKit; tag16, nonce12, без версии
или дополнительного префикса. Payload записи должен быть длиннее 28 байт.

Особая строка `bucket="header"`, `id="root"`, position 0:
indexID буквально `root`, HMAC не применяется; realID внутри packed тоже
`root`. AAD hex: `68656164657200726f6f740030`.

Читатель расшифровывает с AAD из колонок, распаковывает realID, заново
вычисляет HMAC и сравнивает его с колонкой id. Подмена bucket, position,
indexID или ciphertext должна приводить к отказу, а не пропуску записи.

## 4. Header

Расшифрованный JSON:

```json
{"state":{"version":1,"ownerID":"..."},"counts":{"contacts":0}}
```

`state` имеет тот же Codable ShumDatabase, что полный snapshot. Массивы
и словари, вынесенные в строки, очищены. Для optional полей сохраняется
разница между отсутствующим полем и пустым массивом/словарём. Counts содержит
число строк **каждой** корзины из таблиц ниже, включая нулевые значения.
`header` не входит в counts.

В state остаются ownerID/version, ownProfileCard, unviewedEncounterIDs,
pinnedDirectoryEntries и metadata legacyHistory без массива messages.
Нельзя подменять header минимальным объектом и терять неизвестные метаданные.
Основная root строка обязательна и должна иметь position 0.

## 5. Корзины массивов

| bucket | Тип JSON записи | realID |
|---|---|---|
| contacts | ShumContact | card.id |
| requests | ShumContactCard | card.id |
| conversations | ShumConversation | id |
| messages | ShumStoredMessage | envelope.id |
| relay | ShumRelayCopy | envelope.id |
| receipts | ShumStoredReceipt | receipt.key |
| encounters | ShumEncounter | card.id |
| savedProfiles | ShumSavedProfile | card.id |
| invitationOutbox | ShumStoredInvitationControl | control.id |
| profileOutbox | ShumProfileDelivery | recipientID |
| retractOutbox | ShumStoredRetract | control.id |
| reactionOutbox | ShumStoredReaction | control.id |
| legacyMessages | ShumLegacyMessage | id |

Чтение ORDER BY bucket,position. Внутри массива position строго возрастает,
первый > -1. Разрывы допустимы; сплошная нумерация от 0 не обязательна.
Дубли realID запрещены при записи. При удалении позиции остальных строк
сохраняются. Новые элементы после хвоста получают lastPosition+1.
Перестановка существующих элементов или вставка перед ними перенумеровывает
массив и требует нового AEAD для строк, чья position изменилась.

## 6. Корзины словарей

| bucket | Значение JSON | realID |
|---|---|---|
| seenRelay | Date | ключ словаря |
| blocked | ShumContactCard | ID заблокированного |
| deletedMessageIDs | Date | ID сообщения |
| invitationStates | ShumInvitationState | ID контакта |
| reactions | `[String: ShumReactionMark]` | ID сообщения |

Position у каждой записи словаря строго 0. Значение хранится отдельно от
ключа, ключ уже находится в packed realID. Reactions хранит весь внутренний
словарь personID → mark как одну запись для messageID.

## 7. JSON и значения

Payload кодируется `ShumCoding.encode`: JSONEncoder sortedKeys,
withoutEscapingSlashes, Data как **обычный Base64** с padding.
Optional nil обычно отсутствует. Set сериализуется массивом, порядок
элементов не гарантирован и не несёт смысла. Нельзя требовать точного порядка
forwardedTo/sentTo/unviewedEncounterIDs при сравнении снимков.

Swift Date по умолчанию кодируется числом секунд с 2001-01-01T00:00:00Z,
а не Unix milliseconds. Для конвертации:

```text
unixSeconds = jsonDate + 978307200
```

Число может быть дробным или отрицательным, `.distantPast` тоже допустимо.
Отдельные timestamp/expiresAt/updatedAt у wire-пакетов остаются Int64
миллисекундами Unix. Оба формата присутствуют в одном stored message.

ShumStoredMessage сохраняет envelope, расшифрованный text, outgoing,
status (`queued`, `forwarding`, `delivered`, `read`, `expired`, `cancelled`), unread,
hopCount, deliveryTransport, attempts, lastAttempt, forwardedTo,
nostrAccepted, optional lastNostrAttempt/nostrAttempts/deliveredAt/readAt/reply.
Receipt и control outbox сохраняют собственные attempt/transport metadata.
Точные Codable ключи и nil/default значения зафиксированы в state_json
фикстуры и перечисленных Swift типах; значения не являются wire packet.

## 8. Полное чтение и нормализация

Чтение выполняется одним snapshot:

1. Расшифровать root и декодировать Header.
2. Прочитать все записи, проверить размер каждой, AEAD, packed и HMAC.
3. Сверить число записей каждой корзины с counts; лишняя корзина запрещена.
4. Восстановить массивы в position order, словари с position 0.
   Optional массив/словарь заполняется только если соответствующее поле
   присутствовало в header.state.
5. Проверить суммарный размер encrypted payload не больше 100 000 000 байт.
6. Проверить version/owner и применить legacy normalization.

При любой ошибке не публиковать частично прочитанное состояние.
Normalization: отсутствующие invitationStates становятся accepted для
существующих contacts, updatedAt=addedAt в Unix ms, eventID=`legacy-`+contactID;
отсутствующий invitationOutbox превращается в `[]`; savedProfiles всегда nil.
Если normalization изменила состояние, iOS записывает результат транзакцией.

## 9. Запись и атомарность

Commit сравнивает previous/next, пишет только изменённые записи и root,
удаляет исчезнувшие. Неизменённая строка сохраняет ciphertext/nonce.
Повторный commit одинакового состояния не создаёт транзакцию.
Любая изменённая строка получает новый случайный nonce, даже при том же ID.
Все изменения и counts входят в одну SQLite транзакцию; ошибка вызывает
ROLLBACK. Сумма ciphertext после commit ограничена 100 000 000 байтами.

ShumConversationStore сначала делает durable commit, затем публикует память
и допускает ACK. CLI должен сохранить эту последовательность.
Не подтверждать сообщение до успешной записи.

Первое создание: временная база рядом с целевой, полный commit/checkpoint/
close, атомарный rename. iOS исключает каталог из backup и задаёт protection
completeUntilFirstUserAuthentication. На других ОС нужны эквивалентные
локальные ограничения доступа.

## 10. Старые снимки и миграция

SQLite определяется по первым 16 байтам `SQLite format 3\0`.
Если файл иной, iOS пробует старый v1 snapshot:
`ChaChaPoly combined`, AAD пустой, plaintext полный ShumDatabase JSON.
Максимум 100 000 000 байт. После decrypt/normalize создаётся временная
SQLite база, commit/checkpoint/close, старый файл копируется в
`<path>.v1-recovery`, затем atomic rename.
Ошибка не должна уничтожать старый файл.

Legacy история Bitchat мигрируется отдельно в ShumConversationStore.prepare:
AES-GCM combined с отдельным legacy key, JSON ShumLegacyArchive,
предел 100 000 000. Это другой формат, не storage key ChaChaPoly.
`snapshot(at:key:)` читает либо v1 backup, либо проверяет все SQLite записи
перед возвратом переносимого snapshot.

## 11. Примеры и вопросы

Фикстура содержит header, contacts, requests, conversation, message с текстом,
relay, receipt, encounter, словари, profileOutbox и метаданные профиля.
База создана Swift и закрыта с checkpoint; Rust может открыть исходный файл.
Каждая строка содержит indexID, realID, AAD, packed, JSON и combined cipher.
Отрицательные проверки: неверный ключ/owner, подмена position, удаление
контактов без обновления counts, повреждение ciphertext. No-op commit проверен.

`09-rust-roundtrip.json` создан Rust из исходной Swift базы: изменены
stored text, порядок контактов, добавлены дробные Date и tombstone. Настоящий
Swift reader на разрешённом симуляторе проверяет всю базу и portable snapshot
в `rustSQLiteRoundTrip()`. Это проверка хранения; изменённый stored text
не является новым подписанным wire message.
`09-legacy-cipher.json` снят с CryptoKit AES-256-GCM с отдельным legacy key
и фиксированным nonce. Проверяет слой шифрования, а не Codable schema
исторического архива Bitchat.

1. Формат не имеет схемы версии для отдельных JSON payload. Добавления
   Rust должны сохранять неизвестные поля и optional значения при round-trip.
2. Количество/размеры и корзины открыты. Это ожидаемое ограничение формата,
   нельзя обещать шифрование всей SQLite базы.
3. При восстановлении старых снимков savedProfiles намеренно очищается.
   Нужен отдельный продуктовый ответ, следует ли сохранять эту legacy-корзину.
4. Неверные данные в optional корзине, скрытой nil в header, всё равно
   проверяются криптографически и по counts, но могут не попасть в state.
   Не следует молча создавать такие состояния в новом writer.
5. SQLite writer защищает все записи атомарно, но прежний snapshot нужен
   для правильного diff. Одновременная независимая запись двумя процессами
   с устаревшими previous может потерять изменения. CLI должен иметь одного
   владельца базы/daemon и механизм IPC либо блокировку профиля.
