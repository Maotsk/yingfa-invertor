# Changelog

Все значимые изменения проекта.
Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.0.0/),
версии — по [Semantic Versioning](https://semver.org/lang/ru/).

## [1.0.0] — 2026-09-15

### Добавлено

- Мониторинг инвертора Yingfa 6.2kW (PI30) через ESP8266 + MAX3232 + ESPHome.
- QPIGS (20 параметров) + Q1 (2 температуры) — fast poll, по умолчанию 3 с.
- QPIRI (настройки) + QMOD (режим) + QPIWS (ошибки) — slow poll,
  по умолчанию 20 мин.
- QVFW (версия прошивки) с ретраем, если не получена.
- 4 селекта: Output Source Priority (POP), Input Voltage Range (PGR),
  Max Charging Current (MNCHGC), Charger Source Priority (PCP).
- 4 number (поле ввода): PBCV, PBDV, PCVV, PSDV.
- 2 кнопки: ручной slow poll, перезагрузка ESP.
- Защита UART от конфликтов через `uart_busy` / `pi30_running` /
  `slow_poll_active`.
- Диагностические сенсоры: Inverter Mode, Battery Type, Fault Status,
  Firmware Version.
- Интеграция с Home Assistant Energy Dashboard:
  - сенсоры мощности (W) через шаблоны;
  - сенсоры энергии (kWh) через интеграл (метод `Left`).
- Логирование `QPIGS raw` для отладки парсинга.
- Фильтр ложных срабатываний QPIWS при отключённых PV/батарее.
- GitHub Action для валидации конфига ESPHome.
- Двуязычный README (русский + английский).
- Документация по протоколу PI30 в `docs/protocol-notes.md`.

### Известные ограничения

- CRC-ошибки на ~10% опросов QPIGS. Причина физическая (бит-флип
  на линии RS232), ретрай в `fast_poll` компенсирует.
- `Battery Voltage = 12.80 V` при отключённой АКБ — реальное значение
  на клеммах, не баг парсинга.
- `Grid Import Energy` — расчётный сенсор, не прямое измерение.
- Предупреждение `script took a long time` — косметическое,
  не критично.
