# SPO2-PulseBoard
A custom heart rate and SpO2 monitor built on an Elegoo Uno R3 with a MAX30105 sensor and SSD1306 OLED display. Features a custom PCB, mode-switching buttons, and an alarm LED. Uses lightweight R-ratio SpO2 calculation and peak-detection heart rate — all tuned to run within the Arduino Uno's RAM limits.



<img width="4032" height="3024" alt="IMG_5669" src="https://github.com/user-attachments/assets/b8a35f74-d82a-4b67-ad60-d8907aa285ce" />

# PulseBoard

A custom heart rate and SpO2 monitor built on an Elegoo Uno R3 with a MAX30105 sensor and SSD1306 OLED display. Features a custom PCB, mode-switching buttons, and an alarm LED. Uses lightweight R-ratio SpO2 calculation and peak-detection heart rate — all tuned to run within the Arduino Uno's RAM limits.

---

## Hardware

| Component | Details |
|-----------|---------|
| Microcontroller | Elegoo Uno R3 |
| Sensor | MAX30105 optical heart rate / SpO2 |
| Display | SSD1306 128×32 OLED (I2C) |
| PCB | Custom — see `/hardware` for Gerber files |

### Pin Map

| Pin | Function |
|-----|----------|
| D2 | Menu button — cycles between Heart Rate and SpO2 mode |
| D3 | Up button — reserved for future use |
| D4 | Down button — reserved for future use |
| D12 | Alarm LED — flashes when no finger is on the sensor |
| SDA / SCL | Shared I2C bus → MAX30105 + SSD1306 |

### Button Circuit
All three buttons are active-low, held HIGH by 1kΩ pull-up resistors (R1, R2, R3) with 100nF decoupling capacitors for hardware debounce. The alarm LED is driven through a 1kΩ current-limiting resistor (R4).

---

## Features

- **Heart Rate (BPM)** — peak detection on a rolling IR signal window with a 4-beat rolling average
- **SpO2 (%)** — R-ratio method with a quadratic calibration formula for accurate readings in the healthy range
- **Mode switching** — press D2 to toggle between Heart Rate and SpO2 mode; a splash screen confirms the switch
- **Alarm LED** — flashes at 1Hz on D12 when no finger is detected, turns off immediately when a finger is placed
- **Stable display** — 500ms refresh throttle prevents flicker; readings are held back for the first few seconds until the signal stabilizes

---

## How It Works

### Finger Detection
The sketch polls the MAX30105 every loop. An IR value above 50,000 indicates a finger is present (tissue absorbs the light). Below that threshold, the sensor is reading open air. All HR and SpO2 state resets on each new finger placement.

### Heart Rate Detection
A 10-sample rolling window tracks the IR signal's min, max, and range. Beat detection fires when the signal crosses 50% of the range, subject to these parameters:

| Parameter | Value |
|-----------|-------|
| Threshold | 50% of signal range |
| Minimum signal range | 400 |
| Minimum time between beats | 400ms |
| First beat | Always skipped (timing invalid) |
| Valid BPM range | 50 – 120 |
| Rolling average | 4 beats |
| Avg display delay | 8 seconds |

### SpO2 Calculation
AC (peak-to-peak) and DC (midpoint) values are computed for both the Red and IR channels from the same 10-sample window. The R-ratio is calculated as:

```
R = (redAC / redDC) / (irAC / irDC)
```

R is then passed through a quadratic calibration formula fitted to empirical blood oxygen absorption data:

```
SpO2 = -45.060 × R² + 30.354 × R + 94.845
```

The result is clamped between 70–100% and smoothed with a 9-reading rolling average. Readings are displayed after 5 seconds.

> **Note:** This formula is accurate in the healthy range (95–100%). Readings below ~90% should be treated as approximate — the sensor is not designed for clinical low-SpO2 accuracy.

---

## Libraries Required

Install these through the Arduino IDE Library Manager before compiling:

- [Adafruit SSD1306](https://github.com/adafruit/Adafruit_SSD1306)
- [Adafruit GFX](https://github.com/adafruit/Adafruit-GFX-Library)
- [SparkFun MAX3010x](https://github.com/sparkfun/SparkFun_MAX3010x_Sensor_Library) — provides `MAX30105.h`

> **Note:** This project does NOT use `spo2_algorithm.h`. That library was tested but exceeded the Uno's available RAM. SpO2 logic here is a custom lightweight implementation.

---

## Usage

1. Wire up the hardware according to the pin map above or use the custom PCB
2. Install the required libraries
3. Upload `heart_monitor.ino` to your Elegoo Uno R3
4. Place your finger **lightly** on the sensor — pressing too hard flattens the pulse waveform and stops beat detection
5. Press **D2** to switch between Heart Rate and SpO2 mode
6. Wait 8 seconds for a stable heart rate average, or 5 seconds for an SpO2 reading

---

## Known Limitations

- **Finger pressure is critical** — too much pressure drops the signal range below 400 and halts detection
- **INT pin unused** — the MAX30105 interrupt pin is wired on the PCB but unconnected in firmware. Currently the sketch polls the sensor every loop. Future enhancement: wire INT to D7 for event-driven reads
- **SpO2 accuracy** — reliable in the 95–100% range; not suitable for clinical use
- **BPM range** — capped at 50–120 BPM; athletes with resting HR below 50 will see no reading

---

## Project Structure

```
PulseBoard/
├── firmware/
│   └── heart_monitor.ino
├── hardware/
│   └── (Gerber files, schematic)
└── README.md
```

---

## License

MIT
