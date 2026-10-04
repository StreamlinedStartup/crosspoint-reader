# reTerminal E1001 Hardware Notes

Board: v1.2, verified on device (SY6974B at I2C1 0x6B).
Sources: schematic v1.2 (2025-11-20), Zephyr `boards/seeed/reterminal_e1001`, the Seeed wiki, and GxEPD2.
Pins match on both the v1.2 schematic and the Zephyr dts unless the Source column says otherwise.

## Chip

| Item | Value | Status |
|---|---|---|
| SoC | ESP32-S3 (QFN56) rev v0.2 | verified (`flash-id`) |
| PSRAM | 8 MB embedded, octal (uses GPIO33-37) | verified (`flash-id`) |
| Flash | 32 MB W25Q256JV, quad, 3.3 V | verified (`flash-id`, ef/4019) |
| USB serial | CH340C on UART0 (GPIO43/44), DTR/RTS auto-reset | verified |
| Native USB | unusable, because GPIO19/20 carry I2C0 | schematic |

## Pin Map

| Part | Signal | GPIO | Polarity / notes | Source |
|---|---|---|---|---|
| SPI bus | SCK / MOSI | 7 / 9 | Shared by screen and SD | both |
| | MISO | 8 | SD only. The panel is write-only. | v1.2 sch |
| Screen (UC8179) | CS | 10 | Active low | both |
| | DC | 11 | | both |
| | RST | 12 | Active low. **Also enables the panel's 3.3 V load switch.** RST low = panel unpowered. | both |
| | BUSY | 13 | Low = busy | both |
| SD (SPI mode) | CS | 14 | Active low | v1.2 sch |
| | Card detect | 15 | Low = card present (1 M pull-up) | v1.2 sch |
| | Power enable | 16 | Active high (TPS22916) | v1.2 sch |
| Buttons | Right / Middle / Left | 3 / 4 / 5 | Active low, external 10 K pull-ups, RTC-capable. Left-to-right order with buttons on top, verified on device. GPIO3 is the stock power key. | both |
| Battery | ADC | 1 | ADC1_CH0, divider 10 K / 10 K, V_bat = 2 x V_adc | both |
| | Read enable | 21 | Active high. Settle 10 ms before reading. | both |
| LED (green) | | 6 | **Active low** | both |
| Buzzer | | 45 | Active high, magnetic. Needs PWM around 2.7 kHz. | both |
| I2C0 | SDA / SCL | 19 / 20 | PCF8563 @0x51, SHT40 @0x44 | both |
| I2C1 | SDA / SCL | 39 / 40 | SY6974B charger @0x6B (v1.2 only) | both |
| Mic (PDM) | EN / DATA / CLK | 38 / 41 / 42 | Not used by the port | v1.2 sch |
| Header J2 | | 2, 17, 18, 46 + I2C0 | GPIO46 must not be held high at reset | v1.2 sch |
| BOOT button | | 0 | 10 K pull-up | v1.2 sch |

## Screen

- Panel: GoodDisplay GDEY075T7, 800 x 480, 1-bit, landscape native.
- Controller: UC8179, write-only 4-wire SPI. Zephyr runs it at 4 MHz and Seeed's GxEPD2 examples at 2 MHz.
- Zephyr init sequence:
  1. RST low 10 ms, release, wait 10 ms, wait for BUSY high
  2. `01 07 07 3F 3F` (PWR)
  3. `06 17 17 17 17` (BTST)
  4. `00 1F` (PSR, KW mode, OTP LUT)
  5. `61 03 20 01 E0` (TRES 800 x 480)
  6. `50 29 07` (CDI)
  7. `60 22` (TCON)
- Full refresh: `04`, wait for BUSY, `12`, wait for BUSY, `02`.
- Fast refresh (GxEPD2): `E0 02`, then `E5 5A` (fast full) or `E5 6E` (fast partial, OTP LUT).
- Hibernate (GxEPD2): `02`, then `07 A5`. On this board, RST low also cuts panel power.
- After the panel power is cut, both frame RAMs are lost. Rewrite DTM1 (`10`) and DTM2 (`13`) before any partial refresh.
- Seeed ships a 4-gray mode that uses custom register LUTs. The port will not use it at first.

## Power and Sleep

- All three keys can wake the chip through ext1 `ANY_LOW` with mask `(1<<3)|(1<<4)|(1<<5)`.
- RTC INT and charger /INT go only to test points, so neither can wake the ESP.
- Power switch SW1, OFF position: holds ESP_RST to GND.
- Power switch SW1 on v1.2: also powers the CH340C. USB serial likely needs the switch ON. (Inferred, unverified.)

## Strapping Pins

| GPIO | Role | Effect here |
|---|---|---|
| 0 | Boot mode | BOOT button, pulled up |
| 3 | JTAG source | KEY0. Harmless unless the JTAG eFuse is burned (never burn eFuses). |
| 45 | VDD_SPI voltage | Buzzer. Pulled low, so 3.3 V flash. Never drive high at reset. |
| 46 | Boot mode | Header only |

## v1.0 to v1.2 Differences

| Change | v1.0 | v1.2 | Firmware impact |
|---|---|---|---|
| Charger | ETA6003, no I2C | SY6974B on I2C1 @0x6B | Probe 0x6B to identify the revision. The watchdog may need to be disabled if firmware writes registers. |
| CH340C power | Always on from USB | Through SW1 | Serial likely needs the switch ON |
| Q9 boost FET | DMN3023L | DMN3404L | None |

## Unconfirmed

- Whether SW1 must be ON for USB serial.
- Whether the TPS22916 EN pins have internal pull-downs, which affects SD and VBAT rail state in deep sleep.

## Port Findings

- The UC8179 driver must not write register 0xE1 (gate scan). The X4 Pro value 0x02 drives each row onto two gates on this glass, which stretches the image 2x vertically.
- Fast full refresh: TSSET 0x5A, about 1.6 s. Fast partial refresh: TSSET 0x6E, about 0.9 s.
- The X4 Pro grayscale waveforms are off for this panel.
- The CH340C link corrupts data above 230400 baud.
- Build with pioarduino core 6.1.19. PlatformIO 6.2.0 fails with `No module named 'SCons.Tool.FortranCommon'`.
