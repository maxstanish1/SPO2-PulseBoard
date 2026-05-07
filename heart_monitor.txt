/*
 * Heart Rate & SpO2 Monitor
 * Hardware: Elegoo Uno R3 + MAX30105 + SSD1306 OLED
 * 
 * Pin assignments (from PCB schematic):
 *   D2  → Menu button  (cycles between HR and SpO2 mode)
 *   D3  → Up button    (reserved / future use)
 *   D4  → Down button  (reserved / future use)
 *   D12 → Alarm LED    (flashes when no finger on sensor)
 *   SDA/SCL → MAX30105 + SSD1306 on I2C bus
 *
 * Modes:
 *   MODE 0 = Heart Rate (BPM)
 *   MODE 1 = Blood Oxygen (SpO2 %)
 *
 * Button D2 cycles between modes.
 * LED on D12 flashes at 1Hz when no finger is detected, stays OFF when finger is on.
 */

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "MAX30105.h"

// ─── Display ──────────────────────────────────────────────────────────────────
#define SCREEN_WIDTH    128
#define SCREEN_HEIGHT    32
#define OLED_RESET       -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ─── Sensor ───────────────────────────────────────────────────────────────────
MAX30105 particleSensor;

// ─── Pin definitions (from Gerber schematic) ──────────────────────────────────
#define PIN_BTN_MENU  2    // Menu button  → cycles mode
#define PIN_BTN_UP    3    // Up button    → reserved
#define PIN_BTN_DOWN  4    // Down button  → reserved
#define PIN_ALARM_LED 12   // Alarm LED    → flashes when no finger

// ─── Mode ─────────────────────────────────────────────────────────────────────
#define MODE_HR   0
#define MODE_SPO2 1
int currentMode = MODE_HR;

// ─── Button debounce ──────────────────────────────────────────────────────────
bool     lastMenuState     = HIGH;
unsigned long lastMenuTime = 0;
#define DEBOUNCE_MS 50

// ─── LED flash state ──────────────────────────────────────────────────────────
unsigned long lastLedToggle = 0;
bool          ledState       = false;
#define LED_FLASH_INTERVAL 500

// ─── Heart rate detection ─────────────────────────────────────────────────────
const byte RATE_SIZE = 4;
byte       rates[RATE_SIZE];
byte       rateSpot        = 0;
long       lastBeat        = 0;
float      beatsPerMinute  = 0;
int        beatAvg         = 0;
int        displayBPM      = 0;
bool       beatDetected    = false;
bool       firstBeatSkipped = false;

// ─── IR signal buffers ────────────────────────────────────────────────────────
long irValueBuffer[10];
byte bufferIndex = 0;
long irMin = 999999;
long irMax = 0;

// ─── SpO2 calculation buffers ─────────────────────────────────────────────────
const byte SPO2_BUF_SIZE = 9;
long  redBuffer[10];
long  irBuffer_spo2[10];
byte  spo2BufIndex   = 0;
int   displaySpO2    = 0;
float spo2Readings[SPO2_BUF_SIZE];
byte  spo2ReadIndex  = 0;
byte  spo2Count      = 0;

// ─── Finger detection state ───────────────────────────────────────────────────
bool          fingerCurrentlyOn  = false;
unsigned long fingerPlacedTime   = 0;

// ─── Display refresh throttle ─────────────────────────────────────────────────
unsigned long lastDisplayUpdate = 0;
#define DISPLAY_REFRESH_MS 500


