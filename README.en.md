# Yingfa 6.2kW PI30 Inverter — ESPHome Monitoring

[Русский](./README.md) | **English**

![ESPHome](https://img.shields.io/badge/ESPHome-2024.6%2B-blue)
![ESP8266](https://img.shields.io/badge/MCU-ESP8266-green)
![Protocol](https://img.shields.io/badge/protocol-PI30-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

Monitoring and control of a **Yingfa 6.2kW** inverter (PI30 protocol)
via ESP8266 + MAX3232 + ESPHome with Home Assistant integration.

## Hardware

| Component | Model / parameter |
|---|---|
| Microcontroller | ESP8266 (NodeMCU) |
| Converter | MAX3232 (RS232 ↔ TTL) |
| Inverter interface | RS232, 2400 baud, 8N1 |
| TX ESP → RX inverter | GPIO15 |
| RX ESP ← TX inverter | GPIO13 |

**Important:** ESP and inverter GND must be connected.
Power the ESP from a separate source or USB.

## Features

### Monitoring (sensors)

Polling intervals are configurable in Home Assistant
(see the "Scheduler" section):

- **QPIGS** (fast poll, default 3 s): grid voltage/frequency, AC output,
  load, BUS, battery (voltage, charge current, capacity), PV (current,
  voltage, power), discharge current, status bits.
- **Q1** (fast poll, default 3 s): inverter and PV NTC temperatures.
- **QPIRI** (slow poll, default 20 min): settings — priorities, battery
  type, max charge current, battery voltages.
- **QMOD** (slow poll, default 20 min): current mode
  (Line / Battery / Fault / Standby).
- **QPIWS** (slow poll, default 20 min): fault flags with false-positive
  filtering.
- **QVFW** (slow poll, until received): inverter firmware version.
  On failure — one retry, then next slow poll.

### Control

- **Selects:** Output Source Priority (POP), Input Voltage Range (PGR),
  Max Charging Current (MNCHGC), Charger Source Priority (PCP).
- **Number (input field):** battery voltages — PBCV (re-charge),
  PBDV (re-discharge), PCVV (C.V.), PSDV (cut-off).
- **Buttons:** manual slow poll trigger, ESP restart.

## Home Assistant Energy Dashboard Integration

Energy (kWh) calculation and power sensors are created on the
**Home Assistant** side, not on the ESP. This is a deliberate choice:
integration on the ESP8266 adds extra load and additional flash memory
writes, reducing the device's lifespan.

Setting up the Energy Dashboard consists of three steps:

1. **Power (W)** — some sensors already exist in ESPHome, others need to
   be computed via the **Template** helper.
2. **Energy (kWh)** — all sensors are created via the **Integral** helper
   (Riemann sum integral).
3. **Adding to Energy Dashboard**.

### Step 1. Power sensors (W)

#### Already in ESPHome (ready to use)

| Sensor | What it measures |
|---|---|
| `sensor.yingfa_invertor_ac_output_active_power` | AC Output Power — inverter output to load |
| `sensor.yingfa_invertor_pv_charging_power` | PV Power — solar panel generation |

Verify in **Developer Tools → States** that they have:
- `device_class: power`
- `state_class: measurement`

These attributes are already set in the ESPHome config.

#### Battery Power — via the "Template" helper

Home Assistant cannot multiply sensors directly, so battery power
(charge and discharge) is computed with a template.
Formula: **voltage × current**.

**Creation:**

1. **Settings → Devices & Services → Helpers → + Create Helper**
2. Choose **Template** → **Template a Sensor**
3. Fill in the fields per the tables below.

**Battery Charge Power:**

| Field | Value |
|---|---|
| Name | `Battery Charge Power` |
| State template | see code below |
| Unit of measurement | `W` |
| Device class | `power` |
| State class | `measurement` |

```jinja
{% set voltage = states('sensor.yingfa_invertor_battery_voltage') | float(0) %}
{% set current = states('sensor.yingfa_invertor_battery_charging_current') | float(0) %}
{{ (voltage * current) | round(1) }}
```

**Battery Discharge Power:**

| Field | Value |
|---|---|
| Name | `Battery Discharge Power` |
| State template | see code below |
| Unit of measurement | `W` |
| Device class | `power` |
| State class | `measurement` |

```jinja
{% set voltage = states('sensor.yingfa_invertor_battery_voltage') | float(0) %}
{% set current = states('sensor.yingfa_invertor_battery_discharge_current') | float(0) %}
{{ (voltage * current) | round(1) }}
```

#### Grid Import Power — computed, for Energy Dashboard

PI30 **does not provide** a separate sensor for grid input power.
Fields 0 and 1 in QPIGS are grid voltage and frequency — there is no
grid current. So grid power is computed from the power balance:

`Grid Import = AC Output − PV + Battery Charge − Battery Discharge`

**Creation via the "Template" helper:**

| Field | Value |
|---|---|
| Name | `Grid Import Power` |
| State template | see code below |
| Unit of measurement | `W` |
| Device class | `power` |
| State class | `measurement` |

```jinja
{% set ac_out = states('sensor.yingfa_invertor_ac_output_active_power') | float(0) %}
{% set pv = states('sensor.yingfa_invertor_pv_charging_power') | float(0) %}
{% set batt_chg = states('sensor.battery_charge_power') | float(0) %}
{% set batt_dis = states('sensor.battery_discharge_power') | float(0) %}
{% set grid = ac_out - pv + batt_chg - batt_dis %}
{{ [grid, 0] | max | round(1) }}
```

> The last line `[grid, 0] | max` clamps negative values to zero.
> This is needed because when PV fully covers the load, the formula
> may produce a negative result due to rounding error.
> Energy Dashboard does not accept negative values in import sensors.

> **Important:** this is an approximate calculation. PI30 does not
> provide the real grid current, so the formula relies on the power
> balance. Errors may accumulate, especially if the inverter efficiency
> is not 100%. But this is a standard trade-off for Voltronic-compatible
> inverters.

> If the source entity IDs differ (e.g. `sensor.battery_voltage` instead
> of `sensor.yingfa_invertor_battery_voltage`), adjust the template.
> Exact names are visible in **Developer Tools → States**.

### Step 2. Energy sensors (kWh)

All energy sensors are created the same way — via the **Integral**
helper (Riemann sum integral). It takes instantaneous power in watts
and accumulates it into kilowatt-hours.

**General procedure:**

1. **Settings → Devices & Services → Helpers → + Create Helper**
2. Choose **Integral**
3. Fill in the fields per one of the tables below.

The fields **Integration method**, **Unit prefix** and **Time unit** are
the same for all sensors:

| Field | Value |
|---|---|
| Integration method | `Left` |
| Unit prefix | `k` |
| Time unit | `h` |

> **Why `Left` instead of `Trapezoidal`?** Inverter sensors update every
> 3 seconds and often stay at the same value (e.g. PV = 0 at night).
> `Trapezoidal` gives a large error in such cases, `Left` is more
> accurate. `Right` also works but overestimates the result.

---

**1. AC Output Energy — inverter output**

| Field | Value |
|---|---|
| Name | `Yingfa Inverter AC Energy` |
| Input sensor | `sensor.yingfa_invertor_ac_output_active_power` |

---

**2. PV Energy — solar panel generation**

| Field | Value |
|---|---|
| Name | `Yingfa Inverter PV Energy` |
| Input sensor | `sensor.yingfa_invertor_pv_charging_power` |

---

**3. Battery Charge Energy**

| Field | Value |
|---|---|
| Name | `Battery Charge Energy` |
| Input sensor | `sensor.battery_charge_power` |

---

**4. Battery Discharge Energy**

| Field | Value |
|---|---|
| Name | `Battery Discharge Energy` |
| Input sensor | `sensor.battery_discharge_power` |

---

**5. Grid Import Energy — consumption from grid**

| Field | Value |
|---|---|
| Name | `Grid Import Energy` |
| Input sensor | `sensor.grid_import_power` |

### Step 3. Adding to Energy Dashboard

1. **Settings → Dashboards → Energy**
2. Under **Grid**, add:
   - `Grid Import Energy`
3. Under **Solar Panels**, add:
   - `Yingfa Inverter PV Energy`
4. Under **Battery**, add:
   - `Battery Charge Energy`
   - `Battery Discharge Energy`
5. Under **Individual devices**, add:
   - `Yingfa Inverter AC Energy`
6. Save.

### Important

- **Do not use** the `integration` platform in ESPHome — it adds extra
  load on the ESP8266 and wears out flash memory.
- `device_class: energy` and `state_class: total_increasing` are set
  automatically by HA when creating the **Integral** helper.
- Do not add `sensor.battery_voltage` or
  `sensor.battery_charging_current` to the Energy Dashboard directly —
  they are not in kWh and not `total_increasing`.
- `Grid Import Energy` is a **computed** sensor, not a direct measurement.
  Small deviations from real grid consumption are possible.

## Architecture

### Scripts

- `pi30_exchange` — single exchange point. Takes `command` (string),
  builds CRC (CRC-16/XMODEM), sends, reads response, parses.
- `fast_poll` — QPIGS + Q1, with retry on error.
- `slow_poll` — QPIRI + QMOD + QPIWS + QVFW (if version not yet received).

### UART conflict protection

- `uart_busy` — UART is busy.
- `pi30_running` — pi30_exchange script is running.
- `slow_poll_active` — slow poll in progress (blocks selects and numbers).

### Scheduler

`interval: 200ms` checks whether it's time to run fast/slow poll.
Intervals are configurable in HA:

- Fast Poll Interval (1–60 s, default 3)
- Slow Poll Interval (1–120 min, default 20)

### UART response reading

- Wait for the first byte (timeout 1 s).
- Wait for at least 5 bytes (up to 20 ms).
- Read until the line is silent for 15 ms.
- Verify CRC, retry on error (for QPIGS).

## Installation

1. Clone the repository.
2. Copy `secrets.yaml.example` → `secrets.yaml`, fill in:
   - `wifi_ssid`, `wifi_password`
   - `yingfa_api_key` (your own, not the example!)
   - `yingfa_ap_password` (your own, not the example!)
3. Set `device_ip` in `substitutions` — your ESP's IP on the local network.
   If you don't know it, check your router (DHCP leases) or the ESPHome
   Dashboard. Without this, OTA and API connection will not work.
4. Compile and flash:

   ```bash
   esphome run yingfa-invertor.yaml
   ```

## Known Issues

### CRC errors (~10% of QPIGS polls)

Symptom: `PI30 QPIGS: CRC ERROR RX=XXXX CALC=YYYY`, where RX and CALC
differ by 1 bit.

Cause: physical — bit flip on the RS232 line. The retry in `fast_poll`
handles the situation.

What to try:

- Connect ESP and inverter GND.
- Replace MAX3232 (cheap clones often cause this).
- Add 100 nF capacitor on MAX3232 VCC/GND.
- Shorten/shield the cable to the inverter.
- Reduce baud rate to 1200 (if supported).

### `script took a long time (max is 50 ms)`

Cosmetic ESPHome warning. Cause — the UART read loop waits for line
silence. Not critical, can be ignored.

### Battery Voltage = 12.80 V with battery disconnected

Not a parsing bug — the real value at the inverter terminals with no
battery connected. Capacity = 0 %, SCC = 0 V — also normal.

## PI30 Protocol Notes

Full command format see in [`docs/protocol-notes.md`](docs/protocol-notes.md).

Briefly:

- Command: ASCII + CRC-16/XMODEM (2 bytes big-endian) + `0x0D`.
- Response: `(` + ASCII body + CRC + `0x0D`.
- For QPIGS/QPIRI/Q1/QPIWS the response arrives as a single packet,
  inter-byte pause is minimal.

### Commands Used

| Command | Purpose |
|---|---|
| QPIGS | General data (20 fields) |
| QPIRI | Inverter settings |
| QMOD | Operating mode (1 char) |
| QPIWS | Fault flags (32 bits) |
| Q1 | NTC temperatures |
| QVFW | Firmware version |
| POP0X | Output Source Priority |
| PGR0X | Input Voltage Range |
| MNCHGCXXX | Max Charging Current |
| PCP0X | Charger Source Priority |
| PBCVxx.x | Battery Re-charge Voltage |
| PBDVxx.x | Battery Re-discharge Voltage |
| PCVVxx.x | Battery C.V. Voltage |
| PSDVxx.x | Battery Cut-off Voltage |

## Useful Links

- [ESPHome UART](https://esphome.io/components/uart.html)
- [PI30 Protocol (Voltronic)](https://github.com/jblance/mpp-solar)
- [Home Assistant Number](https://www.home-assistant.io/integrations/number/)

## License

This project is distributed under the MIT License.
See the [LICENSE](LICENSE) file for details.
