# 06. Транспорт Bluetooth

Черновик v1, Shum-iOS `5b8efc8`, Bitchat upstream
`9b84b361225facd8e623f25d76f889d3dc54a879`. Приоритет у текущего Swift-кода,
включая изменения Shum поверх upstream. Источники в `Shum-iOS/`:

- `ShumiOS/Vendor/Bluetooth/Services/BLE/BLEService.swift`,
  `BLEService+LinkLayerCentralRole.swift`, `BLEService+LinkLayerPeripheralRole.swift`,
  `BLERadioController.swift`, `BLEAnnounceHandler.swift`, `BLEProximityStore.swift`.
- Там же `BLEOutboundPacketPolicy`, `BLEOutboundFragmentPlanner`,
  `BLEFragmentAssemblyBuffer`, `BLEFragmentHandler`, `BLEIngressPacketGuard`.
- `ShumiOS/Vendor/Bluetooth/Noise/NoiseProtocol.swift`, `NoiseSession.swift`,
  `Services/NoiseEncryptionService.swift`, `Services/RelayController.swift`,
  `Services/TransportConfig.swift`, `Protocols/Packets.swift`, `Models/NoisePayload.swift`.
- Локальная зависимость проекта `localPackages/BitFoundation/Sources/BitFoundation/`:
  `BinaryProtocol`, `BitchatPacket`, `PeerID`, `MessageType`, `MessagePadding`,
  `CompressionUtil`. Путь задан в ShumiOS.xcodeproj; это часть исходников iOS.

## 1. GATT и обнаружение

| Объект | UUID |
|---|---|
| Primary service | `F85FC602-7866-4F4C-90AB-4F14E107792B` |
| Единственная characteristic | `8F85918D-468D-4FBE-825A-CBD890B74D10` |

Это пространство Shum, одинаковое в Debug/Release, не UUID обычного Bitchat.
Characteristic имеет notify, write, writeWithoutResponse, read; permissions
readable/writeable. Пароль/BLE pairing не является уровнем аутентификации
Shum: доверие устанавливает Noise. Peripheral публикует сервис до объявления.

Реклама содержит только service UUID, без Local Name, ключей и Shum ID.
Central сканирует по service UUID, разрешает дубли в foreground. После
подключения находит characteristic, включает notify. Central → peripheral:
ATT writes, peripheral → central: notifications. Оба участника могут быть
central/peripheral, нужна поддержка обеих ролей для взаимного обнаружения.

ATT success означает приём write стеком, не доставку Shum. iOS отвечает
успехом до анализа данных. Для long writes собирает значения по offset
отдельно на central (предел буфера 1 000 000 байт). Для notify используется
backpressure updateValue / readyToUpdateSubscribers. Нельзя терять остаток
фрагментов при временно заполненном буфере.

Максимальный локальный BLE frame budget 512, реальные ограничения получают
от maximumWriteValueLength / maximumUpdateValueLength, не считают всегда
равными 512. Паузы фрагментов 25 мс directed, 30 мс broadcast; одновременно
до двух больших transfer. Радио-политика локальна: максимум 6 central links,
connect timeout 8 с, базовый RSSI порог -90 dBm, расслабляется до -95/-100
при изоляции. Эти значения не добавляются в wire.

## 2. Полная личность и короткий ID

```text
Shum ID / fingerprint = lowerhex(SHA256(noise_static_public_key))   # 64 chars
full contact peerID   = lowerhex(noise_static_public_key)           # 64 chars
routing peerID        = fingerprint[0:16]                           # 8 bytes
```

`PeerID(hexData:)` только переводит байты в hex. `PeerID(publicKey:)`
вычисляет короткий hash ID. `ShumContactCard.peerID` полный; `toShort()` /
`routingData` и BLEService используют 8-байтовый короткий. Это разные
представления одного контакта, их нельзя взаимозаменять в бинарном header.

В текущем BLEService локальный routing ID выводится из постоянного
Noise-ключа и меняется при смене личности / профиля / panic, а не по таймеру
при обычном повторном подключении. Объявленный experimental announceV2
`0x2c` и код rotating IDs существуют, но shipping mesh их не посылает и
не принимает как новую систему обнаружения. Не включать автоматически.

## 3. BitchatPacket binary

Все многобайтовые целые big-endian. Timestamp UInt64 Unix ms.