void setup() {
  Serial.begin(9600);
  delay(500);
  Serial.println("=== Heart Monitor Starting ===");

  pinMode(PIN_BTN_MENU, INPUT);
  pinMode(PIN_BTN_UP,   INPUT);
  pinMode(PIN_BTN_DOWN, INPUT);

  pinMode(PIN_ALARM_LED, OUTPUT);
  digitalWrite(PIN_ALARM_LED, LOW);

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("ERROR: Display failed!");
    for (;;);
  }
  Serial.println("Display OK");

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("ERROR: Sensor failed!");
    while (1);
  }
  Serial.println("Sensor OK");

  particleSensor.setup();
  particleSensor.setPulseAmplitudeRed(0x3F);
  particleSensor.setPulseAmplitudeIR(0x3F);

  for (int i = 0; i < 10;           i++) { irValueBuffer[i] = 0; redBuffer[i] = 0; irBuffer_spo2[i] = 0; }
  for (int i = 0; i < RATE_SIZE;    i++)   rates[i] = 0;
  for (int i = 0; i < SPO2_BUF_SIZE; i++) spo2Readings[i] = 0;

  Serial.println("=== Ready — press finger LIGHTLY ===");
  showModeSplash();
}


void loop() {
  handleMenuButton();

  long irValue  = particleSensor.getIR();
  long redValue = particleSensor.getRed();

  if (irValue > 50000 && !fingerCurrentlyOn) {
    fingerCurrentlyOn = true;
    fingerPlacedTime  = millis();
    lastBeat          = millis();

    beatAvg = 0; rateSpot = 0; displayBPM = 0;
    beatDetected = false; firstBeatSkipped = false;
    for (int i = 0; i < RATE_SIZE; i++) rates[i] = 0;

    displaySpO2 = 0; spo2ReadIndex = 0; spo2Count = 0;
    for (int i = 0; i < SPO2_BUF_SIZE; i++) spo2Readings[i] = 0;

    digitalWrite(PIN_ALARM_LED, LOW);
    ledState = false;
    Serial.println(">>> Finger placed");

  } else if (irValue < 50000 && fingerCurrentlyOn) {
    fingerCurrentlyOn = false;
    Serial.println(">>> Finger removed");
  }

  if (!fingerCurrentlyOn) {
    if (millis() - lastLedToggle >= LED_FLASH_INTERVAL) {
      lastLedToggle = millis();
      ledState = !ledState;
      digitalWrite(PIN_ALARM_LED, ledState ? HIGH : LOW);
    }
  }

  irValueBuffer[bufferIndex] = irValue;
  redBuffer[bufferIndex]     = redValue;
  irBuffer_spo2[bufferIndex] = irValue;
  bufferIndex = (bufferIndex + 1) % 10;

  irMin = 999999; irMax = 0;
  for (int i = 0; i < 10; i++) {
    if (irValueBuffer[i] < irMin) irMin = irValueBuffer[i];
    if (irValueBuffer[i] > irMax) irMax = irValueBuffer[i];
  }

  long range     = irMax - irMin;
  long threshold = irMin + (range * 0.50);

  if (fingerCurrentlyOn && range > 400) {
    if (irValue > threshold && !beatDetected && (millis() - lastBeat > 400)) {
      beatDetected = true;
      long delta = millis() - lastBeat;
      lastBeat   = millis();

      if (!firstBeatSkipped) {
        firstBeatSkipped = true;
        Serial.println("FIRST BEAT — skipped (calibrating)");
      } else {
        beatsPerMinute = 60000.0 / delta;
        if (beatsPerMinute >= 50 && beatsPerMinute <= 120) {
          displayBPM = (int)beatsPerMinute;
          rates[rateSpot++] = (byte)beatsPerMinute;
          rateSpot %= RATE_SIZE;
          int validCount = 0; beatAvg = 0;
          for (byte x = 0; x < RATE_SIZE; x++) {
            if (rates[x] > 0) { beatAvg += rates[x]; validCount++; }
          }
          if (validCount > 0) beatAvg /= validCount;
          Serial.print("BEAT! BPM="); Serial.print(beatsPerMinute, 1);
          Serial.print(" Avg="); Serial.println(beatAvg);
        }
      }
    }
    if (irValue < (threshold - (range * 0.15))) beatDetected = false;
  }

  if (fingerCurrentlyOn) {
    long redWinMin = 999999, redWinMax = 0;
    long irWinMin  = 999999, irWinMax  = 0;
    for (int i = 0; i < 10; i++) {
      if (redBuffer[i] > 0) {
        if (redBuffer[i] < redWinMin) redWinMin = redBuffer[i];
        if (redBuffer[i] > redWinMax) redWinMax = redBuffer[i];
      }
      if (irBuffer_spo2[i] > 0) {
        if (irBuffer_spo2[i] < irWinMin) irWinMin = irBuffer_spo2[i];
        if (irBuffer_spo2[i] > irWinMax) irWinMax = irBuffer_spo2[i];
      }
    }

    float redAC = redWinMax - redWinMin;
    float irAC  = irWinMax  - irWinMin;
    float redDC = (redWinMax + redWinMin) / 2.0;
    float irDC  = (irWinMax  + irWinMin)  / 2.0;

    if (redDC > 0 && irDC > 0 && irAC > 0 && redAC > 0) {
      float R = (redAC / redDC) / (irAC / irDC);
      float spo2Val = -45.060 * R * R + 30.354 * R + 94.845;
      if (spo2Val > 100.0) spo2Val = 100.0;
      if (spo2Val < 70.0)  spo2Val = 70.0;

      spo2Readings[spo2ReadIndex] = spo2Val;
      spo2ReadIndex = (spo2ReadIndex + 1) % SPO2_BUF_SIZE;
      if (spo2Count < SPO2_BUF_SIZE) spo2Count++;

      float sum = 0;
      for (int i = 0; i < spo2Count; i++) sum += spo2Readings[i];
      displaySpO2 = (int)(sum / spo2Count);
    }
  }

  if (millis() - lastDisplayUpdate >= DISPLAY_REFRESH_MS) {
    lastDisplayUpdate = millis();
    updateDisplay(irValue);
  }

  delay(20);
}


