# Solar-irradiance-meter
Code of the project-
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ACS712.h>
#include <EEPROM.h>
#include <BH1750.h>

// ===== LCD SETTINGS =====
#define LCD_ADDRESS 0x27
#define LCD_COLUMNS 16
#define LCD_ROWS 2
#define LCD_REFRESH_MS 1000

// ===== EEPROM SETTINGS =====
#define EEPROM_SIGNATURE_ADDR 0
#define EEPROM_MIDPOINT_ADDR   2
#define EEPROM_SIGNATURE       0xAB42

// ===== PINS =====
#define CURRENT_PIN A0
#define VOLTAGE_PIN A1
#define BUTTON_PIN 2

// ===== SENSOR CONFIG =====
#define ADC_VREF          5.0
#define ADC_STEPS         1023
#define VOLTAGE_DIV_RATIO 3.361 // 4.58k / 1.94k

// ===== OBJECTS =====
ACS712 acs(CURRENT_PIN, ADC_VREF, ADC_STEPS, 100); // Only for calibration
LiquidCrystal_I2C lcd(LCD_ADDRESS, LCD_COLUMNS, LCD_ROWS);
BH1750 lightMeter;

// ===== GLOBAL VARIABLES =====
bool manualCalibrationRequested = false;
bool lastButtonState = HIGH;
unsigned long lastLcdUpdate = 0;

void setup() {
  Serial.begin(9600);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  Wire.begin();
  lcd.init();
  lcd.backlight();
  lcd.clear();

  if (!lightMeter.begin(BH1750::CONTINUOUS_HIGH_RES_MODE)) {
    lcd.print("BH1750 ERROR!");
    while (1); // Halt on sensor error
  }

  loadCalibration();

  if (acs.getMidPoint() == 0) {
    lcd.setCursor(0, 0);
    lcd.print("AUTO CALIB...");
    acs.autoMidPoint();
    saveCalibration();
    delay(1500);
  } else {
    lcd.setCursor(0, 0);
    lcd.print("MID OFFSET:");
    lcd.setCursor(0, 1);
    lcd.print(acs.getMidPoint());
    delay(1500);
  }

  lcd.clear();
  lcd.setCursor(2, 0);
  lcd.print("SYSTEM READY");
  delay(1000);
}

void loop() {
  checkCalibrationButton();

  if (manualCalibrationRequested) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("CALIBRATING...");
    Serial.println("Manual calibration triggered");

    acs.setMidPoint(0);
    delay(100);
    acs.autoMidPoint();
    saveCalibration();

    int mid = acs.getMidPoint();
    lcd.setCursor(0, 1);
    lcd.print("MID:");
    lcd.print(mid);
    Serial.print("New midpoint: ");
    Serial.println(mid);

    delay(2000);  // 2-second delay after calibration
    lcd.clear();
    manualCalibrationRequested = false;
  }

  if (millis() - lastLcdUpdate >= LCD_REFRESH_MS) {
    updateDisplay();
    lastLcdUpdate = millis();
  }
}

void updateDisplay() {
  float vout = analogRead(VOLTAGE_PIN) * (ADC_VREF / ADC_STEPS);
  float voltage = vout * VOLTAGE_DIV_RATIO;

  float current = voltage / 50.0;   // I = V / 50 ohms
  float power = voltage * current;
  float lux = lightMeter.readLightLevel();

  // === LCD Display ===
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("V:");
  lcd.print(voltage, 2);
  lcd.print(" I:");
  lcd.print(current, 3);

  lcd.setCursor(0, 1);
  lcd.print("P:");
  lcd.print(power, 2);
  lcd.print("W Lx:");
  lcd.print((int)lux);

  // === Serial Monitor ===
  Serial.print("Voltage: ");
  Serial.print(voltage, 2);
  Serial.print(" V | Current: ");
  Serial.print(current, 3);
  Serial.print(" A | Power: ");
  Serial.print(power, 2);
  Serial.print(" W | Lux: ");
  Serial.println((int)lux);
}

void checkCalibrationButton() {
  bool currentButtonState = digitalRead(BUTTON_PIN);
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    delay(50); // Debounce
    if (digitalRead(BUTTON_PIN) == LOW) {
      manualCalibrationRequested = true;
    }
  }
  lastButtonState = currentButtonState;
}

void saveCalibration() {
  int currentMid = acs.getMidPoint();
  int storedMid;
  EEPROM.get(EEPROM_MIDPOINT_ADDR, storedMid);
  if (storedMid != currentMid) {
    EEPROM.put(EEPROM_SIGNATURE_ADDR, EEPROM_SIGNATURE);
    EEPROM.put(EEPROM_MIDPOINT_ADDR, currentMid);
    Serial.print("Calibration saved to EEPROM. Midpoint: ");
    Serial.println(currentMid);
  }
}

void loadCalibration() {
  int sig;
  EEPROM.get(EEPROM_SIGNATURE_ADDR, sig);
  if (sig == EEPROM_SIGNATURE) {
    int midpoint;
    EEPROM.get(EEPROM_MIDPOINT_ADDR, midpoint);
    acs.setMidPoint(midpoint);
    Serial.print("Loaded midpoint from EEPROM: ");
    Serial.println(midpoint);
  } else {
    acs.setMidPoint(0);
    Serial.println("No valid calibration found. Using default midpoint.");
  }
}