| Offset | Размер | Поле |
|---:|---:|---|
| 0 | 1 | version, 1 или 2 |
| 1 | 1 | type |
| 2 | 1 | ttl |
| 3 | 8 | timestamp |
| 11 | 1 | flags |
| 12 | 2 для v1, 4 для v2 | payloadLength |
| 14/16 | 8 | senderID |
| далее | 8, если flag 01 | recipientID |
| далее | 1 + 8*n, только v2 и flag 08 | route: count и n IDs |
| далее | payloadLength | payload, включая compression preamble |
| далее | 64, если flag 02 | signature |
| далее | необязательно | padding |

Минимум 22 байта v1 / 24 v2. Комментарии upstream о 21 байте и header=13
устарели: исполняемый код задаёт header=14/16.

Flags: 01 hasRecipient, 02 hasSignature, 04 compressed, 08 hasRoute,
10 isRSR. Route игнорируется в v1. Прочие биты decoder не отвергает.
ID в encoder обрезается до 8 байт или дополняется нулями справа. Клиент
должен передавать корректные 8 байт; полный Noise-ключ нельзя просто
обрезать вместо derivation. Route count <=255, пустой hop недопустим.

Нет recipient либо восемь `ff` означает broadcast. Другой recipient это
directed. PayloadLength не включает IDs, route, signature или padding.
Decoder допускает хвост после объявленного тела; отдельно проверяет наличие
полей и размеры. Сам binary decoder подпись не проверяет.

Нужные Shum типы: announce=01, leave=03, noiseHandshake=10,
noiseEncrypted=11, fragment=20. requestSync=21 относится к gossip Bitchat.
Bitchat courierEnvelope=04 является другим внешним протоколом; Shum relay
из раздела 05 передаёт ShumPacket через Noise payload 41, не type04.

### 3.1. Подпись BitchatPacket

```text
unsigned = packet(signature=nil, ttl=0, isRSR=false)
signing_bytes = BinaryProtocol.encode(unsigned, padding=true)
signature = Ed25519.sign(signing_private_key, signing_bytes)
```

TTL и RSR изменяемы и подписью не защищены. Route входит в подпись, когда
присутствует. Compression и padding применяются при построении signing_bytes
так же, как encoder. Подписаны announce; обычный noiseEncrypted Shum
создаётся с signature=nil, потому что аутентификация внутри Noise.

### 3.2. Compression

Сжимается **payload**, до добавления binary header. Порог 100 байт, а не
упомянутые в старом комментарии 256. Кандидат с count>=100 допускается,
если `uniqueBytes / min(count,256) < 0.9`, и сжатие реально короче исходного.
Используется Apple Compression `COMPRESSION_ZLIB`: проверенный Swift-пример
содержит **raw DEFLATE**, без RFC1950 zlib header / Adler-32. Независимая
проверка `zlib.decompress(bytes, -15)` открывает его, режим `15` отвергает.
При flag04 payload:
originalSize (UInt16/UInt32 BE по version) || compressed bytes.

Header length включает preamble размера originalSize. Декомпрессия должна
вернуть ровно originalSize. Ограничение коэффициента 50 000 и общий
FileTransferLimits.maxFramedFileBytes применяются до выделения памяти.
Несжатый wire также допустим. Для Noise ciphertext эвристика обычно не
включает сжатие. Точный поток сжатия дан в 06-frames, Rust должен уметь
его декодировать; произвольный JSON gzip не подходит.

### 3.3. Padding

