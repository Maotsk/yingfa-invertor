![ESPHome](https://img.shields.io/badge/ESPHome-2024.6%2B-blue)
![ESP8266](https://img.shields.io/badge/MCU-ESP8266-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

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
   - `yingfa_api_key` (свой, не из примера!)
   - `yingfa_ap_password` (свой, не из примера!)
3. Проверить `device_ip` в `substitutions` — должен совпадать с IP ESP.
4. Скомпилировать и залить:
   ```bash
   esphome run yingfa-invertor.yaml

## Известные проблемы

### CRC-ошибки (~10% опросов QPIGS)

Симптом: `PI30 QPIGS: CRC ERROR RX=XXXX CALC=YYYY`, где RX и CALC
отличаются на 1 бит.

Причина: физическая — бит-флип на линии RS232.
Ретрай в `fast_poll` спасает ситуацию.

Что попробовать:
- Соединить GND ESP и инвертора.
- Заменить MAX3232 (дешёвые клоны часто дают эффект).
- Конденсатор 100 нФ на VCC/GND MAX3232.
- Укоротить/экранировать кабель до инвертора.
- Снизить скорость до 1200 бод (если поддерживается).

### `script took a long time (max is 50 ms)`

Косметическое предупреждение ESPHome. Причина — цикл чтения UART
ждёт тишины на линии. Не критично, можно игнорировать.

### Battery Voltage = 12.80 V при отключённой АКБ

Не баг парсинга — реальное значение на клеммах инвертора при
отсутствии батареи. Ёмкость = 0 %, SCC = 0 В — тоже норма.

## Протокол PI30 — заметки

Полный формат команд см. в [`docs/protocol-notes.md`](docs/protocol-notes.md).

Кратко:
- Команда: ASCII + CRC-16/XMODEM (2 байта big-endian) + `0x0D`.
- Ответ: `(` + ASCII тело + CRC + `0x0D`.
- Для QPIGS/QPIRI/Q1/QPIWS ответ приходит цельным пакетом,
  пауза между байтами минимальна.

### Используемые команды

| Команда | Назначение |
|---|---|
| QPIGS | Общие данные (20 полей) |
| QPIRI | Настройки инвертора |
| QMOD | Режим работы (1 символ) |
| QPIWS | Флаги ошибок (32 бита) |
| Q1 | Температуры NTC |
| QVFW | Версия прошивки |
| POP0X | Output Source Priority |
| PGR0X | Input Voltage Range |
| MNCHGCXXX | Max Charging Current |
| PCP0X | Charger Source Priority |
| PBCVxx.x | Battery Re-charge Voltage |
| PBDVxx.x | Battery Re-discharge Voltage |
| PCVVxx.x | Battery C.V. Voltage |
| PSDVxx.x | Battery Cut-off Voltage |

## Полезные ссылки

- [ESPHome UART](https://esphome.io/components/uart.html)
- [PI30 Protocol (Voltronic)](https://github.com/jblance/mpp-solar)
- [Home Assistant Number](https://www.home-assistant.io/integrations/number/)

## Лицензия

Этот проект распространяется под лицензией MIT.
Подробности — в файле [LICENSE](LICENSE).
