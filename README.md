#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "HX711.h"

// =====================
// DISPLAY
// =====================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// =====================
// HX711 - ARDUINO NANO ESP32 PINS
// =====================
HX711 scale1;
HX711 scale2;
// Arduino Nano ESP32 Pin-Belegung:
#define DT1 4     // Wägezelle 1: Data (Gelb auf HX711)
#define SCK1 5    // Wägezelle 1: Clock (Weiß auf HX711)
#define DT2 6     // Wägezelle 2: Data (Gelb auf HX711)
#define SCK2 7    // Wägezelle 2: Clock (Weiß auf HX711)

// Alternative gute Pins (falls Probleme mit den obigen):
// #define DT1 8     // Alternative
// #define SCK1 9    // Alternative
// #define DT2 10    // Alternative
// #define SCK2 11   // Alternative

// =====================
// TASTER
// =====================
#define TARE_PIN 12    // Tarieren (Pullup verwenden)
#define CAL_PIN 13     // Kalibrieren (Pullup verwenden)

// =====================
// RELAIS & SCHALTER
// =====================
#define RELAY_PIN 14   // Relais für Mahlwerk
#define START_SWITCH 15 // Start-Schalter

// =====================
// VARIABLEN
// =====================
float weight1 = 0;
float weight2 = 0;
float totalWeight = 0;
float targetWeight = 18.0;
float offset = 0.3;
float calFactor1 = 1.0;  // Start mit 1.0
float calFactor2 = 1.0;  // Start mit 1.0
const float CALIBRATION_WEIGHT = 105.0;  // Bekanntes Kalibriergewicht

bool scale1OK = false;
bool scale2OK = false;
bool calibrating = false;
int calibrationStep = 0;

unsigned long lastTareTime = 0;
unsigned long lastCalTime = 0;
const unsigned long DEBOUNCE_TIME = 1000;

// =====================
// FUNKTIONEN
// =====================

void initDisplay() {
  // Arduino Nano ESP32: Standard I2C Pins sind D0(SDA) und D1(SCL)
  // Diese werden automatisch von Wire.begin() verwendet
  // Wire.begin(SDA, SCL) - auf Nano ESP32 sind D0/D1 standardmäßig SDA/SCL
  
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3D)) {
      Serial.println("Display nicht gefunden!");
      while(1); // Stoppe bei Display-Fehler
    }
  }
  
  display.ssd1306_command(SSD1306_SETCONTRAST);
  display.ssd1306_command(150);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 25);
  display.println("Arduino Nano ESP32");
  display.setCursor(15, 35);
  display.println("System startet...");
  display.display();
  delay(1000);
}

bool initScale(HX711 &scale, int dtPin, int sckPin, float calFactor, const char* name) {
  Serial.print("Initialisiere ");
  Serial.print(name);
  Serial.print(" (DT=");
  Serial.print(dtPin);
  Serial.print(", SCK=");
  Serial.print(sckPin);
  Serial.print(")... ");
  
  scale.begin(dtPin, sckPin);
  
  // Höhere Timeout für stabilere Initialisierung
  if(scale.wait_ready_timeout(3000)) {
    scale.set_scale(calFactor);
    scale.tare();  // Normales Tarieren für Start
    
    // Test ob die Zelle vernünftige Werte liefert
    delay(300);
    float test = scale.get_units(3);
    
    if (test > 10000 || test < -10000) {
      Serial.println("FEHLER: Ungültiger Testwert");
      return false;
    }
    
    Serial.println("OK");
    Serial.print("  Test: ");
    Serial.print(test, 1);
    Serial.println(" g");
    
    return true;
  } else {
    Serial.println("FEHLER - Nicht bereit");
    return false;
  }
}

