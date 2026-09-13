# Заметки по протоколу PI30 (Yingfa 6.2kW)

## Формат кадра

### Запрос (TX)

```
[ASCII команда] [CRC_H] [CRC_L] [0x0D]
```

CRC-16/XMODEM:
- init = 0x0000
- poly = 0x1021
- без отражения, без финального XOR
- big-endian в кадре

### Ответ (RX)

```
( [ASCII тело] [CRC_H] [CRC_L] [0x0D]
```

- Начинается с `(` (0x28), заканчивается `0x0D`.
- CRC считается от `(` до последнего байта тела включительно.

## QPIGS — 21 поле

Пример: `221.0 50.0 221.0 50.0 0950 0939 015 308 12.80 000 000 0031 00.0 000.0 00.00 00000 00010000 00 00 00000 010`

| # | Поле | Ед. |
|---|---|---|
| 0 | Grid voltage | V |
| 1 | Grid frequency | Hz |
| 2 | AC output voltage | V |
| 3 | AC output frequency | Hz |
| 4 | AC output apparent power | VA |
| 5 | AC output active power | W |
| 6 | Output load | % |
| 7 | BUS voltage | V |
| 8 | Battery voltage | V |
| 9 | Battery charging current | A |
| 10 | Battery capacity | % |
| 11 | Inverter heat sink temperature | °C |
| 12 | PV input current | A |
| 13 | PV input voltage | V |
| 14 | Battery voltage from SCC | V |
| 15 | Battery discharge current | A |
| 16 | Device status bits | — |
| 17 | Battery voltage offset (fan) | — |
| 18 | EEPROM version | — |
| 19 | PV charging power | W |
| 20 | Device status | — |

В проекте парсятся поля 0–16 и 19.

## QPIRI — настройки

Пример: `230.0 26.9 230.0 50.0 26.9 6200 6200 48.0 47.1 41.9 58.4 54.0 2 010 010 1 1 2 1 01 0 0 54.0 0 1 47.1 10 44.0`

Используемые поля:
- 8 — Battery re-charge voltage (PBCV)
- 9 — Battery cut-off voltage (PSDV)
- 10 — Battery C.V. voltage (PCVV)
- 12 — Battery type (0=AGM, 1=Flooded, 2=User, 3=LIP, 4=LIL, 5=LIB)
- 14 — Max charging current (A)
- 15 — Input voltage range (0=Appliance, 1=UPS)
- 16 — Output source priority (0=Utility, 1=Solar, 2=SBU)
- 17 — Charger source priority (0/1=S+U, 2=Only Solar, 3=Solar First)
- 22 — Battery re-discharge voltage (PBDV)

## QMOD — режим (1 символ)

| Символ | Режим |
|---|---|
| P | Power On |
| S | Standby |
| L | Line Mode |
| B | Battery Mode |
| F | Fault Mode |
| H | Power Saving |
## QPIWS — 32 бита флагов

Порядок: бит 0 — старший (первый символ строки).

| # | Флаг |
|---|---|
| 0 | Inverter Fault |
| 1 | Bus Over |
| 2 | Bus Under |
| 3 | Bus Soft Fail |
| 4 | Line Fail |
| 5 | Output Short Circuit |
| 6 | Inverter Voltage Too Low |
| 7 | Inverter Voltage Too High |
| 8 | Over Temperature |
| 9 | Fan Locked |
| 10 | Battery Voltage High |
| 11 | Battery Low Alarm |
| 12 | Battery Under Shutdown |
| 13 | Over Load |
| 14 | EEPROM Fault |
| 15 | Inverter Soft Fail |
| 16 | Self Test Fail |
| 17 | OP DC Voltage Over |
| 18 | Battery Open |
| 19 | Current Sensor Fail |
| 20 | Battery Short |
| 21 | Power Limit |
| 22 | PV Voltage High |
| 23 | MPPT Overload Fault |
| 24 | MPPT Overload Warning |
| 25 | Battery Too Low To Charge |

В проекте: биты 0 и 22 игнорируются, если PV < 5 В и батарея < 20 В
(защита от ложных срабатываний при отключённом оборудовании).

## Команды управления

| Команда | Формат | Примечание |
|---|---|---|
| POP | `POP0X` | X: 0=Utility, 1=Solar, 2=SBU |
| PGR | `PGR0X` | X: 0=Appliance, 1=UPS |
| MNCHGC | `MNCHGCXXX` | XXX: 010–120 A |
| PCP | `PCP0X` | X: 1=S+U, 2=Only Solar, 3=Solar First |
| PBCV | `PBCVxx.x` | Напряжение перезаряда |
| PBDV | `PBDVxx.x` | Напряжение повторного разряда |
| PCVV | `PCVVxx.x` | Напряжение C.V. |
| PSDV | `PSDVxx.x` | Напряжение отключения |

Ответ: `(ACK` или `(NAK`.

## Замечания по отладке

- Логи ESPHome удобно смотреть через `esphome logs yingfa-invertor.yaml`.
- При CRC-ошибках в логе видно `RX=` и `CALC=` — по разнице видно, какой бит «слетел».
- `QPIGS raw:` в логе показывает тело ответа — удобно для проверки парсинга.
- `QPIWS bits:` в логе показывает 32-битную строку — по ней видно, какие флаги подняты.
