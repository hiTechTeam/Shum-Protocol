# Shum Protocol

[English](README.md) · Русский

Спецификация протокола Shum и тестовые примеры совместимости. Личность задают ключи на устройстве; зашифрованные сообщения передаются по Bluetooth или через релеи Nostr.

## Состояние

Стабильная версия ещё не выпущена. Разделы 01–09 описывают текущий черновик v1, который используют iOS и ядро на Rust. Несколько устройств, синхронизация и релеи профиля входят в будущую v1 stable; форматы передачи ещё нужно определить.

Все разделы доступны на русском и английском, в `spec/ru/` и `spec/en/`. Строки UTF-8 в примерах данных сохранены без перевода.

## Спецификация

| Тема | Документ |
| :--- | :--- |
| Обзор | [00](spec/ru/00-overview.md) |
| Ключи и личность | [01](spec/ru/01-keys-identity.md) |
| Карточка и приглашения | [02](spec/ru/02-contact-card.md) |
| Конверт сообщения | [03](spec/ru/03-envelope.md) |
| Виды пакетов | [04](spec/ru/04-packets.md) |
| Приём и слияние | [05](spec/ru/05-rules.md) |
| Bluetooth | [06](spec/ru/06-transport-bluetooth.md) |
| Nostr | [07](spec/ru/07-transport-nostr.md) |
| Push API | [08](spec/ru/08-push-api.md) |
| Хранилище | [09](spec/ru/09-storage.md) |
| Устройства, план | [10](spec/ru/10-devices.md) |
| Релеи профиля, план | [11](spec/ru/11-relays.md) |
| Состав v1 stable и открытые решения | [12](spec/ru/12-v1-stable.md) |

«Должен» и «нельзя» задают требования совместимости. Описание текущего iOS фиксирует поведение кода. Предлагаемые изменения и открытые вопросы указаны в спецификации; это ещё не реализованные функции.

## Тестовые примеры

[JSON-примеры](vectors/) содержат входные данные Swift и ожидаемые результаты. Для случайного шифрования и подписей клиент проверяет или расшифровывает сохранённый результат. [Shum Core](https://github.com/hiTechTeam/Shum-Core) закрепляет ревизию этого репозитория и проверяет совместимость по примерам.

Генератор: `ShumTests/Protocol/ShumProtocolVectorTests.swift` в [Shum iOS](https://github.com/hiTechTeam/Shum-iOS). Последняя полная проверка: 21 тест прошёл на ревизии iOS `5b8efc8`. Повторная генерация меняет случайные подписи и nonce.

```sh
xcodebuild test -project ShumiOS.xcodeproj -scheme Shum \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -parallel-testing-enabled NO \
  -only-testing:ShumTests/ShumProtocolVectorTests \
  -disableAutomaticPackageResolution -skipPackageUpdates
```

Запуск из Shum-iOS. Bluetooth-сетка и Noise взяты из [Bitchat](https://github.com/permissionlesstech/bitchat); ревизии и лицензии записаны в репозитории iOS.

## Связанные проекты

[Shum Core](https://github.com/hiTechTeam/Shum-Core) · [Shum CLI](https://github.com/hiTechTeam/Shum-CLI) · [Shum iOS](https://github.com/hiTechTeam/Shum-iOS) · [Issues](https://github.com/hiTechTeam/Shum-Protocol/issues)

## Лицензия

[MIT](LICENSE), copyright 2026 hiTechTeam.
