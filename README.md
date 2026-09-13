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
- **QVFW** (при slow poll, пока не получена): версия прошивки инвертора.
  При неудаче — один ретрай, затем повтор при следующем slow poll.

### Управление

- **Селекты:** Output Source Priority (POP), Input Voltage Range (PGR),
  Max Charging Current (MNCHGC), Charger Source Priority (PCP).
- **Number (поле ввода):** напряжения батареи PBCV (re-charge),
  PBDV (re-discharge), PCVV (C.V.), PSDV (cut-off).
- **Кнопки:** ручной запуск slow poll, перезагрузка ESP.

## Интеграция с Energy Dashboard Home Assistant

Расчёт энергии (кВт·ч) и сенсоры мощности батареи создаются на стороне
**Home Assistant**, а не на ESP. Это осознанное решение: интеграция на
ESP8266 создаёт лишнюю нагрузку и дополнительные записи во флеш-память,
сокращая срок службы устройства.

### Сенсоры, которые нужно создать в HA

Используй хелпер **Интеграция** (Riemann sum integral) для преобразования
мгновенной мощности (W) в энергию (kWh):

| Источник (W) | Целевой сенсор (kWh) | Назначение |
|---|---|---|
| `sensor.yingfa_invertor_ac_output_active_power` | `sensor.yingfa_invertor_ac_energy` | Потребление от инвертора |
| `sensor.yingfa_invertor_pv_charging_power` | `sensor.yingfa_invertor_pv_energy` | Выработка солнечных панелей |
| `sensor.battery_charge_power` | `sensor.battery_charge_energy` | Заряд батареи |
| `sensor.battery_discharge_power` | `sensor.battery_discharge_energy` | Разряд батареи |

### Как создать хелпер

1. **Настройки → Устройства и службы → Помощники → + Создать помощника**
2. Нажми **Интеграция**
3. Заполни:
   - **Название:** `Yingfa Inverter AC Energy`
   - **Входной сенсор:** `sensor.yingfa_invertor_ac_output_active_power`
   - **Метод интегрирования:** `Trapezoidal`
   - **Префикс единицы:** `k`
   - **Единица времени:** `h`
4. Повтори для PV (`sensor.yingfa_invertor_pv_charging_power`).

### Требования к исходным сенсорам

Исходные сенсоры (мощность в W) должны иметь:

- `device_class: power`
- `state_class: measurement`

Эти атрибуты уже заданы в конфиге ESPHome — проверь в
**Developer Tools → States**, что они на месте.

### Сенсоры батареи — мощность (W)

Home Assistant не умеет перемножать сенсоры напрямую, поэтому мощность
батареи (заряд и разряд) нужно вычислить через помощник **Шаблон**.

**Создание через веб-интерфейс:**

1. **Настройки → Устройства и службы → Помощники → + Создать помощника**
2. Выбери **Шаблон** → **Шаблонный сенсор**
3. Заполни поля (см. таблицы ниже).

**Battery Charge Power (мощность заряда):**

| Поле | Значение |
|---|---|
| Название | `Battery Charge Power` |
| Шаблон состояния | см. код ниже |
| Единица измерения | `W` |
| Класс устройства | `power` |
| Класс состояния | `measurement` |

```jinja
{% set voltage = states('sensor.yingfa_invertor_battery_voltage') | float(0) %}
{% set current = states('sensor.yingfa_invertor_battery_charging_current') | float(0) %}
{{ (voltage * current) | round(1) }}
```

**Battery Discharge Power (мощность разряда):**

| Поле | Значение |
|---|---|
| Название | `Battery Discharge Power` |
| Шаблон состояния | см. код ниже |
| Единица измерения | `W` |
| Класс устройства | `power` |
| Класс состояния | `measurement` |

```jinja
{% set voltage = states('sensor.yingfa_invertor_battery_voltage') | float(0) %}
{% set current = states('sensor.yingfa_invertor_battery_discharge_current') | float(0) %}
{{ (voltage * current) | round(1) }}
```

> Если entity_id исходных сенсоров отличаются (например,
> `sensor.battery_voltage` вместо `sensor.yingfa_invertor_battery_voltage`),
> поправь в шаблоне. Точные имена видны в **Developer Tools → States**.

### Сенсоры батареи — энергия (kWh)

Из мощностей (W) нужно получить энергию (kWh) — через помощник
**Интеграция** (Riemann sum integral).

1. **Настройки → Устройства и службы → Помощники → + Создать помощника**
2. Выбери **Интеграция**
3. Заполни:

| Поле | Battery Charge | Battery Discharge |
|---|---|---|
| Название | `Battery Charge Energy` | `Battery Discharge Energy` |
| Входной сенсор | `sensor.battery_charge_power` | `sensor.battery_discharge_power` |
| Метод интегрирования | `Trapezoidal` | `Trapezoidal` |
| Префикс единицы | `k` | `k` |
| Единица времени | `h` | `h` |

### Добавление в Energy Dashboard

1. **Настройки → Панели → Энергия**
2. В разделе **Individual devices** добавь созданные хелперы:
   - `Yingfa Inverter AC Energy`
   - `Yingfa Inverter PV Energy`
3. В разделе **Батарея** добавь:
   - `Battery Charge Energy`
   - `Battery Discharge Energy`
4. Сохрани.

### Важно

- **Не используй** платформу `integration` в ESPHome — это создаёт
  дополнительную нагрузку на ESP8266 и изнашивает флеш-память.
- `device_class: energy` и `state_class: total_increasing` HA
  проставит автоматически при создании хелпера **Интеграция**.
- Не добавляй `sensor.battery_voltage` или
  `sensor.battery_charging_current` в Energy Dashboard напрямую —
  они не в кВт·ч и не `total_increasing`.

## Архитектура

### Скрипты

- `pi30_exchange` — единая точка обмена. Принимает `command` (string),
  формирует CRC (CRC-16/XMODEM), отправляет, читает ответ, парсит.
- `fast_poll` — QPIGS + Q1, с ретраем при ошибке.
- `slow_poll` — QPIRI + QMOD + QPIWS + QVFW (если версия ещё не получена).

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
   ```

## Известные проблемы

### CRC-ошибки (~10% опросов QPIGS)

Симптом: `PI30 QPIGS: CRC ERROR RX=XXXX CALC=YYYY`, где RX и CALC
отличаются на 1 бит.

Причина: физическая — бит-флип на линии RS232. Ретрай в `fast_poll`
спасает ситуацию.

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
