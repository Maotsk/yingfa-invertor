# Yingfa 6.2kW PI30 Inverter — ESPHome Monitoring

Мониторинг и управление инвертором **Yingfa 6.2kW** (протокол PI30) через
ESP8266 + MAX3232 + ESPHome с интеграцией в Home Assistant.

## Аппаратная часть

| Компонент | Модель / параметр |
|---|---|
| Микроконтроллер | ESP8266 (NodeMCU) |
| Преобразователь | MAX3232 (RS232 ↔ TTL) |
| Интерфейс инвертора | RS232, 2400 бод, 8N1 |
| TX ESP → RX инвертора | GPIO15 |
| RX ESP ← TX инвертора | GPIO13 |

**Важно:** GND ESP и GND инвертора должны быть соединены.
Питание ESP — от отдельного источника или USB.

## Возможности

### Мониторинг (сенсоры)
- **QPIGS** (каждые 3 с): сетевое напряжение/частота, выход AC, нагрузка,
  BUS, батарея (напряжение, ток заряда, ёмкость), PV (ток, напряжение,
  мощность), ток разряда, статусные биты.
- **Q1** (каждые 3 с): температуры NTC инвертора и PV.
- **QPIRI** (каждые 20 мин): настройки — приоритеты, тип батареи, макс. ток
  заряда, напряжения батареи.
- **QMOD** (каждые 20 мин): текущий режим (Line / Battery / Fault / Standby).
- **QPIWS** (каждые 20 мин): флаги ошибок с фильтром ложных срабатываний.
- **QVFW** (один раз при старте): версия прошивки инвертора.

### Управление
- **Селекты:** Output Source Priority (POP), Input Voltage Range (PGR),
  Max Charging Current (MNCHGC), Charger Source Priority (PCP).
- **Number (поле ввода):** напряжения батареи PBCV (re-charge),
  PBDV (re-discharge), PCVV (C.V.), PSDV (cut-off).
- **Кнопка:** ручной запуск slow poll.

## Архитектура

### Скрипты
- `pi30_exchange` — единая точка обмена. Принимает `command` (string),
  формирует CRC (CRC-16/XMODEM), отправляет, читает ответ, парсит.
- `fast_poll` — QPIGS + Q1, с ретраем при ошибке.
- `slow_poll` — QPIRI + QMOD + QPIWS.

### Защита от конфликтов UART
- `uart_busy` — UART занят обменом.
- `pi30_running` — скрипт pi30_exchange выполняется.
- `slow_poll_active` — идёт slow poll (блокирует селекты и number).

### Планировщик
`interval: 200ms` проверяет, не пора ли запустить fast/slow poll.
Интервалы настраиваются в HA:
- Fast Poll Interval (1–60 с, по умолчанию 3)
- Slow Poll Interval (1–120 мин, по умолчанию 20)

### Приём ответа UART
- Ждём первый байт (timeout 1 с).
- Ждём набора ≥5 байт (до 20 мс).
- Читаем до «тишины» на линии 15 мс.
- Проверяем CRC, при ошибке — ретрай (для QPIGS).

## Установка

1. Клонировать репозиторий.
2. Скопировать `secrets.yaml.example` → `secrets.yaml`, заполнить:
   - `wifi_ssid`, `wifi_password`
   - `api_encryption_key` (свой, не из примера!)
3. Проверить `device_ip` в `substitutions` — должен совпадать с IP ESP.
4. Скомпилировать и залить:
   ```bash
   esphome run yingfa-invertor.yaml