Политика BLE включает padding только для noiseHandshake / noiseEncrypted.
Алгоритм MessagePadding выбирает первый bucket из 256,512,1024,2048,
вмещающий `encodedLength + 16`. Для большего пакета target=encodedLength.
Дополняет до target только когда разница 1…255: каждый добавленный байт
равен разнице (PKCS#7 style). При разнице >255 остаётся без padding.
Это внешний padding всего binary frame, не Noise payload.

Decoder сначала читает as-is, затем пытается unpad при неудаче. Он обычно
читает padding как допустимый хвост. Нельзя передавать хвост в Noise AEAD:
длина ciphertext определяется payloadLength, подпись отдельным flag.

## 4. Announce и привязка

TLV: type UInt8, length UInt8, затем length байт. Encoder порядок:

| TLV | Значение |
|---|---|
| 01 | nickname UTF-8, <=255 байт |
| 02 | Noise public key, 32 байта |
| 03 | Ed25519 public key, 32 байта |
| 04 | Необязательно до 10 соседей по 8 байт |
| 05 | Необязательно capabilities, минимальное little-endian UInt64 |
| 06 | Необязательно bridge geohash, 1…12 UTF-8 байт |

Обязательны 01–03; unknown TLV пропускаются. Decode структуры ещё не
является доверием. Handler связывает senderID с hash(Noise key), проверяет
подпись announce и сохранённый signing-key pin, отбрасывает stale/self.
TTL=7 используется как признак direct, но не является криптографическим
доказательством физического соседства. Сессия Noise должна подтвердить
владение Noise-ключом до приёма карточки и Shum-содержимого.

После XX посылается authenticatedPeerState, Noise payload21:
version01 || TLV01 capabilities || TLV02 signing key32. Здесь unknown TLV
пропускаются, duplicates известных полей и неканонические capabilities
отклоняются. Это привязывает signing key к подтверждённому Noise key.
Объявленная возможность в публичном announce сама по себе такой связи не даёт.

## 5. Noise XX между узлами

`Noise_XX_25519_ChaChaPoly_SHA256` (32 ASCII байта), prologue пустой,
premessage нет. Инициализация h=protocol_name (ровно 32), ck=h, затем
MixHash(prologue), даже если это пустые Data. HKDF/MixHash/MixKey/AEAD как
в разделе 03. Шаблон:

```text
→ e                  + encrypted_or_clear_payload
← e, ee, s, es       + encrypted_payload
→ s, se              + encrypted_payload
```

Обычные handshake payload пустые. Фреймы содержат соответственно 32, 96,
64 байта, включая теги пустых зашифрованных payload. Каждый e32, encrypted
s48. Nonce после каждого MixKey сбрасывается; каждый EncryptAndHash
использует текущий h как AAD и обновляет h ciphertext. Нельзя опускать tag
пустого payload, когда уже есть cipher key.

Handshake bytes помещаются прямо в payload Bitchat noiseHandshake10 с
адресатом. После завершения split=HKDF2(ck, empty): initiator send=c1,
receive=c2; responder send=c2, receive=c1. Hash transcript сохраняется для
channel binding. Remote static key сверяется с известным контактом /
последующими карточками. При необходимости payload сначала ставится в
pending session queue, затем запускается handshake, чтобы не потерять
быстрый ответ. Конфликт одновременных handshake разрешает Noise service;
клиент должен уметь responder независимо от того, кто установил GATT link.

### 5.1. Transport cipher

```text
nonce12 = 00 00 00 00 || LE64(counter)
wire = BE32(counter) || ChaCha20Poly1305(ciphertext || tag)
AAD = empty
```

Начало counter=0 для каждого направления. На отправке максимум
UInt32.max-1; дальше нужна новая сессия. Принимающая сторона извлекает BE32,
создаёт тот же 12-байтовый nonce, проверяет AEAD и только после успеха
обновляет replay state. Заявлено окно 1024 counter, допускающее out-of-order.
Transport wire минимум 20 байт. Handshake cipher отличается отсутствием
четырёхбайтового counter prefix и использованием transcript h как AAD.

### 5.2. Shum typed payload

```text
plaintext = 41 || ShumPacket JSON          # до 24000 байт JSON
plaintext = 40 || ShumProfilePacket JSON   # до 6144 байт JSON
```

Один type byte без длины, остальное Data. После encrypt это binary packet
noiseEncrypted11, signature=nil, version1, ttl7, recipient=routing ID.
Shum ciphertext из Noise X внутри envelope остаётся дополнительным слоем.
Курьер Shum сначала открывает собственную XX-сессию депонента, затем
сохраняет всё ещё закрытый для адресата envelope.

## 6. Fragment wire и сборка

Сначала полностью кодируется исходный packet вместе с его padding/signature.
Потом его байты режутся на chunks. Каждый fragment type20 имеет payload:

```text
fragmentID[8] || BE16(index) || BE16(total) || originalType[1] || chunk
```

Index с 0, total 1…10000, index<total. Header fragments 13 байт. Ключ сборки
(senderID8, fragmentID8). Sender, timestamp, ttl, recipient берутся из
original; signature=nil. Directed override может задать другого link peer.
Для original с route используется fragment version2 и копия route.

Default chunk=469. По реальному link budget `max(64, limit - 42)`;
route sizing учитывает v2 header, IDs, count+8*n route, fragment header и
запас16. Явный maxChunk не меньше64. Packet можно получить в обратном
порядке; сборка склеивает chunks по index и снова декодирует исходный frame.

До128 сборок, при переполнении вытесняется самая старая. Срок30 с с начала
сборки, не с последнего дубля. Duplicate index заменяет bytes, но не
сбрасывает stall timer. Broadcast stalls через5 с могут запросить отсутствующий
stream через requestSync, повтор не чаще10 с; directed так не восстанавливаются.
Предел размера сборки зависит от originalType (file/noise допускают framed
file limit). После реконструкции действуют проверки Noise и Shum, а не
доверие к originalType в непроверенном заголовке fragment.

## 7. Mesh и два разных счётчика

Внешний Bitchat ttl начинается с7 и уменьшается на каждом mesh relay.
Он не подписан. Source routing v2 позволяет route из промежуточных8-byte IDs;
это version бинарного транспорта, **не версия Shum и не v2-devices**.
На входе текущий BLEIngressPacketGuard требует timestamp skew<=120000 мс,
кроме разрешённого RSR ответа синхронизации. Нельзя присваивать RSR без
реального запроса. Loopback собственного ID отбрасывается.

Отдельный Shum hopCount начинается с1 на передаче конверта между курьерами
и ограничен envelope.hopLimit<=4. Несколько физических mesh hops внутри
XX-сессии не превращаются в несколько Shum courier hops. Свои receipt,
очереди и tombstones из раздела05 не заменяются gossip-дедупликацией Bitchat.

## 8. Расстояние и список рядом

Расстояние берётся только из локального RSSI физического BLE peripheral,
привязанного к peer. Relay-пакеты не являются замерами расстояния.
BLEProximityStore принимает RSSI -127…-1, игнорирует127 и0; до30 последних
замеров на peripheral, до256 peripheral. Учитываются замеры age>=0 и<30 с.

```text
readings = sorted(all current RSSI samples of peripherals bound to peer)
median = readings[count / 2]             # верхняя медиана при чётном числе
distance = max(1, round(10^((-59 - median) / 20)))  # целые метры
```

Нет действительного RSSI → nil, не0 метров. Это приближённая оценка с
-59 dBm на1 метр и показателем2. Стены и положение устройств меняют RSSI.
Присутствие в nearby Shum дополнительно требует card, чей Noise key
совпадает с установленной сессией. Просто увидеть service UUID недостаточно.

## Тестовые примеры

Генератор `bluetoothFrames`, `bluetoothNoise`, без реального радио.

| Файл | Содержание |
|---|---|
| `06-frames.json` | v1/v2 binary, padding, compression, route, signing bytes, announce и оба peerID |
| `06-fragments.json` | Фиксированный stream ID, реальные frame bytes, сборка в обратном порядке |
| `06-noise-xx.json` | Три deterministic handshake frames, transcript hash, transport в обе стороны, проверка replay |
| `06-proximity.json` | Медиана RSSI, округление, отсутствующий сигнал, точная граница30 с |

## Вопросы

1. **Уточнение раздела01 про смену короткого ID.** Текущее shipping BLE
   выводит его из постоянного Noise-key. Фраза «меняются со временем» в01
   не означает периодическую ротацию в iPhone1.0; experimental rotating
   announceV2 не задействован. CLI должен следовать shipping derivation.
2. **Compression и старые комментарии.** Реальный порог100, header14/16.
   Комментарии о256/21/13 нельзя использовать для совместимости. Поток
   Apple Compression подтверждён fixture и независимой raw-DEFLATE проверкой.
3. **Подтверждённый дефект replay window.** В Swift смещение bitmap при
   повышении counter направлено в неверную сторону. После успешного приёма
   counter0 и counter1 повтор counter0 проходит AEAD и принимается снова:
   `swift_replay_zero_after_one_accepted=true` в 06-noise-xx. Исправление iOS не входит
   в эту работу; решение о строгом отклонении в Rust нужно зафиксировать.
4. **Fragment metadata.** Buffer использует incoming total для условия
   завершения, не сверяя total/originalType с первой частью, и заменяет
   duplicate index. Нужна отдельная оценка ограничения против подмены;
   финальная AEAD всё равно обязательна, неполная сборка недоверенна.
5. **Подписи без domain у части v1.** Общее утверждение раздела01 «все подписи
   с меткой» не относится к legacy card, envelope, controls и binary announce.
   У каждого типа использовать фактические signing bytes своего раздела.
