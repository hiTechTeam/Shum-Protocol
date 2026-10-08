# 08. API push-сервера

Источники: iOS `5b8efc8`,
`ShumiOS/App/Services/Notifications/ShumPushService.swift`;
`Shum-Push-Server/internal/api/server.go`, `internal/identity/auth.go`,
`internal/apns/client.go`, `internal/store/`.
Генератор `pushAPI()` перехватывает настоящие запросы ShumPushService через
локальный URLProtocol. `vectors/08-requests.json` не использует интернет,
APNs или реальные ключи пользователя.

## 1. Назначение и адрес

Push пробуждает получателя для чтения Nostr. Он не содержит сообщение,
шифротекст, реакцию, карточку или ключи чата. Доставка через APNs не является
ShumReceipt. Доставку и прочтение подтверждает только раздел 04.

Базовый HTTPS URL берётся iOS из `ShumPushAPIBaseURL` в Bundle.
Пустое значение и неразрешённая подстановка `$(...)` отключают запросы.
Пути ниже относительные; hostname в подпись не входит. CLI должен иметь
настройку URL. Привязки к сетевому интерфейсу в протоколе нет.

## 2. Карточка HTTP API

Это отдельный DTO, не JSON сетевой карточки Shum:

```json
{"version":1,"noise_key":"base64url32","signing_key":"base64url32",
 "nostr_key":"lowercase-hex32","name":"Имя","bio":"",
 "signature":"base64url64"}
```

Ключи и legacy signature кодируются Base64URL без padding. Расширения seed,
avatarVersion, profileRevision и profileSignature не передаются.
Сервер проверяет legacy подпись карточки по разделу 02: реконструирует
camelCase объект с обычным Base64 ключей, `signature:""`, сортирует поля,
не экранирует HTML и `/`, удаляет финальный LF Go encoder.
Никакой дополнительной domain label у legacy подписи нет.

Сервер требует version 1; noise 32 байта, не весь нулевой; signing 32;
signature 64; Nostr ровно 64 **строчных ASCII hex**. Имя после Go TrimSpace
непустое, исходное имя до 64 байт UTF-8. Bio до 72 Unicode **runes**.
ID = lowercase hex SHA-256(raw noise key).
Важно: runes здесь отличаются от Swift Character/graphemes.

## 3. Подпись каждого HTTP-запроса

Заголовки:

| Заголовок | Значение |
|---|---|
| Content-Type | `application/json` |
| X-Shum-Public-Key | Base64URL без padding, Ed25519 public key 32 байта |
| X-Shum-Timestamp | десятичные целые секунды Unix |
| X-Shum-Nonce | новый UUID в нижнем регистре у iOS |
| X-Shum-Signature | Base64URL без padding, подпись Ed25519 64 байта |

Точные подписываемые UTF-8 байты:

```text
SHUM1\nMETHOD\nPATH\nTIMESTAMP\nNONCE\nBODY_SHA256_HEX
```

Здесь `\n` означает один LF `0a`. В конце строки LF **нет**. `SHUM1`
без NUL. METHOD в верхнем регистре. PATH начинается с `/`, без hostname
и query; сервер использует `URL.EscapedPath()`. Для трёх фиксированных путей
процентное кодирование отсутствует. Timestamp и nonce включаются как текст
из заголовков; сервер не пересериализует timestamp перед проверкой.
BODY_SHA256_HEX: 64 строчных hex от **исходных байтов HTTP body**.

iOS сериализует body через JSONEncoder `.sortedKeys` и
`.withoutEscapingSlashes`. Подписывается получившийся JSON, а не объект после
парсинга на сервере. Поэтому замена пробелов или порядка полей ломает подпись.
Ключ заголовка обязан совпасть с ключом проверенной карточки.

Сервер принимает timestamp Int64 с отклонением не больше 5 минут от своих
часов. Nonce 16–128 байт; UUID не обязателен. Успешно проверенный nonce
запоминается на 10 минут отдельно для каждого public key; повтор отвергается.
Кеш находится в памяти процесса, после перезапуска пропадает.

## 4. Регистрация и удаление устройства

```text
POST /v1/devices
DELETE /v1/devices
```

Оба имеют подписанный JSON body:

```json
{"card":{},"device_token":"64-hex","environment":"production",
 "app_version":"1.0"}
```

`device_token`: 32 байта APNs token как 64 hex; сервер допускает оба регистра,
хранит нижний. `environment` POST строго `sandbox` или `production`.
У iOS DEBUG выбирает sandbox, Release production. DELETE это поле не
валидирует. `app_version` необязателен для сервера, iOS передаёт bundle version.

POST сохраняет устройство для ID карточки, отвечает 201
`{"status":"registered"}`. DELETE удаляет token этой личности, отвечает 204
с пустым body. CLI не должен придумывать APNs token для компьютера.

