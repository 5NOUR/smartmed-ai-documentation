# Hardware and Data Collection

## 1. Hardware interfaces found in source

GPIO numbering is BCM, not physical header numbering.

| Function | BCM GPIO / bus | Physical pin note |
|---|---:|---:|
| Compartment 1 green LED | GPIO17 | Physical 11 |
| Compartment 1 red LED | GPIO27 | Physical 13 |
| Compartment 2 green LED | GPIO22 | Physical 15 |
| Compartment 2 red LED | GPIO23 | Physical 16 |
| Compartment 3 green LED | GPIO24 | Physical 18 |
| Compartment 3 red LED | GPIO25 | Physical 22 |
| Compartment 4 green LED | GPIO5 | Physical 29 |
| Compartment 4 red LED | GPIO6 | Physical 31 |
| Compartment 1 MC-38 | GPIO12 | Physical 32 |
| Compartment 2 MC-38 | GPIO13 | Physical 33 |
| Compartment 3 MC-38 | GPIO19 | Physical 35 |
| Compartment 4 MC-38 | GPIO16 | Physical 36 |
| Buzzer control | GPIO26 | Physical 37 |
| HX711 DT | GPIO20 | Physical 38 |
| HX711 SCK | GPIO21 | Physical 40 |
| DS3231 | I²C bus 1, address 0x68 | SDA/SCL pins depend on Pi header |

The buzzer source comment specifies a 5V active buzzer driven through a 2N2222 transistor and a 1kΩ base resistor. Do not connect a 5V buzzer directly to a GPIO pin.

## 2. Sensor behavior

### MC-38 compartment sensors
The source configures four `gpiozero.Button` inputs with pull-up enabled and 0.1-second debounce. The code treats:
- pressed as `closed`
- released as `open`

Each transition is recorded in the `events` table with source `hardware`.

### HX711 and load cell
The weight sensor:
- Reads 24-bit HX711 values.
- Applies a tare offset and calibration factor stored in `system_config`.
- Takes multiple samples and returns their median.
- Is connected in software only to compartment 4.

When compartment 4 opens, the service records a baseline weight. When it closes, it reads again. An absolute difference of at least 5 grams is recorded as `possible_removal` or `possible_addition`.

A weight change does not prove a person took the medication. Container movement, vibration, placement, or calibration drift can also change the reading.

### DS3231 RTC
The RTC manager reads and writes the DS3231 over I²C at address `0x68`. The current startup code attempts to sync the RTC from the system UTC clock when the device responds.

## 3. Data required from real Raspberry Pi experiments

For each test session, collect:
- Date/time and timezone.
- Software Git commit or version.
- Raspberry Pi OS/kernel/Python version.
- Sensor wiring revision.
- Compartment ID.
- Scheduled time and actual lid-open/lid-close timestamps.
- Whether the event was expected, late, missed, or a wrong-compartment action.
- Human-reviewed ground-truth label.
- Weight before and after, if compartment 4 is tested.
- Calibration factor, tare offset, reference mass, and measurement repeatability.
- Whether the RTC was available and its offset from a trusted clock.
- Any sensor bounce, missed transition, false trigger, or power interruption.

Do not store patient names, phone numbers, diagnoses, or medication details in public experiment files.

## 4. Suggested controlled tests

| Test | Procedure | Record |
|---|---|---|
| Sensor open/close reliability | 30 open-close cycles per compartment | Detected transitions / 30 |
| Debounce behavior | Open/close slowly and quickly | False/missed transitions |
| Schedule matching | Test before, within, and after dose window | Correct state and alert |
| Wrong compartment | Open a non-due compartment | Alarm and database record |
| Missed window | Do not open the active compartment | Missed alert timing |
| HX711 repeatability | 20 readings at zero and at known masses | Mean, standard deviation, range |
| Weight-change threshold | Add/remove known masses around 5g | Detection sensitivity |
| RTC accuracy | Compare RTC with trusted clock over 24h | Time drift in seconds |
| Long-duration operation | Run continuously for 8–24h | Crashes, memory, CPU temp, event loss |

## 5. Data export

The SQLite database is stored at `data/smartmed.db`. For controlled testing, export only the minimum required columns and replace any identifying IDs with random experiment IDs.

Example schema inspection:

```bash
sqlite3 data/smartmed.db ".tables"
sqlite3 data/smartmed.db "SELECT id, compartment_id, event_type, occurred_at, source FROM events ORDER BY occurred_at DESC LIMIT 20;"
```

Before sharing logs, inspect and remove personal data. Never upload the live database to a public GitHub repository.
