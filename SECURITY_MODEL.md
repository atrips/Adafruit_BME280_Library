# Threat model: atrips/Adafruit_BME280_Library

Status: inferred from the code (v1.0.6, `library.properties`); no attacker or asset priorities have been confirmed by the maintainers yet.

## System overview and trust boundaries
An Arduino C++ driver for the Bosch BME280 temperature/humidity/pressure sensor. It is compiled into a user's sketch and runs on a microcontroller (AVR, SAMD, ESP8266/ESP32, ...), with no OS, MMU or network stack of its own.

Trust boundary: **MCU firmware <-> sensor over the I2C or SPI bus.** Everything read back from the bus (chip ID, status register, calibration coefficients, ADC samples) crosses from the sensor/bus into the firmware. The calling sketch is trusted (same firmware image).

## Components
- **Bus I/O** - `Adafruit_BME280.cpp`: `init()`, `write8`, `read8`, `read16`, `read24`, `spixfer` (I2C via `TwoWire`, hardware SPI, bit-banged software SPI).
- **Calibration** - `Adafruit_BME280.cpp`: `readCoefficients()`, `isReadingCalibration()`; struct `bme280_calib_data` in `Adafruit_BME280.h`.
- **Compensation** - `Adafruit_BME280.cpp`: `readTemperature`, `readPressure`, `readHumidity`, `readAltitude`, `seaLevelForAltitude` (fixed-point/int64 math on calibration + ADC values).
- **Public API** - `Adafruit_BME280.h`: constructors, `begin()` overloads, `setSampling()`, `takeForcedMeasurement()`, `Adafruit_BME280_Unified` declaration.
- **Examples** - `examples/bme280test`, `examples/advancedsettings` (sample sketches, not shipped code).

## Entry points and who can reach them
- Sensor/bus data: reachable by whoever controls the sensor or the wires (faulty, counterfeit or tampered sensor; physical attacker on the bus; another device on a shared I2C bus answering at 0x76/0x77).
- API parameters (`begin(addr)`, pin numbers, `setSampling` enums, `seaLevel`/`altitude` floats): reachable only by the sketch author (trusted).
- No network, serial-command, filesystem or persistent-storage surface in the library.

## Assets
- Availability of the firmware (a hang in the driver blocks the whole sketch).
- Integrity of sensor readings (may feed control/safety logic in the end product).
- Memory safety / defined behaviour of the MCU firmware.
- No secrets, credentials or personal data are handled.

## Attacker profiles
1. Faulty or counterfeit sensor / noisy bus returning arbitrary bytes.
2. Physical attacker with access to the sensor or the I2C/SPI lines (can spoof any response).
3. Malicious or buggy sketch code (trusted by design; out of the library's control).

## Existing controls
- Chip-ID check (`0x60`) in `init()` before use.
- Division-by-zero guard in `readPressure()` (`var1 == 0`).
- "Measurement disabled" sentinels return `NAN` (`0x800000`, `0x8000`).
- Humidity result is clamped to `[0, 419430400]`.

## Notable areas (from code reading, not user-confirmed)
- Unbounded busy-waits on sensor status: `while (isReadingCalibration()) delay(100);` in `init()` and `while (read8(STATUS) & 0x08) delay(1);` in `takeForcedMeasurement()`.
- Signed shifts/multiplications on attacker-influenced calibration and ADC values in the compensation code (possible signed overflow = UB).
- No return-value checks on `endTransmission()` / `requestFrom()`.

## Out of scope / accepted risk
None stated by the user yet.

## Open questions
- Which attackers matter (all, physical only, robustness only, none)?
- Which asset matters most (availability, reading integrity, memory safety)?
- Any product firmware / infrastructure repository for context? (none found; the scan covers the library alone)