iOS регистрируется после configure(card,signer) и получения token. Ключ
дедупликации `card.id:token:environment`. Повторные регистрации планируются
с задержками 0, 15, 60, 300 секунд. Задержки ожидания последовательные.
Смена карточки сбрасывает registration key.

## 5. Запрос уведомления

```text
POST /v1/notifications
```

```json
{"card":{},"recipient_id":"64-hex","event_id":"uuid-or-other-id",
 "kind":"message"}
```

Kind строго `message`, `invitation` или `reaction`. Recipient ID 64 ASCII hex,
сервер допускает верхний регистр, но поиск в хранилище использует исходный
текст. Для совместимости отправлять нижний регистр.
Event ID: 16–64 ASCII символа `A-Z a-z 0-9 - _`. Это ID Shum-пакета,
не обязательно 64-символьный ID Nostr wrap; UUID из 36 символов допустим.

После проверки сервер пытается послать push каждому устройству получателя.
Ответ **всегда 202 `{"status":"accepted"}`**, даже если устройств нет или
все попытки APNs неуспешны. Счётчики sent/failed/removed доступны только в
серверном логе. Некорректные APNs token удаляются из хранилища.

iOS дедуплицирует уведомления в памяти по eventID, без kind и recipientID.
Одновременно допускает один task на этот ID. Последовательные задержки
повторов 0, 2, 5, 15, 30, 60 секунд. Любой 2xx завершает попытки успехом;
4xx, кроме 429, останавливает; 429, 5xx и сетевые ошибки повторяются.
Смена/удаление личности отменяет задачи. Удержание push при BLE доставке
описано в разделе 05: 8 секунд для сообщения, 5 для реакции.

## 6. Ограничения и ответы ошибок

Body максимум 16 KiB. Неизвестные JSON поля Go декодер игнорирует.
Невалидный JSON/поля: 400; карточка, подпись, время или повтор nonce: 401;
превышение 60 запросов в минуту на IP либо неверный gateway secret: 429;
ошибка persistence: 500. Без настроенного APNs notify возвращает 503 ещё
до чтения body и аутентификации.
Если задан gateway secret, требуется `X-Shum-Gateway`; клиент iOS сам
секрет не передаёт. Предполагается настроенный gateway перед сервисом.

Все ответы получают `Cache-Control: no-store`, `X-Content-Type-Options:
nosniff`. JSON отвечает с Content-Type application/json и финальным LF.
GET `/healthz`: 200 `{"status":"ok"}`. GET `/readyz`: status ready и число
identities/devices либо 503 waiting_for_apns_credentials.
GET `/invite` обслуживает страницу приглашения, к подписанному API не относится.

## 7. APNs payload и пробуждение

```json
{"aps":{"alert":{"title":"Имя","loc-key":"PUSH_NEW_MESSAGE_BODY"},
 "sound":"default","content-available":1},
 "shum":{"event_id":"id","kind":"message"}}
```

Для реакции loc-key `PUSH_NEW_REACTION_BODY`, для приглашения
`Новое приглашение`. Title получается из имени: TrimSpace, удалить control
символы, U+202A–202E и U+2066–2069, взять первые 64 runes, снова trim;
пустой результат заменяется на `Shum`.
Текст сообщения и выбранная реакция отсутствуют.

APNs: HTTP/2 POST `/3/device/TOKEN` к production или sandbox host;
push-type alert, priority 10, topic bundle ID, expiration now+24h,
collapse-id eventID. Авторизация ES256 JWT сервера, кеш до 50 минут.
410, BadDeviceToken, DeviceTokenNotForTopic, Unregistered удаляют token.
Секреты APNs никогда не нужны CLI.

Получив push, iOS передаёт eventID background handler. До установки handler
ожидают до 16 событий, таймаут 10 секунд. Remote dedup до 1000 ID.
Реальное содержимое загружается из Nostr.

## 8. Примеры и вопросы

`08-requests.json`: регистрация, три вида уведомлений и DELETE, исходные
body, заголовки, подписываемые байты и проверка подписи Swift. Timestamp и
nonce записываются как сгенерированы; для воспроизведения серверного времени
нужны часы из фикстуры, а не текущее время запуска клиента.

1. Сервер считает bio в runes, iOS в graphemes. Валидный iOS bio из emoji
   или combining marks может получить 401 от сервера.
2. Go JSON encoder экранирует U+2028/U+2029 даже при SetEscapeHTML(false).
   Нужна отдельная совместимость с каноническими байтами Swift для таких имён.
3. 202 не доказывает успешную передачу APNs. Интерфейс CLI должен показывать
   только подтверждённое им событие, без вывода о получении сообщения.
4. IP limiter доверяет первому X-Forwarded-For. Публичный backend должен
   находиться за доверенным gateway, который заменяет этот заголовок.
5. API требует legacy signature карточки даже при наличии новой profile
   подписи. При создании push DTO нельзя выбрасывать legacy подпись.
