# Yingfa 6.2kW PI30 Inverter — ESPHome Monitoring

**Русский** | [English](./README.en.md)

![ESPHome](https://img.shields.io/badge/ESPHome-2024.6%2B-blue)
![ESP8266](https://img.shields.io/badge/MCU-ESP8266-green)
![Protocol](https://img.shields.io/badge/protocol-PI30-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

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

Интервалы опроса настраиваются в Home Assistant
(см. раздел «Планировщик»):

- **QPIGS** (fast poll, по умолчанию 3 с): сетевое напряжение/частота,
  выход AC, нагрузка, BUS, батарея (напряжение, ток заряда, ёмкость),
  PV (ток, напряжение, мощность), ток разряда, статусные биты.
- **Q1** (fast poll, по умолчанию 3 с): температуры NTC инвертора и PV.
- **QPIRI** (slow poll, по умолчанию 20 мин): настройки — приоритеты,
  тип батареи, макс. ток заряда, напряжения батареи.
- **QMOD** (slow poll, по умолчанию 20 мин): текущий режим
  (Line / Battery / Fault / Standby).
- **QPIWS** (slow poll, по умолчанию 20 мин): флаги ошибок с фильтром
  ложных срабатываний.
- **QVFW** (slow poll, пока не получена): версия прошивки инвертора.
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

Настройка Energy Dashboard состоит из трёх шагов:

1. **Мощности (W)** — часть сенсоров уже есть в ESPHome, часть нужно
   вычислить через помощник **Шаблон**.
2. **Энергии (kWh)** — все сенсоры создаются через помощник
   **Интегральный** (в некоторых версиях русской локализации — **Интеграл**).
3. **Добавление в Energy Dashboard**.

### Шаг 1. Сенсоры мощности (W)

#### Уже есть в ESPHome (готовы к использованию)

| Сенсор | Что измеряет |
|---|---|
| `sensor.yingfa_invertor_ac_output_active_power` | Мощность на выходе инвертора |
| `sensor.yingfa_invertor_pv_charging_power` | Мощность от солнечных панелей |

Проверь в **Developer Tools → States**, что у них есть:
- `device_class: power`
- `state_class: measurement`

Эти атрибуты уже заданы в конфиге ESPHome.

#### Нужно создать в HA через помощник «Шаблон»

Home Assistant не умеет перемножать сенсоры напрямую, поэтому мощность
батареи (заряд и разряд) вычисляется шаблоном. Формула: **напряжение × ток**.

**Создание:**

1. **Настройки → Устройства и службы → Помощники → + Создать помощника**
2. Выбери **Шаблон** → **Шаблонный сенсор**
3. Заполни поля по таблицам ниже.

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

### Шаг 2. Сенсоры энергии (kWh)

Все четыре сенсора создаются одинаково — через помощник **Интегральный**
(Riemann sum integral). Он берёт мгновенную мощность в ваттах и накапливает
её в киловатт-часах.

**Общий порядок создания:**

1. **Настройки → Устройства и службы → Помощники → + Создать помощника**
2. Выбери **Интегральный** (в некоторых версиях — **Интеграл**)
3. Заполни поля по одной из таблиц ниже.

Поля **Метод интегрирования**, **Префикс единицы** и **Единица времени**
одинаковые для всех четырёх сенсоров:

| Поле | Значение |
|---|---|
| Метод интегрирования | `Left` (Левая сумма) |
| Префикс единицы | `k` |
| Единица времени | `h` |

> **Почему `Left`, а не `Trapezoidal`?** Сенсоры инвертора обновляются
> раз в 3 секунды и часто остаются на одном значении (например, PV = 0
> ночью). `Trapezoidal` в таких случаях даёт большую ошибку, `Left` —
> точнее. `Right` тоже работает, но завышает результат.

---

**1. AC Energy — потребление от инвертора**

| Поле | Значение |
|---|---|
| Название | `Yingfa Inverter AC Energy` |
| Входной сенсор | `sensor.yingfa_invertor_ac_output_active_power` |

---

**2. PV Energy — выработка солнечных панелей**

| Поле | Значение |
|---|---|
| Название | `Yingfa Inverter PV Energy` |
| Входной сенсор | `sensor.yingfa_invertor_pv_charging_power` |

---

**3. Battery Charge Energy — заряд батареи**

| Поле | Значение |
|---|---|
| Название | `Battery Charge Energy` |
| Входной сенсор | `sensor.battery_charge_power` |

---

**4. Battery Discharge Energy — разряд батареи**

| Поле | Значение |
|---|---|
| Название | `Battery Discharge Energy` |
| Входной сенсор | `sensor.battery_discharge_power` |

### Шаг 3. Добавление в Energy Dashboard

1. **Настройки → Панели → Энергия**
2. В разделе **Individual devices** (Отдельные устройства) добавь:
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
  проставит автоматически при создании помощника **Интегральный**.
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