void handleMenuButton() {
  bool menuState = digitalRead(PIN_BTN_MENU);
  if (menuState == LOW && lastMenuState == HIGH &&
      (millis() - lastMenuTime > DEBOUNCE_MS)) {
    lastMenuTime = millis();
    currentMode = (currentMode == MODE_HR) ? MODE_SPO2 : MODE_HR;
    Serial.print("Mode changed → ");
    Serial.println(currentMode == MODE_HR ? "HEART RATE" : "SpO2");
    showModeSplash();
  }
  lastMenuState = menuState;
}


void showModeSplash() {
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(1);
  display.setCursor(20, 12);
  display.print(currentMode == MODE_HR ? ">> Heart Rate Mode" : ">> SpO2 Mode");
  display.display();
  delay(800);
}


void updateDisplay(long irValue) {
  display.clearDisplay();
  display.setTextColor(WHITE);

  if (irValue > 50000) {
    unsigned long timeOn = millis() - fingerPlacedTime;

    if (currentMode == MODE_HR) {
      if (millis() - lastBeat < 200) {
        display.setTextSize(2);
        display.setCursor(0, 8);
        display.print("<3");
      }
      display.setTextSize(1);
      display.setCursor(35, 0);  display.print("Live:");
      display.setCursor(70, 0);
      if (displayBPM > 0) display.print(displayBPM);
      else                display.print("--");
      display.setCursor(35, 16); display.print("Avg :");
      display.setTextSize(2);
      display.setCursor(70, 12);
      if (timeOn >= 8000 && beatAvg > 0) display.print(beatAvg);
      else                                display.print("--");

    } else {
      display.setTextSize(1);
      display.setCursor(0, 0);   display.print("SpO2");
      display.setTextSize(2);
      display.setCursor(0, 14);
      if (timeOn >= 5000 && displaySpO2 > 0) {
        display.print(displaySpO2);
        display.setTextSize(1); display.print("%");
      } else {
        display.print("--");
      }
      display.setTextSize(1);
      display.setCursor(80, 14); display.print("O2 Sat");
    }

  } else {
    display.setTextSize(1);
    display.setCursor(15, 4);  display.print("Place Finger");
    display.setCursor(15, 20);
    display.print(currentMode == MODE_HR ? "[Heart Rate]" : "[SpO2]");
  }

  display.display();
}