void updateDisplay() {
  display.clearDisplay();
  
  // Titel
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("NANO ESP32 Kaffee");
  
  // Status-LEDs
  display.setCursor(100, 0);
  if(scale1OK) display.print("✓");
  else display.print("✗");
  
  display.setCursor(110, 0);
  if(scale2OK) display.print("✓");
  else display.print("✗");
  
  display.drawLine(0, 10, 128, 10, SSD1306_WHITE);
  
  if (calibrating) {
    display.setTextSize(1);
    display.setCursor(0, 15);
    display.println("KALIBRIERUNG");
    
    switch(calibrationStep) {
      case 1:
        display.setCursor(0, 25);
        display.println("Zelle 1 leer lassen");
        display.setCursor(0, 35);
        display.println("CAL-Taste druecken");
        break;
      case 2:
        display.setCursor(0, 25);
        display.print(CALIBRATION_WEIGHT, 0);
        display.println("g auf Zelle 1");
        display.setCursor(0, 35);
        display.println("CAL-Taste druecken");
        display.setCursor(0, 45);
        display.print("Gemessen: ");
        display.print(weight1, 1);
        display.println(" g");
        break;
      case 3:
        display.setCursor(0, 25);
        display.println("Zelle 2 leer lassen");
        display.setCursor(0, 35);
        display.println("CAL-Taste druecken");
        break;
      case 4:
        display.setCursor(0, 25);
        display.print(CALIBRATION_WEIGHT, 0);
        display.println("g auf Zelle 2");
        display.setCursor(0, 35);
        display.println("CAL-Taste druecken");
        display.setCursor(0, 45);
        display.print("Gemessen: ");
        display.print(weight2, 1);
        display.println(" g");
        break;
      case 5:
        display.setCursor(0, 25);
        display.println("KALIBRIERT!");
        display.setCursor(0, 35);
        display.print("Z1-Faktor: ");
        display.println(calFactor1, 0);
        display.setCursor(0, 45);
        display.print("Z2-Faktor: ");
        display.print(calFactor2, 0);
        break;
    }
  } else {
    // Normale Anzeige
    display.setTextSize(2);
    display.setCursor(0, 15);
    display.print(totalWeight, 1);
    display.println(" g");
    
    display.setTextSize(1);
    display.setCursor(0, 35);
    if(scale1OK) {
      display.print("Z1: ");
      display.print(weight1, 1);
      display.println(" g");
    } else {
      display.println("Z1: ---");
    }
    
    display.setCursor(64, 35);
    if(scale2OK) {
      display.print("Z2: ");
      display.print(weight2, 1);
      display.println(" g");
    } else {
      display.println("Z2: ---");
    }
    
    display.setCursor(0, 45);
    display.print("Ziel: ");
    display.print(targetWeight, 1);
    display.println(" g");
    
    // Aktuelle Faktoren anzeigen
    if (scale1OK && scale2OK) {
      display.setCursor(0, 55);
      display.print("CF1:");
      display.print(calFactor1, 0);
      display.print(" CF2:");
      display.print(calFactor2, 0);
    }
  }
  
  display.display();
}

void readScales() {
  static unsigned long lastRead = 0;
  unsigned long now = millis();
  
  if(now - lastRead > 200) {
    if(scale1OK && scale1.is_ready()) {
      weight1 = scale1.get_units(2);
      if (weight1 < 0) weight1 = 0;  // Negative Werte vermeiden
    }
    
    if(scale2OK && scale2.is_ready()) {
      weight2 = scale2.get_units(2);
      if (weight2 < 0) weight2 = 0;  // Negative Werte vermeiden
    }
    
    totalWeight = weight1 + weight2;
    lastRead = now;
  }
}

void tareScales() {
  if (calibrating) return;
  
  Serial.println("\n=== TARIERUNG ===");
  
  if(scale1OK) {
    scale1.tare();
    Serial.println("Wägezelle 1 tariert");
  }
  
  if(scale2OK) {
    scale2.tare();
    Serial.println("Wägezelle 2 tariert");
  }
  
  weight1 = 0;
  weight2 = 0;
  totalWeight = 0;
  
  Serial.println("Tarierung abgeschlossen");
  updateDisplay();
}

