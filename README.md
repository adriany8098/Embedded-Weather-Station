# Autonomous Multi-Sensor Environmental Analytics & Weather Station
> **Academic Benchmark**: Awarded **57/60 (95%)** | International Foundation Programme, University of York, UK

---

## 📌 Executive Summary
This project delivers a field-deployable, high-precision environmental monitoring station engineered around an **Arduino microcontroller**. The system executes real-time data acquisition for ambient temperature, relative humidity, barometric pressure, and localized wind velocity across an uninterrupted 8-hour testing window (09:00 - 17:00). 

A core objective of this study was conducting a rigorous **cross-sensor evaluation between the DHT22 and BME280 sensor modules**, assessing thermal inertia, bus protocol efficiency, sampling latency, and signal-to-noise ratios under variable environmental terrains.

---

## 🛠️ Detailed Hardware & Sensor Architecture

### 1. Temperature & Humidity Probe: DHT22 
* **Operating Mechanism**: Uses a capacitive humidity sensing element and a high-precision Negative Temperature Coefficient (NTC) thermistor.
* **Interface Protocol**: Single-bus custom digital signal protocol (OneWire-style timing).
* **Specifications**:
  * **Temperature Range**: -40°C to +80°C (Accuracy: ±0.5°C)
  * **Humidity Range**: 0% to 100% RH (Accuracy: ±2% RH)
  * **Sampling Rate**: Low frequency (Maximum 0.5 Hz / 1 sample every 2 seconds)
* **Engineering Context**: Deployed as the secondary/baseline environmental probe to evaluate response lag against thermal fluctuations.

### 2. Micro-Electromechanical System (MEMS) Sensor: BME280
* **Operating Mechanism**: Integrated environmental chip combining piezo-resistive pressure, capacitive humidity, and low-noise temperature sensing.
* **Interface Protocol**: High-speed **I2C Bus Protocol** (Device Address: `0x76` / `0x77`).
* **Specifications**:
  * **Temperature Accuracy**: ±1.0°C (Optimized for internal thermal drift compensation)
  * **Humidity Accuracy**: ±3% RH with minimal hysteresis
  * **Barometric Pressure**: 300 hPa to 1100 hPa (Absolute accuracy: ±1.0 hPa)
  * **Sampling Rate**: Up to 157 Hz (Programmed via non-blocking timing loops)
* **Engineering Context**: Primary sensor for barometric pressure and high-frequency thermal response testing.

### 3. Auxiliary Peripherals & Hardware Integration
* **Anemometer**: Pulse-counting optoelectronic sensor for measuring wind dynamics.
* **Display Output**: LCD 1602 driven via PCF8574 I2C Backpack (reducing pin count from 6 GPIOs down to 2 I2C lines: `SDA` / `SCL`).
* **Power Architecture**: Portable 5V DC regulated lithium power bank enabling standalone, untethered outdoor operation.
* **Enclosure Engineering**: Custom IP-rated weather-proof housing with external sensor probe exposure to eliminate thermal insulation bias.

---

## 💻 Firmware Logic & Algorithm Design

The code is optimized for reliability and long-term data logging:

```cpp
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BME280.h>
#include "DHT.h"

// Hardware Pin Definitions
#define DHTPIN 2
#define DHTTYPE DHT22

// Sensor Instances
DHT dht(DHTPIN, DHTTYPE);
Adafruit_BME280 bme; // I2C Communication

// Non-blocking Timing Control
unsigned long previousMillis = 0;
const long sampleInterval = 5000; // Log data every 5 seconds

void setup() {
  Serial.begin(9600);
  
  // Initialize Sensors
  dht.begin();
  bool bme_status = bme.begin(0x76);
  
  if (!bme_status) {
    Serial.println(F("[ERROR] BME280 initialization failed. Check I2C wiring."));
    while (1); // Halt execution on critical hardware error
  }
  
  Serial.println(F("Timestamp(ms),DHT_Temp(C),BME_Temp(C),DHT_Hum(%),BME_Hum(%),Pressure(hPa)"));
}

void loop() {
  unsigned long currentMillis = millis();

  // Execution loop running without delay() to prevent bus blocking
  if (currentMillis - previousMillis >= sampleInterval) {
    previousMillis = currentMillis;

    // Acquire DHT22 Data
    float dhtTemp = dht.readTemperature();
    float dhtHum  = dht.readHumidity();

    // Acquire BME280 Data via I2C
    float bmeTemp = bme.readTemperature();
    float bmeHum  = bme.readHumidity();
    float bmePres = bme.readPressure() / 100.0F; // Convert Pa to hPa

    // Sanity Check & Data Logging
    if (!isnan(dhtTemp) && !isnan(dhtHum)) {
      Serial.print(currentMillis); Serial.print(",");
      Serial.print(dhtTemp);       Serial.print(",");
      Serial.print(bmeTemp);       Serial.print(",");
      Serial.print(dhtHum);        Serial.print(",");
      Serial.print(bmeHum);        Serial.print(",");
      Serial.println(bmePres);
    }
  }
}
