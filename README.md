# Shum Protocol

Спецификация протокола мессенджера Shum и тестовые примеры к ней.

Цель: первая стабильная версия протокола (**v1 stable**), в которой все
клиенты понимают друг друга, а один профиль синхронизирован на всех своих
устройствах. Это основа, на которую нанизываются следующие версии. Что в неё
входит и когда она готова: [spec/12-v1-stable.md](spec/12-v1-stable.md).

## Статус

Протокол ещё не выпущен. Разделы 01–09 описывают **черновик v1**: поведение
iOS-приложения, снятое с кода `Shum-iOS`. Разделы 10 и 11 добавляют несколько
устройств и релеи профиля, раздел 12 собирает всё в v1 stable.

| Раздел | Файл | Состояние |
|---|---|---|
| Обзор | [spec/00-overview.md](spec/00-overview.md) | черновик |
| Ключи и личность | [spec/01-keys-identity.md](spec/01-keys-identity.md) | черновик, примеры есть |
| Карточка контакта и приглашения | [spec/02-contact-card.md](spec/02-contact-card.md) | черновик, примеры есть |
| Конверт сообщения | [spec/03-envelope.md](spec/03-envelope.md) | черновик, примеры проверены Swift |
| Виды пакетов | [spec/04-packets.md](spec/04-packets.md) | черновик, примеры проверены Swift |
| Правила приёма и слияния | [spec/05-rules.md](spec/05-rules.md) | черновик, примеры проверены Swift |
| Транспорт Bluetooth | [spec/06-transport-bluetooth.md](spec/06-transport-bluetooth.md) | черновик, примеры проверены Swift |
| Транспорт Nostr | [spec/07-transport-nostr.md](spec/07-transport-nostr.md) | черновик, примеры проверены Swift |
| API push-сервера | [spec/08-push-api.md](spec/08-push-api.md) | черновик, примеры проверены Swift |
| Хранилище на устройстве (SQLite) | [spec/09-storage.md](spec/09-storage.md) | черновик, примеры проверены Swift |
| Несколько устройств | [spec/10-devices.md](spec/10-devices.md) | часть v1 stable, решения согласованы, нужны форматы |
| Релеи профиля и сети релеев | [spec/11-relays.md](spec/11-relays.md) | часть v1 stable, решения согласованы, нужны форматы |
| Первая стабильная версия | [spec/12-v1-stable.md](spec/12-v1-stable.md) | состав, исправления, открытые решения, готовность |
| Тестовые примеры | [vectors/](vectors/) | для разделов 01–09 |

## Как читать

- **Должен / нельзя** означают обязательные требования к совместимому клиенту.
- **Сейчас в iOS** описывает фактическое поведение текущего кода, даже если оно
  выглядит странно. До выпуска v1 stable его можно менять: список исправлений
  в [разделе 12](spec/12-v1-stable.md), пункт 4. Где разделы 01–09 откладывают
  что-то «до v2» или «до новой версии», это теперь решается до v1 stable.
- **Вопрос** отмечает место, где код неоднозначен или требует решения. Такие
  места собраны в конце каждого раздела.
- Ссылки на код даны в виде `Файл.swift` и имени типа или функции в репозитории
  `Shum-iOS`.

## Источники

- `Shum-iOS/ShumiOS/Vendor/Messaging/` — карточка, конверт, логика доставки Shum.
- `Shum-iOS/ShumiOS/Vendor/Bluetooth/` — Bluetooth-сетка и Noise-сессии, взяты из
  проекта Bitchat (Unlicense), ревизия указана в `Shum-iOS/Upstreams/versions.json`.
- `Shum-iOS/ShumiOS/Vendor/Nostr/` — личность и события Nostr.
- `Shum-iOS/docs/seed-profile-protocol.md` — заметка о профилях и приглашениях.
- `Shum-iOS/ShumiOS/Vendor/Messaging/ShumSQLitePersistence.swift` — формат базы на
  устройстве. Раздел 09 не про сеть: он нужен, чтобы ядро открыло базу,
  созданную Swift, без переноса данных.

## Тестовые примеры

В папке `vectors/` лежат JSON-файлы, снятые с работающего Swift-кода:
фиксированные входные данные и ожидаемый результат. Совместимый клиент
должен проходить их все.

Подписи Ed25519 в CryptoKit и шифрование со случайными одноразовыми числами
дают разные байты при каждом запуске. Для них пример устроен как «вот подпись
или шифротекст из Swift, клиент должен их проверить или расшифровать», а не
«должно получиться ровно это».

Генератор: `Shum-iOS/ShumTests/Protocol/ShumProtocolVectorTests.swift`.
Последняя полная проверка: **21 тест в одной suite, passed**, iOS `5b8efc8`,
симулятор iPhone 17 Pro `9374725C-1A19-4568-A06B-AAB02E8DBCD2`.
Запуск из `Shum-iOS`:

```sh
xcodebuild test -project ShumiOS.xcodeproj -scheme Shum \
  -destination 'platform=iOS Simulator,id=9374725C-1A19-4568-A06B-AAB02E8DBCD2' \
  -parallel-testing-enabled NO \
  -only-testing:ShumTests/ShumProtocolVectorTests \
  -disableAutomaticPackageResolution -skipPackageUpdates
```

Повторный запуск перезаписывает случайные подписи/nonce в фикстурах.
Ошибки аутентификации в логе отрицательных тестов ожидаемы.
Известный дефект replay-window Swift записан в разделе 06 и отдельно в
`06-noise-xx.json`; он требует явного решения для реализации Rust.