void calibrateScale() {
  static unsigned long lastButtonPress = 0;
  unsigned long now = millis();
  
  // Kalibrierung starten
  if (!calibrating && digitalRead(CAL_PIN) == LOW && (now - lastCalTime) > DEBOUNCE_TIME) {
    lastCalTime = now;
    calibrating = true;
    calibrationStep = 1;
    Serial.println("\n=== KALIBRIERUNG GESTARTET ===");
    Serial.println("Schritt 1: Zelle 1 leer lassen");
    updateDisplay();
    return;
  }
  
  // Kalibrierungsschritte
  if (calibrating && digitalRead(CAL_PIN) == LOW && (now - lastButtonPress) > DEBOUNCE_TIME) {
    lastButtonPress = now;
    
    switch(calibrationStep) {
      case 1:  // Zelle 1 leer
        Serial.println("\n--- Zelle 1 leer ---");
        if (scale1OK) {
          scale1.power_down();
          delay(100);
          scale1.power_up();
          delay(500);
          
          long rawZeroSum = 0;
          for (int i = 0; i < 20; i++) {
            rawZeroSum += scale1.read();
            delay(10);
          }
          long rawZero1 = rawZeroSum / 20;
          
          Serial.print("Mittelwert leer: ");
          Serial.println(rawZero1);
          
          scale1.set_scale(1.0);
          scale1.set_offset(rawZero1);
        }
        calibrationStep = 2;
        Serial.println("\nSchritt 2: " + String(CALIBRATION_WEIGHT) + "g auf Zelle 1 legen");
        updateDisplay();
        break;
        
      case 2:  // Zelle 1 mit 105g
        Serial.println("\n--- Zelle 1 mit " + String(CALIBRATION_WEIGHT) + "g ---");
        if (scale1OK) {
          delay(1000);
          
          long rawWeightSum = 0;
          for (int i = 0; i < 20; i++) {
            rawWeightSum += scale1.read();
            delay(10);
          }
          long rawWithWeight1 = rawWeightSum / 20;
          
          Serial.print("Mittelwert mit Gewicht: ");
          Serial.println(rawWithWeight1);
          
          long rawDiff1 = rawWithWeight1 - scale1.get_offset();
          Serial.print("Reine Gewichts-Differenz: ");
          Serial.println(rawDiff1);
          
          calFactor1 = (float)rawDiff1 / CALIBRATION_WEIGHT;
          
          Serial.print("Neuer Kalibrierfaktor Zelle 1: ");
          Serial.println(calFactor1, 3);
          
          scale1.set_scale(calFactor1);
          scale1.tare();
          
          delay(300);
          float testReading = scale1.get_units(5);
          Serial.print("Test: ");
          Serial.print(testReading, 1);
          Serial.println(" g");
        }
        calibrationStep = 3;
        Serial.println("\nSchritt 3: Zelle 2 leer lassen");
        updateDisplay();
        break;
        
      case 3:  // Zelle 2 leer
        Serial.println("\n--- Zelle 2 leer ---");
        if (scale2OK) {
          scale2.power_down();
          delay(100);
          scale2.power_up();
          delay(500);
          
          long rawZeroSum = 0;
          for (int i = 0; i < 20; i++) {
            rawZeroSum += scale2.read();
            delay(10);
          }
          long rawZero2 = rawZeroSum / 20;
          
          Serial.print("Mittelwert leer: ");
          Serial.println(rawZero2);
          
          scale2.set_scale(1.0);
          scale2.set_offset(rawZero2);
        }
        calibrationStep = 4;
        Serial.println("\nSchritt 4: " + String(CALIBRATION_WEIGHT) + "g auf Zelle 2 legen");
        updateDisplay();
        break;
        
      case 4:  // Zelle 2 mit 105g
        Serial.println("\n--- Zelle 2 mit " + String(CALIBRATION_WEIGHT) + "g ---");
        if (scale2OK) {
          delay(1000);
          
          long rawWeightSum = 0;
          for (int i = 0; i < 20; i++) {
            rawWeightSum += scale2.read();
            delay(10);
          }
          long rawWithWeight2 = rawWeightSum / 20;
          
          Serial.print("Mittelwert mit Gewicht: ");
          Serial.println(rawWithWeight2);
          
          long rawDiff2 = rawWithWeight2 - scale2.get_offset();
          Serial.print("Reine Gewichts-Differenz: ");
          Serial.println(rawDiff2);
          
          calFactor2 = (float)rawDiff2 / CALIBRATION_WEIGHT;
          Serial.print("Neuer Kalibrierfaktor Zelle 2: ");
          Serial.println(calFactor2, 3);
          
          scale2.set_scale(calFactor2);
          scale2.tare();
          
          delay(300);
          float testReading = scale2.get_units(5);
          Serial.print("Test: ");
          Serial.print(testReading, 1);
          Serial.println(" g");
        }
        
        calibrationStep = 5;
        Serial.println("\n=== KALIBRIERUNG ABGESCHLOSSEN ===");
        Serial.print("Kalibrierfaktor Zelle 1: ");
        Serial.println(calFactor1, 3);
        Serial.print("Kalibrierfaktor Zelle 2: ");
        Serial.println(calFactor2, 3);
        
        updateDisplay();
        delay(3000);
        
        calibrating = false;
        calibrationStep = 0;
        
        tareScales();
        break;
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(2000);
  
  Serial.println("\n=== ARDUINO NANO ESP32 KAFFEEMUEHLE ===");
  Serial.println("Pin-Belegung:");
  Serial.println("HX711 Z1: DT=4, SCK=5");
  Serial.println("HX711 Z2: DT=6, SCK=7");
  Serial.println("Taster: TARE=12, CAL=13");
  Serial.println("Relais: 14, Start: 15");
  
  // Pin-Modi setzen
  pinMode(TARE_PIN, INPUT_PULLUP);
  pinMode(CAL_PIN, INPUT_PULLUP);
  pinMode(START_SWITCH, INPUT_PULLUP);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  
  initDisplay();
  
  Serial.println("\nInitialisiere Wägezellen...");
  scale1OK = initScale(scale1, DT1, SCK1, 1.0, "Wägezelle 1");
  scale2OK = initScale(scale2, DT2, SCK2, 1.0, "Wägezelle 2");
  
  // Alternative Pins versuchen, falls erste nicht funktionieren
  if (!scale1OK) {
    Serial.println("Versuche alternative Pins für Zelle 1...");
    scale1OK = initScale(scale1, 8, 9, 1.0, "Wägezelle 1 (Alt)");
  }
  
  if (!scale2OK) {
    Serial.println("Versuche alternative Pins für Zelle 2...");
    scale2OK = initScale(scale2, 10, 11, 1.0, "Wägezelle 2 (Alt)");
  }
  
  Serial.println("\n=== KALIBRIERUNGS-PROZESS ===");
  Serial.println("1. CAL-Taste: Start");
  Serial.println("2. Zelle 1 leer -> CAL");
  Serial.println("3. " + String(CALIBRATION_WEIGHT) + "g auf Zelle 1 -> CAL");
  Serial.println("4. Zelle 2 leer -> CAL");
  Serial.println("5. " + String(CALIBRATION_WEIGHT) + "g auf Zelle 2 -> CAL");
  
  Serial.println("\n=== SYSTEM BEREIT ===");
  updateDisplay();
}

void loop() {
  unsigned long now = millis();
  
  // TARE Taste
  if(!calibrating && digitalRead(TARE_PIN) == LOW && (now - lastTareTime) > DEBOUNCE_TIME) {
    lastTareTime = now;
    tareScales();
  }
  
  // Kalibrierung
  calibrateScale();
  
  // Wägezellen lesen
  readScales();
  
  // Mahlung starten
  static bool grinding = false;
  if(!calibrating && digitalRead(START_SWITCH) == LOW && !grinding) {
    grinding = true;
    digitalWrite(RELAY_PIN, HIGH);
    Serial.println("Mahlung gestartet");
    updateDisplay();
    
    unsigned long grindStartTime = millis();
    const unsigned long MAX_GRIND_TIME = 30000; // 30 Sekunden max
    
    while(totalWeight < (targetWeight - offset) && grinding) {
      readScales();
      
      // Sicherheits-Timeout
      if(millis() - grindStartTime > MAX_GRIND_TIME) {
        Serial.println("SICHERHEIT: Timeout!");
        break;
      }
      
      // Zu schweres Sicherheitssystem
      if(totalWeight > targetWeight * 1.5) {
        Serial.println("SICHERHEIT: Zu schwer!");
        break;
      }
      
      // Display aktualisieren
      static unsigned long lastGrindUpdate = 0;
      if(millis() - lastGrindUpdate > 200) {
        updateDisplay();
        lastGrindUpdate = millis();
      }
      
      delay(50);
    }
    
    digitalWrite(RELAY_PIN, LOW);
    grinding = false;
    Serial.print("Mahlung beendet: ");
    Serial.print(totalWeight, 1);
    Serial.println(" g");
    updateDisplay();
    
    delay(1000);
  }
  
  // Display aktualisieren
  static unsigned long lastUpdate = 0;
  if(now - lastUpdate > 300) {
    updateDisplay();
    lastUpdate = now;
  }
  
  delay(50);
}
